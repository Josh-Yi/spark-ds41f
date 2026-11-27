# 2026-11-27: Site upgrade from spark-ds41f:1107 to spark-ds41f:1126

Promotion of the site profile from the 1107-era recipe to the released
`spark-ds41f-1126` (`releases/spark-ds41f-1126`, tag `spark-ds41f-1126` =
`890f8654`). This is a release promotion, not a single-variable tuning
experiment: several upstream changes land together, so the comparison below is
a version-to-version one, and each claim is reported against the recorded
reference run rather than against an A/A repeat of 1107.

The site's 1107 deployment is in
`experiments/2026-11-18-site-deploy-tp4-ring/README.md`.

## Base and variables

- Base commit: `890f8654` ("release: record tp3-triangle and tp3-triangle-4k
  qualification in 1126 baseline summary"), the `spark-ds41f-1126` release tag.
  Not `upstream/main` head, which carries unreleased candidate-75/76 work.
- Site changes kept on top of the release: the deployment block
  (`Josh-Yi/spark-ds41f`, branch `site-1126`, `{home}/Projects/Scripts/spark-ds41f`)
  and `--limit-mm-per-prompt {"image":256}`
  (`manifests/runtime-defaults/2026-11-27-images-256.json`).
- Site changes deliberately not carried:
  - spark-attic display-carveout lending for KV (`SPARK_ATTIC_*`, the four
    attic mounts, `VLLM_PLUGINS=spark_attic`). 1126 allocates 16.50 GiB of KV
    per rank itself, 4.00 GiB more than the 1107 profile's 12.50 GiB pool once
    its 2,044 MiB borrow is counted, and its tp4-ring profile explicitly keeps
    weights in regular memory (`SPARK_DS41F_DISPLAY_CARVEOUT_WEIGHTS=0`).
    Borrowing on top of 1126 is a separate, later experiment.
  - The inherited 1107 `docker_run_args` DRM card bind and device cgroup rule:
    1126 sizes the carve-out through RM `/dev/nvidiactl` (vLLM 0821) and no
    longer runs with either.

## Preconditions

- Host stack is unchanged between 1107 and 1126: kernel
  `7.0.0-1019-nvidia-64k` (`kho=off`), driver 580.178.04, memory-saver 0.4.1,
  64 KiB pages. The site's disabled `nvmodule-guard` (finding 1 of the 1107
  deployment record) is unchanged by this promotion.
- The GLP vector changes from the gated GLP-39 (`alpha` 0.75) to the public
  `GLP-27 L11-37`, served at `alpha` 2.0 with generation defaults
  `temperature 0.3 / top_p 0.95`. The file must be in the hub cache at revision
  `4d570926e9dd` on all four nodes and match
  `sha256:24e97d41a9a4f77adad78ff4cb888cb4e339eb62151dbf37a7fcd3ea30eb442e`;
  1126 warns and serves the stock name only when a node lacks it, so a missing
  file would silently drop the steered name.
- New in 1126: `host.defragment_before_start` sweeps 2 MiB hugepages before
  compaction. The site's `min_free_kbytes=45166`/THP=never profile interacts
  with it; watch the first start.

## Exact commands (to run under a cluster window)

```sh
# local, before touching the cluster
bin/spark doctor
git diff --check

# image: the profile names spark-ds41f:1126-site1, so the release tag is never
# overwritten. build image refuses to run beside a live service, so this step
# needs the same window as the cluster work below.
bin/spark build prepare
bin/spark build image --apply        # -> spark-ds41f:1126-site1
bin/spark build check
bin/spark build distribute --apply

# cluster (needs the window hold in ~/spark-hold.json)
bin/spark cluster sync --apply --cluster-config config/cluster.json   # read live:/shape: first
bin/spark --cluster-config config/cluster.json cluster start --apply
bin/spark doctor --live
bin/spark bench --suite release --shape tp4-ring
```

## Runs

| # | when | image | boots | notes |
| - | ---- | ----- | ----- | ----- |
| 1 | pending | `spark-ds41f:1126-site1` | 1 | first boot after sync; record ready time, min MemAvailable per node, KV line, swap growth |
| 2 | pending | `spark-ds41f:1126-site1` | 2 | repeat boot; `bin/spark bench` release suite, tp4-ring |

Nothing has been run yet: no window taken, no image built, no cluster
operation attempted.

## To measure

- `cluster ready` time and the KV/`MemAvailable` line at startup, against the
  promotion's 13.68-16.09 GiB minimum.
- `bin/spark bench` release suite against
  `manifests/suites/2026-11-26-spark-ds41f-1126-tp4`; the 1107 site reference is
  `manifests/suites/2026-11-07-spark-ds41f-1107-tp4`.
- The 256-image limit under real multi-image load (64/128/256 images): head host
  memory peak, request KV occupancy, and TTFT, since the request holds KV blocks
  for every image's tokens for its lifetime and the head decodes every image in
  unified memory.
- Behaviour of the steered name under the new GLP vector: the site's clients
  pinned `deepseek-v4.1-flash-weightless`, and the vector, alpha and generation
  defaults all changed.

## Known divergences from the release recipe

Five recipe-invariant checks fail on this branch and pass at the release tag; all
of them encode "`config/cluster.json` is the owner's release", which a site fork
breaks by naming its own deployment and image:

- `tests/test_releases.py::test_production_is_the_newest_release` allows a site
to differ from the frozen recipe only through `serve_options` (admission limits)
and `site_fields` (`host.home`). This profile also changes
`deployment.repository`, `deployment.branch`, `deployment.path` and
`container.image`, which `docs/setup.md` tells a site to set.
- `tests/test_site_defaults.py::NodeHomeTest.test_checkout_and_mounts_follow_it`
asserts the recipe's `{home}/projects/spark-ds41f`; this site's checkout is
`{home}/Projects/Scripts/spark-ds41f`.
- `tests/test_topology.py::TopologyTest::test_former_repository_url_names_the_renamed_repository`
asserts the recipe's `deployment.repository` is the owner's URL.
- `tests/test_agentx.py::ArmTest::test_the_arm_is_production_but_its_port_and_usage_details`
compares a committed experiment arm against the recipe, so the 256-image cap and
the site image tag show up as differences.
- `tests/test_releases.py::test_generated_docs_and_pages_match_manifests` passes
once the generated pages are regenerated for this profile; it is listed here
because regenerating them is a site-only change.

The 256-image change itself is handled by the existing record mechanism. The
clean fix is for `site_fields` to cover the `deployment` keys and the image tag;
that belongs in a pull request to the owner rather than carried here, so this
branch leaves the tests as released and records the failures instead. The
release tag itself is not green in this checkout either: seven tests fail there
for unrelated reasons (the candidate patch series are not prepared in `.work/`,
and the lab plan tests expect the head's node map).

## Conclusion

Pending the runs above.

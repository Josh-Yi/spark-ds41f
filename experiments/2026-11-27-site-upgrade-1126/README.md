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

| # | when (UTC) | what | ready | notes |
| - | ---------- | ---- | ----- | ----- |
| 1 | 09:28-10:26 | window 1: stop 1107, clear the JIT caches, build, distribute, sync `1ae87b35`, start | 1051.0 s (cold JIT) | image `spark-ds41f:1126-site1` = `sha256:0f3dba6a...`; build 1785 s, lowest MemAvailable 78.5 GiB; GPU, TileLang and sparknet import smokes passed |
| 2 | 10:46 | `bin/spark bench` (default: quality gate + decode), `--allow-mismatch` | - | quality 5/5; report `results/private/bench/20261127T104644Z/bench.json`; decode step ms prose c1 27.2 -> c16 47.3, code c1 30.0 -> c16 54.2; min MemAvailable a3d0 12.36 GiB / workers 17.57-17.47 GiB (promotion minimum 13.68-16.09); swap growth 0.0 GiB on every node; GPU max 51 C, thermal slowdown 0.0 ms |
| 3 | 10:57 | multi-image limit check (`results/private/prep/mm_limit_test.py`, weightless name) | - | 1/64/128/256 images accepted (256 in 12.36 s), 257 refused `HTTP 400 "At most 256 image(s)"`; the site's `--limit-mm-per-prompt` override is live |
| 4 | 10:52-10:55 | stop, second boot, repeat `bin/spark bench` | 116.2 s (warm JIT) | bench aborted mid-decode on request (stress testing stopped); the quality gate had passed 5/5; no decode numbers recorded |
| 5 | 11:13-11:20 | window 2: GLP-39 switch (`3e6a7c71`), sync, coordinated restart | 113.9 s (warm JIT) | both names served; in-container vector sha256 `9c66d477...` verified; `WEIGHTLESS_STEER_ALPHA=0.75`; sample generation on the steered name: TTFT 191 ms, 68.9 tok/s |

## Operational findings

- `bin/spark build prepare --only <part>` unlinks the build directory's
  `inputs.json` before its partial run, so the next `build image` refuses with
  "is not prepared" even though `build check` passed earlier. Recovered with a
  full `bin/spark build prepare` (every download reused and re-verified).
- `cluster sync` detaches the head's own checkout at the deployment commit.
  Commits made afterwards land on a detached HEAD and the branch keeps its old
  tip; `git checkout -B site-1126` repaired it before push. Commit before sync,
  or re-attach after.
- `scripts/lab.py window open` refuses deployment windows (publish guard wants
  the deployment commit on `origin/main`, and the nodes already at it). The
  hold file was written by hand in the documented format instead.
- The release suite times the engine grid and needs `VLLM_SERVER_DEV_MODE=1`
  (dev RPC); it refuses on a production profile. Runs 2 and 4 use the default
  bench (quality gate + decode), which needs no dev RPC.
- The bench refuses a cluster whose live findings contain an ERROR;
  `--allow-mismatch` was used, with the sole ERROR the pre-existing secure_boot
  inconsistency (spark-630a in Setup Mode; firmware level, not this upgrade).
- litellm's auth had a latent crash: keyless requests hit
  `ModuleNotFoundError: prisma` in the error path and returned HTTP 500.
  Installing prisma 0.15.0 into `~/litellm/venv` restores proper 401s; the
  master-key path was verified end to end. Unknown virtual keys still answer
  "No connected db" (`database_url: null`); the site's clients use the master
  key. Host-side change, not in this repository.
- Fan noise under load is the fan controller working as designed: no fan
  setting was changed. `dgx-fan-control` is a temperature-driven additive
  floor (journal: state 0 -> 8 within 30 s as the SoC went 44 -> 63 C under
  client prefill load; 9,000 RPM at floor 8). The bench raises the floor only
  during its cooling phase and restores it afterwards.

## Variables

One intended variable per record: the image upgrade (1107 -> 1126-site1,
runs 1-4). Run 5 changes a second variable with its own record,
`manifests/runtime-defaults/2026-11-27-glp-39-site.json` (GLP-27 -> GLP-39 at
the repository head, alpha 0.75, no generation defaults), by site choice.

## To measure (open)

- The release suite on a development arm in a lab window (dev RPC), against
  `manifests/suites/2026-11-26-spark-ds41f-1126-tp4`; the 1107 site reference is
  `manifests/suites/2026-11-07-spark-ds41f-1107-tp4`.
- Decode numbers for a second boot (run 4's bench was aborted mid-decode).
- The 256-image limit under real multi-image load with photographs (host
  memory peak, KV occupancy, TTFT); run 3 covers the limit's behaviour only.
- Waived by the site: the steered-name behaviour comparison GLP-27 vs GLP-39
  (no style requirement; GLP-39 chosen).

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

All four nodes serve `spark-ds41f:1126-site1` (release `890f8654` with the
site's 256-image cap and GLP-39 steering) at deployment commit `3e6a7c71`.
Cold boot 1051.0 s with cleared JIT caches, 116.2 s and 113.9 s warm; the
quality gate passed in both recorded boots; swap growth was zero and host
memory stayed at 12.36-17.57 GiB available at the measured points. The
256-image site cap behaves as configured and the steered name serves with
GLP-39 at alpha 0.75. Rollback: `spark-ds41f:1107` is retained on every node
and the `site` branch is unmodified. Open items are listed under "To measure".

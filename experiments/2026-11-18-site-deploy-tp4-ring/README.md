# 2026-11-18: Site deployment of spark-ds41f:1107 on a four-node ring

First deployment of this recipe at this site (spark-a3d0 / spark-d403 /
spark-1d1b / spark-630a, switchless CX7 ring). Base commit `323840d3`
("Set the deployment path to the site checkout location"; note: this commit is
not on GitHub yet — the head's key authenticates as `Josh-Yi`, which cannot
push `christopherowen/spark-ds41f`. The commit was distributed to every node by
git bundle; push or grant access to reconcile).

Shape `tp4-ring`, image `spark-ds41f:1107` (built on the head from
`build prepare` inputs, distributed over the fiber `transfer_host` addresses).
Final state: `cluster ready` in 634.6 s, both served names answering
(`deepseek-v4.1-flash`, `deepseek-v4.1-flash-weightless`), litellm repointed
from the retired tensorfold endpoint (192.168.31.49:8888) to
192.168.31.49:8000.

## Findings worth keeping

1. **Site module guard conflicted with the recipe's driver flavor.**
   `/usr/local/sbin/nvmodule-guard.sh` (installed 2026-09-28 on all four nodes,
   `nvmodule-guard.service`) renames the `nvidia-*-open` module directories at
   boot: its comment records the open NVIDIA kernel module as the root cause of
   a GB10 silent-wedge incident at this site. The recipe's memory-saver is
   validated only against the open build (`module_package()` selects
   `linux-modules-nvidia-580-open-<kernel>` and verifies the open source
   hashes). The owner decided to follow the recipe; the guard is disabled
   (`systemctl disable`) on all four nodes and the recipe's open 580.178.04
   modules are loaded. If a silent wedge recurs under load, the guard's claim
   and this decision are the first place to look.

2. **fan-control: FF-A probe fails after warm reboots; cold power cycle cures.**
   `dgx-ec-fan-control` probe returned `status=0x5` / `-EIO` ("refusing
   fan-floor cooling device") on every node after warm reboots, matching the
   upstream troubleshooting note (eSPI mailbox-status read failure). One cold
   power cycle per node (halt, unplug, replug) restored the transport
   every time (`state=0/12` afterwards). Warm reboot is not a cure.

3. **Cross-subnet routes for NFS were runtime-only and vanished at reboot.**
   The model hub is served from the head's NFS export (`/srv/nfs4/huggingface`);
   the worker mounts use the head's link-subnet address (10.10.10.x).
   `tf-ring-setup.sh` persists only the 10.0.0.x overlay; the 10.10.x
   cross-subnet routes were runtime state. Added
   `/usr/local/sbin/site-ring-routes.sh` + `site-ring-routes.service`
   (two routes per node, remote link subnets via both ring neighbours,
   idempotent, enabled at boot) on all four nodes.

4. **The 64 KiB host profile is required at startup, not just for capacity.**
   First `cluster start` failed on every rank:
   `Free memory on device cuda:0 (100.68/123.79 GiB) ... less than desired GPU
   memory utilization (0.82, 101.5 GiB)`. The host ran THP=madvise and
   min_free_kbytes=6469696 (a site wedge precaution). The recipe's
   `scripts/configure-swap` + `spark-ds41f-kernel-memory.service` set THP=never
   ("Avoid the PMD-sized THP reserve on the 64 KiB kernel"), min_free_kbytes
   45166, and activate `/swap-64k.img` (the 4 KiB `/swap.img` must be swapped
   off on the 64 KiB kernel). With that profile applied the start succeeded
   with no other change. The site's high min_free_kbytes wedge precaution is
   superseded by the memory-saver profile; the memory guard service
   (`nvidia-memguard`) is still the alerting layer.

5. **`scripts/install-fan-control` is broken for its pinned v0.1.4** (the tag
   has no `install.sh`). Installed manually per the upstream installation
   guide (DKMS add/build/install against `/usr/src/dgx-spark-fan-control-0.1.4`,
   userspace tool, `dgx_ec_fan_control.conf`, `dgx-fan-control.service`).
   DKMS signing uses the enrolled key at
   `/root/.local/share/dgx-spark-fan-control/keys/` via
   `/etc/dkms/framework.conf`; one MOK enrollment per node (console, firmware
   screen; warm reboots show the MOK manager, cold boots do not light the
   display at that stage).

6. **GPU package flap kept `nvidia_uvm` busy on one node** (`nvidia-persistenced`
   held the module; `modprobe -r` failed mid-install). Stop
   `nvidia-persistenced` before the memory-saver install's module swap.

## Exact commands (abridged)

- `bin/spark build prepare` / `build image --apply` / `build distribute --apply`
- per node: `scripts/install-64k-kernel`, `scripts/configure-swap`,
  `scripts/install-memory-saver` (after `dkms` + `/etc/dkms/framework.conf`
  signing), manual fan-control DKMS (see finding 5)
- `bin/spark --cluster-config config/cluster.json cluster start --apply`
  (failed once — finding 4 — logs in
  `results/private/failed-starts/20261118T105140Z-spark-ds41f`)
- litellm: `config.yaml` repointed to `http://192.168.31.49:8000/v1`, models
  `openai/deepseek-v4.1-flash` and `openai/deepseek-v4.1-flash-weightless`;
  `systemctl restart litellm`; both answered end to end.

## Conclusion

The qualified tp4-ring profile (64 KiB pages, memory-saver, fan-control,
1M context / 16 sequences) is deployed and serving. All four nodes run the
same kernel, driver, module builds and memory policy; `doctor --live` kernel
and memory findings clear. Remaining site divergence to watch: the disabled
module guard (finding 1) and the fan-control cold-boot quirk (finding 2).

## Addendum (2026-11-19): image limit 256 and the display carveout

The site branch (Josh-Yi/spark-ds41f) raises `--limit-mm-per-prompt` to
`{"image":256}` (record: `manifests/runtime-defaults/2026-11-19-images-256.json`)
for multi-image agent compaction, whose clients resend history images.

spark-attic 0.1.4 (display-carveout lender) is wired into the engine and lends
2,044 MiB per rank to the KV cache (`SPARK_ATTIC_PLACE=kv`): the pool is now
12.50 GiB = 10.50 GiB device + 2,044 MiB carveout on every rank. Three findings
worth keeping:

1. The carveout is lent on demand; installing atticd alone leaves `lent 0 MiB`.
   The engine borrows through spark-attic's vLLM plugin, which needs the key
   (`SPARK_ATTIC_KEY_FILE`) and the plugin package visible in the container.
2. Our vLLM fork gates general plugins behind `VLLM_PLUGINS` as a whitelist;
   the recipe's empty string parses as `['']` and silently loads NO plugin.
   The site profile sets `VLLM_PLUGINS=spark_attic`.
3. The container image sets `PYTHONPATH` to its candidate overlays; a site env
   entry of the same name replaces it. The plugin package is mounted into
   site-packages instead of touching PYTHONPATH.

Xid 43 ("GPU stopped processing", one per worker node) appeared once, at the
moment `cluster stop` tore down the vllm processes; no recurrence across the
subsequent starts, and each node's journal holds no other Xid.

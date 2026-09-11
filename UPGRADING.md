# Upgrading Ceph

This document describes how to upgrade a cephadm-managed cluster **by hand**,
outside of the role. The role can drive the same sequence for you (bump
`ceph_version` and re-run — see *Letting the role drive it* at the end), but
for a major-version jump such as Squid → Tentacle, driving it manually gives
you a checkpoint between every phase.

Everything here applies only to clusters already managed by cephadm
(`ceph_deployment_mode == 'cephadm'`). A cluster still running native packages
must be migrated to cephadm first — see the migration section of `README.md` —
and the role deliberately refuses to migrate and upgrade in the same run.

## The important thing to understand

**The role does not implement the rolling upgrade. The cephadm mgr module
does.** `ceph orch upgrade start` restarts daemons in a fixed order:

```
mgr → mon → crash → osd → mds → rgw → rbd-mirror → cephfs-mirror →
ceph-exporter → iscsi → nfs → nvmeof → smb → node-exporter → prometheus →
alertmanager → grafana → loki → promtail
```

and calls `ok-to-stop` before restarting each mon, OSD and MDS, so it will
stall rather than break quorum or take PGs inactive. It also sets
`require-osd-release` for you once the OSDs are done — unlike the package-based
upgrade path, that is not a manual post-step.

The corollary is the thing to be careful about: **never use
`ceph orch daemon redeploy` to push daemons onto a new image.** It performs no
`ok-to-stop` check and ignores the ordering above. That is what the role's
`cephadm-normalize-images.yml` does, which is why it now refuses to run while
an upgrade is in progress.

## Supported paths

| From | To | Supported |
|---|---|---|
| Squid 19.2.z | Tentacle 20.2.z | yes |
| Reef 18.2.z | Tentacle 20.2.z | yes |
| Quincy 17.2.z or older | Tentacle | **no** — upgrade to Reef or Squid first |

There is **no downgrade.** Once mons have written a Tentacle-format map there
is no supported path back to Squid. Snapshot/backup accordingly.

## 1. Pre-flight

```bash
# Every daemon on the same starting version -- note the PLURAL.
# `ceph version` (singular) reports only the mon that answered, and mons
# are upgraded early, so it will lie to you about a partial upgrade.
ceph versions

# Cluster stable: no down/recovering/incomplete/undersized/backfilling PGs.
ceph -s
ceph health detail

# All OSDs up and in.
ceph osd stat

# All hosts online and all daemons running.
ceph orch host ls
ceph orch ps

# At least one standby mgr, or the upgrade halts at UPGRADE_NO_STANDBY_MGR
# after failing over the active one.
ceph mgr dump | jq '.standbys | length'      # must be >= 1
```

The role labels every mon host `mgr` (`cephadm-host-labels.yml`), so on a
3-mon cluster you have 3 mgr daemons and this is satisfied.

Also confirm you are not about to trip over the environment-specific items in
*Known role-specific gotchas* below.

## 2. Quiesce the PG autoscaler

PG splits or merges landing in the middle of an upgrade can stall daemon
restarts for a long time on a large cluster. The role turns the autoscaler on
for every pool it creates, so this applies:

```bash
ceph osd pool set noautoscale
```

Remember to undo this in step 6.

## 3. Large-CephFS option

By default cephadm reduces each filesystem to `max_mds 1` for the MDS phase and
restores it afterwards, which means reduced metadata throughput for that window.
On a large CephFS deployment you can instead have it fail the filesystem
outright and upgrade all MDS daemons together:

```bash
ceph config set mgr mgr/orchestrator/fail_fs true
```

That trades a short hard interruption for a much shorter overall MDS phase.
Leave it unset for a small cluster. Either way, expect CephFS and anything
layered on it (the role's NFS exports) to pause during the MDS phase.

## 4. Dry run

```bash
ceph orch upgrade check --image quay.io/ceph/ceph:v20.2.3
```

This validates that the image exists and is pullable and reports which daemons
would be updated, without restarting anything. Fix a bad tag or an unreachable
registry here rather than at `UPGRADE_FAILED_PULL` with daemons already moving.

Pre-pulling the image on every host shortens the window considerably. The role
does this in `cephadm-prereqs.yml`, so the cheapest way to get it is to run the
role once with `ceph_version` still at the *old* value after pointing
`ceph_container_image` at the new tag — or just do it directly:

```bash
ansible -m command -a 'cephadm --image quay.io/ceph/ceph:v20.2.3 pull' <ceph hosts>
```

## 5. Run the upgrade

```bash
ceph orch upgrade start --image quay.io/ceph/ceph:v20.2.3
```

Monitor with any of:

```bash
ceph orch upgrade status     # in_progress / is_paused / progress / message
ceph -s                      # progress bar with an ETA
ceph -W cephadm              # verbose orchestrator log -- most useful
ceph versions                # watch daemons move across, by type
```

Expect `HEALTH_WARN` for the duration. Expect it to take hours, not minutes, on
a cluster with a real OSD count — every OSD is restarted one at a time behind
an `ok-to-stop` check.

Controls:

```bash
ceph orch upgrade pause
ceph orch upgrade resume
ceph orch upgrade stop       # cancels; does NOT roll back already-upgraded daemons
```

### If it stalls

`ceph orch upgrade status` reporting `is_paused: true` means the orchestrator
gave up on a step and is waiting for you.

| Symptom | Cause | Action |
|---|---|---|
| `UPGRADE_NO_STANDBY_MGR` | no standby mgr to fail over to | `ceph orch apply mgr 2`, then `ceph orch upgrade resume` |
| `UPGRADE_FAILED_PULL` | image unreachable from a host | fix registry/network, `ceph orch upgrade stop` then start again |
| stuck on one OSD | `ok-to-stop` refusing — PGs would go inactive | fix the underlying PG health, it proceeds on its own |
| a host is offline | orchestrator pauses until it returns | bring the host back, `ceph orch upgrade resume` |

A cluster left mid-upgrade is a normal, safe state — mixed-version daemons
interoperate. Fix the cause and resume; do not try to force daemons forward
with `ceph orch daemon redeploy`.

## 6. Verify and unwind

```bash
ceph orch upgrade status                    # in_progress: false, is_paused: false
ceph versions                               # EVERY daemon on the new version
ceph orch ps                                # all running, all on the new image
ceph osd dump | grep require_osd_release    # should read 'tentacle'
ceph health detail                          # back to HEALTH_OK

ceph osd pool unset noautoscale             # undo step 2
ceph config rm mgr mgr/orchestrator/fail_fs # undo step 3, if you set it
```

`require_osd_release` is set automatically by cephadm once every OSD is
upgraded. If it still reads `squid` after `ceph versions` shows all OSDs on
20.2.z, set it manually:

```bash
ceph osd require-osd-release tentacle
```

## 7. Hand back to the role

Only after `ceph versions` is uniformly on the new release:

```yaml
# inventory/<env>/group_vars/<cluster>.yaml
ceph_version: "20.2.3"
```

```bash
ansible-playbook deploy.yml --tags ceph -l <ceph hosts>
```

The role will see the deployed version already matches, skip the upgrade
phase, re-pin `mgr/cephadm/container_image_base` to the new tag, and find
nothing for image normalization to do. Leaving `ceph_version` stale after a
manual upgrade is the failure mode to avoid: the cluster and the inventory
disagree, and every subsequent run will try to normalize daemons back onto the
old image.

## Letting the role drive it

Bump `ceph_version` and run the role. `cephadm-version.yml` will, on the
bootstrap node:

1. read `ceph versions` and compare *every* daemon against `ceph_version`;
2. assert health, OSD up/in counts, absent blocking PG states, and a standby
   mgr (steps 1 above) before starting anything;
3. run `ceph orch upgrade check` against the target image;
4. set `noautoscale`;
5. `ceph orch upgrade start`, then poll `ceph orch upgrade status` until it is
   idle — failing fast on `is_paused` rather than spinning out the timeout;
6. clear `noautoscale` and assert `ceph versions` is uniformly on the target.

A run that ends on a paused upgrade fails at step 5 and deliberately leaves
`noautoscale` set — the upgrade is still pending. The next run that sees it
through clears it; clear it by hand if you abandon the upgrade instead.

Under the `linear` strategy that poll is a play-wide barrier: every other host
parks there until the orchestrator finishes, so no per-host task in the role
touches daemons mid-upgrade. An upgrade left running by an earlier interrupted
play is waited on too, and image normalization refuses to run while an upgrade
is in progress.

Relevant variables (`defaults/main.yml`):

| Variable | Default | Purpose |
|---|---|---|
| `ceph_upgrade_timeout_minutes` | `360` | ceiling on the upgrade wait |
| `ceph_upgrade_poll_delay` | `30` | seconds between status polls |
| `ceph_upgrade_allowed_health` | `[HEALTH_OK, HEALTH_WARN]` | health states an upgrade may start under |
| `ceph_upgrade_blocking_pg_states` | `[down, recovering, incomplete, undersized, backfilling]` | PG states that block a start |
| `ceph_upgrade_precheck` | `true` | run `ceph orch upgrade check` first |
| `ceph_upgrade_pause_autoscale` | `true` | hold `noautoscale` for the duration |

The role does **not** stagger by daemon type, host or CRUSH bucket. If you want
`--daemon-types`, `--hosts`, `--services`, `--limit` or `--crush_bucket_type`,
drive the upgrade manually per the steps above.

## Known role-specific gotchas

**Debian apt suites.** `cephadm-prereqs.yml` adds
`download.ceph.com/debian-<release>/<codename>` with `update_cache: true` on
Debian hosts. `debian-tentacle` currently publishes `bookworm` and `jammy`
only — on any other codename `apt update` fails and the play dies in prereqs
before touching Ceph. Check
<https://download.ceph.com/debian-tentacle/dists/> first.

**Client packages are not upgraded.** `cephadm` and `ceph-common` are installed
with `state: present`, not `latest`, so an already-installed client stays at
its current version after the cluster moves. Upstream recommends updating them
after an upgrade. Do it deliberately:

```bash
ansible -m apt -a 'name=cephadm,ceph-common state=latest' -b <ceph hosts>
```

**Monitoring stack images.** `cephadm bootstrap` deploys prometheus, grafana,
alertmanager and node-exporter, which run their own (non-Ceph) images, while
`cephadm-normalize-images.yml` asserts that every daemon in `ceph orch ps` runs
the configured Ceph image. On a cluster with a monitoring stack that assertion
will fail. Check `ceph orch ps --daemon-type prometheus` before relying on a
full role run to complete after the upgrade.

## References

- [cephadm upgrade](https://docs.ceph.com/en/latest/cephadm/upgrade/)
- [v20.2.0 Tentacle release notes](https://ceph.io/en/news/blog/2025/v20-2-0-tentacle-released/)
- [cephadm adoption (native packages → cephadm)](https://docs.ceph.com/en/tentacle/cephadm/adoption/)

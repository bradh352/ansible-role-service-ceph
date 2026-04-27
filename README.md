# Ceph Role

Author: Brad House<br/>
License: MIT<br/>
Original Repository: https://github.com/bradh352/ansible-role-service-ceph

## Overview

This role deploys and manages Ceph via **cephadm** — Ceph's container-based
management framework — targeting hyperconverged deployments. It can deploy
Monitors, Managers, OSDs, MDS, RadosGW, and NFS-Ganesha.

This role targets Ubuntu and is tested on 24.04 LTS. It supports both fresh
installations and migration from legacy native-package installations.

## Core variables used by this role
* `ceph_cluster_name`: This name is used as an organization tool in Ansible.
  There are groups (specified in the next section) that are used to identify
  what services are deployed on each node in the cluster, and this name is
  used to reference the correct group.  This name may contain alphanumerics
  and underscores only.  This does not attempt to change the cluster name of the
  installed ceph instance, which is always "ceph" as unique cluster names are
  deprecated as per:
  https://docs.ceph.com/en/latest/rados/configuration/common/#naming-clusters-deprecated
* `ceph_public_network`: The ipv4 subnet to use for public client communication.
  This is the network where ceph clients talk to the ceph backend.  This may
  be the same as `ceph_cluster_network`.  This should be a group var for the
  cluster.
* `ceph_cluster_network`: The ipv4 subnet to use for cluster communication. This
  is the network where ceph OSDs communication with eachother for syncronization
  tasks.  This may be the same as `ceph_public_network`.  This should be a group
  var for the cluster.
* `ceph_uuid`: A unique UUID to reference the cluster.  Use `uuidgen` to
  generate this value.  This should be a group var for the cluster.
  `ceph_mon_ip`: Must be unique for each monitor node in the cluster.  This
  is a node-specific value.
* `ceph_osd_room`: Optional. Short alphanumeric (no spaces) name of the room in
  which the ceph host resides.  Specifying this will place the OSD in this
  bucket to assist in determining the proper failure domains.
* `ceph_osd_row`: Optional. Short alphanumeric (no spaces) name of the row in
  which the ceph host resides.  Specifying this will place the OSD in this
  bucket to assist in determining the proper failure domains.
* `ceph_osd_rack`: Optional. Short alphanumeric (no spaces) name of the rack in
  which the ceph host resides.  Specifying this will place the OSD in this
  bucket to assist in determining the proper failure domains.
* `ceph_osd_chassis`: Optional. Short alphanumeric (no spaces) name of the
  chassis (such as when hosts share a single chassis, e.g. Supermicro microcloud)
  in which the ceph host resides.  Specifying this will place the OSD in this
  bucket to assist in determining the proper failure domains.
* `ceph_default_pool_size`: Optional. Default pool size.  Default is `3`
* `ceph_default_min_pool_size`: Optional. Default min pool size.  Default is `2`.

### Variables for configuring resources

* `ceph_pools`: List of ceph pools to create.  If a ceph pool by the same name
  already exists, it will not be modified with the possible settings.  It is
  up to the user to correct any settings post-creation.  Pools created here are
  always configured to be used as `rbd` pools.  The autoscaler is also automatically
  enabled.  Does not currently support creating erasure coded pools.
  See `ceph_fs`, which will create its own pools for CephFS.
  * `name`: Name of the ceph pool.  Can contain alpha-numerics, hypens, periods,
    and underscores (`[A-Za-z0-9_.-]`).
  * `replica`: Number of replicas for the data.  Defaults to `3`.
  * `min_size`: Minimum number of replicas online before the pool goes offline.
    Recommended to be at least `2`.
  * `bulk`: Whether or not the pool is expected to be large (contain a lot of
    data).  Defaults to `true`.
* `ceph_fs`: List of ceph filesystems to create.  IMPORTANT: each filesystem
  created will use a dedicated MDS node, you must ensure you have enough
  MDS nodes to support your use case.
  * `name`: Name of ceph filesystem. Can contain alpha-numerics, hypens, periods,
    and underscores (`[A-Za-z0-9_.-]`).
  * `nfs`: Boolean.  Defaults to false.  If set to true, will deploy NFS
    services on each member of the mds group.  It is recommended to use a
    Virtual IP such as through Keepalived High Availability/Fail Over.
  * `nfs_root`: Subtree within the CephFS for the mountpoint.  It is often
    desirable to share a single CephFS instance with multiple users and delegate
    permissions per subtree.  Defaults to '/'.  Should specify a path like
    '/export'.
  * `nfs_export`: Name to export the filesytem as. If not specified, the same as
    `name`.
* `ceph_rgw`: Define this dictionary with the below members if using rgw.
  * `realm`: Name of the realm.
  * `zonegroup`: Name of the zone group.
  * `zone`: Name of the zone.
  * `port`: Port for RGW service to listen on.  If not specified, defaults to
    `7480`.
* `ceph_prometheus`: Boolean. Default `false`. Whether or not to enable prometheus
  in the ceph mgr.  Will listen on port 9283 on mgr nodes.  Automatically
  enables performance counters and stats for all pools.

## Container / cephadm variables

This role deploys Ceph using **cephadm** — Ceph's container-based management
framework. All daemons run as OCI containers managed by podman.

* `ceph_version`: The Ceph version tag to deploy, e.g. `19.2.3`. On a new
  cluster this controls what image is pulled. On an existing cephadm-managed
  cluster, **changing this value triggers a rolling upgrade** via
  `ceph orch upgrade`. Do not change this at the same time as migrating from
  native packages — migrate first, upgrade separately. Default: `19.2.3`.
* `ceph_container_image`: Container image path, without the tag. Default:
  `quay.io/ceph/ceph`. Override this when using a private registry mirror
  (see *Using a private registry mirror* below).
* `ceph_cephadm_user`: System user created on all nodes that cephadm uses
  for SSH management. Defaults to `cephadm`. This user is given passwordless
  sudo on all cluster nodes and holds the SSH keypair used for inter-node
  communication. The private key lives on all MON nodes (any MON may host the
  active orchestrator); the public key is distributed to all cluster nodes.
* `ceph_osd_min_size`: Minimum disk size in bytes for OSD auto-discovery.
  Disks smaller than this are ignored. Default: `1099511627776` (1 TiB),
  mirroring the existing 1 TB filter.
* `ceph_osd_rotational`: Drive type filter. `~` (null) accepts any drive type,
  `false` selects only SSDs/NVMe, `true` selects only HDDs. Default: `~`.
* `ceph_osd_memory_target_autotune`: Whether to let cephadm autotune
  per-OSD memory based on host RAM. Pinned to `false` by default because
  the autotuned value (~0.7 × host_RAM / num_osds) crowds out colocated
  workloads on hyperconverged hosts.
* `ceph_osd_memory_target`: Per-OSD memory target in bytes when
  autotune is disabled. Default: `4294967296` (4 GiB), matching the
  upstream `osd_memory_target` default.
* `ceph_container_registry`: Registry hostname used as a prefix for the
  container image. Default: `quay.io`. Override when pulling from a mirror.
* `ceph_container_registry_user`: Username for registry authentication.
  Leave empty for unauthenticated (public) registries. Default: `""`.
* `ceph_container_registry_password`: Password for registry authentication.
  Default: `""`.

### Using a private registry mirror

If your nodes cannot reach `quay.io` directly, mirror the Ceph image to an
internal registry and point the role at it:

```yaml
# group_vars/ceph_mycluster_cluster.yaml
ceph_container_registry: "registry.mycompany.com"
ceph_container_image: "registry.mycompany.com/ceph/ceph"
# Optional, only if the mirror requires auth:
ceph_container_registry_user: "robot"
ceph_container_registry_password: "{{ vault_registry_password }}"
```

Mirror the image with:
```bash
podman pull quay.io/ceph/ceph:v19.2
podman tag  quay.io/ceph/ceph:v19.2 registry.mycompany.com/ceph/ceph:v19.2
podman push registry.mycompany.com/ceph/ceph:v19.2
```

The role configures `/etc/containers/registries.conf.d/ceph-mirror.conf` on
every node to redirect pulls automatically.

---

## Migrating from native packages to cephadm

### What changes

| Aspect | Before | After |
|---|---|---|
| Daemon execution | systemd units running binaries | systemd units running containers via podman |
| Config file | `/etc/ceph/ceph.conf` | Stored in cluster config DB (`ceph config set`) |
| NFS config | `/etc/ganesha/ganesha.conf` | Stored in `.nfs` RADOS pool |
| Binary tools | `/usr/bin/ceph`, `/usr/bin/radosgw-admin`, etc. | Available via `cephadm shell` |
| Upgrade path | `apt upgrade ceph` | `ceph orch upgrade start --image ...` (or bump `ceph_version`) |
| All role variables | — | Unchanged |
| All cluster features | — | Unchanged |

No data is moved, deleted, or reformatted during migration. OSD LVM metadata is
preserved exactly. CephFS and RGW data are not touched.

### Do all hosts need to migrate at the same time?

**No.** Migration is incremental and rolling. The cluster continues serving I/O
throughout. A cluster where some nodes are containerized and others are still
running native packages is a normal intermediate state — the Ceph protocol is
agnostic to how each daemon is launched.

When you run the playbook against all nodes, the role automatically:
- Sequences MON adoptions one at a time to maintain quorum
- Adopts OSDs, MDS, and RGW across multiple nodes in parallel
- Defers NFS migration until all other daemons are adopted
- Skips any daemon already converted on a previous run

You can also target a subset of nodes with `--limit` and the role picks up
where it left off on the next full run.

### Prerequisites

Before starting, verify:

1. **Cluster is healthy**: `ceph health` must return `HEALTH_OK`. Do not begin
   migration with a degraded or recovering cluster.
2. **All OSDs are up**: `ceph osd stat` — `num_up_osds` must equal `num_osds`.
3. **Ceph version ≥ Octopus (v15)**: `ceph version`. Earlier releases do not
   support `cephadm adopt`.
4. **Set `ceph_version` to match what is currently installed**:
   ```bash
   ceph version   # e.g. "ceph version 19.2.3 ..."
   ```
   Then in your group vars:
   ```yaml
   ceph_version: "19.2.3"   # must match major.minor.rel of installed version
   ```
   The role asserts this match and fails early if it does not. This prevents
   accidentally upgrading and migrating simultaneously.
5. **Internet or mirror access**: All nodes must be able to pull the container
   image (see *Using a private registry mirror* if needed).

### Migration procedure

**Step 1 — Add the new variables to your inventory**

In your cluster group vars, add at minimum:
```yaml
ceph_version: "19.2.3"          # match your currently installed version
ceph_container_image: "quay.io/ceph/ceph"
```

All other new variables have safe defaults and are optional.

**Step 2 — Run a dry-run check (optional but recommended)**

```bash
ansible-playbook deploy.yml --tags ceph --check
```

This will report what the role would do without making changes. Watch for any
assertion failures from the version check or health check tasks.

**Step 3 — Run the migration**

```bash
ansible-playbook deploy.yml --tags ceph
```

The role detects the legacy installation and runs the migration automatically.
You do not need a separate migration playbook.

What happens, in order:

1. The `cephadm` system user is created on all nodes, SSH keys are distributed,
   podman is installed, the container image is pre-pulled.
2. Pre-flight checks run on the bootstrap node (health, OSD counts, version).
3. The bootstrap MON and MGR are adopted into containers.
4. The cephadm orchestrator is enabled and pointed at the `cephadm` SSH user.
5. All cluster hosts are registered with the orchestrator.
6. Remaining MONs are adopted one at a time with quorum verification between
   each. This is the most time-sensitive sequence (~2 min per MON for a typical
   node).
7. MDS and RGW nodes are adopted in parallel across nodes.
8. OSDs are adopted in parallel across nodes (~30 s per OSD).
9. **NFS migration** (if used): all NFS nodes stop the legacy Ganesha service
   simultaneously, the cephadm NFS container starts, and exports are recreated
   via `ceph nfs export`. Client interruption target: under 5 minutes.
10. Legacy packages are removed.
11. CRUSH topology is validated and corrected where needed.
12. Nice-level systemd overrides are applied to the new container service units.

**Step 4 — Verify the migration**

```bash
ceph health
ceph orch ps           # all daemons should show 'running'
ceph osd stat          # same OSD count as before
ceph fs status         # CephFS healthy
ceph orch host ls      # all hosts registered
```

For NFS:
```bash
ceph nfs export ls <fs_name>    # exports present
showmount -e <nfs_vip>          # exports visible from a client
```

### What to do if the migration stops partway

The migration is re-entrant. If the playbook fails or is interrupted at any
point, simply fix the underlying issue and re-run:

```bash
ansible-playbook deploy.yml --tags ceph
```

Already-migrated daemons are detected and skipped. The role resumes from
where it stopped.

If you want to check the state before re-running:
```bash
# Shows which daemons are cephadm-managed vs still legacy
ceph orch ps                              # cephadm-managed daemons
systemctl list-units 'ceph-[^@]*@*'       # legacy daemons (if any remain)
```

### There is no automated rollback

Once `cephadm adopt` converts a daemon, reverting it to native packages is a
manual process and not recommended. Verify `ceph health` is `HEALTH_OK` at
each checkpoint before continuing to the next group of nodes. The pre-flight
assertions at the start of each run enforce this automatically.

### Upgrading Ceph version after migration

After migration is complete, upgrade by bumping `ceph_version` in your group
vars and re-running the playbook:

```yaml
ceph_version: "20.0.1"   # or the next release
```

```bash
ansible-playbook deploy.yml --tags ceph
```

The role detects that the deployed version differs from `ceph_version`, runs
`ceph orch upgrade start --image quay.io/ceph/ceph:v20.0.1`, and polls until the
upgrade is complete before proceeding. The upgrade is a rolling restart — no
downtime for RBD or RGW; CephFS and NFS experience brief per-MDS pauses during
MDS restarts.

---

## Groups used by this role

NOTE: When `ceph_cluster_name` is specified below, all hyphens (`-`) will be
      replaced with underscores (`_`) to comply with group name requirements.
* `ceph_{{ ceph_cluster_name }}_mon`: All members of this group will deploy
  monitors.
* `ceph_{{ ceph_cluster_name }}_mds`: All members of this group will deploy
  mds daemons.
* `ceph_{{ ceph_cluster_name }}_osd`: All members of this group will deploy
  OSDs. Disks available on the host will automatically be created as OSDs
  as long as they are non-removable, do not currently contain a partition
  table, and are at least 1TB in size.
* `ceph_{{ ceph_cluster_name }}_rgw`: All members of this group will deploy
  rados gateways.



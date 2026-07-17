# IDC Phase 1: What's Supported and Known Issues

This page explains what you can and cannot do with cluster-configs in Phase 1, and the
known issues we are shipping with.

- **Section 1, Scope:** things Phase 1 does not do, by design.
- **Section 2, Known issues:** bugs we know about and have accepted for Phase 1, each
  with a workaround.
- **Section 3, Coming soon:** gaps we plan to fix.

If you hit something on this page, you don't need to report it.

---

## 1. Scope

### 1.1 What you can build

| Machine type | GKE | Slurm | VMs (`jbvm`) |
| :-- | :-: | :-: | :-: |
| `a3-megagpu-8g` (H100 Mega) | ✅ | — | — |
| `a3-ultragpu-8g` (H200) | ✅ | ✅ | ✅ |
| `a4-highgpu-8g` (B200) | ✅ | ✅ | ✅ |
| `a4x-highgpu-4g` (GB200) | ✅ | ✅ | ✅ |
| `ct6e-standard-*` (TPU v6e, e.g. `ct6e-standard-4t`) | ✅ | — | — |
| Any other CPU machine type (e.g. `n2-standard-8`, `c2-standard-60`, `h3-standard-88`) | ✅ | ✅ | ✅ |

- **Cluster types:** `gke`, `slurm` and `jbvm`. Cloud Batch and HTCondor are not
  supported.
- **Not yet supported:** A2, G2, G4, H4D, other A3 variants (e.g. `a3-highgpu-8g`),
  A4X Max, and TPU v4, v5e and v5p. Using one gives a clear error.
- **GKE only:** the `gke-hyperdisk`, `kueue-jobset` and `gke-job-template` features, and
  all job overlays.
- **Slurm and VMs only:** the `netapp-volume` and `startup-script` features.
- **Pricing model:** everything defaults to **on-demand**. Spot, DWS flex-start and
  reservations are opt-in, and each cluster-config shows how to turn them on.

### 1.2 One accelerator type per cluster

A cluster can use **one** GPU or TPU machine type, on as many pools as you like, plus any
number of CPU pools. Mixing two GPU types (e.g. H200 and B200), or GPUs and TPUs, is
rejected:

```text
combining different accelerator archetypes in one cluster is not currently supported: ...
```

**Workaround:** deploy each accelerator type as its own cluster.

### 1.3 Pool settings change that pool only

A setting under a pool's `settings:` changes that pool and nothing else. Parts of the
cluster the pools share, such as the Slurm controller, the login node, the network or the
GKE cluster itself, are not touched. To change those, use `config_base` settings (see
[Config Base](../v2/config-base/README.md)).

This includes sizing values such as boot disk size, node count, number of TPU slices and
number of VMs. Priority order is: pool setting, then top-level `vars:`, then the built-in
default.

```yaml
vars:
  disk_size_gb: 100         # used by any pool that does not set its own
compute_archetypes:
  - name: big
    machine_type: n2-standard-8
    settings:
      disk_size_gb: 300     # this pool only
  - name: small
    machine_type: n2-standard-4   # gets 100
```

To target a specific sub-module within a multi-module pool or feature, nest under its
module ID (e.g. `"{name}_partition": { exclusive: false }`).

### 1.4 Attaching features to pools, login, or controller

- Storage features (`filestore`, `managed-lustre`, `cloud-storage`, `netapp-volume`,
  `aiml-gcsfuse`, `pre-existing-network-storage`) provision a **single shared storage
  instance** in `cluster-env` and mount it across all pools by default, or only on the
  pools listed in `attach_to`.
- The `startup-script` feature (on `slurm` and `jbvm`) can target specific compute pools,
  `login`, or `controller` via `attach_to: [controller, login, <pool-name>]`.
- The names `controller`, `login` and `compute` are reserved and cannot be used as
  compute pool names.

Phase 1 still has **no**:

- Spack or Ramble feature. On Slurm and VMs, install software with a `startup-script`
  feature instead.
- more than one login node

### 1.5 Overlays

- An overlay can add to a cluster but can't remove or replace anything.
- Shipped job overlays (`fio-bench-job`, `nccl-jobset-test`, `run-nvidia-smi`) preset
  `gke-job-template` and are GKE-only. There are no Slurm or VM overlays yet.
- **Pool targeting:** In a mixed GKE cluster with both accelerator (GPU/TPU) and CPU pools,
  an overlay without `attach_to` automatically targets the accelerator pool(s). Use
  `attach_to` to target a specific pool explicitly.
- On GKE there is no way yet to install arbitrary software on nodes.

### 1.6 What a cluster-config cannot express

A cluster-config has five top-level sections: `config_base`, `vars`,
`compute_archetypes`, `features` and `overlays`. Anything else, including `aliases`, is
rejected.

| You can't… | What to do instead |
| :-- | :-- |
| Set inputs that the underlying Terraform modules do not expose | Only inputs declared in the underlying Terraform module `variables.tf` (or catalog `vars:`) are accepted |
| Add arbitrary third-party Terraform modules inline | Add a catalog feature under `v2/features/` or use a regular (non-IDC) blueprint |
| Build custom images with Packer | Accelerator types that need extra software install it when the node boots, or use `startup-script` |
| Set `terraform_backend_defaults` inside `cluster-config.yaml` | Pass a deployment file via `gcluster create -d deployment.yaml` (which supports `terraform_backend_defaults`) or use `gcluster deploy --backend-config` |
| Put two pools in one Slurm partition | Each pool becomes its own partition |

**Command-line vars & deployment files:** `-d` (`--deployment-file`) and `-v` (`--vars key=value`)
both work with `v2` cluster configs (`-v` takes highest precedence, followed by `-d`,
then `cluster-config` `vars:`). Values passed via `--vars` that look like numbers become
numbers (`--vars version_prefix=1.30` becomes `1.3`), so put string-typed numeric prefixes
in `vars:` with quotes: `version_prefix: "1.30"`.

---

## 2. Known issues (accepted for Phase 1)

### 2.1 Machine-type typos aren't always caught when offline

CPU machine types are checked against Google Cloud when you compile. That check is
**skipped without a warning** when `gcluster` can't reach Google Cloud (not logged in,
Compute API disabled, missing permissions), or when `project_id` or `zone` isn't set. A typo
such as `n2-standrad-8` then compiles fine and only fails at deploy time.

**Workaround:** run `gcloud auth application-default login`, and set `project_id` and
`zone`.

### 2.2 Don't use underscores in pool or feature names

IDC uses `_` internally to tell which pool something belongs to. Names that contain `_`
can get mixed up. For example, with pools `gpu_spot` and `gpu` (in that order), a
`startup-script` feature with `attach_to: [gpu]` runs on `gpu_spot` instead of `gpu`.

**Workaround:** use hyphens, e.g. `gpu-spot` or `training-data`.

---

## 3. Coming soon (fix planned)

### 3.1 Nested mount paths aren't caught

Exact duplicate mount paths (including trailing slashes like `/home` vs `/home/` and GKE `pv_mount_path` vs `local_mount`) are rejected when you compile. However, nested mount paths where one directory is inside another (such as `/data` and `/data/scratch`) are not caught at compile time.

**Workaround:** give every storage feature its own, separate mount path.

### 3.2 GKE job overlays mount every storage volume

```yaml
features:
  - {name: home,    type: filestore,      settings: {local_mount: /home}}
  - {name: scratch, type: managed-lustre, settings: {local_mount: /scratch}}
overlays:
  - {name: bench, type: fio-bench-job}
```

The `bench` job mounts **both** `home` and `scratch`. There is no way to limit this yet.

**Workaround:** none needed in most cases. The extra mount is harmless unless its path
clashes with one the job uses.

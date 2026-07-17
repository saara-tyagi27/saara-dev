# Tab 1 — Intent-Driven Compiler (IDC): How to Use It

## 1\. Setup & Installation

On your cloudtop, pull the CL and build the compiler:

```sh
git clone sso://cluster-toolkit/cluster-toolkit idc-bug-bash
cd idc-bug-bash
git fetch origin refs/changes/00/14300/17
git checkout FETCH_HEAD -b idc-bug-bash
make
```

### Verify your build

```sh
./gcluster --version
ls v2/
```

You should see five directories: `cluster-configs`, `compute-archetypes`, `config-base`, `features`, `overlays`. That is the IDC catalog.

> [!IMPORTANT]
> Run `./gcluster` from the repo root. The compiler loads the catalog from the `v2/` folder next to the binary.
>
> Even `expand` talks to GCP to look up your machine type. Use a real `project_id` and a `zone` where that machine type exists.

---

## 2\. The Mental Model ([go/ctk-idc](http://goto.google.com/ctk-idc))&nbsp;

An IDC cluster configuration is built from five stacked layers. You write the top one (unless you want your own layer specifications); the compiler pulls in the rest.

| Layer | What it is |
| :---- | :---- |
| **1\. Cluster Config** | Your file. \~20 lines. Names the base, machines, and capabilities you want. |
| **2\. Config Base** | The skeleton — networking, controller, login, cluster wiring. |
| **3\. Compute Archetype** | Hardware fact sheet — GPU counts, NIC topology, images, placement. |
| **4\. Feature** | Something the cluster **has** — storage, dashboards, job queues. |
| **5\. Overlay** | User specific customisations — a benchmark, a test job, a training job. (limited to additions — patching to lower layers not permitted. *This means an overlay cannot modify or override the settings of a Config Base or Archetype; it can only add new functionality.*) |

---

## 3\. Start From an Existing Config

You do not have to start from scratch. `v2/cluster-configs/` already contains 12 validated starting points.  
Open one and fill them in.

### 3.1 Example A — fill in a shipped config

`v2/cluster-configs/a4high-slurm.yaml`

```yaml
config_base: slurm

vars:
  deployment_name: a4high-slurm # supply unique deployment name
  project_id:                   # supply your personal GCP project id
  region:                       # supply region with a4-highgpu-8g capacity
  zone:                         # supply zone with a4-highgpu-8g capacity
  a4_cluster_size: 2            # supply node count
  a4_reservation_name: ""       # optional: supply a4-highgpu-8g reservation name
  # Or use one of these instead of a reservation (pick only one):
  # a4_dws_flex_enabled: true
  # a4_enable_spot_vm: true

compute_archetypes:
  - name: a4high-compute        # becomes the partition name
    machine_type: a4-highgpu-8g
    settings:
      node_count_static: $(vars.a4_cluster_size)

features:
  - name: homefs                # becomes the storage module name
    type: managed-lustre
    settings:
      size_gib: 36000
      per_unit_storage_throughput: 500
      local_mount: /home

  - name: ml-data
    type: aiml-gcsfuse
```

**Compile it:**

```sh
# Compile into expanded blueprint
./gcluster expand v2/cluster-configs/a4high-slurm.yaml -o out.yaml

./gcluster create v2/cluster-configs/a4high-slurm.yaml -w
```

Open `out.yaml` to see what those ~25 lines expanded into.  
> [!TIP]
> Use `expand`/`create` for fast iteration while authoring a config — it is the quickest way to see whether the compiler accepts your file and compiled into the intended blueprint.

**Optionally:**
*(Note: If you choose to deploy, you will be using your own personal GCP project. Please ensure you have supplied your project ID in the config variables and have the necessary quota/permissions.)*

```sh
./gcluster deploy a4high-slurm     # actually deploy
./gcluster destroy a4high-slurm    # tear it down
```

&nbsp;

---

### 3.2 Example B — swap one feature for another

Same file. The cluster above puts `/home` on Managed Lustre. To put it on Filestore instead, change the `type:` and give it the settings Filestore expects. **Nothing else in the file changes.**

**Replace managed lustre feature with filestore:**

```yaml
features:
  - name: homefs
    type: filestore
    settings:                # settings are optional here
      filestore_tier: HIGH_SCALE_SSD
      size_gb: 10240
      local_mount: /home
```

Expand again and diff the output against the Lustre version — the storage modules and all their wiring are swapped for you.

**To add a feature**, append another entry. Features stack:

```yaml
  - name: cluster-metrics
    type: monitoring-dashboard
```

**To remove a feature**, delete its block. Delete the whole `features:` key if you want none.

---

### 3.3 Example C — GKE with two node pools and an overlay

`v2/cluster-configs/a4high-gke.yaml` has one GPU pool. Here is a trimmed version with a **second CPU pool** added and an **overlay** that runs `nvidia-smi`.

```yaml
config_base: gke

vars:
  deployment_name: a4high-gke
  project_id: my-gcp-project
  region: us-central1
  zone: us-central1-b
  authorized_cidr: 1.2.3.4/32   # your IP as IP_ADDRESS/32, or 0.0.0.0/0 for all
  a4_enable_spot_vm: true

compute_archetypes:
  - name: a4high-compute
    machine_type: a4-highgpu-8g
    settings:
      static_node_count: 1
      spot: $(vars.a4_enable_spot_vm)

  - name: cpu-compute           # <-- second pool, just append another entry
    machine_type: n2-standard-8
    settings:
      static_node_count: 2

features:
  - name: ai-workload
    type: kueue-jobset
    settings:
      kueue:
        install: true
        config_path: $(vars.kueue_configuration_path)
        config_template_vars:
          num_gpus: ((module.a4high-compute_pool.static_gpu_count))
          accelerator_type: nvidia-b200
      jobset:
        install: true

overlays:
  - name: gpu-check
    type: run-nvidia-smi
```

**Two things to notice:**

1. **Adding a pool is one block.** You get a second GKE node pool. Any storage or monitoring feature you declared is wired to **both** pools automatically.  
2. **`attach_to:` pins a feature or overlay to specific pools.** In a mixed GPU + CPU GKE cluster, an unattached overlay automatically targets the accelerator pool(s), or you can set `attach_to` explicitly:

| Config | Resulting module | Pools it targets |
| :---- | :---- | :---- |
| with `attach_to: [a4high-compute]` | `a4high-compute_gpu-check` | GPU pool only ✅ |
| without `attach_to` (mixed GPU + CPU) | `gpu-check` | GPU pool(s) automatically ✅ |

> [!NOTE]
> Overlays are GKE-only. So are the `kueue-jobset`, `gke-hyperdisk` and `gke-job-template` features.

---

### 3.4 Switching a config to a different base

To take `a4high-slurm.yaml` to GKE (or the reverse).

#### 1. The base line

```yaml
-config_base: slurm
+config_base: gke
```

**2\. The `vars:` block.** This is the complete list of what each base requires — nothing else is mandatory.

| Variable | `gke` | `slurm` | `jbvm` |
| :---- | :---: | :---: | :---: |
| `deployment_name` | required | required | required |
| `project_id` | required | required | required |
| `region` | required | required | required |
| `zone` | required | required | required |
| `authorized_cidr` | optional | — | — |

> [!IMPORTANT]
> On GKE, `authorized_cidr` defaults to `0.0.0.0/0` (IAM still applies). Set it to `<YOUR-IP>/32` to lock the control plane down to your machine.

**3\. The pool `settings:`.** These go straight to each base's node-pool module, so the key names differ. For example, the node count is `node_count_static` on `slurm` and `static_node_count` on `gke`. If you forget, the compiler tells you the setting is invalid and lists the inputs the module accepts.

```yaml
    settings:
-     node_count_static: $(vars.a4_cluster_size)
+     static_node_count: $(vars.a4_cluster_size)
```

&nbsp;

---

## 4\. Build Your Own Config

Same shape as what you just edited. Start an empty file and add one block at a time.

**Step 1 — pick the base.** One line.

```yaml
config_base: slurm              # gke | slurm | jbvm
```

**Step 2 — say where it lives.**

```yaml
vars:
  deployment_name: my-cluster
  project_id: my-gcp-project
  region: us-central1
  zone: us-central1-a
```

If you picked `gke`, add `authorized_cidr: 1.2.3.4/32` to restrict control-plane access (it defaults to `0.0.0.0/0`).

**Step 3 — declare your compute.** One entry per node pool / partition.

```yaml
compute_archetypes:
  - name: my-nodes              # becomes the pool / partition name
    machine_type: a4-highgpu-8g
    settings:
      node_count_static: 2      # static_node_count on gke
```

Add more entries for more pools. This block is required.

**Stop here and check it works:**

```sh
./gcluster expand my-config.yaml -o out.yaml
```

That is already a complete cluster. Everything below is optional.

**Step 4 — add storage or other features.**

```yaml
features:
  - name: shared-home
    type: filestore
    settings:
      size_gb: 2560
      local_mount: /home
```

Expand again. Add one feature at a time so you know which one broke it.

**Step 5 — (gke only) add a workload.**

```yaml
overlays:
  - name: gpu-check
    type: run-nvidia-smi
```

Expand a final time, then `./gcluster create my-config.yaml -w`.

> [!TIP]
> Anything under a `settings:` block is passed straight through to the underlying Terraform module, so **any input that module accepts, you can set here**. *(Note: A Catalog Reference tab detailing these settings will be provided soon).*

---

---

## 5\. Things to Try During the Bug Bash

Please mix and match. *(Note: A Catalog Reference detailing all available combinations will be provided soon).*

1. **Swap the base.** Take `a4high-slurm.yaml` → `gke` using 3.4. Does it still work? Do the storage mounts follow?  
2. **Stack storage features.** Three filesystems on one cluster — do they all mount?

   ```yaml
   features:
     - {name: home, type: filestore, settings: {local_mount: /home}}
     - {name: scratch, type: managed-lustre, settings: {local_mount: /scratch}}
     - {name: data, type: cloud-storage, settings: {local_mount: /data}}
   ```

3. **Override deep module settings.** Push an unusual value into any `settings:` — a nonstandard tier, a custom mount path, a version pin. Does it land in the expanded output?

4. **Deliberately break things.**

| Try | Expected |
| :---- | :---- |
| A machine type that doesn't exist (`z9-megaflop-99`) | Clear error naming the machine type and zone, and listing the available archetypes |
| An unsupported base (e.g. `a3-megagpu-8g` on `config_base: jbvm`) | Error stating the archetype does not support the base, and which bases it does support |
| `type: gke-hyperdisk` on `config_base: slurm` | Error naming which bases it supports |
| An overlay (e.g. `run-nvidia-smi`) on `config_base: slurm` | Error saying the overlay only supports `gke` |
| `type: gke-job-template` under `features:` on `gke`, with no `image` | Error asking for `settings.image`, or pointing you to a workload overlay instead |
| Two storage features with the same `local_mount` | Error naming both colliding modules |
| Name a pool `controller` | Error saying the name is reserved |
| Delete the `project_id` key from `vars:` | Error: "`vars.project_id` is required and has no default" |
| Delete the `config_base:` line entirely | Error: "`config_base` is required: pick one of batch, gke, jbvm, slurm" |
| Misspell a setting (e.g. `size_gib` instead of `size_gb` on `filestore`) | A typo suggestion: `did you mean "size_gb"?` |
| `type: pre-existing-network-storage` with no settings on `gke` | Validation error demanding one of `gcs_bucket_name`, `filestore_id`, or `lustre_id` |
| `type: pre-existing-network-storage` with no settings on `slurm` / `jbvm` | Validation error demanding `server_ip` and `remote_mount` |

---

## 6\. How to File a Bug

Please add bugs directly to hotlist [b/hotlists/8922304](https://b.corp.google.com/hotlists/8922304).

Include as much as you can:

- The exact CLI command you ran  
- Your cluster-config YAML (paste to [gpaste](https://paste.googleplex.com) and link it)  
- The full error output  
- What you expected to happen instead  
- Whether it reproduces with `expand` alone, or only at `create` / `deploy`

> [!TIP]
> Before filing, please skim the **IDC Known Limitations** document (which will be provided soon alongside the Bug Bash materials). Several behaviours that look like bugs are documented Phase 1 scope decisions — but if you hit one and think it should be prioritised, file it anyway and say so.

---

### Thanks for helping us harden the Intent-Driven Compiler! 🎉

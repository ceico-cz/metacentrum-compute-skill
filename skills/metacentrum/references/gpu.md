# GPU Jobs and Discovery

Read this reference for GPU resource selection, availability, and diagnostics.

## Request GPUs by requirements

Start with the least restrictive request:

```bash
#PBS -l select=1:ncpus=4:mem=16gb:ngpus=1
#PBS -l walltime=02:00:00
```

Add `gpu_mem`, `gpu_cap`, or `cuda_version` only when the application truly requires them. Verify current syntax and queue policy with PBS because advertised resources can change. Do not pin a node merely because it currently looks idle.

**Do not trust `gpu_cap` (or other custom node resources) to actually gate placement without verifying it.** These are host-level custom resources, and enforcement has been observed to be unreliable in practice: a job requesting `gpu_cap=sm_80` matched and started within seconds on a node whose `resources_available.gpu_cap` was `sm_86` (an A40, not the requested A100 class) — this on a queue where the median wait for a real match is on the order of days, so a near-instant start is strong evidence the constraint was not enforced for that request, not luck. Because typical queue backlog means a few minutes of queueing proves nothing either way, use this test instead: submit a throwaway job with an impossible value for the resource (e.g. `gpu_cap=sm_999_bogus`) and confirm it stays `Q` indefinitely; if it does, the resource is at least being evaluated. Whatever the outcome, always make GPU-architecture-sensitive jobs self-verifying — check `nvidia-smi --query-gpu=compute_cap --format=csv,noheader` (or equivalent) at the very start of the job script and exit non-zero immediately on a mismatch, so a wrong-architecture placement fails fast instead of silently running the real workload on the wrong hardware.

## Interpret availability

- Raw-free GPUs are configured GPUs minus assigned GPUs.
- A GPU is not runnable for a new job if its node has no schedulable CPU, system memory, or other requested resource left.
- Node state `free` means some compatible work may fit; it does not mean idle.
- A snapshot does not guarantee a start because priority, fair-share, reservations, queue access, and concurrent submissions also matter.

Inspect live data on a frontend:

```bash
date --iso-8601=seconds
pbsnodes -av
qstat -Q
qstat -Q -f QUEUE@SERVER
```

For a candidate node, compare these `pbsnodes -av NODE` fields:

```text
state
resources_available.ncpus
resources_assigned.ncpus
resources_available.mem
resources_assigned.mem
resources_available.ngpus
resources_assigned.ngpus
resources_available.gpu_mem
resources_available.gpu_cap
resources_available.cuda_version
resources_available.queue_list
```

Compute a conservative one-CPU-per-GPU estimate as:

```text
cpu_free = available.ncpus - assigned.ncpus
gpu_free = available.ngpus - assigned.ngpus
runnable_gpus = max(0, min(cpu_free, gpu_free))
```

A real job may still not fit due to memory, scratch, walltime, queue, or architecture constraints.

## Restricted clusters and ACL-gated queues

A node showing free GPUs is not necessarily reachable by the current user. `resources_available.queue_list` on each node lists which queues actually route to it (values are prefixed `q_`; strip the prefix to get the real queue name shown by `qstat -Q`, e.g. `q_gpu_dgx` → queue `gpu_dgx`). Do not assume the general public queue (`gpu`) reaches every cluster — check this per node/cluster before recommending it:

```bash
pbsnodes -av | grep -B20 'cluster = CLUSTER_NAME' | grep -E 'cluster =|queue_list'
```

Two failure modes observed on MetaCentrum (node and queue names below come from a 2026 snapshot; re-check them live before relying on them):

- **A cluster's only route is an ACL-restricted queue.** Example: the `capy` node (H100 80GB ×8) has `resources_available.queue_list = q_gpu_dgx` and nothing else — it is not in the general `gpu` queue at all. `gpu_dgx` itself has `acl_group_enable = True` with `acl_groups = dgx,pbs-admins`, so it is invisible to anyone outside that group even though `pbsnodes` reports it fully idle.
- **A node has no queue at all.** Example: `elbi1-1` (an A100 40GB node) has no `resources_available.queue_list` entry whatsoever — it cannot be reached through a normal `qsub` regardless of group membership.

Several other clusters are reachable through the general `gpu` queue but *also* have a second, group-restricted queue layered on top (e.g. `cucam` on `fau`/`fer`/`cha`, `iti` on `alfrid`/`konos`, `zcu` on `alfrid`, `luna` on `luna`, `galaxy_gpu` on part of `galdor` restricted to named users, not a group). Membership in one of these usually means a dedicated/less-contended path into that cluster, not just the shared `gpu` pool.

Check whether a queue is restricted, and to whom, with:

```bash
qstat -Qf QUEUE_NAME | grep -E 'acl_group_enable|acl_groups|acl_user_enable|acl_users|resources_min|resources_max'
```

Check which groups the current user actually has before claiming access to a restricted queue — never assume from the queue name alone:

```bash
ssh FRONTEND id
```

Cross-reference the groups/username against `acl_groups`/`acl_users`. Only recommend a cluster gated behind a restricted queue if the user's own `id` output actually contains a matching group (or their username is in `acl_users`); otherwise point them at the general `gpu` queue (or another cluster they do qualify for) instead, even if the restricted cluster looks idle.

## Validate inside the allocation

PBS constrains visibility with `CUDA_VISIBLE_DEVICES`. Check the assigned hardware from inside the job:

```bash
printf 'CUDA_VISIBLE_DEVICES=%s\n' "${CUDA_VISIBLE_DEVICES:-unset}"
nvidia-smi
nvidia-smi --query-gpu=index,name,driver_version,memory.total,memory.free,compute_cap --format=csv
```

After completion, inspect scheduler-recorded GPU utilization and peak GPU memory:

```bash
qstat -fw JOB_ID | grep -E 'gpupercent|gpupowerusage|gpumemmaxpercent'
```

Official GPU guidance: <https://docs.metacentrum.cz/en/docs/computing/gpu-comput/gpu-job>

# PBS Jobs

Read this reference when creating or operating OpenPBS jobs.

## Resource model

Common resources are `ncpus`, `mem`, `walltime`, `ngpus`, and one of `scratch_local`, `scratch_ssd`, or `scratch_shared`. Confirm queues with `qstat -Q`; organization-specific queues require membership.

## General batch template

```bash
#!/bin/bash
#PBS -N JOB_NAME
#PBS -l select=1:ncpus=2:mem=4gb
#PBS -l walltime=01:00:00
#PBS -j oe

set -euo pipefail
cd "$PBS_O_WORKDIR"

echo "job=$PBS_JOBID host=$(hostname -f) cpus=$PBS_NCPUS"
PROGRAM_COMMAND
```

For a GPU workload, add `ngpus=1` to `select`, request enough system memory, and load or activate the required software inside the script. Do not assume a module name; discover it with `module avail` on the frontend when possible.

## Scratch pattern

Scratch is temporary and must be explicitly requested. Copy inputs in, run there, and copy results back:

```bash
submit_dir="$PBS_O_WORKDIR"
work_dir="$SCRATCHDIR"
cp -a "$submit_dir/input" "$work_dir/"
cd "$work_dir"
PROGRAM_COMMAND
mkdir -p "$submit_dir/results"
cp -a result.txt "$submit_dir/results/"
```

Allow walltime for transfers and protect important partial results when practical.

## Paths inside a job

`$HOME` inside a job is the native home of the **execution node**, not of the frontend the job was submitted from. Verified 2026-10-12 for a job submitted on `metafzu.fzu.cz` (home `/storage/praha1/home/USERNAME`): on `oven.grid.cesnet.cz` `$HOME` was `/storage/brno2/home/USERNAME`, and on `alfrid2.meta.zcu.cz` it was `/storage/plzen1/home/USERNAME`. A script that uses `$HOME/tools/prog` fails with `No such file or directory` on such nodes, even though the file exists on the frontend.

In job scripts, refer to programs and data by absolute `/storage/SITE/home/USERNAME/...` paths, which are mounted on every node, or relative to `$PBS_O_WORKDIR`. On `metafzu.fzu.cz`, `$PBS_O_WORKDIR` resolves to `/auto/vestec1-elixir/home/USERNAME/...`, which is the same storage as `/storage/praha1/`. Tools that create their output directories themselves may also need the parent to exist first (`mkdir -p`).

## Long-running service jobs: the oven queue

The `oven` queue is for long, lightweight jobs such as a workflow manager or task-scheduler server. Values seen on 2026-10-12 (check with `qstat -Qf oven`): maximum walltime 720 hours, default 1 CPU and 48 hours, default chunk memory 100 MB, open to normal users. Jobs run on the single host `oven.grid.cesnet.cz`, whose native home is `/storage/brno2`. Request it explicitly:

```bash
#PBS -q oven
#PBS -l select=1:ncpus=1:mem=1gb
#PBS -l walltime=168:00:00
```

Do not put computation there. The node is shared by everyone's long-running services.

## Inspect a running job on its node

From a frontend you can SSH to a node where you have a running job (the delegated Kerberos ticket is used), for example to check processes or files the job created:

```bash
qstat -f JOB_ID | grep exec_host    # e.g. exec_host = oven/7
ssh oven.grid.cesnet.cz 'pgrep -u "$USER" -a PROGRAM; echo "$HOME"'
```

Keep this to short, read-only checks.

## Many small tasks and multiple clusters: HyperQueue

For thousands of short tasks, or one set of tasks spread over MetaCentrum and Slurm clusters, use [HyperQueue](https://it4innovations.github.io/hyperqueue/stable/) instead of one PBS job per task. An `hq server` runs as a job in the `oven` queue, and `hq worker` runs as ordinary PBS (and Slurm) jobs that pull tasks from it. A tested walkthrough for MetaCentrum, Phoebe and Koios is at <https://hpc.ceico.cz/systems/metacentrum/hyperqueue/>. Two MetaCentrum specifics:

- Start PBS workers with `hq worker start --manager pbs --detect-resources cpus --resource "mem=sum(MiB)"`, where MiB matches the job's `mem=` request. Without this the worker offers the whole node's GPUs and memory (see "Diagnose and tune").
- When workers run on several clusters, submit tasks with `--cwd /tmp --stdout none --stderr none` and let each task write its output to a path it builds on the worker. Otherwise tasks inherit the submit host's paths and fail on other clusters.

## Job lifecycle

```bash
qsub job.pbs                 # submit; capture the returned job ID
qstat -u "$USER"             # list current jobs
qstat -f JOB_ID             # inspect one job
qstat -x -f JOB_ID          # inspect history and Exit_status
qstat -f QUEUE_NAME          # inspect all jobs in one queue (e.g. `qstat -f gpu`), without dumping the whole cluster
qdel JOB_ID                 # cancel only when the user requests it
```

Use `qdel -W force JOB_ID` only after ordinary deletion fails and the user confirms the exact job. For short interactive work, use `qsub -I` with explicit resources and walltime.

## Diagnose and tune

For a queued job, inspect its comment, queue, requested resources, reservations, and eligibility with `qstat -f`. Apparent capacity does not override fair-share, priority, queue access, maintenance, or reservations.

For a finished job, compare `resources_used` with the request and examine stdout/stderr. Tools such as `lscpu`, `free`, and `df` show the shared node, not necessarily the allocation. The same holds for software that auto-detects hardware: on 2026-10-12 a HyperQueue worker in a job with `ncpus=2:mem=2gb` and no `ngpus` reported 2 NVIDIA GPUs and 1.5 TB of memory on a GPU node. Pass the allocation to such tools explicitly instead of letting them probe the node. Within a job prefer PBS variables and:

```bash
qstat -f "$PBS_JOBID"
```

Official references: [resources](https://docs.metacentrum.cz/en/docs/computing/resources/resources), [PBS commands](https://docs.metacentrum.cz/en/docs/computing/resources/pbs-commands), and [queues](https://docs.metacentrum.cz/en/docs/computing/resources/queues).

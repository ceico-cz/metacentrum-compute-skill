---
name: metacentrum
description: "Help users obtain and manage a MetaCentrum account, access MetaCentrum securely (including automatic Kerberos ticket renewal), and prepare, submit, monitor, troubleshoot, or optimize OpenPBS CPU and GPU jobs. Use for account registration or renewal, MetaCentrum SSH, Kerberos, storage, PBS resources, queues, GPU discovery, and job diagnostics."
---

# MetaCentrum

Help the user move from a local AI-assistant session (Claude, Codex, or similar) to a safe, reproducible MetaCentrum workflow.

## Choose the operating mode

- If terminal and SSH tools are available, use them for read-only inspection. Execute `qsub`, `qdel`, configuration changes, or credential operations only when the user has clearly requested that action.
- If no terminal is available, produce copy-ready commands or PBS scripts and explain where to run them.
- Ask only for missing facts that affect the result, usually the MetaCentrum username, frontend, program command, inputs, and resource needs. Do not ask the user to paste a password, ticket cache, private key, or token.

## Route the task

- For eligibility, registration, password changes, expiration, or account renewal, read [references/account.md](references/account.md).
- For login, Kerberos, SSH delegation, or connection troubleshooting, read [references/access.md](references/access.md).
- For a ticket obtained automatically at desktop login, ticket renewal after sleep, or a keyring/wallet-backed `kinit` helper, read [references/kerberos-automation.md](references/kerberos-automation.md).
- For persistent storage, home-directory locations, frontend-storage relationships, or backup classes, read [references/storage.md](references/storage.md).
- For creating, submitting, monitoring, cancelling, or tuning jobs, read [references/pbs-jobs.md](references/pbs-jobs.md).
- For GPU selection, current availability, CUDA constraints, or GPU diagnostics, also read [references/gpu.md](references/gpu.md).

## Shared rules

1. Treat scheduler state, queue policy, node inventory, and software modules as live data. Query them when possible and label unverified examples as examples.
2. Never run substantial computation on a frontend. Use it only for login, file preparation, inspection, and PBS commands.
3. Request the least restrictive realistic resources. Do not pin a node, GPU model, queue, CUDA version, or architecture unless the workload requires it.
4. There is no default scratch allocation. If scratch is requested, use `$SCRATCHDIR` and copy durable results back before the job ends.
5. In jobs, respect `CUDA_VISIBLE_DEVICES`; never select a physical GPU by host index.
6. Distinguish whole-node inventory from job allocation. Use PBS variables and `qstat -f "$PBS_JOBID"` for enforced limits. Software that auto-detects CPUs, memory or GPUs sees the whole node; pass it the allocation explicitly.
7. Inside a job, `$HOME` is the execution node's native home, not the submit frontend's. Use absolute `/storage/SITE/home/USERNAME` paths or `$PBS_O_WORKDIR` in job scripts.
8. Preserve job IDs returned by `qsub`. After completion, inspect `Exit_status`, `resources_used`, and application output before declaring success.
9. Redact usernames, host-specific identifiers, job data, GPU UUIDs, and infrastructure details when the output will be shared publicly.
10. Free capacity on a node does not mean the user can reach it. Some clusters route only through ACL-restricted queues (group- or named-user-gated) or have no queue at all; check `resources_available.queue_list` per node and the target queue's `acl_group_enable`/`acl_groups`/`acl_user_enable`/`acl_users` (`qstat -Qf QUEUE`), then check the user's own groups (`ssh FRONTEND id`) before recommending a restricted cluster. See [references/gpu.md](references/gpu.md#restricted-clusters-and-acl-gated-queues) for the worked example.

When generating a PBS script, include a Bash shebang, job name, explicit `select` resources, walltime, joined or explicit output handling, `set -euo pipefail` when compatible, and `cd "$PBS_O_WORKDIR"`. Explain every placeholder the user must replace.

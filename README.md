# MetaCentrum compute skill

An agent skill that helps you use [MetaCentrum](https://metacentrum.cz/), the
Czech national grid run by CESNET (e-INFRA CZ), from Claude or Codex. It covers
the whole path from account to finished job:

- **Account**: eligibility, registration, password changes, annual renewal.
- **Access**: choosing a frontend, Kerberos tickets, SSH with GSSAPI
  credential delegation, connection troubleshooting.
- **Kerberos automation** (optional): renewable tickets, `krb5-auth-dialog`,
  keyring-backed `kinit` at desktop login.
- **Storage**: `/storage/SITE/home` layout, frontend-to-storage mapping,
  backup classes, scratch.
- **PBS jobs**: batch templates, scratch staging, the `oven` queue for
  long-running services, HyperQueue for many small tasks, job lifecycle and
  diagnostics.
- **GPUs**: resource requests, live availability, ACL-restricted queues,
  in-job verification.

The skill uses terminal and SSH tools for read-only inspection when they are
available. It runs `qsub`, `qdel`, configuration changes or credential
operations only when you ask for them. It never asks for your password, ticket
cache or private keys.

## Install

### Claude Code (plugin marketplace)

```text
/plugin marketplace add ceico-cz/metacentrum-compute-skill
/plugin install metacentrum@ceico-metacentrum
```

### Claude Code (manual)

```bash
git clone https://github.com/ceico-cz/metacentrum-compute-skill.git
cp -r metacentrum-compute-skill/skills/metacentrum ~/.claude/skills/
```

Use `.claude/skills/` inside a project instead to enable it for that project
only.

### Claude apps (claude.ai, desktop)

Build a zip with `SKILL.md` inside the top-level `metacentrum/` folder:

```bash
cd skills && zip -r ../metacentrum-skill.zip metacentrum
```

Then upload `metacentrum-skill.zip` under **Settings → Capabilities → Skills**.

### Codex

```bash
cp -r skills/metacentrum ~/.codex/skills/
```

`skills/metacentrum/agents/openai.yaml` provides the Codex display metadata.

## Usage

The skill triggers on MetaCentrum-related requests, for example:

- "Set up Kerberos SSH to a MetaCentrum frontend for user `alice`."
- "Write a PBS script that runs my Python training on one GPU with 32 GB RAM
  for 12 hours, using scratch."
- "Which GPU clusters can I actually submit to right now?"
- "My job 12345678.pbs-m1.metacentrum.cz finished but produced no output;
  find out why."

Scheduler state, queues, node inventory and modules change over time. The
skill treats them as live data and checks them on a frontend when it can.

## Repository layout

```text
.claude-plugin/        Claude Code plugin and marketplace manifests
skills/metacentrum/    the skill (SKILL.md + references/ + agents/openai.yaml)
AGENTS.md              guidelines for contributors and coding agents
```

## Contributing

See [AGENTS.md](AGENTS.md). Pull requests with corrections from real
MetaCentrum use are welcome. Please redact usernames, job IDs, GPU UUIDs and
other identifying details.

Official MetaCentrum documentation: <https://docs.metacentrum.cz/>

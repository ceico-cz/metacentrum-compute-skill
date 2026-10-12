# Repository Guidelines

This repository packages the `metacentrum` agent skill as a Claude Code plugin
and also supports manual installation for Claude apps and Codex.

## Structure

- `.claude-plugin/plugin.json`: plugin manifest. Bump `version` on
  user-visible changes.
- `.claude-plugin/marketplace.json`: single-plugin marketplace pointing at the
  repository root.
- `skills/metacentrum/SKILL.md`: entry point. Frontmatter `name` and
  `description` control triggering. The body routes to references.
- `skills/metacentrum/references/*.md`: detailed topic guides loaded on demand.
- `skills/metacentrum/agents/openai.yaml`: Codex display metadata.

`.gitignore` is an allowlist. To track a new top-level file or directory, add
a `!/NAME` line.

## Writing the skill

- Keep `SKILL.md` short: operating rules and routing. Put details in a
  reference file and link it from the routing list.
- Write for any user. Use placeholders such as `USERNAME`, `FRONTEND`,
  `ALIAS`, `JOB_ID` and `QUEUE_NAME`. Do not add personal hostnames,
  usernames, home paths, or descriptions of one person's machine.
- Treat queues, nodes, limits and modules as live data. When recording an
  observation, give the date and say how to re-check it (`qstat -Qf`,
  `pbsnodes -av`).
- Keep the agent-neutral wording: the skill is used from Claude and Codex.
- Use fenced code blocks with language tags and commands that can be copied
  directly.

## Validation

There is no automated test suite. Before committing:

```bash
claude plugin validate .
python3 -m json.tool .claude-plugin/plugin.json >/dev/null
grep -rn '](references/' skills/metacentrum/SKILL.md   # check that the links resolve
cd skills && zip -r /tmp/metacentrum-skill.zip metacentrum && unzip -l /tmp/metacentrum-skill.zip
```

Test behaviour changes by installing the skill locally (see README) and running
a few realistic prompts. Check PBS examples on a frontend with a short,
low-resource job, and never run heavy commands on a frontend.

## Commits and pull requests

Use short, imperative commit subjects, for example
`docs: clarify GPU queue selection` or `skill: route storage questions`. Keep
each commit focused. In pull requests, list the validation you did and include
sanitized output where it helps.

## Security

Never commit passwords, Kerberos caches, keytabs, tokens, private keys, or
machine-local configuration (`~/.ssh/config`, `/etc/krb5.conf`, wallet
exports). Redact usernames, job data, GPU UUIDs and infrastructure details in
examples and logs.

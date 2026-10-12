# MetaCentrum Access

Read this reference for login, SSH, Kerberos, and delegated credentials.

## Safe discovery

Use one of the documented login servers below. Prefer a site-specific server when
the user is entitled to it or needs convenient access to that site's storage;
otherwise choose a general-purpose server. Host aliases reach the same listed
login server, but use the primary hostname in generated examples unless the user
already has a working alias.

| Primary hostname | Aliases | Native storage | Site | Notes |
| --- | --- | --- | --- | --- |
| `charon.nti.tul.cz` | `charon.metacentrum.cz` | `/storage/liberec3-tul` | Liberec | General login server |
| `elmo.elixir-czech.cz` | `elmo.metacentrum.cz` | `/storage/praha5-elixir` | Praha | Elixir users |
| `nympha.meta.zcu.cz` | `nympha.metacentrum.cz`, `nympha.zcu.cz`, `minos.zcu.cz`, `minos.meta.zcu.cz`, `alfrid.meta.zcu.cz` | `/storage/plzen1` | Plzen | General login server |
| `metafzu.fzu.cz` | `metafzu.metacentrum.cz` | `/storage/praha1` | Praha | Dedicated to FZU users |
| `oven.metacentrum.cz` | none | `/storage/brno2` | Brno | Oven node only |
| `perian.grid.cesnet.cz` | `perian.metacentrum.cz`, `onyx.metacentrum.cz` | `/storage/brno2` | Brno | General login server |
| `skirit.grid.cesnet.cz` | `skirit.metacentrum.cz` | `/storage/brno2` | Brno | General login server |
| `skirit-lite.ics.muni.cz` | `skirit-lite.metacentrum.cz` | `/storage/brno2` | Brno | Lightweight server for quick job management, script editing, and job-status checks |
| `tarkil.grid.cesnet.cz` | `tarkil.metacentrum.cz` | `/storage/praha1` | Praha | General login server |
| `tilia.ibot.cas.cz` | `tilia.metacentrum.cz` | `/storage/pruhonice1-ibot` | Pruhonice | General login server |
| `zenith.cerit-sc.cz` | `zenith.metacentrum.cz` | `/storage/brno12-cerit` | Brno | General login server |

All listed servers run Debian 12. Treat availability and access restrictions as
live operational data if a connection fails or current status matters.

Use placeholders only until the user supplies their username or chooses a
frontend:

```bash
klist -f
ssh USERNAME@FRONTEND
ssh USERNAME@FRONTEND klist -f
```

A normal MetaCentrum workflow uses a forwardable Kerberos ticket locally and GSSAPI credential delegation to a trusted frontend. A minimal host-specific SSH entry is:

```sshconfig
Host metacentrum
    HostName FRONTEND
    User USERNAME
    GSSAPIAuthentication yes
    GSSAPIDelegateCredentials yes
    GSSAPIKeyExchange yes
```

Keep delegation limited to trusted MetaCentrum hosts. Do not recommend a wildcard `Host *` delegation rule.

## Verification

Test Kerberos authentication without silently falling back to keys or passwords:

```bash
ssh -o BatchMode=yes \
  -o PreferredAuthentications=gssapi-with-mic \
  -o PubkeyAuthentication=no \
  -o PasswordAuthentication=no \
  USERNAME@FRONTEND hostname
```

Then confirm the remote session received a delegated ticket:

```bash
ssh USERNAME@FRONTEND klist -f
```

If this fails, inspect `klist -f`, system time, DNS, and the effective settings from `ssh -G metacentrum` — **use the configured `Host` alias, not the raw `FRONTEND` hostname.** SSH `Host` blocks match the literal argument given on the command line, not the resolved hostname, so `ssh -G FRONTEND` silently skips the alias's `GSSAPIAuthentication`/`GSSAPIDelegateCredentials`/`GSSAPIKeyExchange` settings and reports misleading (effectively default/disabled) values even though the alias itself is configured correctly. The same applies to the actual `ssh` invocation: always connect with `ssh metacentrum`, never `ssh FRONTEND` directly, or delegation silently won't happen and the frontend session will fall back to publickey/password auth with no ticket to use for `qsub`/`qstat`/etc.

A ticket that has simply expired is the most common cause of `debug1: Unspecified GSS failure. Minor code may provide more information` during `gssapi-with-mic`/`gssapi-keyex` in `ssh -v` output — check `klist -f` against the current `date` first before investigating anything else. `kinit -R` only works while the ticket is still valid. Once it has expired it fails with `Ticket expired while renewing credentials`, even if `klist` still shows a `renew until` date in the future (verified 2026-10-12: a ticket that expired on 9 Oct with `renew until` 15 Oct could not be renewed on 12 Oct). A full reacquisition is then required: password-based `kinit`, or the site's wallet/keytab-backed helper if one exists. A renewal daemon such as `krb5-auth-dialog` must therefore run before the ticket expires, for example when the laptop resumes from sleep.

## Credential handling

Never request, display, log, or embed a MetaCentrum password. Prefer the system Kerberos client, OS credential storage, and interactive prompts. Do not place credentials in scripts, environment files, command-line arguments, or skill resources. Destroy an unwanted local ticket with `kdestroy`; do not delete unrelated credential caches.

Official security guidance: <https://docs.metacentrum.cz/en/docs/access/security/kerberos>

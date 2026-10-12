# MetaCentrum Storage

Read this reference for persistent storage, home-directory locations,
frontend-storage relationships, and backup classes. For temporary job scratch,
also read [pbs-jobs.md](pbs-jobs.md).

## Persistent storage inventory

Storage server names describe their physical site and an instance number. A
storage server is a large disk array containing user data, including one of a
user's possible home directories. It is not necessarily an SSH login server.

| Server | Directory | Backup class | Notes |
| --- | --- | ---: | --- |
| `storage-brno2.metacentrum.cz` | `/storage/brno2/` | 2 | |
| `storage-brno11-elixir.metacentrum.cz` | `/storage/brno11-elixir/` | 2 | Dedicated to ELIXIR-CZ |
| `storage-brno12-cerit.metacentrum.cz` | `/storage/brno12-cerit/` | 2 | |
| `storage-plzen1.metacentrum.cz` | `/storage/plzen1/` | 2 | |
| `storage-plzen2-zcu.metacentrum.cz` | `/storage/plzen2-zcu/` | 3 | Dedicated to ZČU groups |
| `storage-plzen4-ntis.metacentrum.cz` | `/storage/plzen4-ntis/` | 3 | Dedicated to `iti`/`kky` groups |
| `storage-praha2-natur.metacentrum.cz` | `/storage/praha2-natur/` | 0 | |
| `storage-praha6-fzu.metacentrum.cz` | `/storage/praha6-fzu/` | 0 | |
| `storage-praha5-elixir.metacentrum.cz` | `/storage/praha5-elixir/` | 3 | |
| `storage-budejovice1.metacentrum.cz` | `/storage/budejovice1/` | 3 | |
| `storage-liberec3-tul.metacentrum.cz` | `/storage/liberec3-tul/` | 0 | |
| `storage-pruhonice1-ibot.metacentrum.cz` | `/storage/pruhonice1-ibot/` | 3 | |
| `storage-vestec1-elixir.metacentrum.cz` | `/storage/vestec1-elixir/` | 2 | Also provides `/storage/praha1/` |

The class numbers above are inventory values, not self-explanatory guarantees.
Do not infer backup frequency, retention, recoverability, or service level from
a number alone; verify the current definition before advising on data safety.
Treat server availability, access restrictions, and storage policy as live data.

## Frontends, storage, and home directories

A frontend is a login machine for file preparation, light pre- and
post-processing, and scheduler commands. Each frontend has exactly one native
storage: its `/home` points to that storage's `home` directory. For example, on
the Skirit frontend these paths refer to the same home directory:

```text
~
/storage/brno2/home/USERNAME
```

Several frontend hostnames can share the same native storage. For example,
Nympha and its associated login aliases can place a user's home at:

```text
/storage/plzen1/home/USERNAME
```

Users can have home directories on several loosely interconnected storages.
The home selected by `~` depends on the frontend, so do not assume that logging
into another frontend exposes the same path as `~`.

## Moving between storage homes

Mounted storage paths can be accessed from a frontend regardless of which
storage supplies that frontend's native home. A user normally does not need to
log in to a particular frontend just to reach another storage. Inspect the
current native home and change to a different mounted home explicitly:

```bash
pwd
printf 'home=%s\n' "$HOME"
cd "/storage/brno2/home/$USER"
pwd
```

For example, if Skirit is unavailable, a user can log in through Tarkil and
then change from Tarkil's native `/storage/praha1/home/USERNAME` to
`/storage/brno2/home/USERNAME`.

Before copying or computing against a path, verify that it exists and that the
user has access. Do not run substantial processing on a frontend; stage the
data for a PBS job and use explicitly requested scratch for temporary job I/O.

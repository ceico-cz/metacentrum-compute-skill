# Automating the MetaCentrum Kerberos Ticket

Read this reference when the user wants a ticket obtained automatically at
desktop login, kept renewed, or restored after sleep or reboot. It is optional;
[access.md](access.md) covers the basic `kinit` and SSH delegation workflow.

Change local system or desktop configuration only when the user has asked for
it. Show the planned commands and files first, and back up any file you replace.

## Client setup

Install a Kerberos client and, on Linux desktops, a ticket monitor:

```bash
sudo apt install krb5-user krb5-auth-dialog   # Debian/Ubuntu
```

Other platforms: `krb5-workstation` on Fedora/RHEL, `krb5` on Arch, and the
system Heimdal client on macOS.

MetaCentrum publishes its realm configuration on every frontend. Copy it and
keep the previous file:

```bash
scp USERNAME@FRONTEND:/etc/krb5.conf /tmp/krb5.conf.metacentrum
sudo cp -a /etc/krb5.conf "/etc/krb5.conf.before-metacentrum-$(date +%Y%m%d)"
sudo install -o root -g root -m 0644 /tmp/krb5.conf.metacentrum /etc/krb5.conf
```

If the machine already uses another realm in `/etc/krb5.conf`, merge the
`META` realm and its `domain_realm` entries instead of replacing the file, or
give the other realm its own `KRB5_CONFIG` and `KRB5CCNAME`.

## Forwardable, renewable ticket

```bash
kinit -f -r 7d USERNAME@META
klist -f
```

The KDC decides the actual lifetime. `-r 7d` requests a renewable ticket, so a
monitor can extend it without the password until the renewable limit. Renewal
only works while the ticket is still valid. Once it has expired, the user must
run `kinit` again (see the expiry note in [access.md](access.md)).

## Renewal monitor

`krb5-auth-dialog` runs in the desktop session, renews the ticket before it
expires and prompts for the password when it cannot. Make sure it starts at
login and check it with:

```bash
pgrep -af krb5-auth-dialog
```

A laptop that sleeps through the expiry time comes back with an expired ticket.
The monitor then asks for the password again, or a keyring-backed helper
(below) obtains a new ticket.

## Keyring-backed login helper (pattern)

To avoid typing the password at each login, a small user-level helper can
store the MetaCentrum password in the OS secret store and run `kinit` from it.
Write such a helper only on request, for the user's actual desktop:

| Platform | Secret store | Typical access |
| --- | --- | --- |
| KDE Plasma | KDE Wallet | D-Bus `org.kde.kwalletd6` / `kwalletd5` API |
| GNOME and others | Secret Service (GNOME Keyring, KeePassXC) | `secret-tool lookup service metacentrum user USERNAME` |
| macOS | Keychain | `security find-generic-password -s metacentrum -a USERNAME -w` |

The helper should:

1. Exit without changes if `klist -s` reports a valid ticket.
2. Read the password from the secret store into process memory only.
3. Pass it to `kinit -f -r 7d USERNAME@META` on standard input, never as an
   argument, environment variable or file.
4. Start the renewal monitor after a ticket exists.
5. Offer a `--store` mode that prompts for the password, validates it with
   `kinit` into a temporary cache (`KRB5CCNAME=$(mktemp)`), and only then
   writes it to the secret store. Use this after a MetaCentrum password change.

Start it at login with an autostart entry, for example
`~/.config/autostart/metacentrum-kinit.desktop` on freedesktop desktops or a
LaunchAgent on macOS. If the monitor package also ships a global autostart
entry, mask it with a user-level copy that contains `Hidden=true`. The helper
then gets the ticket before the monitor starts, and the user is not asked for
the password twice.

If the secret store does not unlock with the desktop login (PAM unlocking for
KDE Wallet or GNOME Keyring), the user sees an unlock prompt at login.

## SSH settings

Delegate credentials only to the chosen frontend (see [access.md](access.md)):

```sshconfig
Host ALIAS
    HostName FRONTEND
    User USERNAME
    GSSAPIAuthentication yes
    GSSAPIDelegateCredentials yes
    GSSAPIKeyExchange yes
```

`GSSAPIKeyExchange` requires an OpenSSH build with GSSAPI key-exchange support,
which Debian and Ubuntu ship. Other builds may not recognize the option.

## Troubleshooting

```bash
klist -f                                  # local ticket, flags, expiry
pgrep -af krb5-auth-dialog                # is the monitor running?
ssh -G ALIAS | grep -Ei '^(user|hostname|gssapi)'
ssh ALIAS klist -f                        # did the frontend get a ticket?
```

- No ticket after login: run the helper manually and read its error.
- `Password incorrect` from the helper: the stored password is outdated. Run
  the helper's `--store` mode.
- Destroy the local cache with `kdestroy`. The next helper run recreates it.

## Security considerations

- An unlocked wallet or keyring can be read by other processes running as the
  same desktop user. This is the normal trust model for keyring automation.
- Never put the password in a script, environment file, shell history,
  command-line argument or skill resource.
- A delegated ticket stays usable on the frontend until it expires, so keep
  `GSSAPIDelegateCredentials` limited to trusted MetaCentrum hosts.

Official guidance: <https://docs.metacentrum.cz/en/docs/access/security/kerberos>

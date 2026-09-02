# Temporary polkit access

Repository layout: `sbin/pkauth-grant-admin` is the Bash administrator command, `bin/pkauth-verify-token` is the POSIX verifier run by polkit, and `share/10-e3da-temp-admin.rules` is the polkit rule.

This repository grants a time-limited polkit `pkexec` authorization to a local account. It does not add the account to sudoers or change its Unix groups. Because the rule authorizes the generic `org.freedesktop.policykit.exec` action, a valid token is equivalent to temporary root access: it must be protected and audited accordingly.

## Revision policy

Keep development changes folded into one commit. Create a single Git commit only for a major revision after the code, documentation, deployment behavior, and tests have been checked. Decisions and structural changes must be synchronized across the scripts, their usage output, README examples, deployment notes, and operational guidance before committing. Amend that commit during the revision cycle instead of accumulating small or unverified commits.

## Design

The management script signs the expiration timestamp with an OpenSSH private key. The server stores the timestamp and SSH signature in `/var/run/pkauth` (normally a `/run` filesystem), which disappears at boot. The polkit rule invokes the fixed verifier at `/home/root/e3da/pkauth/bin/pkauth-verify-token` through `polkit.spawn()`. The verifier runs as the unprivileged `polkitd` account, reads the token and public signing key from `/home/root/e3da/pkauth/conf/pkauth.conf`, verifies the signature, and returns the timestamp. Expiry is checked on every request, so no cleanup service is required.

The timestamp is signed, not encrypted. Signing provides integrity and authenticity; it does not hide the expiry time. The token is published as one file with an atomic rename, preventing polkit from observing a half-written pair.

## Configuration

The grant script keeps the remote policy values fixed:

| Value | Setting |
| --- | --- |
| Token directory | `/var/run/pkauth` |
| Audit log | `/var/log/e3da/pkauth.log` |
| PKAUTH root | `/home/root/e3da/pkauth` |
| Default administrator SSH/signing-key directory | `$HOME/.ssh/keys` |
| Signing key name | `admin@e3da.local` |
| Manager GID | `7000` |
| Default SSH port | `10022` |

The verifier and configuration use the fixed `PKAUTH_ROOT` layout: `bin/pkauth-verify-token` and `conf/pkauth.conf`. This leaves room for additional executables or keys in the same shared directories without exposing each internal path as a separate setting.

The verifier uses built-in defaults and optionally sources `/home/root/e3da/pkauth/conf/pkauth.conf`. The trusted config may change the verifier’s `TOKEN_DIR`, `NAMESPACE`, `IDENTITY`, and `ADMIN_PUBLIC_KEY`; the verifier’s no-argument usage output shows both the built-in defaults and the values loaded from config. Keep `TOKEN_DIR`, `NAMESPACE`, and `IDENTITY` aligned with the grant script when changing them, because the grant script does not load this config. Without the file, the verifier fails closed because no signing key is trusted. The rule’s verifier path must match the installed verifier path.

Install these files on each target:

- `sbin/pkauth-grant-admin` as `/home/root/e3da/pkauth/sbin/pkauth-grant-admin`, root-owned and mode `0755`
- `share/10-e3da-temp-admin.rules` as `/etc/polkit-1/rules.d/10-e3da-temp-admin.rules`
- `bin/pkauth-verify-token` as `/home/root/e3da/pkauth/bin/pkauth-verify-token`, root-owned and mode `0755`
- `conf/pkauth.conf` as `/home/root/e3da/pkauth/conf/pkauth.conf`, mode `0644`, containing verifier overrides and `ADMIN_PUBLIC_KEY`; `/home/root` is a trusted NFS policy area

The executable and share files include `#author`/`#version` or syntax-equivalent metadata at the top. Version tags belong to individual files: retain the existing tag for every file that did not change, and update the date only in files that changed. Keep the `share/` folder and its installed counterparts consistent; a changed share-file tag means the corresponding file in `/etc` must be refreshed. Review all share-file tags together before deployment so no changed file is missed.
The rules directory must be readable by the polkit service. The token directory must be root-owned and mode `0755`; token files are mode `0644` so `polkitd` can read them. The log directory should be root-owned and mode `0755`, and the log file root-owned mode `0640`. The complete OpenSSH public key line is stored in the config as:

```text
ADMIN_PUBLIC_KEY="<key-type> AAAA..."
```

Some distributions run polkit with systemd `ProtectHome=yes`, which hides `/home` from the verifier even when `polkitd` can read it in a shell. When using the trusted `/home/root` config, add a read-only service override and restart polkit:

```ini
# /etc/systemd/system/polkit.service.d/10-pkauth-home.conf
[Service]
ProtectHome=read-only
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart polkit.service
```

### Deployment checklist

Generate the public-key config from the public half of the administrator key. The private key stays on the workstation; no separate signer file is needed:

```bash
sudo install -d -o yrpeng -g root -m 0755 /home/root/e3da/pkauth/{bin,conf}
printf 'ADMIN_PUBLIC_KEY="%s"\n' "$(ssh-keygen -y -f "/home/root/.ssh/keys/admin@e3da.local")" \
	| sudo tee /home/root/e3da/pkauth/conf/pkauth.conf >/dev/null
sudo chown yrpeng:root /home/root/e3da/pkauth/conf/pkauth.conf
sudo chmod 0644 /home/root/e3da/pkauth/conf/pkauth.conf
```

For a remote RHEL 9 deployment from the administrator workstation:

```bash
public_key=$(ssh-keygen -y -f "/home/root/.ssh/keys/admin@e3da.local")
encoded_key=$(printf '%s\n' "$public_key" | base64 -w0)
ssh -p 10022 -i "/home/root/.ssh/keys/admin@e3da.local" root@e3da-srv2.e3da.wx /bin/sh -s -- "$encoded_key" <<'REMOTE'
set -eu
encoded_key=$1
install -d -o yrpeng -g root -m 0755 /home/root/e3da/pkauth/bin /home/root/e3da/pkauth/conf
printf '%s' "$encoded_key" | base64 -d | awk '{printf "ADMIN_PUBLIC_KEY=\"%s\"\n", $0}' > /home/root/e3da/pkauth/conf/pkauth.conf
chown yrpeng:root /home/root/e3da/pkauth/conf/pkauth.conf
chmod 0644 /home/root/e3da/pkauth/conf/pkauth.conf
REMOTE
```

The explicit `/bin/sh -s` keeps deployment independent of the remote account's login shell, including `tcsh`.

The optional `/home/root/e3da/pkauth/conf/pkauth.conf` may override verifier defaults and must contain the public key. If `TOKEN_DIR`, `NAMESPACE`, or `IDENTITY` is changed, the same value must also be changed in the grant script so both sides use the same token and signature parameters:

```text
TOKEN_DIR=/var/run/pkauth
NAMESPACE=pkauth@e3da.local
IDENTITY=admin@e3da.local
ADMIN_PUBLIC_KEY="<key-type> AAAA..."
```

Install the verifier as `/home/root/e3da/pkauth/bin/pkauth-verify-token` with mode `0755`, and the rule as `/etc/polkit-1/rules.d/10-e3da-temp-admin.rules` with mode `0644`. The config file must be readable by `polkitd` and must not be group- or other-writable. Its NFS ownership and parent-directory trust are operational assumptions; the verifier does not check every parent directory. Ensure the rules directory is readable by the installed polkit service: Ubuntu/Mint commonly uses root-owned `0755`; RHEL may use `polkitd:root` mode `0700`. Verify the local package convention instead of changing it blindly.

Before the first grant, verify:

```bash
sudo stat -c '%U:%G %a %n' /home/root/e3da/pkauth /home/root/e3da/pkauth/bin /home/root/e3da/pkauth/conf /home/root/e3da/pkauth/conf/pkauth.conf \
	/home/root/e3da/pkauth/bin/pkauth-verify-token /var/run/pkauth
sudo -u polkitd /home/root/e3da/pkauth/bin/pkauth-verify-token cto
systemctl is-active polkit.service
```

To display verifier defaults and the values loaded from `/home/root/e3da/pkauth/conf/pkauth.conf`, run it without a username:

```bash
sudo -u polkitd /home/root/e3da/pkauth/bin/pkauth-verify-token
```

The verifier must return the token expiry timestamp. A missing or unreadable config/public key stops the grant before token publication.

The verifier also provides an administrative `--preflight` mode for the grant script. It receives the target username, manager GID, token directory, and log file as arguments; validates the account and manager membership, prepares remote paths, and checks polkit access diagnostics. Normal polkit calls use only the `<username>` interface.

The SSH key is always both the login and signing key. Only its public key belongs in the trusted config; the private key stays on the workstation. Anyone trusted to modify the NFS config can authorize temporary root access.

## Usage

```bash
sbin/pkauth-grant-admin cto
sbin/pkauth-grant-admin cto sudoer@e3da-srv1.e3da.wx
sbin/pkauth-grant-admin cto sudoer@e3da-srv1.e3da.wx 6

# Use a local operator key directory for testing
sbin/pkauth-grant-admin cto sudoer@e3da-srv1.e3da.wx 3 10022 admin@e3da.local /home/yarui/.ssh/keys

# Use another SSH port for a specific host
sbin/pkauth-grant-admin cto sudoer@e3da-srv2.e3da.wx 3 22
```

The first argument is the mandatory target account receiving temporary access. The second argument is an optional SSH destination in `[ssh_user@]host` form and defaults to `root@localhost`; it is the remote transport account, not necessarily the target account. The third argument is the grant duration, defaulting to 3 hours; the fourth is the SSH port, defaulting to `10022`; the fifth is the signing identity; and the sixth is the local SSH key directory, defaulting to `$HOME/.ssh/keys`. Running the command without arguments prints usage and all defaults. Token storage, logging, verifier location, manager GID, namespace, and signing-key relationship are fixed grant-script values and are not environment overrides.

When the SSH destination user is not `root`, the script first runs `sudo -n true` and uses `sudo -n` for all remote privileged operations. A password prompt, missing sudo permission, or non-root account without noninteractive sudo stops the grant before a token is published. The SSH destination user and target account may differ, for example `cto sudoer@e3da-srv1.e3da.wx`.

When no host or user is supplied, the command prints the usage and all defaults, including the assembled verifier and SSH-key paths.

The default grant is 3 hours and the maximum is 24 hours. The target account must exist and either have manager GID `7000` as its primary group or be listed in that group. A rejected account is recorded in `/var/log/e3da/pkauth.log`.

The command is shell-independent for recipients and remote SSH accounts. Users may run it from Bash, tcsh, or another shell, and remote commands are executed through `/bin/sh`:

```text
pkexec /usr/bin/id
pkexec /usr/bin/dnf update -y
```

The grant script itself requires Bash, OpenSSH `ssh-keygen`, GNU `date`, and `awk`. It checks that the selected local SSH key directory and identity file are readable before contacting the target, then prints the selected private-key path and derived public key for diagnostics without printing private-key contents. With `sudo`, two arguments are sufficient when the default `$HOME/.ssh/keys/admin@e3da.local` resolves to the signing key. The signed expiration is a UTC-independent Unix epoch in milliseconds, and the verifier compares it with `Date.now()`. Grant output and audit log timestamps use the local timezone of the workstation or target host, respectively. Keep clocks synchronized with NTP. SSH `from=` restrictions remain enforced by OpenSSH and do not affect signature verification; the signing public key is trusted separately. Grant, rejection, and publication errors are logged to `/var/log/e3da/pkauth.log` on the target.

## Testing and operations

Validate locally with `bash -n sbin/pkauth-grant-admin` and `sh -n bin/pkauth-verify-token`. On a target, check `systemctl status polkit.service`, `pkaction`, and `tail -f /var/log/e3da/pkauth.log`. Expired tokens are intentionally retained as small volatile records; expiry is enforced during every polkit check and all tokens disappear when `/var/run` is cleared at boot. Test both a manager-group account and an account outside the group, a modified timestamp, an invalid signature, expiry, and reboot cleanup. Do not test by granting generic pkexec access on production first.

### Deployment lessons

- If the token verifies manually as `polkitd` but `pkexec` says `Not authorized`, inspect the polkit service sandbox. `ProtectHome=yes` hides `/home/root` from `polkit.spawn()`. With the trusted NFS policy area, install `10-pkauth-home.conf` with `ProtectHome=read-only`, run `systemctl daemon-reload`, and restart polkit.
- The grant preflight checks the local SSH key directory and invokes the installed verifier’s `--preflight` mode on the target. It validates the target account and manager group, prepares the token and log paths, and warns when `polkit.service` reports `ProtectHome=yes` or `tmpfs`, or when `polkitd` cannot execute the verifier. A warning is diagnostic; apply the `ProtectHome=read-only` override and restart polkit before testing authorization.
- Polkit rules must be readable by the polkit service. Ubuntu/Mint commonly uses a root-owned `0755` rules directory; RHEL commonly uses `polkitd:root` mode `0700`. Preserve the package-specific directory mode and verify it after deployment.
- Deploy the verifier and rule together. The rule contains the fixed verifier path `/home/root/e3da/pkauth/bin/pkauth-verify-token`; a missing, non-executable, or differently installed verifier causes authorization to fail closed.
- Verify the complete path as `polkitd`, not only the token file: `sudo -u polkitd /home/root/e3da/pkauth/bin/pkauth-verify-token cto`. Then test `pkexec /usr/bin/id` as the target account before using a sensitive command.

Ubuntu 24.04 and RHEL 9 generally provide the required polkit and OpenSSH components, but package paths and polkit service packaging differ. Ubuntu 26.04 and RHEL 10+ may change `pkexec` defaults, the JavaScript engine, or package layout. Re-test `polkit.spawn`, the rule with the current ECMA-262 Edition 5 syntax, `ssh-keygen -Y verify`, and the `pkexec` action on each release. A dedicated root D-Bus service with a narrowly scoped polkit action is the better long-term design than authorizing generic `pkexec`.

# Ubuntu Secret Service setup

This setup is performed by the human operator as the same unprivileged Linux user that runs Hermes. Hermes must not enroll or receive the database password.

## Packages

Install the command-line clients and a Secret Service provider:

```bash
sudo apt update
sudo apt install keepassxc libsecret-tools gnome-keyring dbus-user-session
```

`secret-tool` is a client. A D-Bus user session, a running Secret Service provider, and an unlocked collection must also exist in the Hermes process environment. On a headless server, do not assume a desktop login has created or unlocked them.

## Files

Place the encrypted database and local key file at these defaults, or set `KPXC_DB` and `KPXC_KEY` to different paths in the Hermes runtime configuration:

```text
~/.local/share/keepassxc/vault.kdbx
~/.keys/vault.keyx
```

Keep the key file local to the server and distribute it separately from the database. Restrict both paths to the Hermes user:

```bash
chmod 700 "$HOME/.local/share/keepassxc" "$HOME/.keys"
chmod 600 "$HOME/.local/share/keepassxc/vault.kdbx" "$HOME/.keys/vault.keyx"
```

## Enroll the database password

Run this interactively as the Hermes user:

```bash
secret-tool store --label='KeePassXC vault' \
  service keepassxc-vault \
  account "$(id -un)"
```

Enter the database password only at the hidden terminal prompt. Do not pipe it from shell history, paste it into chat, place it in an environment variable, or ask Hermes to store it.

Check lookup without displaying the password:

```bash
secret-tool lookup service keepassxc-vault account "$(id -un)" >/dev/null
```

A zero exit status confirms lookup. It does not confirm that KeePassXC can open the database with both factors.

## Headless service requirement

Linux Secret Service is session-scoped. Hermes must run as the same user and inherit that user's D-Bus session address. The selected collection must already be unlocked when Hermes starts.

If `secret-tool` reports that no Secret Service is available, cannot connect to D-Bus, or cannot unlock the collection, fix the user session and keyring lifecycle before enabling vault access. Do not fall back to `.env`, process arguments, environment variables, or plaintext files. For unattended startup after reboot, choose and provision a deliberate service-level secret design before relying on this skill.

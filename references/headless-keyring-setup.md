# Headless keyring setup (no desktop login)

Linux Secret Service is session-scoped. On a server where no desktop login has
ever run, there is no persistent collection, so the database password has nowhere
to live until one is created. Creating and unlocking it is the operator's job,
not the agent's.

## Symptom

```text
$ secret-tool store --label='KeePassXC vault' service keepassxc-vault account "$(id -un)"
secret-tool: *** does not exist at path "/org/freedesktop/secrets/collection/login"
```

The service name being *activatable* is not enough. Activating it starts
`gnome-keyring-daemon --start --foreground --components=secrets`, and the only
collection that daemon exposes is `session`: in memory, discarded when the daemon
exits. Nothing is persisted, so a lookup after a restart finds nothing.

## Create and unlock the login collection

Run once, as the same unprivileged user that runs the agent, with no other
keyring daemon already running:

```bash
printf '%s\n' "$KEYRING_PW" | gnome-keyring-daemon --unlock --components=secrets
```

- The password must be newline-terminated on stdin. Without the trailing newline
  the request does not take effect.
- The first run creates `~/.local/share/keyrings/login.keyring` (0600).
- Later runs unlock that existing collection; they do not recreate it.

## Verify functionally, never by exit status

`--unlock` exits 0 even when nothing was unlocked. Check the collection itself:

```bash
busctl --user call org.freedesktop.secrets /org/freedesktop/secrets \
  org.freedesktop.DBus.Properties Get ss org.freedesktop.Secret.Service Collections
# expect /org/freedesktop/secrets/collection/login among the results

busctl --user get-property org.freedesktop.secrets \
  /org/freedesktop/secrets/collection/login \
  org.freedesktop.Secret.Collection Locked
# v b false = unlocked
```

Or round-trip a throwaway item and require non-empty output back.

## Re-unlocking after a reboot is timing-flaky

The same command, same password, same host has been observed both to unlock on
the first attempt and to leave the collection locked on the first attempt and
succeed on a later one. Never assume a single invocation worked: retry in a loop
(3-4 attempts, ~2s apart) and stop on a successful functional check.

## Never pre-start the daemon before unlocking

```bash
gnome-keyring-daemon --start --daemonize --components=secrets   # do NOT do this first
printf '%s\n' "$PW" | gnome-keyring-daemon --unlock --components=secrets
```

That leaves two daemons competing for `org.freedesktop.secrets`. Every store and
lookup then fails with:

```text
secret-tool: *** Object does not exist at path "/org/freedesktop/secrets/collection/login"
```

while the Collections property still advertises the login collection - a
confusing failure that looks like a configuration problem rather than a
double-daemon problem. One single `--unlock` invocation is what creates and
starts what it needs.

## Clearing stray daemons safely

If a competing daemon is suspected, kill only daemons that belong to your own
HOME. Identify them by `/proc/<pid>/exe`, and read HOME from `/proc/<pid>/environ`:

```bash
for p in /proc/[0-9]*; do
  exe=$(readlink "$p/exe" 2>/dev/null) || continue
  case "$exe" in */gnome-keyring-daemon)
    h=$(tr '\0' '\n' < "$p/environ" 2>/dev/null | sed -n 's/^HOME=//p')
    case "$h" in ""|"$HOME") kill "${p#/proc/}" 2>/dev/null;; esac;;
  esac
done
```

Killing unscoped is destructive whenever another context is running: it takes
down that session's daemon and re-locks its keyring, silently breaking its vault
access until the operator unlocks again. Two related traps: `pkill -x
gnome-keyring-daemon` matches nothing (process names are truncated to 15
characters, so the pattern is longer than the name it is compared against), and
`pkill -f gnome-keyring-daemon` can match the calling shell's own command line
and kill the caller.

## Empty-password keyring: unattended, but unprotected at rest

```bash
printf '\n' | gnome-keyring-daemon --unlock --components=secrets
```

This creates the login collection with an empty password, so it auto-unlocks on
every daemon start and stored items survive restarts with no operator action.
The trade-off is that the keyring file is then effectively unprotected: the
database password inside it is only as safe as the 0600 mode on the file, and a
copy of the disk yields the password without the key file, defeating the point of
keeping the two factors apart. Choose it only when unattended operation matters
more than at-rest protection, and record that choice deliberately.

## The agent process needs the session bus environment

Secret Service access is per-session, so the agent's process must carry both:

```text
XDG_RUNTIME_DIR=/run/user/<uid>
DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus
```

Check with `systemctl --user show-environment`. A systemd `--user` service
inherits both; a shell or process started without a session bus does not, and
secret-tool then fails with:

```text
secret-tool: *** autolaunch D-Bus without X11 $DISPLAY
```

Fix the process environment rather than defaulting these variables inside a
wrapper script, so the failure stays visible instead of being papered over.

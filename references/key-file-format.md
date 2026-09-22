# Key file format

KeePassXC decides a key file's format from its content, not its extension:

- **XML key file v2** - recommended, and what a `.keyx` extension implies.
  Self-checking.
- **32 raw bytes** - used directly as a 256-bit key.
- **64 hex characters** - decoded to a 256-bit key.
- **anything else** - hashed into a key.

A `.keyx` holding 32 raw bytes works, but KeePassXC reports on every unlock:

```text
WARNING: You are using an old key file format which KeePassXC may
stop supporting in the future.
Please consider generating a new key file.
```

Prefer XML v2:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<KeyFile>
    <Meta>
        <Version>2.0</Version>
    </Meta>
    <Key>
        <Data Hash="0123ABCD">
            64 uppercase hex characters
        </Data>
    </Key>
</KeyFile>
```

`Hash` is the first 4 bytes of the SHA-256 of the key, in uppercase hex.
KeePassXC enforces it:

```text
Failed to load key file <path>: Checksum mismatch! Key file may be corrupt.
```

That check is the reason to prefer XML v2: it is the only format that detects a
damaged, truncated or partially-synced key file, which is exactly the failure you
cannot afford on the second factor. Generate it in place, never printing key
material to a terminal, chat, or log:

```bash
python3 - <<'PY'
import hashlib, pathlib, secrets
key = secrets.token_bytes(32)
xml = ('<?xml version="1.0" encoding="UTF-8"?>\n<KeyFile>\n    <Meta>\n'
       '        <Version>2.0</Version>\n    </Meta>\n    <Key>\n'
       f'        <Data Hash="{hashlib.sha256(key).digest()[:4].hex().upper()}">\n'
       f'            {key.hex().upper()}\n        </Data>\n    </Key>\n</KeyFile>\n')
p = pathlib.Path.home()/".keys"/"vault.keyx"
p.parent.mkdir(mode=0o700, parents=True, exist_ok=True)
p.write_text(xml)
p.chmod(0o600)
PY
```

Verify before trusting it, using a throwaway database:

1. `keepassxc-cli db-create -p --set-key-file <keyfile> /tmp/t.kdbx`
   (interactive password prompt; `-k` is deprecated in favour of
   `--set-key-file`)
2. reopen it with the same password plus the key file and require exit 0
3. tamper with the `Hash` attribute and confirm KeePassXC refuses with
   `Checksum mismatch!`

## db-create requires an explicit key

`keepassxc-cli db-create` aborts with `No key is set.` unless `--set-key-file`
or `-p` is given, and it does not treat piped stdin as a password. The database
password prompt comes from `-p`.

# `bw-env`

Generic wrapper that pulls the **notes** field of a Bitwarden item, parses it
as a `.env`-style file, exports each key into the environment, and `exec`s a
command. Other scripts in this repo (`aws`, `start`) are thin layers on top.

## Usage

```sh
bw-env <secret-name> <command> [args...]
```

## Vault layout

The Bitwarden item named `<secret-name>` must have its env vars in the
**notes** field, one per line, in `KEY=value` form:

```
DATABASE_URL=postgres://app:hunter2@db.local/app
STRIPE_SECRET=sk_live_...
FEATURE_X=on
```

The username/password/custom-fields slots are ignored.

## Parsing rules

Each line is processed independently:

- Blank lines and lines whose first non-whitespace char is `#` are skipped.
- Optional leading `export ` is stripped (so `export FOO=bar` works too).
- The line must match `KEY=VALUE` where `KEY` is `[A-Za-z_][A-Za-z0-9_]*`. Lines that don't match are silently skipped.
- If `VALUE` is wrapped in matching `"..."` or `'...'`, the quotes are stripped. **No** backslash-escape or `$VAR` interpolation is performed inside quotes — the value is taken literally.

## Examples

```sh
bw-env stripe-prod -- some-script               # script sees STRIPE_SECRET etc.
bw-env github-bot gh api repos/foo/bar          # gh sees GITHUB_TOKEN etc.
bw-env myproj-env-prod env | grep DATABASE_URL  # quick verify
```

## Failure modes

- `rbw not installed` — install rbw and `rbw login` / `rbw unlock` first.
- `could not read notes for vault item '<name>'` — item missing, vault locked, or wrong name. Try `rbw get --field notes <name>` directly to diagnose.
- `vault item '<name>' has empty notes` — item exists but its notes blob is empty. Add at least one `KEY=value` line.

## Notes

- `bw-env` only reads from `--field notes`. If you have legacy items that store creds in username/password, migrate them first (or use a different wrapper).
- The exec model means the spawned command **replaces** the bw-env process — exit codes and signals propagate naturally.

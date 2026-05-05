# `upload-env`

Move local `.env*` files into Bitwarden so they're not sitting on disk. Run
from inside a project directory.

## Usage

```sh
upload-env
```

No args. Operates on the current directory only.

## What it picks up

| Filename            | Vault item created             | Notes                                    |
| ------------------- | ------------------------------ | ---------------------------------------- |
| `.env`              | `<project>-env-default`        |                                          |
| `.env.<name>`       | `<project>-env-<name>`         | e.g. `.env.prod` -> `myproj-env-prod`    |
| `.env-<name>`       | `<project>-env-<name>`         | e.g. `.env-staging` -> `myproj-env-staging` |
| `.env.example`      | (skipped)                      | always ignored                           |
| `.env-example`      | (skipped)                      | always ignored                           |

`<project>` is the basename of the current directory.

## Placeholder filter

Files containing any of the following are **skipped** (not uploaded, not
deleted) — the heuristic is intentionally a bit aggressive to avoid uploading
template values:

- `<...>` — angle-bracket placeholders (`<your-key>`, `<API_KEY>`)
- `${VAR}` — unresolved shell-style references
- The tokens `CHANGEME`, `PLACEHOLDER`, `TODO`, `XXX`, `FIXME`, or `your_*_here` (case-insensitive, when surrounded by non-word chars)

Resolve the placeholders (or remove the offending lines) and re-run.

## Conflict prompt

If a vault item already exists at the target name, you'll be prompted per
file:

```
myproj-env-prod already exists. [s]kip / [o]verwrite / [c]ancel?
```

- **skip** — leave the existing vault item and the local file alone, move on to the next file.
- **overwrite** — replace the vault item's notes with the local file's contents (preserving whatever's in the password slot), then delete the local file.
- **cancel** — abort the entire run; nothing further is uploaded or deleted.

## Local file deletion

The local file is `rm`'d **only after** a successful `rbw add` / `rbw edit`.
If `rbw` fails (e.g. vault locked mid-run), the file stays on disk.

## Bitwarden item shape

Created items have:

- **Name**: `<project>-env-<env-name>`
- **Password**: empty placeholder (rbw requires a first line; `bw-env` reads only `--field notes`)
- **Notes**: the literal contents of the `.env` file

This matches what [`bw-env`](bw-env.md), [`start`](start.md), and
[`edit-env`](edit-env.md) expect.

## How rbw is driven

`rbw add` and `rbw edit` only support entry through `$EDITOR`, so this script
sets `$EDITOR` to a tiny shim that overwrites the temp file rbw provides with
the prepared content. The shim is created with `mktemp` and removed on exit.

## Failure modes

- `rbw not installed` — install rbw and `rbw login` / `rbw unlock`.
- Hangs at the conflict prompt with no output — the prompt reads from `/dev/tty`. Run interactively, not under CI or pipes.
- `rbw edit` opens your real editor instead of the shim — the shim wasn't honored. Check that `$EDITOR` and `$VISUAL` aren't set readonly somewhere in your shell config.

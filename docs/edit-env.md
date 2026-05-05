# `edit-env`

Open a project env's notes blob in your `$EDITOR`, save, and persist back to
Bitwarden — without seeing rbw's first-line-is-password convention.

## Usage

```sh
edit-env [env-name]
```

`env-name` defaults to `default`. The vault item read/written is
`<project-name>-env-<env-name>`, where `<project-name>` is the basename of
the current directory. The item must already exist; `edit-env` errors if it
doesn't (use [`upload-env`](upload-env.md) or `rbw add` to create it).

## Editor experience

You see only the env vars — no leading password line, no rbw chrome:

```
DATABASE_URL=postgres://app:hunter2@db.local/app
STRIPE_SECRET=sk_live_...
FEATURE_X=on
```

Save and quit. If the contents are unchanged, `edit-env` exits early with
`no changes` and doesn't touch the vault.

`$VISUAL` wins over `$EDITOR`; falls back to `vi` if neither is set.

## What's preserved

- The password slot is preserved verbatim (so anything you set there manually survives).
- The notes blob is replaced wholesale with what you saved.

## How it works

1. `rbw get --field notes <item>` -> temp file.
2. Open temp file in your `$EDITOR`.
3. If unchanged, exit.
4. Otherwise, prepare a payload `<existing-password>\n<new-notes>`, set `$EDITOR` to a shim that overwrites rbw's temp file with that payload, and run `rbw edit <item>`.

## Failure modes

- `rbw not installed` — install and unlock rbw.
- `vault item '<name>' not found` — create it first via `upload-env` or `rbw add`.
- Editor opens with rbw's password+notes layout instead of just notes — the shim's `$EDITOR` override didn't take. Check your shell isn't forcing `$EDITOR` after the script sets it.

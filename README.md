# workstation-utils

Personal collection of small wrapper scripts that make day-to-day project work
faster on this workstation. The common thread is **secrets via Bitwarden**: env
vars, AWS creds, and project run-config all flow through `rbw` so nothing
sensitive sits on disk.

## Scripts

All live in [`bin/`](bin/). Each has its own page in [`docs/`](docs/).

| Script                              | What it does                                                                                       |
| ----------------------------------- | -------------------------------------------------------------------------------------------------- |
| [`bw-env`](docs/bw-env.md)          | Generic shim: load env vars from a Bitwarden item's notes blob, then `exec` a command.             |
| [`aws`](docs/aws.md)                | Wrapper around `aws` CLI; injects credentials from BW item `aws-<profile>`.                        |
| [`start`](docs/start.md)            | Detect project type (Go / Python / Node) in the current dir and run it, with env loaded from BW.   |
| [`upload-env`](docs/upload-env.md)  | Upload `.env` / `.env.<name>` / `.env-<name>` files to BW, then delete the local copies.            |
| [`edit-env`](docs/edit-env.md)      | Open a project env's notes blob in `$EDITOR`, hiding rbw's first-line-is-password quirk.            |

## Requirements

- [`rbw`](https://github.com/doy/rbw) (unofficial Bitwarden CLI). Must be configured (`rbw register`, `rbw login`) and unlocked (`rbw unlock`) before any script that reads from the vault is invoked.
- Bash 4+ (the scripts use `${var,,}`, namerefs, etc.).
- Tools relevant to whatever you're running: `go`, `poetry`, `uv`, `yarn`, `node`, etc.

## Install

The repo ships with an `install` script that creates symlinks from a directory
on your `$PATH` back to the scripts in this repo. It is safe to re-run any
time — for example after `git pull` adds a new utility.

```sh
./install
```

### What it does

1. Walks `$PATH` looking for `~/bin` or `~/.local/bin`.
2. Picks whichever it finds first; if neither is on `$PATH`, defaults to `~/.local/bin` and prints a one-liner you can paste into your shell rc to add it.
3. `mkdir -p` the chosen directory.
4. For each executable in `bin/`, creates a symlink at `<target>/<name>` pointing at the absolute path inside this repo.

### Idempotency rules

- If a symlink already exists and points at the same file (compared by inode), it is left untouched and reported as `already up to date`.
- If a symlink exists but points somewhere else, it is **left alone** and reported as `skipped`. The script never overwrites a non-matching link — resolve manually.
- If a regular file exists with the same name, it is **left alone** and reported as `skipped`. Same reason.
- New executables in `bin/` are linked on subsequent runs; you don't need to re-bootstrap.

### After install

If you saw the "not in PATH" warning, add the suggested line to `~/.zshrc` (or
`~/.bashrc`) and start a new shell. Then verify any one of the scripts is on
your path, e.g.:

```sh
which start
start --help    # (no --help flag yet; just confirms PATH resolution)
```

### Uninstall

```sh
for f in <repo>/bin/*; do rm -f "$HOME/.local/bin/$(basename "$f")"; done
```

(Or `~/bin/` if that's where install picked.) The `install` script has no
dedicated uninstall mode on purpose — there are few enough symlinks that a
one-liner is clearer than a flag.

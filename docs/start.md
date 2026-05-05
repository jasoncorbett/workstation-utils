# `start`

Run from any project directory; figures out the project type, builds if
necessary, and runs it — all with environment variables loaded from a
Bitwarden item via [`bw-env`](bw-env.md).

## Usage

```sh
start [env-name]
```

`env-name` defaults to `default`. The vault item read is
`<project-name>-env-<env-name>`, where `<project-name>` is the basename of the
current directory.

So in `~/code/checkout`:

| Command         | Vault item read           |
| --------------- | ------------------------- |
| `start`         | `checkout-env-default`    |
| `start prod`    | `checkout-env-prod`       |
| `start staging` | `checkout-env-staging`    |

The vault item must exist (even for `default`) — the env-vars notes blob can
be a single placeholder line if the project doesn't actually need any.

## Project-type detection

First marker found wins:

| Order | Type    | Marker                          | What runs                                  |
| ----- | ------- | ------------------------------- | ------------------------------------------ |
| 1     | Go      | `go.mod`                        | `go build -o <name>` then `./<name>`       |
| 2     | Python  | `poetry.lock`                   | `poetry run python main.py`                |
| 3     | Python  | `uv.lock` (if no `poetry.lock`) | `uv run python main.py`                    |
| 4     | Node    | `package.json` + `yarn.lock`    | see [Node](#node-specifics) below          |

If none match: error. If `package.json` is present without `yarn.lock`: error
(npm and pnpm are intentionally out of scope right now).

The Go binary is forced to `<project-name>` via `-o` so it always matches the
directory name regardless of what the `go.mod` module is called.

## `.start-commands` (per-project hook)

If `./.start-commands` exists in the project dir, it runs **first**, before
the type-specific build/run. It must be executable (otherwise `start` errors
out — it won't silently skip an unreadable hook).

Use it for project-specific bring-up that doesn't fit the generic detector:
seeding a database, starting docker compose, generating code, etc. Because
the whole sequence runs under a single `bw-env` invocation, your hook sees
the same env vars as the main process.

## Node specifics

After verifying `yarn.lock` is present:

1. If `yarn --version`'s major number is `> 1`, prepend `corepack enable` (safe to repeat).
2. Inspect `package.json` `.scripts` and pick:
   - has both `build` and `dev` -> `yarn build && yarn dev`
   - has `dev` only -> `yarn dev`
   - has `start` only -> `yarn start`
   - none of the above -> error

Script enumeration happens via `node -e ...` against `./package.json`, so
`node` must be on `$PATH`.

## Composition

The full command chain is composed and handed to a single
`bw-env <project>-env-<env-name> bash -c "<chain>"`:

```
[./.start-commands && ] <type-specific commands chained with &&>
```

Any non-zero exit aborts the rest. The `bw-env` lookup happens once per
`start` invocation regardless of how many sub-commands the chain has.

## Failure modes

- `no supported project type detected in <dir>` — none of the markers matched.
- `Node project without yarn.lock is not supported (yarn only)` — yarn-only by design.
- `package.json has no build/dev/start script` — add one.
- `.start-commands exists but is not executable` — `chmod +x .start-commands`.
- `bw-env: could not read notes for vault item '<name>'` — make sure the vault item exists. For new projects, create it at minimum with one placeholder env var.

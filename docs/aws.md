# `aws`

Drop-in wrapper for the AWS CLI that pulls credentials from Bitwarden on
demand instead of leaving them in `~/.aws/credentials`. Built on top of
[`bw-env`](bw-env.md).

## Usage

Identical to the real `aws` CLI. Profiles are still selected with `--profile`
or `$AWS_PROFILE`:

```sh
aws s3 ls
aws --profile prod ec2 describe-instances
AWS_PROFILE=staging aws sts get-caller-identity
```

## Vault layout

For each AWS profile, create a Bitwarden item named `aws-<profile>` whose
**notes** field contains the credentials in `.env` form:

```
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
```

Optionally include `AWS_DEFAULT_REGION=us-east-1` and any other `AWS_*` vars
you want injected per profile. Anything `bw-env` can parse is fair game.

## Profile resolution

In order of precedence:

1. `--profile X` or `--profile=X` on the command line (stripped before forwarding).
2. `$AWS_PROFILE` environment variable.
3. The literal `default`.

The resolved profile name is interpolated into the vault lookup as
`aws-<profile>`.

## Subcommands that bypass the vault

These pass straight through to the real `aws` without touching Bitwarden — no
unlock needed:

- `aws` (no args)
- `aws help` / `aws --help`
- `aws --version`
- `aws configure ...`

## Locating the real `aws`

The wrapper finds the real `aws` by walking `$PATH` and skipping any entry
whose inode matches its own. So you can install it as `~/.local/bin/aws` (in
front of `/opt/homebrew/bin/aws`), or symlink it elsewhere — both work.

## How env-var creds interact with profile config

When the wrapper hands off, it:

- Strips `--profile <x>` from the args, and
- `unset`s `AWS_PROFILE`,

so the injected `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` env vars are the
sole credential source the real `aws` sees. Anything in `~/.aws/credentials`
under `[<profile>]` is ignored. Profile-specific *config* (region, output
format) in `~/.aws/config` under `[profile <name>]` is also ignored unless
those values are also placed in the vault item.

## Migrating from a `~/.aws/credentials` setup

For each `[<profile>]` section:

1. `rbw add aws-<profile>` (the editor will open — see [`upload-env`](upload-env.md) for a friendlier path if you have a lot of profiles).
2. Put the keys in the notes field as `KEY=value` lines.
3. Delete the section from `~/.aws/credentials` (or leave it — it'll be ignored).

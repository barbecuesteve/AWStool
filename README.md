# AWStool

CLI for standing up a new personal project from scratch: GitHub repo, AWS
accounts, CDK app, and CI, in one pass. Built for a single-owner AWS
Organization managed from the `default` profile, with repos under
[barbecuesteve](https://github.com/barbecuesteve).

```
cd ~/Code/my-new-project
awstool onboard
```

Run with no flags to be prompted for each step:

```
Create GitHub repo? [y/N]
Create AWS OU + prod account? [y/N]
Also create a staging account? [y/N]
Initialize a CDK app (TypeScript)? [y/N]
Set up GitHub Actions deploys (OIDC roles + workflow)? [y/N]
```

Or select steps with flags: `--with-repo`, `--with-account`,
`--with-staging-account`, `--with-cdk` (plus `--cdk-language ts|py`),
`--with-ci`.

Always available: `--dry-run` prints every command and file it would
write, after running the read-only preflight checks, without changing
anything. `--name` overrides the project name (default: folder name);
`--profile` overrides the management-account profile; `--yes` skips the
account-creation confirmation.

## What each step does

**repo** - `git init` if needed, initial commit if none, create the
GitHub repo, add `origin`, push.

**account / staging** - create an Organizations OU named after the
project, create `<name>-prod` (and `<name>-staging`) accounts inside it,
and append matching profiles to `~/.aws/config` that assume
`OrganizationAccountAccessRole` from the management profile. Account
emails default to gmail plus-addressing and are prompted. Account
creation is the one hard-to-undo step, so it gets its own confirmation:
AWS accounts can only be closed, not deleted, and closed accounts count
against org quotas for 90 days.

**cdk** - `cdk init app` in TypeScript or Python (skipped if `cdk.json`
exists), pin each environment's account/region in a generated env module
(`lib/env.ts` or `<package>/deploy_env.py`), rewrite the app entrypoint
to instantiate one env-pinned stack per environment, and `cdk bootstrap`
each new account. Python apps additionally get a `.venv` with
dependencies installed and `cdk.json` pointed at `.venv/bin/python`, so
nothing requires venv activation.

**ci** - generate a `GithubOidcStack` (GitHub OIDC provider plus a
`GitHubDeployRole` that trusts only this repo and can only assume the
`cdk-*` bootstrap roles), deploy it to each account, and write
`.github/workflows/deploy.yml`: push to main deploys staging, prod
deploys on manual dispatch. No stored AWS keys anywhere.

Steps run in that order so the generated files, including a
quick-reference section appended to the project README, land in the
initial commit.

## Preflight

Before doing anything (including in dry-run) the tool verifies: required
CLIs on PATH, `gh` authenticated and the repo name available, the AWS
profile belongs to the org management account, no OU or `~/.aws/config`
profile name collisions, and (for cdk) an empty directory or an existing
`cdk.json`.

## Prerequisites

- `gh` authenticated (`gh auth login`)
- `aws` CLI with a management-account profile (default: `default`)
- `node`/`npx` for CDK steps (`cdk` used if installed, else `npx aws-cdk`)

## Install

```
ln -s ~/Code/AWStool/awstool ~/bin/awstool
```

## Companion

`~/bin/awsconsole <account>` opens the AWS web console for any org
member account by assuming `OrganizationAccountAccessRole` - it looks up
account names live from Organizations, so accounts created by awstool
show up automatically.

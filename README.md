# AWStool

Two standalone scripts for running personal projects in their own AWS
accounts:

- **`awstool`** stands up a new project from scratch in one pass: GitHub
  repo, AWS accounts, CDK app, and CI.
- **`awsconsole`** opens the AWS web console for any account you can
  reach, without stored passwords or switching accounts in the UI.

Both are single-file Python 3 with no third-party packages, so there is
nothing to install beyond the CLIs they drive.

```
cd ~/Code/my-new-project
awstool onboard
```

Run with no flags to be prompted for each step:

```
Create GitHub repo? [y/N]
Create AWS OU + prod account? [y/N]
Also create a staging account? [y/N]
Initialize a CDK app? [y/N]
Set up GitHub Actions deploys (OIDC roles + workflow)? [y/N]
```

Once the steps are chosen and preflight has passed, it prompts for the
details each one needs: repo visibility and description, CDK language
(typescript or python, defaulting to typescript), and an email address
per AWS account.

Or select steps with flags: `--with-repo`, `--with-account`,
`--with-staging-account`, `--with-cdk` (plus `--cdk-language ts|py`),
`--with-ci`.

Always available: `--dry-run` prints every command and file it would
write, after running the read-only preflight checks, without changing
anything. `--name` overrides the project name (default: folder name);
`--owner` sets the GitHub user or organization to create the repo under
(default: the authenticated `gh` user); `--profile` overrides the
management-account profile; `--region` overrides the deploy region
(default: the management profile's configured region, else `us-west-2`);
`--yes` skips the account-creation confirmation.

## Before you start

`awstool`'s account steps assume **an existing AWS Organization, with the
profile you pass owning the management account**. It creates an OU and
member accounts inside that org, so a single standalone AWS account will
not work for `--with-account`; preflight fails early and clearly if the
profile is not the management account. The `--with-cdk` and `--with-repo`
steps have no such requirement and work anywhere.

If you have no organization yet, create one from the AWS console
(Organizations, "Create an organization") in the account you want as the
management account. That account should hold no workloads of its own.

`awsconsole` has no organization requirement at all.

## What each step does

**repo** - `git init` if needed, initial commit if none, create the
GitHub repo, add `origin`, push.

**account / staging** - create an Organizations OU named after the
project, create `<name>-prod` (and `<name>-staging`) accounts inside it,
and append matching profiles to `~/.aws/config` that assume
`OrganizationAccountAccessRole` from the management profile.
`~/.aws/config` is copied to `~/.aws/config.bak` before anything is
appended. Every AWS account needs a globally unique email address; the
default offered is plus-addressed off the management account's email
(`you@example.com` becomes `you+myproject-prod@example.com`), which works
on Gmail and most major providers but not all, and each address is
prompted so it can be overridden. Account creation is the one
hard-to-undo step, so it gets its own confirmation: AWS accounts can only
be closed, not deleted, and closed accounts count against org quotas for
90 days.

**cdk** - `cdk init app` in TypeScript or Python (skipped if `cdk.json`
exists), pin each environment's account/region in a generated env module
(`lib/env.ts` or `<package>/deploy_env.py`), rewrite the app entrypoint
to instantiate one env-pinned stack per environment, and `cdk bootstrap`
each new account. Python apps additionally get a `.venv` with
dependencies installed and `cdk.json` pointed at `.venv/bin/python`, so
nothing requires venv activation.

TypeScript and Python are the only supported languages. `cdk init app`
itself also offers javascript, go, java, csharp and fsharp, but awstool
does more than run it: it generates an env module, rewrites the app
entrypoint, writes the OIDC stack, and parses the generated source to
find the stack class, all in the project's language. That is a set of
code generators per language rather than a list entry, so the supported
set is deliberately small.

For another language, run `cdk init` yourself and use awstool for
`--with-repo` and `--with-account`. You get the repo, the OU, the
accounts and the `~/.aws/config` profiles; bootstrap each account
(`cdk bootstrap aws://<account-id>/<region> --profile <name>-<env>`) and
wire the deploy role and workflow by hand. `--with-cdk` and `--with-ci`
assume TypeScript or Python and will fail on other languages.

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
CLIs on PATH, `gh` authenticated and the repo name available under the
resolved owner, the AWS profile belongs to the org management account, no
OU or `~/.aws/config` profile name collisions, and (for cdk) an empty
directory or an existing `cdk.json`. The region in use is printed before
any work starts.

## Prerequisites

- `gh` authenticated (`gh auth login`)
- `aws` CLI with a management-account profile (default: `default`)
- `node`/`npx` for CDK steps (`cdk` used if installed, else `npx aws-cdk`)
- `python3` 3.7 or newer (both scripts are standard library only, so
  there is no `requirements.txt` and nothing to `pip install`)
- AWS CLI v2.13 or newer for `awsconsole` (it uses
  `aws configure export-credentials`)

## Install

Both scripts are standalone; symlink them onto your PATH.

```
ln -s "$PWD/awstool" ~/bin/awstool
ln -s "$PWD/awsconsole" ~/bin/awsconsole
```

## awsconsole

`awsconsole <account>` opens the AWS web console for any account you can
reach, with no stored passwords and no account switching in the UI.

```
awsconsole --list
awsconsole poker        # unique substring is enough
```

It builds the account list from two sources and merges them:

- **Organizations** - every active member account reachable from the
  management profile, under the name Organizations gives it. Reached by
  assuming `OrganizationAccountAccessRole` (override with `--role`;
  Control Tower organizations usually use `AWSControlTowerExecution`).
- **`~/.aws/config`** - every profile carrying an account id, meaning an
  SSO profile (`sso_account_id`) or an assume-role profile (account id
  read out of `role_arn`). Credentials come from the profile itself via
  `aws configure export-credentials`, so SSO, role chaining and
  `credential_process` all work, and an expired SSO token triggers
  `aws sso login` automatically.

An account in both is reached through its profile, since that is the
explicit choice and may carry a narrower role. Matching is
case-insensitive against both the Organizations name and the profile
name. Accounts created by awstool show up under either name.

No AWS Organization is required: if the management profile cannot read
Organizations, awsconsole falls back to `~/.aws/config` profiles alone.
That also covers accounts in a *different* organization, reached through
their own SSO profile.

Options: `--list`, `--print-url` (print the sign-in URL instead of
opening a browser), `--profile`, `--role`, `--region`. Each has an
environment default: `AWSCONSOLE_PROFILE`, `AWSCONSOLE_ROLE`,
`AWSCONSOLE_REGION`. Region otherwise follows the profile in use, then
`AWS_DEFAULT_REGION`, then `us-east-1`.

The generated sign-in URL grants console access to whoever holds it, so
it is only printed with `--print-url`. Commercial AWS partition only;
GovCloud and China use different sign-in endpoints.

## License

MIT. See [LICENSE](LICENSE).

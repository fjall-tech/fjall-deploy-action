# Fjall Deploy Action

A GitHub Action to deploy, destroy, or build [Fjall](https://fjall.io) applications on AWS.

This is a thin shim onto `fjall ci run` — it installs the CLI and delegates. The CLI owns command and flag validation, so an invalid combination (for example `mode` with `command: destroy`) fails with a directional error from `fjall` rather than being silently dropped.

A tier `target` (`organisation`, `platform`, or `account`) runs the tier's own command, such as `fjall org deploy` or `fjall platform deploy`, and accepts only the flags that tier honours (`force`, `verbose`, and, for `organisation`, `no-cascade`). From fjall 39.0.12, an `account` or `platform` destroy is refused, because it destroyed every stack in the organisation, and from fjall 40 it removes only that tier's stack. Any app property, or one only a deploy reads, paired with a tier target is refused at the plugin boundary rather than silently dropped.

## Quick Start

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-24.04-arm
    steps:
      - uses: actions/checkout@v4

      - uses: fjall-tech/fjall-deploy-action@v31
        with:
          target: my-app
          deploy-target: production-use1
        env:
          FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

The deploy token is also the deploy's AWS credential (Option 1 under [Authentication](#authentication)), so the job exports no AWS keys.

If the deploy job sets `timeout-minutes`, make it 360, the timeout `fjall ci setup` writes, GitHub's default, and the most a job on a GitHub-hosted runner may run: from fjall 40, a deploy waits out each database engine upgrade, which AWS has held for up to three hours. When a plan's upgrades could wait longer than 360 minutes, the plan warns before anything is upgraded (`database-engine-ci-job-limit`). Fjall cannot read the job's own `timeout-minutes`, so it reads 360 on a GitHub-hosted and a self-hosted runner alike. As of 2026-10-11 that has two costs: a lower `timeout-minutes` goes unwarned, and a job on a self-hosted runner that raised its `timeout-minutes` above 360 is still warned, and cannot silence the warning. A job stopped mid-upgrade leaves the upgrade running in RDS, and re-running the deploy waits for it and resumes.

## Runner Architecture

Fjall builds ECS images for `linux/arm64`, the platform its tasks run on, and a container Lambda's image for its `architecture`. Run the job on a runner of the architecture those images build for: `ubuntu-24.04-arm` on GitHub-hosted runners, which `fjall ci setup` writes from fjall 40. From fjall 40 it writes `ubuntu-24.04` only when the manifest a synth left in the app's `cdk.out` lists only x86_64 images. It stays on `ubuntu-24.04-arm` otherwise. When `cdk.out` holds no current list, a warning names the commands that write one and the rerun, and when the app builds both, a warning names each x86_64 Lambda. Fjall 39's `ci setup` writes `ubuntu-latest`, so in a workflow it generated for an app that builds arm64 images, change `runs-on` by hand.

On an x86 runner such as `ubuntu-latest`, an arm64 image's `RUN` steps can run only under emulation, which the action does not register. The default BuildKit image behind Fjall's `fjall` builder ships one (`buildkit-qemu-aarch64`, checked on BuildKit 0.33.1), and BuildKit documents that most builds need no manual QEMU setup. BuildKit also documents emulation as much slower than a native build, and advises against it for compiling. A `RUN` step can still fail when its container mixes binaries for several architectures. BuildKit's documented cure is to register QEMU on the host, which `docker/setup-qemu-action` does.

## Authentication

Two credentials are involved: a Fjall deploy token (`FJALL_API_KEY`) and AWS credentials.

### Fjall API Key

The CLI authenticates to Fjall via `FJALL_API_KEY` — an app-scoped `fjall_dk_` deploy token used to record the deployment, hold the app's deploy slot, and read organisation config. A deploy token is minted only in a signed in browser session of an owner or admin. From a checkout of the repository, run `fjall ci token create --app <app> --target <target>` (CLI 40.0.1 or later). You approve its request in your own browser session, because only the person whose Fjall credential opened a request can approve it. Then the command stores the token as the `FJALL_API_KEY` repository secret itself, through `gh`. Or mint one on **Settings → CI/CD Tokens** in the [Fjall dashboard](https://fjall.io), the page whose URL `fjall ci setup` prints. Tokens expire after at most 90 days. To rotate, run the command again, or re-mint from the same page and update the secret.

Store the token as a repository secret named `FJALL_API_KEY` (`gh secret set FJALL_API_KEY`) and pass it in the step's `env:` block as in the Quick Start. It takes precedence over any credentials stored on the runner (`~/.fjall/auth.json`).

GitHub hands that env to every step of the action. Every step but Run Fjall takes the token and any AWS keys out of it. Check inputs, Setup Node.js and Summarise image builds blank them in their `env:`, and Install Fjall CLI unsets them before it installs the CLI, which runs no package's install scripts. Run Fjall takes the token out of its own environment too, and hands it to the `fjall` command alone. When the job grants `id-token: write`, the runner also sets GitHub's OIDC request token in every step, after the step's `env:`. Install Fjall CLI and Summarise image builds remove it, while Check inputs, which runs no command but `mktemp`, and Setup Node.js keep it. The runner also gives the commit-pinned `actions/setup-node` step its own runtime token.

### AWS Credentials

The CLI resolves AWS credentials in a fixed order and **the first match wins**:

1. `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` in the step's environment
2. A Fjall-minted session, exchanged from your deploy token

Pick one path. With `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in the
step's environment the CLI uses them and never reaches the Fjall mint. With
fjall 39, an application deploy or destroy then runs in whatever account they
belong to, whatever target `deploy-target` or `environment` picks. A deploy runs
in `AWS_REGION` (else `AWS_DEFAULT_REGION`, else `us-east-2`) unless `region` is
set, and a destroy runs there even when it is. A tier step doesn't check the
keys first either. An account deploy deploys in their account. It refuses only
when that account is the one registered with Fjall as the organisation's
management or platform account, or when Fjall's list of accounts can't be read.
An organisation destroy records its start and reads AWS Organizations in their
account, creating an organisation there if it has none, before it checks the
account. An organisation deploy goes further before that check. It sets up the
organisation in their account, creating member accounts and organisational
units, and registers those accounts with Fjall. A platform deploy refuses keys
for another account, but only after it has recorded its start. From fjall 40,
the CLI first confirms they belong to the account the step acts on. If they
don't, fjall 40 refuses the step before it records or changes anything, and
names both accounts and the variables to unset. If they do, fjall 40 runs an
application step in the target's own region unless `region` is set. Running
`aws-actions/configure-aws-credentials` in a job meant to deploy through the
Fjall mint is the usual way to get this wrong.

#### Option 1: Fjall OIDC (recommended — no AWS setup)

Set `FJALL_API_KEY` and export no AWS credentials. The CLI exchanges your
deploy token with the Fjall API for a short-lived, Fjall-signed web-identity
token, then calls `sts:AssumeRoleWithWebIdentity` against the
`FjallDeploy<orgId>` role Fjall created when you connected your AWS account.
The deploy token _is_ the pipeline's AWS credential.

You need no IAM OIDC provider of your own, no deploy role, no AWS secrets, and
**no `id-token: write` permission** — the identity is minted by Fjall, not by
GitHub.

The session this path mints lasts one hour. From fjall 40, the CLI renews it
through the deploy token before it expires, so a deploy can run past the hour.
Fjall 39 and earlier renew no session on any path, so with them a deploy that
runs longer than an hour fails part-way on this path too.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-24.04-arm
    steps:
      - uses: actions/checkout@v4

      - uses: fjall-tech/fjall-deploy-action@v31
        with:
          target: my-app
          deploy-target: production-use1
        env:
          FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

Every CI deploy token carries the `deploy:oidc:mint` scope in its base set, so
there is nothing extra to grant.

#### Option 2: AWS OIDC

Use GitHub's OIDC provider with `aws-actions/configure-aws-credentials` for
keyless authentication against a role you create and maintain. `id-token: write`
is required here, and the Fjall mint is not used:

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-24.04-arm
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/deploy
          aws-region: us-east-1

      - uses: fjall-tech/fjall-deploy-action@v31
        with:
          target: my-app
          deploy-target: production-use1
        env:
          FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

The CLI cannot renew a session it reads from the environment, so a deploy that
runs longer than the role session (one that upgrades database engines, for
example) fails when the session expires.

#### Option 3: Static IAM credentials

Pass long-lived keys as environment variables from GitHub Secrets. Least
preferred — these do not expire on their own:

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: my-app
    deploy-target: production-use1
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_REGION: us-east-1
```

The CLI cannot renew keys it reads from the environment, so when you pass
temporary session keys here instead (with `AWS_SESSION_TOKEN`), a deploy that
runs longer than their session (one that upgrades database engines, for
example) fails when they expire.

## Inputs

| Input               | Required | Default  | Applies to         | Description                                                                                                                                                                                                                                                                                                                                                    |
| ------------------- | -------- | -------- | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`           | no       | `deploy` | —                  | `deploy`, `destroy`, or `build`                                                                                                                                                                                                                                                                                                                                |
| `target`            | **yes**  | —        | all                | Application name, or one of the tiers `organisation`, `platform` and `account`. App targets run the app, and tier targets run the tier's own command (`fjall org\|platform\|account deploy\|destroy`). From fjall 39.0.12 an `account` or `platform` destroy is refused, and from fjall 40 it removes only that tier's stack. `build` runs on app targets only |
| `service`           | no       | `-`      | deploy, build      | One service, from fjall 40 by its whole name, ignoring case. Fjall 39 also selects every service whose name ends with it. A deploy narrows to it only with `mode: deploy-only` or `image-tag`. Fjall 40 refuses it on any other deploy, and fjall 39 leaves it unread                                                                                          |
| `mode`              | no       | `full`   | deploy             | `full`, `infra-only`, or `deploy-only`                                                                                                                                                                                                                                                                                                                         |
| `environment`       | no       | `-`      | deploy             | Target environment (maps to the `-e` flag)                                                                                                                                                                                                                                                                                                                     |
| `deploy-target`     | no       | `-`      | app targets        | Deploy to a specific `fjall target list` target, e.g. `production-use1`; maps to `--target` (the credential where), distinct from `environment`                                                                                                                                                                                                                |
| `verbose`           | no       | `false`  | all                | Enable verbose output                                                                                                                                                                                                                                                                                                                                          |
| `skip-build`        | no       | `false`  | deploy             | Skip a static-site or OpenNext build command when its output is already built; never skips a Docker build                                                                                                                                                                                                                                                      |
| `no-cascade`        | no       | `false`  | org deploy/destroy | Skip the platform/account cascade around an `organisation` deploy or destroy (rejected for any other target)                                                                                                                                                                                                                                                   |
| `region`            | no       | `-`      | deploy             | Deploy to a specific region within the target's account                                                                                                                                                                                                                                                                                                        |
| `image-tag`         | no       | `-`      | deploy             | Roll the app's services and container image functions to images already in the app's ECR repository, with no build and no template change. Takes precedence over `mode`                                                                                                                                                                                        |
| `plan`              | no       | `false`  | deploy             | Compute and print the change plan, then stop before any mutation; not with `mode: deploy-only` or `image-tag`, which change no template                                                                                                                                                                                                                        |
| `require-approval`  | no       | `false`  | deploy             | Engage the approval gate: refuse to mutate unless approved via `auto-approve` or `approval-token`; not with `mode: deploy-only` or `image-tag`, which change no template                                                                                                                                                                                       |
| `auto-approve`      | no       | `false`  | deploy             | Approve the computed plan without prompting (use with `require-approval`)                                                                                                                                                                                                                                                                                      |
| `approval-token`    | no       | `-`      | deploy             | Resume an approved plan from a prior `plan` run (digest re-verified before applying); not with `mode: deploy-only` or `image-tag`, which change no template                                                                                                                                                                                                    |
| `build-args`        | no       | `-`      | deploy, build      | Newline-separated `KEY=VALUE` pairs, each passed via `--build-arg`                                                                                                                                                                                                                                                                                             |
| `build-secrets`     | no       | `-`      | deploy, build      | Newline-separated `id=ID,ssm=PATH` (or `secretsManager=NAME` \| `env=VAR`) tokens (`--build-secret`)                                                                                                                                                                                                                                                           |
| `force`             | no       | `false`  | deploy, destroy    | Deploy: deploy every stack, including those with no detected changes (CloudFormation leaves an unchanged stack as it is). Destroy: bypass the busy-stack guard and destroy while the target's stacks are mid-operation. App destroys skip destruction confirmation in CI regardless; a tier destroy still needs force to pass its non-interactive confirmation |
| `cli-version`       | no       | `40`     | —                  | Pin the published `fjall` CLI version. Defaults to this action's compatible major (floats across `40`.x, never crossing into the next major); `auto` derives the major from the app's pinned `@fjall/components-infrastructure`; `latest` always installs the newest published release                                                                         |
| `working-directory` | no       | `.`      | —                  | Folder that holds the project's `fjall/` folder, relative to the repository's checkout. `fjall` runs there                                                                                                                                                                                                                                                     |
| `build-record`      | no       | `true`   | deploy, build      | Keep BuildKit's record of the run's image builds for a later step to upload. It holds every build arg value, each Dockerfile's whole source and each build step's log, unmasked, so read [Keeping the build record](#keeping-the-build-record) before you upload it. Needs buildx 0.23.0 or later. A destroy and a tier target keep no record                  |

Boolean inputs take `true` or `false`. Leave one out for its default, which is `false` for every one but `build-record`. Any other value stops the action at its first step, Check inputs, before Node or the CLI is installed, with an error naming the input: `'yes'`, `'True'`, `1`, `yes`, `no`, `on`, `off`, a padded value, or an expression that resolves to anything else.

An input of any kind given an empty value stops the action the same way. The action cannot tell an expression that came out empty (`image-tag: ${{ env.ROLLBACK_TAG }}` with the variable unset) from a value left blank on purpose, and reading it as the input left out would turn that rollback into a build and deploy. Leave the input out, give it a value, or, for an input without a default of its own, pass `-`, which means none. Those inputs are `service`, `environment`, `deploy-target`, `region`, `image-tag`, `approval-token`, `build-args` and `build-secrets`, which default to it. Every other input refuses `-`, `cli-version` and `working-directory` included. Pass `approval-token` the plan's output as it is, never with a `|| '-'` ending: a plan cancelled before it finished leaves the output empty, and the empty value stops the apply step. Handed `-`, the step runs with no token and could deploy with no reviewed plan, unless it sets `require-approval: true` without `auto-approve: true`. To pass an optional value to any other of those inputs in one step, end the expression with `|| '-'`:

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: my-app
    deploy-target: production-use1
    image-tag: ${{ inputs.image-tag || '-' }}
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

The same ending closes a conditional value: `${{ <condition> && '<value>' || '-' }}`.

A plan and the approval gate cover a full or `infra-only` deploy. `mode: deploy-only` and `image-tag` change no template, so they have no plan, and a run of either with `plan`, `require-approval` or `approval-token` is refused before anything is deployed. Through CLI 39.0.8 such a run went ahead without either.

## Outputs

| Output              | Description                                                                                                                  |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `result`            | `success`, `plan-pending` (exit code 2 — see below), or `failure`                                                            |
| `duration`          | Command duration in seconds                                                                                                  |
| `approval-token`    | Token emitted by a `plan` run with pending changes; feed back via the `approval-token` input                                 |
| `build-record-path` | Path of the `.dockerbuild` record a run with `build-record: true` kept, for a later step to upload. Empty when none was kept |

## Exit Codes

| Code | Meaning                                                                                                                                                                                                                         |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `0`  | Success (for `plan`: no changes detected)                                                                                                                                                                                       |
| `1`  | Error                                                                                                                                                                                                                           |
| `2`  | Stopped without failing: a plan with changes pending approval, a plan refused at the approval gate, or a run cancelled before it completed, which can follow a partial change. `approval-token` is set only for pending changes |

The action propagates exit code 2, so a `plan` step with pending changes fails the job unless you set `continue-on-error: true` and branch on the `result` output.

A cancelled run that ended on a failure the cancel didn't cause, such as a call AWS refused or a database a destroy left, sets `result` to `failure` and exits 1.

## Verifying a Deploy

The `result` output answers "did the step succeed". To confirm what actually shipped, the CLI's read commands work anywhere the same `FJALL_API_KEY` is available (including a follow-up step):

- `fjall deployments list` — your organisation's active and recent deployments: who triggered each, from where, and its current state
- `fjall releases <app>` — the app's recorded releases with their image tags: what is live now, and what a rollback can target
- `fjall status <app>` — the deployed infrastructure's current health

## Build summary

After the `fjall` command ends, the action's last step, Summarise image builds, writes one table of the run's image builds to the job's summary page. It runs whether the command passed, failed or stopped at `plan-pending`. It reads the build history of Fjall's `fjall` builder rather than anything the CLI prints, so it works with fjall 39 and 40. It never changes the action's result or exit code, and it adds at most about 30 seconds. A destroy skips it, and a run that built no image, such as a tier deploy or an `image-tag` roll, writes no table. The table needs buildx 0.21.0 or later on the runner.

Each row is one image, named `image 1`, `image 2` and so on in build order, with the platform it was built for in brackets, such as `image 1 (linux/arm64)`. The table shows the first 20 images and counts the rest. It names no service, Dockerfile, build stage or instruction, because those are text the app's authors wrote, and the action has no way to mask a secret inside them. Match a row to a service by build order, or by the CLI's log above it.

| Column        | What it shows                                                                                                                                                                                                     |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Image         | `image N`, in build order, with the image's platform                                                                                                                                                              |
| Built         | `native` when the image's platform is the runner's own, `emulated on <platform>` when the runner built it under QEMU on a host of the platform named, and `unknown` when the step couldn't tell                   |
| Total         | The whole build, as BuildKit timed it. It ends `, failed` for a build that failed and `, cancelled` for one that stopped before it finished, and reads `running` for one still running when the step read it      |
| Context       | Loading the build context, with how much was sent                                                                                                                                                                 |
| Cache lookup  | Looking up the registry cache, with what the lookup found, which is `found`, `none found` (a first build, or a cache nobody has written yet), `refused`, `failed`, or `not finished` when the build stopped first |
| Cache pull    | Pulling and unpacking the layers of the steps the cache answered                                                                                                                                                  |
| Rebuilt steps | Running the steps the cache didn't answer                                                                                                                                                                         |
| Cached        | The share of the Dockerfile's steps the cache answered, leaving out `FROM`, with a count of any step that failed or never finished                                                                                |
| Image push    | Pushing the image to its repository                                                                                                                                                                               |
| Cache export  | Writing the registry cache, with how many layers it wrote                                                                                                                                                         |

Each phase is a window on BuildKit's clock, and BuildKit runs work in parallel, so the windows overlap and can add up to more than Total. Cache pull is an upper bound, because a cached step's window holds time it spent waiting as well as pulling.

Under the table, each image lists its 3 slowest rebuilt steps of at least 1 second, and the first step the cache missed, which broke the cache for every step after it in its stage. A step reads `[n/m] KEYWORD`, such as `[4/9] RUN`, without its instruction, because BuildKit puts build arg values into a `RUN` step's name.

Notes follow the steps. An image built for a platform other than the runner's own was built under emulation, and its note names the runner that builds it natively, `ubuntu-24.04-arm` for `linux/arm64` and `ubuntu-24.04` for `linux/amd64` (see [Runner Architecture](#runner-architecture)). Each image also gets a `docker buildx history logs --builder fjall <ref>` command, which prints that build's whole BuildKit log from a later step of the same job. The job log carries the same commands, and one line saying where the table went.

### Keeping the build record

By default, with buildx 0.23.0 or later, the step also exports BuildKit's record of the run's image builds to one `.dockerbuild` file on the runner, and the `build-record-path` output gives its path. Docker Desktop 4.31 or later opens the file in its Builds view, with each step's timing and log, through Import builds or `docker buildx history import --file <file>`. The action uploads nothing itself, so add an `actions/upload-artifact` step after it, as below. Its `if:` uploads the record after a failed deploy too, and skips the upload when no record was kept.

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  id: deploy
  with:
    target: my-app
    deploy-target: production-use1
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}

- uses: actions/upload-artifact@v7
  if: ${{ !cancelled() && steps.deploy.outputs.build-record-path != '' }}
  with:
    name: fjall-build-record
    path: ${{ steps.deploy.outputs.build-record-path }}
    retention-days: 7
```

`actions/upload-artifact` v6 and later run on Node 24 and need Actions Runner 2.327.1 or later. On an older self hosted runner use v4, and on GitHub Enterprise Server use the release the `actions/upload-artifact` README names for it, which was v3.2.2 on 12 November 2026.

The record holds all of this unmasked. It holds every build arg value, in its `BuildArgs` and inside the names of `RUN` steps. It holds each Dockerfile's whole source, every instruction as written and as run, with the environment each `RUN` step saw, the image's configuration and each build step's log. It also names the absolute paths of the build context and the Dockerfile on the runner. Anyone with read access to the repository can download an uploaded artifact, and on a public repository that's anyone signed in to GitHub. So pass a secret through `build-secrets`, never `build-args`, keep `retention-days` short, and upload the record only where every reader may see what the builds used. Set `build-record: false` to keep no record at all. On a repository GitHub doesn't report as private, the step prints a warning when the record is on. A tier target and a destroy keep no record whatever `build-record` says, because neither builds an image.

## Serialising Deploys

Fjall serialises deployments that touch the same infrastructure (the deploy slot): an application's deploys queue behind an in-flight deploy of the same application, of its account's infrastructure, or of the organisation, while unrelated applications deploy in parallel. Give every workflow that deploys the same app a shared concurrency group — the same shape `fjall ci setup` scaffolds — so overlapping runs queue instead of contending for the slot:

```yaml
concurrency:
  group: fjall-deploy-my-app
  cancel-in-progress: false
```

Keep `cancel-in-progress: false`: cancelling a deploy mid-run can interrupt an in-flight CloudFormation update.

## Deploy Modes

One action runs every kind of deploy, and its inputs choose which. Each row links to a worked example.

| Deploy                                                   | Inputs                                               | What it does                                                                                                                               | On a fresh runner                                                                                                                              |
| -------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| [Full](#full-deploy-default)                             | Leave `mode` out, or `mode: full`                    | Deploys the app's stacks, and builds and rolls the checkout's images                                                                       | Runs                                                                                                                                           |
| [Infrastructure only](#infrastructure-only)              | `mode: infra-only`                                   | Deploys the app's stacks, and builds and rolls the checkout's images only when the Compute stack deploys                                   | Runs, unless a static site or OpenNext app has no built output                                                                                 |
| [Code only](#code-only)                                  | `mode: deploy-only`                                  | Builds and rolls what the assembly in `fjall/<app>/cdk.out` declares, with no template change                                              | Refused, unless a step earlier in the job wrote the assembly ([Split Infrastructure and Code Deploys](#split-infrastructure-and-code-deploys)) |
| [An image already built](#roll-back-to-a-previous-image) | `image-tag`                                          | Rolls the app's services and container image functions to images already in the app's ECR repository, with no build and no template change | Runs, unless the roll reaches a container image function, which needs the assembly                                                             |
| [One service](#deploy-specific-service)                  | `service`, beside `image-tag` or `mode: deploy-only` | Narrows an `image-tag` roll or a code only deploy to that service                                                                          | As the row it narrows                                                                                                                          |

Leave `mode` out when you set `image-tag`. The roll takes precedence over any `mode`, so `mode: infra-only` beside it deploys no infrastructure.

### Full Deploy (default)

Deploys infrastructure and application code:

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: my-app
    deploy-target: production-use1
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

### Infrastructure Only

Deploys the stacks that changed, or every stack with `force`, and builds and rolls the checkout's images only when the Compute stack is among them. It never runs a static site's or OpenNext build command. Where the checkout holds no built output, as on a fresh runner, it refuses such an app, so deploy that app in full.

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: my-app
    deploy-target: production-use1
    mode: infra-only
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

### Code Only

Deploy application code using existing infrastructure (faster for code-only changes). A code-only deploy builds and rolls only what the assembly a full deploy writes to `fjall/<app>/cdk.out` declares, so it needs that assembly in the job's checkout. A fresh runner has none, and there the deploy is refused (through CLI 39.0.8 it rolled nothing and exited 0; an app with capacity-identity pins was refused instead). On a fresh runner, use a full deploy, or `image-tag` for an image already built, which rolls the services the app's Compute stack exports with no assembly; an app with a container-image function in the roll needs the assembly for that too. Where a step earlier in the same job wrote the assembly, such as an `infra-only` deploy (Split Infrastructure and Code Deploys, below), the code-only step is:

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: my-app
    deploy-target: production-use1
    mode: deploy-only
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

## Examples

### Plan Gate (review changes before applying)

Compute the change plan on pull requests without mutating anything:

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  id: plan
  continue-on-error: true
  with:
    target: my-app
    deploy-target: production-use1
    plan: true
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}

- name: Fail unless plan is clean
  if: steps.plan.outputs.result == 'failure'
  run: exit 1
```

Then apply the approved plan (the token is digest-bound — the apply re-verifies that the plan has not drifted). A plan with no changes prints no token, so run the apply only when changes are pending:

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  if: steps.plan.outputs.result == 'plan-pending'
  with:
    target: my-app
    deploy-target: production-use1
    approval-token: ${{ steps.plan.outputs.approval-token }}
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

Or approve in one shot in trusted pipelines:

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: my-app
    deploy-target: production-use1
    require-approval: true
    auto-approve: true
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

### Roll Back to a Previous Image

`image-tag` rolls the app's services and container image functions to images already built, with no build and no template change. Each rolls to its own image with the same tag ending, so `api-sha-4f9c2ab0d1e3` rolls a `worker` service to `worker-sha-4f9c2ab0d1e3`. Each of those images must already be in the app's ECR repository in the target account and region.

The shared ending holds only where the services run the same image, because a tag's ending comes from its image's digest. Where they run different images, the roll is refused and no service rolls. Roll each service in a step of its own instead, with `service` and its own tag (Deploy Specific Service, below).

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: my-app
    deploy-target: production-use1
    image-tag: "api-sha-4f9c2ab0d1e3"
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

Find valid tags with `fjall releases my-app` (each release records the image tags it shipped), from the original deploy's output, or from the app's ECR repository. An image-tag rollback re-pins application code only — it does not revert infrastructure changes or roll back database migrations.

### Build-Time Args and Secrets

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: my-app
    deploy-target: production-use1
    build-args: |
      GIT_SHA=${{ github.sha }}
      PUBLIC_API_URL=https://api.example.com
    build-secrets: |
      id=npm-token,ssm=/ci/npm-token
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

### Destroy with Force

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    command: destroy
    target: my-app
    deploy-target: production-use1
    force: true
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

### Deploy Specific Service

`service` narrows an `image-tag` roll, or a code only deploy, to one service. From fjall 40 it selects the service by its whole name, ignoring case, so `service: worker` rolls `worker`, never `scan-worker`, and a name that matches no service is refused, listing the services there are. Fjall 39 also selects every service whose name ends with the name given, so with it `service: worker` rolls `scan-worker` too. A code only deploy needs the assembly a full deploy wrote (Code Only, above), so on a fresh runner roll an image already built.

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: my-app
    deploy-target: production-use1
    service: api
    image-tag: "api-sha-4f9c2ab0d1e3"
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

### Deploy a Tier

A tier `target` routes to the noun-verb tier command. This runs `fjall org deploy --non-interactive --no-cascade` (the organisation root stack without its platform/account cascade):

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: organisation
    no-cascade: true
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

### Pin CLI Version

To pin an exact CLI release, put a version that `npm view fjall versions` lists in place of the placeholder.

```yaml
- uses: fjall-tech/fjall-deploy-action@v31
  with:
    target: my-app
    deploy-target: production-use1
    cli-version: "<exact version>"
  env:
    FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

### Split Infrastructure and Code Deploys

Run both steps in one job: the code step builds and rolls what the assembly the infrastructure step synthesised to `fjall/<app>/cdk.out` declares. A code step in a job of its own starts on a fresh runner with no assembly, and is refused there (Code Only, above).

```yaml
jobs:
  deploy:
    runs-on: ubuntu-24.04-arm
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/deploy
          aws-region: us-east-2
      - uses: fjall-tech/fjall-deploy-action@v31
        with:
          target: my-app
          deploy-target: production-use2
          mode: infra-only
        env:
          FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
      - uses: fjall-tech/fjall-deploy-action@v31
        with:
          target: my-app
          deploy-target: production-use2
          mode: deploy-only
        env:
          FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

### Staging to Production Promotion

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  staging:
    runs-on: ubuntu-24.04-arm
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-2
      - uses: fjall-tech/fjall-deploy-action@v31
        with:
          target: my-app
          environment: staging
        env:
          FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}

  production:
    needs: staging
    runs-on: ubuntu-24.04-arm
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-2
      - uses: fjall-tech/fjall-deploy-action@v31
        with:
          target: my-app
          environment: production
        env:
          FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

The production job builds what it deploys. `skip-build` would not reuse what staging built: it skips only a static-site or OpenNext build command, never a Docker build.

A step that names only `environment` needs exactly one connected account on that stage. Where a stage has more, name the target with `deploy-target`, as `fjall target list` prints it.

## Versioning

Two pins interact, and they move together:

- **The action ref** — `fjall-tech/fjall-deploy-action@v31` is a moving major tag, repointed at the latest 31.x release on each republish; pin an exact tag (`@v31.0.0`) for maximum determinism.
- **`cli-version`** — the `fjall` CLI the action installs per run; it defaults to the action major's compatible CLI major.

Upgrade majors deliberately. A new action major either defaults to a new CLI major, or changes the action's own contract on the same CLI major, as v30 did: read its entry under [Breaking changes](#breaking-changes) before moving the ref, and when the CLI major changes, move any explicit `cli-version` pin in the same change.

## Breaking changes

### v30.0.0

v30 is a new action major, so a workflow pinned to `@v29` keeps the old behaviour until its owner moves the pin. A workflow on `@main` runs the action's `main` branch, which moved to v30 when v30 was published and moves to v31 with the fjall 40 release. v30 installs the same CLI major as v29, 39; v31 carries these changes with the CLI 40 default.

An input given an empty value, and a boolean that is not exactly `true` or `false`, now stops the action at its first step, Check inputs, before Node or the CLI is installed, with an error naming the input and never repeating the value. Before, an empty value read as the input left out, or worse (an empty `cli-version` installed the newest release of any major), and a boolean spelled any other way took the false arm: a step asked to plan deployed, an approval gate was dropped, an organisation deploy or destroy cascaded. An input left out takes its default as before, and `-` means none for every input without a default of its own.

- **An empty value of any input:** an expression that came out empty (`${{ env.ROLLBACK_TAG }}` with the variable unset), or a value left blank on purpose, now stops the action. Leave the input out, give it a value, or, for an input without a default, end the expression with `|| '-'` (`image-tag: ${{ inputs.image-tag || '-' }}`), which also closes `<condition> && '<value>'`. Pass `approval-token` the plan's output as it is: with `|| '-'`, an apply after a cancelled plan runs with no token and could deploy with no reviewed plan, unless the step sets `require-approval: true` without `auto-approve: true`.
- **A plan step's token handed to an apply step:** after a plan with no changes the `approval-token` output is empty, so the apply step now stops. Run the apply step only when the plan's `result` output is `plan-pending`, as the Plan Gate example does.
- **A false not written as `false`:** quoted `'False'`, `'no'` or `'0'`; unquoted `0`, `no`, `off` or `n`; a padded value; or an expression that resolves to one of these. These read as false, and now stop the action. Write `false`, or leave the input out.
- **A true not written as `true`:** quoted `'True'` or `'yes'`; unquoted `1`, `yes`, `on` or `y`. These ran the false arm before. Write `true`, and check what earlier runs did. Unquoted `True` and `TRUE` reach the action as `true`.
- **`-`:** the inputs with no value of their own (`service`, `environment`, `deploy-target`, `region`, `image-tag`, `approval-token`, `build-args`, `build-secrets`) now declare `-` as their default in `action.yml`, in place of `""`, and read it as left out, as the Buildkite plugin does. Every other input refuses it in Check inputs, `cli-version` and `working-directory` included, which handed it to npm as a version and to the Run Fjall step as its directory.
- **Caller variables in the CLI:** the CLI, and with it `infrastructure.ts` and the image build, saw an `INPUT_*` variable for each input but `cli-version` and `working-directory` (`INPUT_TARGET`, `INPUT_ENVIRONMENT` and the rest), holding the action's own value whether or not the calling step exported that name. It also saw the action's own value of `LOG_FILE`, `START` and `TIER_NOUN`, which Run Fjall set for itself before running the CLI, when the calling step exported the same name, and, when the step exported `ARGS`, no `ARGS` at all, nor on a tier target `INCOMPATIBLE`. It now sees each of those upper-case names as the calling step exported it, and no `INPUT_*` variable the step did not export: code that read one must read a variable the calling step sets in its own `env:` instead. Run Fjall's own names are now lower-case, so a calling step's variable that shares one it sets before running the CLI (`argv_file`, `arg`, `log_file` or `start`) reaches the CLI with the action's value, or, for `args`, not at all.
- **The CLI install:** the CLI now installs with `--ignore-scripts`. On a runner whose npm configuration omits optional dependencies (`omit=optional`), esbuild's install script used to fetch its binary; now the CLI fails the first time it needs esbuild. Let npm install optional dependencies.
- **Check inputs and setup-node:** validation runs in a new first step, Check inputs, and Setup Node.js now runs `actions/setup-node` v4.4.0, pinned by commit.

Rollback: the workflow's owner pins the previous major, `v29`, which keeps the old behaviour.

## Caching

The action restores no cache: it installs the CLI with `npm install -g` on every run, which on a GitHub-hosted runner downloads the whole CLI each time. What a restored cache holds decides what runs in a job that has `FJALL_API_KEY` and AWS credentials, and any workflow run on the default branch that a push, a schedule or a dispatch started can write an entry the deploy job would restore, with contents GitHub does not verify (GitHub's [dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching) docs, read 2026-10-15).

npm's own cache is no exception: npm checks a package against the integrity its metadata names, and reads that metadata from the same cache. Measured on npm 11.17.0 (2026-10-15), metadata and a tarball planted in npm's cache made `npm install -g` install a version the registry never published, and run its install script. This action installs with `--ignore-scripts`, so no install script runs here, but the CLI a planted entry installs is the one Run Fjall then runs, holding the token. With `--prefer-online`, npm asked the registry, which answered the ETags the planted entries carried with 304 Not Modified, and npm kept the planted copies. So put no npm cache into a job that runs this action: no `actions/cache` step over `~/.npm`, and on an `actions/setup-node` step no `cache` input and, from setup-node v5, `package-manager-cache: false`. A self-hosted runner that keeps its home directory between jobs keeps npm's cache too, and every workflow it runs can write it: a runner that runs untrusted workflows should not run deploys.

## Troubleshooting

### Blocked deploy slot

> Blocked: Jane has been deploying my-app from CI since 03/08/2026, 14:02:11 (deployment cmd0a1b2c…). View progress in the Fjall dashboard: …

Fjall serialises deployments that touch the same infrastructure. An application deploy or destroy that starts while a conflicting deployment is in flight — the same application, its account's infrastructure, or the organisation — first queues behind it: the step re-claims the slot every 30 seconds for up to 20 minutes, logging who holds it, and only then fails with the message above. Tier targets (`organisation`, `platform`, `account`) route to the noun-verb commands and are not queued: they fail immediately on a held slot. Wait for the active deployment to finish (or cancel it from the dashboard), then retry. `fjall deployments list` shows your organisation's active deployments. Prevent the contention with a concurrency group ([Serialising Deploys](#serialising-deploys)).

### "This deployment requires a newer fjall CLI"

The Fjall API refuses deploys from a CLI older than the app's engine floor. The refusal suggests `npm install -g fjall@latest` — correct for a workstation, wrong here: the action installs the CLI per run from `cli-version`, so nothing on the runner needs upgrading. Fix it in the workflow instead:

- bump `cli-version` to the required major (or a newer exact version), or
- set `cli-version: auto` to derive the major from the app's pinned `@fjall/components-infrastructure`, or
- move to a newer action major (each action major defaults `cli-version` to its compatible CLI major).

### Authentication failures

Two different credentials can fail — the error tells you which:

- **Fjall API errors** (401/403 from the Fjall API) mean `FJALL_API_KEY` is missing, expired, or revoked. Deploy tokens live at most 90 days. Run `fjall ci token create` again for a replacement, or mint one in the dashboard (**Settings → CI/CD Tokens**) and update the repository secret.
- **AWS SDK errors** (`ExpiredToken`, `AccessDenied`, credential-chain failures naming AWS services) mean the workflow's AWS credentials are wrong — see [Authentication](#authentication).

### Build summary missing or partial

Summarise image builds prints one line in the job log for each thing that stopped it, naming the cause and the cure. The deploy's result never depends on it.

- **buildx older than 0.21.0.** The step prints a line naming 0.21.0 and writes no table. Update Docker on a self hosted runner, or move the job to a GitHub hosted one.
- **buildx older than 0.23.0, with `build-record` on, as it is by default.** The table still appears, and a line naming 0.23.0 says why no record was kept. Update buildx.
- **No image build found.** A run that built no image, such as a tier deploy or an `image-tag` roll, writes no table. If the CLI's log shows a build, the Docker daemon's clock may run more than 5 seconds behind the runner's, which makes this run's builds look older than the run. Sync the clock of the daemon's host.
- **Builds left out.** The step keeps a build only when its context is inside the repository's checkout, and the notes count the builds it left out. That leaves out a build another job ran on the same self hosted runner, and a build whose record names no context. Keep each image's build context inside the checkout.
- **Images missing from the end of the table.** The step reads for at most 30 seconds, then shows what it read and counts the rest, and keeps no record when no time is left. Run `docker buildx history ls --builder fjall` in a later step of the same job to list every build.
- **A line saying the build history couldn't be read.** buildx answered in a shape the step doesn't know, or `docker` isn't on the runner. The step's reading of buildx has been checked against buildx 0.37.2 only. Run `docker buildx history ls --builder fjall` in a later step to see what buildx answers, and update buildx.

## License

MIT

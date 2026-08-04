# Fjall Deploy Action

A GitHub Action to deploy, destroy, or build [Fjall](https://fjall.io) applications on AWS.

This is a thin shim onto `fjall ci run` — it installs the CLI and delegates. The CLI owns command and flag validation, so an invalid combination (for example `mode` with `command: destroy`) fails with a directional error from `fjall` rather than being silently dropped.

A tier `target` (`organisation`, `platform`, or `account`) routes to the noun-verb tier command — `fjall org deploy`, `fjall platform destroy`, and so on — and accepts only the flags that tier honours (`force`, `verbose`, and, for `organisation`, `no-cascade`). Any app or deploy-only property paired with a tier target is rejected at the plugin boundary rather than silently dropped.

## Quick Start

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: fjall-tech/fjall-deploy-action@v3
        with:
          target: my-app
        env:
          FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: us-east-2
```

## Authentication

Two credentials are involved: a Fjall deploy token (`FJALL_API_KEY`) and AWS credentials.

### Fjall API Key

The CLI authenticates to Fjall via `FJALL_API_KEY` — an app-scoped `fjall_dk_` deploy token used to record the deployment, hold the app's deploy slot, and read organisation config. Deploy tokens are minted from a signed-in browser session by an owner or admin (**Settings → CI/CD Tokens** in the [Fjall dashboard](https://fjall.io)); CLI mint paths are refused by design, so `fjall ci token create` can only direct you there. Tokens expire after at most 90 days — re-mint from the same page and update the secret to rotate.

Store the token as a repository secret named `FJALL_API_KEY` (`gh secret set FJALL_API_KEY`) and pass it in the step's `env:` block as in the Quick Start. It takes precedence over any credentials stored on the runner (`~/.fjall/auth.json`).

### AWS Credentials

AWS credentials must be configured **before** this action runs. The action does not manage credentials.

#### Option 1: IAM Credentials

Pass credentials as environment variables from GitHub Secrets:

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: my-app
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_REGION: us-east-2
```

#### Option 2: AWS OIDC (Recommended)

Use GitHub's OIDC provider with `aws-actions/configure-aws-credentials` for keyless authentication:

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/deploy
          aws-region: us-east-2

      - uses: fjall-tech/fjall-deploy-action@v3
        with:
          target: my-app
```

#### Option 3: Fjall OIDC

If your app is registered with Fjall, the CLI auto-detects GitHub's OIDC tokens via the `ACTIONS_ID_TOKEN_REQUEST_URL` environment variable. Just grant `id-token: write` permission and set your API key ([Fjall API Key](#fjall-api-key) above):

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: fjall-tech/fjall-deploy-action@v3
        with:
          target: my-app
        env:
          FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

## Inputs

| Input               | Required | Default  | Applies to         | Description                                                                                                                                                                                                                                                                           |
| ------------------- | -------- | -------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`           | no       | `deploy` | —                  | `deploy`, `destroy`, or `build`                                                                                                                                                                                                                                                       |
| `target`            | **yes**  | —        | all                | Application name, or a tier: `organisation`, `platform`, `account`. App targets run the app; tier targets route to the noun-verb tier command (`fjall org\|platform\|account deploy\|destroy`). `build` is app-only                                                                   |
| `service`           | no       | —        | deploy, build      | Specific service name                                                                                                                                                                                                                                                                 |
| `mode`              | no       | `full`   | deploy             | `full`, `infra-only`, or `deploy-only`                                                                                                                                                                                                                                                |
| `environment`       | no       | —        | deploy             | Target environment (maps to the `-e` flag)                                                                                                                                                                                                                                            |
| `deploy-target`     | no       | —        | app targets        | Deploy to a specific `fjall target list` target, e.g. `production-use1`; maps to `--target` (the credential where), distinct from `environment`                                                                                                                                       |
| `verbose`           | no       | `false`  | all                | Enable verbose output                                                                                                                                                                                                                                                                 |
| `skip-build`        | no       | `false`  | deploy             | Skip Docker build (use with `mode: deploy-only` when the image is already pushed)                                                                                                                                                                                                     |
| `skip-migrations`   | no       | `false`  | deploy             | Skip database migrations during this deployment                                                                                                                                                                                                                                       |
| `no-cascade`        | no       | `false`  | org deploy/destroy | Skip the platform/account cascade around an `organisation` deploy or destroy (rejected for any other target)                                                                                                                                                                          |
| `region`            | no       | —        | deploy             | Deploy to a specific region within the target's account                                                                                                                                                                                                                               |
| `image-tag`         | no       | —        | deploy             | Roll forward/back to an existing image tag (skips build, implies `deploy-only`)                                                                                                                                                                                                       |
| `plan`              | no       | `false`  | deploy             | Compute and print the change plan, then stop before any mutation                                                                                                                                                                                                                      |
| `require-approval`  | no       | `false`  | deploy             | Engage the approval gate: refuse to mutate unless approved via `auto-approve` or `approval-token`                                                                                                                                                                                     |
| `auto-approve`      | no       | `false`  | deploy             | Approve the computed plan without prompting (use with `require-approval`)                                                                                                                                                                                                             |
| `approval-token`    | no       | —        | deploy             | Resume an approved plan from a prior `plan` run (digest re-verified before applying)                                                                                                                                                                                                  |
| `build-args`        | no       | —        | deploy, build      | Newline-separated `KEY=VALUE` pairs, each passed via `--build-arg`                                                                                                                                                                                                                    |
| `build-secrets`     | no       | —        | deploy, build      | Newline-separated `id=ID,ssm=PATH` (or `secretsManager=NAME` \| `env=VAR`) tokens (`--build-secret`)                                                                                                                                                                                  |
| `force`             | no       | `false`  | deploy, destroy    | Deploy: redeploy all stacks even when unchanged. Destroy: skip destruction confirmation                                                                                                                                                                                               |
| `cli-version`       | no       | `7`      | —                  | Pin the published `fjall` CLI version. Defaults to this action's compatible major (floats across `7`.x, never crossing into the next major); `auto` derives the major from the app's pinned `@fjall/components-infrastructure`; `latest` always installs the newest published release |
| `working-directory` | no       | `.`      | —                  | Directory containing `fjall-config.json`                                                                                                                                                                                                                                              |

## Outputs

| Output           | Description                                                                                  |
| ---------------- | -------------------------------------------------------------------------------------------- |
| `result`         | `success`, `plan-pending` (exit code 2 — see below), or `failure`                            |
| `duration`       | Command duration in seconds                                                                  |
| `approval-token` | Token emitted by a `plan` run with pending changes; feed back via the `approval-token` input |

## Exit Codes

| Code | Meaning                                                                                    |
| ---- | ------------------------------------------------------------------------------------------ |
| `0`  | Success (for `plan`: no changes detected)                                                  |
| `1`  | Error                                                                                      |
| `2`  | Plan computed with changes pending approval, or refused at the approval gate — no mutation |

The action propagates exit code 2, so a `plan` step with pending changes fails the job unless you set `continue-on-error: true` and branch on the `result` output.

## Verifying a Deploy

The `result` output answers "did the step succeed". To confirm what actually shipped, the CLI's read commands work anywhere the same `FJALL_API_KEY` is available (including a follow-up step):

- `fjall deployments list` — your organisation's active and recent deployments: who triggered each, from where, and its current state
- `fjall releases <app>` — the app's recorded releases with their image tags: what is live now, and what a rollback can target
- `fjall status <app>` — the deployed infrastructure's current health

## Serialising Deploys

Fjall allows one active deployment per app (the deploy slot). Give every workflow that deploys the same app a shared concurrency group — the same shape `fjall ci setup` scaffolds — so overlapping runs queue instead of contending for the slot:

```yaml
concurrency:
  group: fjall-deploy-my-app
  cancel-in-progress: false
```

Keep `cancel-in-progress: false`: cancelling a deploy mid-run can interrupt an in-flight CloudFormation update.

## Deploy Modes

### Full Deploy (default)

Deploys infrastructure and application code:

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: my-app
```

### Infrastructure Only

Deploy infrastructure changes without rebuilding/deploying the application:

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: my-app
    mode: infra-only
```

### Code Only

Deploy application code using existing infrastructure (faster for code-only changes):

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: my-app
    mode: deploy-only
```

## Examples

### Plan Gate (review changes before applying)

Compute the change plan on pull requests without mutating anything:

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  id: plan
  continue-on-error: true
  with:
    target: my-app
    plan: true

- name: Fail unless plan is clean
  if: steps.plan.outputs.result == 'failure'
  run: exit 1
```

Then apply the approved plan (the token is digest-bound — the apply re-verifies that the plan has not drifted):

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: my-app
    approval-token: ${{ steps.plan.outputs.approval-token }}
```

Or approve in one shot in trusted pipelines:

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: my-app
    require-approval: true
    auto-approve: true
```

### Roll Back to a Previous Image

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: my-app
    image-tag: "sha-4f9c2ab"
```

Find valid tags with `fjall releases my-app` (each release records the image tags it shipped), from the original deploy's output, or from the app's ECR repository. An image-tag rollback re-pins application code only — it does not revert infrastructure changes or roll back database migrations.

### Build-Time Args and Secrets

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: my-app
    build-args: |
      GIT_SHA=${{ github.sha }}
      PUBLIC_API_URL=https://api.example.com
    build-secrets: |
      id=npm-token,ssm=/ci/npm-token
```

### Destroy with Force

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    command: destroy
    target: my-app
    force: true
```

### Deploy Specific Service

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: my-app
    service: api
    mode: deploy-only
```

### Deploy a Tier

A tier `target` routes to the noun-verb tier command. This runs `fjall org deploy --non-interactive --no-cascade` (the organisation root stack without its platform/account cascade):

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: organisation
    no-cascade: true
```

### Pin CLI Version

```yaml
- uses: fjall-tech/fjall-deploy-action@v3
  with:
    target: my-app
    cli-version: "7.0.0"
```

### Split Infrastructure and Code Deploys

```yaml
jobs:
  infra:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/deploy
          aws-region: us-east-2
      - uses: fjall-tech/fjall-deploy-action@v3
        with:
          target: my-app
          mode: infra-only

  code:
    needs: infra
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/deploy
          aws-region: us-east-2
      - uses: fjall-tech/fjall-deploy-action@v3
        with:
          target: my-app
          mode: deploy-only
```

### Staging to Production Promotion

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-2
      - uses: fjall-tech/fjall-deploy-action@v3
        with:
          target: my-app
          environment: staging

  production:
    needs: staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-2
      - uses: fjall-tech/fjall-deploy-action@v3
        with:
          target: my-app
          environment: production
          skip-build: true
```

## Versioning

Two pins interact, and they move together:

- **The action ref** — `fjall-tech/fjall-deploy-action@v3` is a moving major tag, repointed at the latest 3.x release on each republish; pin an exact tag (`@v3.0.0`) for maximum determinism.
- **`cli-version`** — the `fjall` CLI the action installs per run; it defaults to the action major's compatible CLI major.

Upgrade majors deliberately: a new action major defaults to a new CLI major, so bump the action ref and any explicit `cli-version` pin in the same change.

## Troubleshooting

### Blocked deploy slot

> Blocked: Jane has been deploying my-app from CI since 03/08/2026, 14:02:11 (deployment cmd0a1b2c…). View progress in the Fjall dashboard: …

Fjall allows one active deployment per app. A deploy that starts while another is in flight is refused with the message above — wait for the active deployment to finish (or cancel it from the dashboard), then retry. `fjall deployments list` shows your organisation's active deployments. Prevent the contention with a concurrency group ([Serialising Deploys](#serialising-deploys)).

### "This deployment requires a newer fjall CLI"

The Fjall API refuses deploys from a CLI older than the app's engine floor. The refusal suggests `npm install -g @fjall/cli@latest` — correct for a workstation, wrong here: the action installs the CLI per run from `cli-version`, so nothing on the runner needs upgrading. Fix it in the workflow instead:

- bump `cli-version` to the required major (or a newer exact version), or
- set `cli-version: auto` to derive the major from the app's pinned `@fjall/components-infrastructure`, or
- move to a newer action major (each action major defaults `cli-version` to its compatible CLI major).

### Authentication failures

Two different credentials can fail — the error tells you which:

- **Fjall API errors** (401/403 from the Fjall API) mean `FJALL_API_KEY` is missing, expired, or revoked. Deploy tokens live at most 90 days: mint a replacement in the dashboard (**Settings → CI/CD Tokens**) and update the repository secret.
- **AWS SDK errors** (`ExpiredToken`, `AccessDenied`, credential-chain failures naming AWS services) mean the workflow's AWS credentials are wrong — see [Authentication](#authentication).

## License

MIT

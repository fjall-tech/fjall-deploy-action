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

      - uses: fjall-tech/fjall-deploy-action@v1
        with:
          target: my-app
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: us-east-2
```

## Authentication

AWS credentials must be configured **before** this action runs. The action does not manage credentials.

### Option 1: IAM Credentials

Pass credentials as environment variables from GitHub Secrets:

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    target: my-app
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_REGION: us-east-2
```

### Option 2: AWS OIDC (Recommended)

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

      - uses: fjall-tech/fjall-deploy-action@v1
        with:
          target: my-app
```

### Option 3: Fjall OIDC

If your app is registered with Fjall, the CLI auto-detects GitHub's OIDC tokens via the `ACTIONS_ID_TOKEN_REQUEST_URL` environment variable. Just grant `id-token: write` permission and set your API key:

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: fjall-tech/fjall-deploy-action@v1
        with:
          target: my-app
        env:
          FJALL_API_KEY: ${{ secrets.FJALL_API_KEY }}
```

The `FJALL_API_KEY` environment variable takes precedence over any credentials stored on the runner (`~/.fjall/auth.json`).

## Inputs

| Input               | Required | Default  | Applies to         | Description                                                                                                                                                                                                         |
| ------------------- | -------- | -------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`           | no       | `deploy` | —                  | `deploy`, `destroy`, or `build`                                                                                                                                                                                     |
| `target`            | **yes**  | —        | all                | Application name, or a tier: `organisation`, `platform`, `account`. App targets run the app; tier targets route to the noun-verb tier command (`fjall org\|platform\|account deploy\|destroy`). `build` is app-only |
| `service`           | no       | —        | deploy, build      | Specific service name                                                                                                                                                                                               |
| `mode`              | no       | `full`   | deploy             | `full`, `infra-only`, or `deploy-only`                                                                                                                                                                              |
| `environment`       | no       | —        | deploy             | Target environment (maps to the `-e` flag)                                                                                                                                                                          |
| `deploy-target`     | no       | —        | app targets        | Deploy to a specific `fjall target list` target, e.g. `production-use1`; maps to `--target` (the credential where), distinct from `environment`                                                                     |
| `verbose`           | no       | `false`  | all                | Enable verbose output                                                                                                                                                                                               |
| `skip-build`        | no       | `false`  | deploy             | Skip Docker build (use with `mode: deploy-only` when the image is already pushed)                                                                                                                                   |
| `skip-migrations`   | no       | `false`  | deploy             | Skip database migrations during this deployment                                                                                                                                                                     |
| `no-cascade`        | no       | `false`  | org deploy/destroy | Skip the platform/account cascade around an `organisation` deploy or destroy (rejected for any other target)                                                                                                        |
| `region`            | no       | —        | deploy             | Deploy to a specific region within the target's account                                                                                                                                                             |
| `image-tag`         | no       | —        | deploy             | Roll forward/back to an existing image tag (skips build, implies `deploy-only`)                                                                                                                                     |
| `plan`              | no       | `false`  | deploy             | Compute and print the change plan, then stop before any mutation                                                                                                                                                    |
| `require-approval`  | no       | `false`  | deploy             | Engage the approval gate: refuse to mutate unless approved via `auto-approve` or `approval-token`                                                                                                                   |
| `auto-approve`      | no       | `false`  | deploy             | Approve the computed plan without prompting (use with `require-approval`)                                                                                                                                           |
| `approval-token`    | no       | —        | deploy             | Resume an approved plan from a prior `plan` run (digest re-verified before applying)                                                                                                                                |
| `build-args`        | no       | —        | deploy, build      | Newline-separated `KEY=VALUE` pairs, each passed via `--build-arg`                                                                                                                                                  |
| `build-secrets`     | no       | —        | deploy, build      | Newline-separated `id=ID,ssm=PATH` (or `secretsManager=NAME` \| `env=VAR`) tokens (`--build-secret`)                                                                                                                |
| `force`             | no       | `false`  | deploy, destroy    | Deploy: redeploy all stacks even when unchanged. Destroy: skip destruction confirmation                                                                                                                             |
| `cli-version`       | no       | `4`      | —                  | Pin the published `fjall` CLI version. Defaults to this action's compatible major (floats across `4`.x, never crossing into the next major); set `latest` to always install the newest published release            |
| `working-directory` | no       | `.`      | —                  | Directory containing `fjall-config.json`                                                                                                                                                                            |

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

## Deploy Modes

### Full Deploy (default)

Deploys infrastructure and application code:

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    target: my-app
```

### Infrastructure Only

Deploy infrastructure changes without rebuilding/deploying the application:

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    target: my-app
    mode: infra-only
```

### Code Only

Deploy application code using existing infrastructure (faster for code-only changes):

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    target: my-app
    mode: deploy-only
```

## Examples

### Plan Gate (review changes before applying)

Compute the change plan on pull requests without mutating anything:

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
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
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    target: my-app
    approval-token: ${{ steps.plan.outputs.approval-token }}
```

Or approve in one shot in trusted pipelines:

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    target: my-app
    require-approval: true
    auto-approve: true
```

### Roll Back to a Previous Image

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    target: my-app
    image-tag: "sha-4f9c2ab"
```

### Build-Time Args and Secrets

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
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
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    command: destroy
    target: my-app
    force: true
```

### Deploy Specific Service

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    target: my-app
    service: api
    mode: deploy-only
```

### Deploy a Tier

A tier `target` routes to the noun-verb tier command. This runs `fjall org deploy --non-interactive --no-cascade` (the organisation root stack without its platform/account cascade):

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    target: organisation
    no-cascade: true
```

### Pin CLI Version

```yaml
- uses: fjall-tech/fjall-deploy-action@v1
  with:
    target: my-app
    cli-version: "2.23.1"
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
      - uses: fjall-tech/fjall-deploy-action@v1
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
      - uses: fjall-tech/fjall-deploy-action@v1
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
      - uses: fjall-tech/fjall-deploy-action@v1
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
      - uses: fjall-tech/fjall-deploy-action@v1
        with:
          target: my-app
          environment: production
          skip-build: true
```

## License

MIT

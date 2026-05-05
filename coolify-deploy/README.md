# Coolify Deploy

Trigger a deployment on a [Coolify](https://coolify.io) instance (e.g. `https://hostingcloud.relybytes.com`) from a GitHub Actions workflow.

Supports two trigger modes:

- **API mode** — `coolify_url` + `api_token` + `uuid` (recommended: lets the action poll the deployment status)
- **Webhook mode** — `webhook_url` (+ optional bearer secret); status polling needs the API credentials too

## Features

- **API or webhook trigger** with the same interface
- **Wait-for-completion** with configurable timeout and polling interval
- **Force rebuild**, **tag** and **PR preview** support
- **Strict failure handling** with optional warning-only mode

## Usage

### API mode (recommended)

```yaml
- uses: relybytes/actions/coolify-deploy@v1
  with:
    coolify_url: https://hostingcloud.relybytes.com
    api_token: ${{ secrets.COOLIFY_TOKEN }}
    uuid: ${{ secrets.COOLIFY_APP_UUID }}
    wait: "true"
    wait_timeout: "600"
```

### Webhook mode

```yaml
- uses: relybytes/actions/coolify-deploy@v1
  with:
    webhook_url: ${{ secrets.COOLIFY_DEPLOY_WEBHOOK }}
    webhook_secret: ${{ secrets.COOLIFY_WEBHOOK_TOKEN }}
    wait: "false"
```

## Inputs

| Input            | Required | Default | Description                                                              |
| ---------------- | -------- | ------- | ------------------------------------------------------------------------ |
| `coolify_url`    | API mode | —       | Base URL of the Coolify instance (e.g. `https://hostingcloud.relybytes.com`) |
| `api_token`      | API mode | —       | Coolify API bearer token                                                 |
| `uuid`           | API mode | —       | UUID of the application/service to deploy                                |
| `webhook_url`    | webhook  | —       | Full Coolify deploy webhook URL (alternative to API mode)                |
| `webhook_secret` | no       | —       | Optional bearer token for the deploy webhook                             |
| `force`          | no       | `false` | Force rebuild without cache                                              |
| `tag`            | no       | —       | Optional Git tag to deploy                                               |
| `pr_number`      | no       | —       | Optional PR number for preview deployments                               |
| `wait`           | no       | `true`  | Wait for the deployment to finish                                        |
| `wait_timeout`   | no       | `600`   | Wait timeout in seconds                                                  |
| `poll_interval`  | no       | `10`    | Polling interval in seconds                                              |
| `fail_on_error`  | no       | `true`  | Fail the workflow on `failed`/`cancelled`/`timeout`                      |

## Outputs

| Output              | Description                                            |
| ------------------- | ------------------------------------------------------ |
| `deployment_uuid`   | UUID of the triggered deployment (when API responds with one) |
| `deployment_status` | Final status: `finished`, `failed`, `cancelled`, `timeout`, `unknown` |
| `deployment_time`   | UTC timestamp of the trigger                           |

## Examples

### Deploy on push to main

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Coolify deploy
        uses: relybytes/actions/coolify-deploy@v1
        with:
          coolify_url: https://hostingcloud.relybytes.com
          api_token: ${{ secrets.COOLIFY_TOKEN }}
          uuid: ${{ secrets.COOLIFY_APP_UUID }}
          force: "true"
```

### Deploy multiple services in parallel

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service:
          - { name: api, uuid: "${{ secrets.COOLIFY_API_UUID }}" }
          - { name: web, uuid: "${{ secrets.COOLIFY_WEB_UUID }}" }
    steps:
      - name: Deploy ${{ matrix.service.name }}
        uses: relybytes/actions/coolify-deploy@v1
        with:
          coolify_url: https://hostingcloud.relybytes.com
          api_token: ${{ secrets.COOLIFY_TOKEN }}
          uuid: ${{ matrix.service.uuid }}
```

### Trigger a tag-based deployment

```yaml
on:
  push:
    tags: ["v*"]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: relybytes/actions/coolify-deploy@v1
        with:
          coolify_url: https://hostingcloud.relybytes.com
          api_token: ${{ secrets.COOLIFY_TOKEN }}
          uuid: ${{ secrets.COOLIFY_APP_UUID }}
          tag: ${{ github.ref_name }}
          wait_timeout: "900"
```

### Preview deployment for pull requests

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: relybytes/actions/coolify-deploy@v1
        with:
          coolify_url: https://hostingcloud.relybytes.com
          api_token: ${{ secrets.COOLIFY_TOKEN }}
          uuid: ${{ secrets.COOLIFY_APP_UUID }}
          pr_number: ${{ github.event.pull_request.number }}
```

### Fire-and-forget via webhook

```yaml
- uses: relybytes/actions/coolify-deploy@v1
  with:
    webhook_url: ${{ secrets.COOLIFY_DEPLOY_WEBHOOK }}
    wait: "false"
```

## How to get the inputs

1. **`coolify_url`** — the base URL of your Coolify dashboard (no trailing path).
2. **`api_token`** — generate it from `Profile -> Keys & Tokens -> API tokens` in Coolify.
3. **`uuid`** — open your application in Coolify; the UUID appears in the URL (`/applications/<uuid>`) and in the resource details.
4. **`webhook_url`** — open the application's `Webhooks` (or `Deployments`) section to copy the deploy webhook.

Store the token, UUID and any webhook URL as repository or organization **secrets** — never hard-code them in the workflow.

## Notes

- The action uses `jq` (preinstalled on `ubuntu-latest` runners) to parse Coolify responses. On runners without `jq`, the trigger still works but the deployment UUID and status polling are skipped.
- In webhook mode the action cannot poll status unless you also pass `coolify_url` + `api_token`. Without those, it returns `deployment_status=unknown`.
- The poll considers the deployment terminal on these statuses: `finished`/`success`/`succeeded` (success), `failed`/`error`/`cancelled`/`canceled` (failure).

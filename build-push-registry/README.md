# Build & Push to Registry

Build a Docker image and push it to **any container registry** using a fixed naming convention:

```
{registry}/{image_name}-{suffix}:YYYY-MM-DD.shortsha
```

When no credentials are passed the action falls back to **GitHub Container Registry** (`ghcr.io`) authenticated with `GITHUB_TOKEN`, so a zero-config build on the current repo "just works".

The environment marker is appended as a **suffix on the image repository**, not as a prefix on the tag — so each environment gets its own image with a clean `:latest` tag.

The `suffix` is automatically derived from the current Git ref (branch / tag / PR) and can be overridden, just like `image_name`, `version`, and the registry credentials.

## Naming convention

- **Image repository**: `{base_name}-{suffix}` (e.g. `relybytes/myapp-prod`)
- **Tag**: `YYYY-MM-DD.shortsha` (UTC date + first 7 chars of `github.sha`)
- **Examples**:
  - `ghcr.io/relybytes/myapp-prod:2026-05-05.a464688`
  - `registry.example.com/team/myapp-dev:2026-05-05.a464688`
  - `docker.io/relybytes/myapp-pr-42:2026-05-05.a464688`

### Default suffix mapping

| Source ref / event   | Suffix          |
| -------------------- | --------------- |
| `main`, `master`     | `prod`          |
| `develop`, `dev`     | `dev`           |
| `staging`            | `staging`       |
| `release/*`          | `rc`            |
| `hotfix/*`           | `hotfix`        |
| `feature/*`          | `feat`          |
| Pull request         | `pr-{number}`   |
| Git tag push         | `release`       |
| Other branch         | sanitized branch name (lowercase, `/` → `-`) |

Override with the `suffix` input when the default doesn't fit, e.g. `suffix: canary`. Pass `suffix: none` to disable the suffix entirely.

### Extra tags published automatically

All extra tags share the same suffixed repository:

- `:latest` — only on `main`/`master` non-PR builds (toggle via `latest`)
- The Git tag itself (e.g. `v1.2.3`) — on tag push events
- Anything passed via `additional_tags`

## Usage

### Default — push to GHCR for the current repo

```yaml
permissions:
  contents: read
  packages: write   # required for the default GITHUB_TOKEN to push to ghcr.io

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: relybytes/actions/build-push-registry@v1
```

A push to `main` produces:

```
ghcr.io/<owner>/<repo>-prod:2026-05-05.a464688
ghcr.io/<owner>/<repo>-prod:latest
```

### Push to a different registry

```yaml
- uses: relybytes/actions/build-push-registry@v1
  with:
    registry: registry.example.com
    username: ${{ secrets.REGISTRY_USER }}
    password: ${{ secrets.REGISTRY_PASS }}
    image_name: team/myapp
```

### Push to Docker Hub

```yaml
- uses: relybytes/actions/build-push-registry@v1
  with:
    registry: docker.io
    username: ${{ secrets.DOCKERHUB_USER }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
    image_name: relybytes/myapp
```

## Inputs

| Input             | Required | Default                  | Description                                                          |
| ----------------- | -------- | ------------------------ | -------------------------------------------------------------------- |
| `registry`        | no       | `ghcr.io`                | Container registry host                                              |
| `username`        | no       | `${{ github.actor }}`    | Registry username                                                    |
| `password`        | no       | `${{ github.token }}`    | Registry password / access token                                     |
| `image_name`      | no       | repo (lowercased)        | Base image name. The suffix is appended as `-{suffix}`.              |
| `dockerfile`      | no       | `Dockerfile`             | Path to the Dockerfile (relative to context)                         |
| `context`         | no       | `.`                      | Build context directory                                              |
| `suffix`          | no       | derived from ref         | Override the suffix. Use `none` to disable.                          |
| `version`         | no       | auto-generated           | Override the tag (skips the `YYYY-MM-DD.shortsha` generation)        |
| `platforms`       | no       | `linux/amd64`            | Comma-separated target platforms                                     |
| `build_args`      | no       | —                        | Newline-separated build args (`KEY=VALUE`)                           |
| `target`          | no       | —                        | Target build stage for multi-stage Dockerfiles                       |
| `labels`          | no       | OCI auto-labels          | Newline-separated labels (override the auto-generated set)           |
| `push`            | no       | `true`                   | Push to the registry (set to `false` for PR validation)              |
| `additional_tags` | no       | —                        | Comma-separated additional tags on the suffixed image                |
| `latest`          | no       | `auto`                   | `true` / `false` / `auto`. `auto` enables `:latest` only on main/master non-PR builds |
| `cache`           | no       | `true`                   | Enable BuildKit inline cache                                         |
| `no_cache`        | no       | `false`                  | Disable build cache entirely                                         |

## Outputs

| Output             | Description                                                |
| ------------------ | ---------------------------------------------------------- |
| `image`            | Full image reference (`registry/name-suffix:version`)      |
| `image_repository` | Repository portion (`registry/name-suffix`) without tag    |
| `version`          | Resolved tag (`YYYY-MM-DD.shortsha` or override)           |
| `suffix`           | Resolved environment suffix                                |
| `branch`           | Branch used to derive the suffix                           |
| `tags`             | All tags published (newline-separated)                     |
| `digest`           | Image digest after push                                    |
| `build_time`       | UTC timestamp of the build                                 |

## Examples

### Multi-arch with build args

```yaml
- uses: relybytes/actions/build-push-registry@v1
  with:
    platforms: linux/amd64,linux/arm64
    build_args: |
      NODE_ENV=production
      VERSION=${{ github.ref_name }}
```

### Custom suffix

```yaml
- uses: relybytes/actions/build-push-registry@v1
  with:
    suffix: canary
```

Produces: `ghcr.io/<owner>/<repo>-canary:2026-05-05.a464688`

### Override the image name (private registry, custom path)

```yaml
- uses: relybytes/actions/build-push-registry@v1
  with:
    registry: registry.example.com
    username: ${{ secrets.REGISTRY_USER }}
    password: ${{ secrets.REGISTRY_PASS }}
    image_name: platform/myapp
```

Produces: `registry.example.com/platform/myapp-prod:2026-05-05.a464688`

### Disable suffix (single-image repo)

```yaml
- uses: relybytes/actions/build-push-registry@v1
  with:
    suffix: none
```

Produces: `ghcr.io/<owner>/<repo>:2026-05-05.a464688`

### PR build without push (validation only)

```yaml
on:
  pull_request:

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: relybytes/actions/build-push-registry@v1
        with:
          push: "false"
```

### Pipe into a deploy step

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Build & push
        id: image
        uses: relybytes/actions/build-push-registry@v1

      - name: Deploy to Coolify
        uses: relybytes/actions/coolify-deploy@v1
        with:
          coolify_url: https://hostingcloud.relybytes.com
          api_token: ${{ secrets.COOLIFY_TOKEN }}
          uuid: ${{ secrets.COOLIFY_APP_UUID }}
          tag: ${{ steps.image.outputs.version }}

      - name: Deploy to Kubernetes
        uses: relybytes/actions/k8s-deploy@v1
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
          namespace: production
          manifests: k8s/
          image: ${{ steps.image.outputs.image }}
          image_placeholder: __IMAGE__
```

## Permissions

To push to `ghcr.io` with the default `GITHUB_TOKEN`, the workflow must declare:

```yaml
permissions:
  contents: read
  packages: write
```

For cross-repo / org-level pushes use a Personal Access Token (classic) with `write:packages` and pass it via `password`. For external registries (Docker Hub, AWS ECR public, private registries) use the registry's own credentials.

## Notes

- Image names are forced to **lowercase** because GHCR rejects uppercase and Docker Hub strongly prefers it; this is safe across registries.
- Each environment gets its **own image repository** (e.g. `myapp-prod`, `myapp-dev`, `myapp-pr-42`). This keeps `:latest` clean and lets you set per-environment retention/visibility on registries that support it.
- The `org.opencontainers.image.{source,revision,created,version}` labels are added automatically when `labels` is empty — pass your own to opt out.
- `latest=auto` deliberately publishes `:latest` only on main/master non-PR builds to avoid polluting the registry.

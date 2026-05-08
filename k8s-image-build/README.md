# Kubernetes Image Build

Build a Docker image and push it to a container registry, ready to be referenced by a Kubernetes Deployment.

Pairs with [`k8s-deploy`](../k8s-deploy/) to form a complete build-and-deploy pipeline.

## Features

- **Multi-platform builds** with Docker Buildx (amd64, arm64, etc.)
- **Multi-tag publishing** with one primary tag plus arbitrary additional tags
- **Build-time arguments and OCI labels** as multi-line inputs
- **Inline BuildKit cache** for faster repeated builds
- **Optional push** so the same action can be used for PR-only builds
- **Image digest output** for downstream pinning

## Usage

```yaml
- name: Build and push image
  id: image
  uses: relybytes/actions/k8s-image-build@v1
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
    image_name: myorg/api
    image_tag: ${{ github.sha }}
    additional_tags: latest
    dockerfile: Dockerfile
    context: .
```

## Inputs

| Input             | Required | Default        | Description                                                              |
| ----------------- | -------- | -------------- | ------------------------------------------------------------------------ |
| `registry`        | yes      | —              | Container registry URL (e.g. `docker.io`, `ghcr.io`)                     |
| `username`        | yes      | —              | Registry username                                                        |
| `password`        | yes      | —              | Registry password or access token                                        |
| `image_name`      | yes      | —              | Image name (e.g. `myorg/myapp`)                                          |
| `image_tag`       | no       | short SHA      | Primary image tag                                                        |
| `additional_tags` | no       | —              | Comma-separated additional tags (e.g. `latest,stable`)                   |
| `dockerfile`      | no       | `Dockerfile`   | Path to the Dockerfile (relative to context)                             |
| `context`         | no       | `.`            | Build context directory                                                  |
| `build_args`      | no       | —              | Newline-separated build arguments (`KEY=VALUE`)                          |
| `target`          | no       | —              | Target build stage for multi-stage Dockerfiles                           |
| `platforms`       | no       | `linux/amd64`  | Comma-separated target platforms                                         |
| `push`            | no       | `true`         | Push the built image to the registry                                     |
| `cache`           | no       | `true`         | Enable BuildKit inline cache                                             |
| `no_cache`        | no       | `false`        | Disable build cache entirely                                             |
| `labels`          | no       | —              | Newline-separated OCI labels (`KEY=VALUE`)                               |

## Outputs

| Output       | Description                              |
| ------------ | ---------------------------------------- |
| `image`      | Full image reference (`registry/name:tag`) |
| `image_tag`  | Resolved primary tag                     |
| `digest`     | Image digest after push                  |
| `build_time` | UTC timestamp of the build               |

## Examples

### Multi-arch with build args

```yaml
- uses: relybytes/actions/k8s-image-build@v1
  with:
    registry: registry.example.com
    username: ${{ secrets.REGISTRY_USER }}
    password: ${{ secrets.REGISTRY_PASS }}
    image_name: platform/api
    image_tag: ${{ github.ref_name }}
    platforms: linux/amd64,linux/arm64
    build_args: |
      NODE_ENV=production
      VERSION=${{ github.ref_name }}
    labels: |
      org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
      org.opencontainers.image.revision=${{ github.sha }}
```

### Build only (no push) for PR validation

```yaml
- uses: relybytes/actions/k8s-image-build@v1
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
    image_name: myorg/api
    push: "false"
```

### Pipe into k8s-deploy

```yaml
- name: Build image
  id: image
  uses: relybytes/actions/k8s-image-build@v1
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
    image_name: ${{ github.repository }}
    image_tag: ${{ github.sha }}

- name: Deploy to Kubernetes
  uses: relybytes/actions/k8s-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG }}
    namespace: production
    manifests: k8s/
    image: ${{ steps.image.outputs.image }}
    image_placeholder: __IMAGE__
```

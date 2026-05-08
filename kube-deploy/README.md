# Kubernetes Deploy

Deploy services to a Kubernetes cluster by applying YAML manifests, with namespace management, generic placeholder replacement and rollout monitoring.

Pairs with [`k8s-image-build`](../k8s-image-build/) for a full build-and-deploy pipeline.

## Features

- **Kubeconfig from secret**: raw YAML or base64-encoded
- **Automatic namespace creation** when missing
- **Namespace-scoped kubeconfig support** for restricted CI/CD ServiceAccounts
- **Manifest pre-processing**:
  - generic `KEY=VALUE` placeholder replacement
  - optional `envsubst`
- **Kustomize support** with `kubectl apply -k`
- **In-place image updates** with `kubectl set image`
- **Rollout wait** for Deployments / StatefulSets / DaemonSets
- **Dry-run modes**: `client` / `server`
- **Prune** for declarative cleanup of removed resources
- **Optional kubectl version pinning**

## Usage

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: k8s/
    replacements: |
      __IMAGE__=ghcr.io/myorg/api:${{ github.sha }}
      __VERSION__=${{ github.sha }}
      __APP_ENV__=production
      __HOST__=api.example.com
    wait: "true"
    wait_timeout: 5m
```

## Kubeconfig secret

The `kubeconfig` input accepts either:

- raw kubeconfig YAML
- base64-encoded kubeconfig YAML

Recommended approach:

```bash
base64 -w 0 kubeconfig.yaml
```

Save the output as a GitHub secret, for example:

```text
KUBECONFIG_B64
```

Then use:

```yaml
kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
```

If the kubeconfig is generated for a specific namespace and ServiceAccount, for example:

```yaml
contexts:
  - name: rely-platform
    context:
      cluster: k3s
      namespace: rely-platform
      user: rely-platform-deploy
```

use:

```yaml
create_namespace: "false"
```

Namespace-scoped ServiceAccounts usually cannot create namespaces because that requires cluster-level permissions.

## Inputs

| Input              | Required | Default     | Description                                                                            |
| ------------------ | -------- | ----------- | -------------------------------------------------------------------------------------- |
| `kubeconfig`       | yes      | —           | Kubeconfig content, raw YAML or base64-encoded YAML                                    |
| `context`          | no       | —           | Kubeconfig context to switch to                                                        |
| `namespace`        | yes      | —           | Target Kubernetes namespace                                                            |
| `create_namespace` | no       | `true`      | Create the namespace if missing. Requires cluster-level permissions                    |
| `manifests`        | yes      | —           | File(s) or directory of manifests, newline or comma separated                          |
| `kustomize`        | no       | `false`     | Apply with `kubectl apply -k`                                                          |
| `replacements`     | no       | —           | Newline-separated placeholder replacements in `KEY=VALUE` format                       |
| `set_image`        | no       | —           | Newline-separated entries in `resource=container=image` format for `kubectl set image` |
| `env_substitution` | no       | `false`     | Run `envsubst` over manifests before apply                                             |
| `validate`         | no       | `true`      | Pass `--validate=true` to `kubectl apply`                                              |
| `dry_run`          | no       | `none`      | One of `none`, `client`, `server`                                                      |
| `prune`            | no       | `false`     | Enable `kubectl apply --prune`                                                         |
| `prune_label`      | no       | —           | Label selector required when `prune` is `true`                                         |
| `wait`             | no       | `true`      | Wait for rollout to finish                                                             |
| `wait_resources`   | no       | auto-detect | Resources to wait on, for example `deployment/api`                                     |
| `wait_timeout`     | no       | `300s`      | Timeout for rollout wait                                                               |
| `kubectl_version`  | no       | latest      | Pin kubectl version, for example `v1.30.0`                                             |

## Outputs

| Output              | Description                      |
| ------------------- | -------------------------------- |
| `applied_resources` | Resources that were applied      |
| `rollout_status`    | `success`, `failed` or `skipped` |
| `deployment_time`   | UTC timestamp of the apply       |

## Placeholder replacement

Use `replacements` for all dynamic values: image, version, environment, host, domain, labels, annotations or runtime configuration.

Workflow:

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: k8s/
    replacements: |
      __IMAGE__=ghcr.io/myorg/api:${{ github.sha }}
      __VERSION__=${{ github.sha }}
      __APP_ENV__=production
      __HOST__=api.example.com
```

Manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app.kubernetes.io/version: "__VERSION__"
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/version: "__VERSION__"
    spec:
      containers:
        - name: app
          image: "__IMAGE__"
          env:
            - name: APP_ENV
              value: "__APP_ENV__"
            - name: APP_VERSION
              value: "__VERSION__"
```

Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
spec:
  ingressClassName: nginx
  rules:
    - host: "__HOST__"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

Replacement format:

```text
PLACEHOLDER=value
```

Empty lines and lines starting with `#` are ignored:

```yaml
replacements: |
  # Release metadata
  __IMAGE__=ghcr.io/myorg/api:${{ github.sha }}
  __VERSION__=${{ github.sha }}
  __RUN_ID__=${{ github.run_id }}

  # Environment
  __APP_ENV__=production
```

Values may contain `=`:

```yaml
replacements: |
  __DATABASE_URL__=postgres://user:password@postgres:5432/app?sslmode=require
```

## envsubst

Use `env_substitution: "true"` when your manifests use shell-style environment placeholders such as `${APP_ENV}` or `${HOST}`.

Workflow:

```yaml
- uses: relybytes/actions/kube-deploy@v1
  env:
    APP_ENV: production
    HOST: api.example.com
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: k8s/
    env_substitution: "true"
```

Manifest:

```yaml
env:
  - name: APP_ENV
    value: "${APP_ENV}"
```

Ingress:

```yaml
rules:
  - host: "${HOST}"
```

## Examples

### Apply a directory of manifests with generic replacements

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: api-prod
    create_namespace: "false"
    manifests: |
      k8s/base/
      k8s/overlays/prod/
    replacements: |
      __IMAGE__=ghcr.io/myorg/api:${{ github.sha }}
      __VERSION__=${{ github.sha }}
      __APP_ENV__=production
      __HOST__=api.example.com
```

A manifest using the placeholders:

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          image: "__IMAGE__"
          env:
            - name: APP_VERSION
              value: "__VERSION__"
```

### Apply manifests with replicas, host and environment values

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: api-prod
    create_namespace: "false"
    manifests: k8s/
    replacements: |
      __IMAGE__=ghcr.io/myorg/api:${{ github.sha }}
      __VERSION__=${{ github.sha }}
      __APP_ENV__=production
      __HOST__=api.example.com
      __REPLICAS__=2
```

Manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: api
    version: "__VERSION__"
spec:
  replicas: __REPLICAS__
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
        version: "__VERSION__"
    spec:
      containers:
        - name: app
          image: "__IMAGE__"
          env:
            - name: APP_ENV
              value: "__APP_ENV__"
```

### Update an existing Deployment with kubectl set image

Use `set_image` when you prefer to update container images after applying manifests.

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: k8s/deployment.yml
    set_image: |
      deployment/api=app=ghcr.io/myorg/api:${{ github.sha }}
      deployment/worker=worker=ghcr.io/myorg/worker:${{ github.sha }}
```

This runs:

```bash
kubectl --namespace production set image deployment/api app=ghcr.io/myorg/api:<sha>
kubectl --namespace production set image deployment/worker worker=ghcr.io/myorg/worker:<sha>
```

### Kustomize with environment-specific overlay

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: staging
    create_namespace: "false"
    manifests: k8s/overlays/staging
    kustomize: "true"
```

When `kustomize` is `true`, manifests are passed directly to:

```bash
kubectl apply -k <path>
```

Placeholder replacement and `envsubst` are skipped because Kustomize has its own patching, image and overlay model.

### Server-side dry run

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: k8s/
    dry_run: server
    wait: "false"
```

### Client-side dry run

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: k8s/
    dry_run: client
    wait: "false"
```

### Multiple manifest paths

Newline-separated:

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: |
      k8s/config/
      k8s/secrets/
      k8s/app/
      k8s/ingress.yaml
```

Comma-separated:

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: k8s/config/,k8s/app/,k8s/ingress.yaml
```

### Explicit rollout resources

By default, the action waits for rollout-capable resources detected from `kubectl apply -o name`.

You can also pass them manually:

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: k8s/
    wait_resources: |
      deployment/api
      deployment/worker
      statefulset/queue
    wait_timeout: 5m
```

### Prune removed resources

Use prune only when all resources managed by this action share a safe label.

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: k8s/
    prune: "true"
    prune_label: app.kubernetes.io/managed-by=relybytes-actions
```

Every managed manifest should include the same label:

```yaml
metadata:
  labels:
    app.kubernetes.io/managed-by: relybytes-actions
```

### Pin kubectl version

```yaml
- uses: relybytes/actions/kube-deploy@v1
  with:
    kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
    namespace: production
    create_namespace: "false"
    manifests: k8s/
    kubectl_version: v1.30.0
```

## Full CI/CD pipeline

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Build image
        id: image
        uses: relybytes/actions/k8s-image-build@v1
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          image_name: ${{ github.repository }}
          image_tag: ${{ github.sha }}
          additional_tags: latest

      - name: Deploy to cluster
        uses: relybytes/actions/kube-deploy@v1
        with:
          kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
          namespace: production
          create_namespace: "false"
          manifests: k8s/
          replacements: |
            __IMAGE__=${{ steps.image.outputs.image }}
            __VERSION__=${{ github.sha }}
            __APP_ENV__=production
            __HOST__=api.example.com
          wait: "true"
          wait_timeout: 5m
```

## Example for Rely Platform on K3s

Example using a namespace-scoped kubeconfig generated for:

```text
namespace: rely-platform
serviceAccount: rely-platform-deploy
```

```yaml
name: Deploy Rely Platform

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ghcr.io/${{ github.repository }}
  IMAGE_TAG: ${{ github.sha }}

jobs:
  deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u "${{ github.actor }}" --password-stdin

      - name: Build image
        run: |
          docker build -t "$IMAGE_NAME:$IMAGE_TAG" .
          docker push "$IMAGE_NAME:$IMAGE_TAG"

      - name: Deploy to K3s
        uses: relybytes/actions/kube-deploy@v1
        with:
          kubeconfig: ${{ secrets.KUBECONFIG_B64 }}
          namespace: rely-platform
          create_namespace: "false"
          manifests: k8s/
          replacements: |
            __IMAGE__=${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
            __VERSION__=${{ github.sha }}
            __APP_ENV__=production
            __HOST__=rely-platform.relybytes.com
          wait: "true"
          wait_timeout: 300s
```

Example `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rely-platform-api
  labels:
    app: rely-platform-api
    app.kubernetes.io/managed-by: relybytes-actions
    app.kubernetes.io/version: "__VERSION__"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: rely-platform-api
  template:
    metadata:
      labels:
        app: rely-platform-api
        app.kubernetes.io/version: "__VERSION__"
    spec:
      containers:
        - name: api
          image: "__IMAGE__"
          ports:
            - containerPort: 3000
          env:
            - name: APP_ENV
              value: "__APP_ENV__"
            - name: APP_VERSION
              value: "__VERSION__"
```

Example `k8s/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rely-platform-api
  labels:
    app.kubernetes.io/managed-by: relybytes-actions
spec:
  selector:
    app: rely-platform-api
  ports:
    - name: http
      port: 80
      targetPort: 3000
  type: ClusterIP
```

Example `k8s/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rely-platform-api
  labels:
    app.kubernetes.io/managed-by: relybytes-actions
spec:
  ingressClassName: nginx
  rules:
    - host: "__HOST__"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rely-platform-api
                port:
                  number: 80
```

## Repository structure

Recommended structure:

```text
.
├── .github
│   └── workflows
│       └── deploy.yml
├── k8s
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
├── Dockerfile
└── README.md
```

For multiple environments:

```text
.
├── k8s
│   ├── base
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── overlays
│       ├── staging
│       └── production
```

## Notes

- The kubeconfig is written to `~/.kube/config` with `0600` permissions and removed at the end of the run.
- Prefer storing kubeconfig as base64 in GitHub secrets to avoid multiline formatting issues.
- Namespace-scoped kubeconfigs usually cannot create namespaces. Use `create_namespace: "false"` when using limited ServiceAccounts.
- Auto-detected wait targets cover `deployment`, `statefulset` and `daemonset` resources extracted from the apply output.
- For other workloads, pass rollout targets explicitly via `wait_resources`.
- When `prune` is enabled you **must** provide a `prune_label`; the action refuses to run otherwise to avoid accidentally pruning unrelated resources.
- When `kustomize` is `true`, placeholder replacement and `envsubst` are skipped.
- Use `dry_run: server` to validate manifests against the Kubernetes API before applying for real.
- Use generic `replacements` for simple placeholder substitution, and prefer Kubernetes `Secret` resources or external secret managers for sensitive values.

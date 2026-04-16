# Kubernetes ONVM Release Action

A reusable GitHub Action for releasing Kubernetes manifests to a remote server over SSH.

## Features

- SSH password-based deployment with `sshpass`
- Manifest directory passed as an input
- Optional registry secret refresh
- Manifest patching for image reference and pull policy
- Deployment restart and rollout wait
- Remote cleanup after release

## Usage

```yaml
- uses: relybytes/actions/kubernetes-onvm-release@v1
  with:
    ssh_host: ${{ vars.SSH_HOST }}
    ssh_user: ${{ vars.SSH_USER }}
    ssh_password: ${{ secrets.SSH_PASSWORD }}
    manifests_dir: k8s/prod
    namespace: my-namespace
    deployment_name: my-app
    pod_label_selector: app=my-app
    image_reference: ghcr.io/my-org/my-app:${{ github.sha }}
    registry_username: ${{ github.actor }}
    registry_password: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Input | Required | Default |
|-------|----------|---------|
| `ssh_host` | Yes | - |
| `ssh_port` | No | `22` |
| `ssh_user` | Yes | - |
| `ssh_password` | Yes | - |
| `manifests_dir` | Yes | - |
| `remote_dir` | No | `/tmp/k8s-release` |
| `namespace` | Yes | - |
| `deployment_name` | Yes | - |
| `pod_label_selector` | Yes | - |
| `deployment_file` | No | `app/01-deployment.yaml` |
| `image_reference` | Yes | - |
| `image_pull_policy` | No | `Always` |
| `create_registry_secret` | No | `true` |
| `registry_secret_name` | No | `registry-secret` |
| `registry_server` | No | `ghcr.io` |
| `registry_username` | No | - |
| `registry_password` | No | - |
| `registry_email` | No | `noreply@example.com` |
| `restart_deployment` | No | `true` |
| `wait_timeout` | No | `120s` |
| `cleanup_remote` | No | `true` |

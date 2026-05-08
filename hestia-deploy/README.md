# HestiaCP Deploy Action

[![GitHub release](https://img.shields.io/github/release/relybytes/actions.svg)](https://github.com/relybytes/actions/releases)
[![GitHub marketplace](https://img.shields.io/badge/marketplace-hestia--deploy-blue?logo=github)](https://github.com/marketplace/actions/hestiacp-deploy)

A production-ready GitHub Action for deploying static sites to HestiaCP servers via SSH with rsync, backup support, and healthcheck capabilities.

## Features

- **Secure SSH deployment** with key-based authentication
- **Backup before deployment** with configurable retention
- **Rsync-based file transfer** with delete option
- **Post-deployment healthcheck** to verify deployment success
- **Comprehensive logging** and error handling
- **Production-ready** with input validation

## Usage

### Basic Usage

```yaml
- name: Deploy to HestiaCP
  uses: relybytes/actions/hestia-deploy@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    target: /home/user/web/domain.com/public_html/
```

### Advanced Usage

```yaml
- name: Deploy to HestiaCP
  uses: relybytes/actions/hestia-deploy@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    port: "22"
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: dist/
    target: /home/user/web/domain.com/public_html/
    delete: "true"
    backup: "true"
    backup_retention: "7"
    healthcheck_url: https://domain.com/health
    healthcheck_timeout: "30"
    healthcheck_retries: "3"
    rsync_options: '-avz --progress --exclude="*.log"'
    ssh_options: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
```

## Inputs

| Input                 | Description                          | Required | Default                                                       |
| --------------------- | ------------------------------------ | -------- | ------------------------------------------------------------- |
| `host`                | SSH host address                     | Yes      | -                                                             |
| `port`                | SSH port                             | No       | `22`                                                          |
| `username`            | SSH username                         | Yes      | -                                                             |
| `private_key`         | SSH private key                      | Yes      | -                                                             |
| `source`              | Source directory to deploy           | No       | `dist/`                                                       |
| `target`              | Target directory on server           | Yes      | -                                                             |
| `delete`              | Delete files in target not in source | No       | `false`                                                       |
| `backup`              | Create backup before deployment      | No       | `true`                                                        |
| `backup_retention`    | Number of backups to keep            | No       | `7`                                                           |
| `healthcheck_url`     | URL to check after deployment        | No       | -                                                             |
| `healthcheck_timeout` | Healthcheck timeout in seconds       | No       | `30`                                                          |
| `healthcheck_retries` | Healthcheck retry attempts           | No       | `3`                                                           |
| `rsync_options`       | Additional rsync options             | No       | `-avz --progress`                                             |
| `ssh_options`         | Additional SSH options               | No       | `-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null` |

## Outputs

| Output               | Description                            |
| -------------------- | -------------------------------------- |
| `deployment_time`    | Timestamp of deployment                |
| `backup_path`        | Path of created backup (if applicable) |
| `healthcheck_status` | Healthcheck result                     |

## Setup

### 1. Generate SSH Key

```bash
ssh-keygen -t rsa -b 4096 -C "github-actions" -f ~/.ssh/relybytes_deploy
```

### 2. Add Public Key to HestiaCP

Copy the public key to your HestiaCP server:

```bash
ssh-copy-id -i ~/.ssh/relybytes_deploy.pub user@your-server.com
```

Or manually add it to `~/.ssh/authorized_keys` on the server.

### 3. Configure GitHub Secrets

Add the following secrets to your repository:

- `SSH_HOST`: Your server hostname or IP
- `SSH_USER`: SSH username
- `SSH_PRIVATE_KEY`: Content of your private key file

### 4. Verify Permissions

Ensure the SSH user has:

- Write permissions to the target directory
- Sufficient disk space for backups
- Network access for healthcheck URLs

## Examples

### Simple Static Site Deployment

```yaml
name: Deploy Static Site

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18"

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Deploy to HestiaCP
        uses: relybytes/actions/hestia-deploy@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          private_key: ${{ secrets.SSH_PRIVATE_KEY }}
          target: /home/user/web/mysite.com/public_html/
          delete: "true"
          healthcheck_url: https://mysite.com
```

### Multi-Environment Deployment

```yaml
name: Deploy

on:
  push:
    branches: [main, develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: npm run build

      - name: Deploy to Staging
        if: github.ref == 'refs/heads/develop'
        uses: relybytes/actions/hestia-deploy@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          private_key: ${{ secrets.STAGING_KEY }}
          target: /home/user/web/staging.mysite.com/public_html/
          delete: "true"
          healthcheck_url: https://staging.mysite.com/health

      - name: Deploy to Production
        if: github.ref == 'refs/heads/main'
        uses: relybytes/actions/hestia-deploy@v1
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USER }}
          private_key: ${{ secrets.PRODUCTION_KEY }}
          target: /home/user/web/mysite.com/public_html/
          delete: "true"
          backup_retention: "14"
          healthcheck_url: https://mysite.com/health
```

## Security Best Practices

- **Never commit private keys** to your repository
- Use **GitHub Secrets** for all sensitive data
- **Rotate SSH keys** regularly
- **Limit SSH user permissions** to only what's needed
- Use **read-only keys** for non-production environments
- **Monitor access logs** on your server

## Troubleshooting

### Common Issues

#### SSH Connection Failed

- Verify host and port are correct
- Check SSH key format and permissions
- Ensure server is accessible from GitHub Actions

#### Permission Denied

- Verify SSH user has write permissions to target directory
- Check directory ownership and permissions
- Ensure SSH key is properly authorized

#### Rsync Failed

- Check source directory exists and contains files
- Verify target directory path is correct
- Check available disk space

#### Healthcheck Failed

- Verify healthcheck URL is accessible
- Check timeout and retry settings
- Ensure application is properly configured

### Debug Mode

Enable debug logging by adding this step before deployment:

```yaml
- name: Enable debug logging
  run: |
    echo "ACTIONS_STEP_DEBUG=true" >> $GITHUB_ENV
    echo "ACTIONS_RUNNER_DEBUG=true" >> $GITHUB_ENV
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request

## License

MIT License - see [LICENSE](../../LICENSE) file for details.

## Support

- Create an [issue](https://github.com/relybytes/actions/issues) for bug reports
- Start a [discussion](https://github.com/relybytes/actions/discussions) for questions
- Check [existing issues](https://github.com/relybytes/actions/issues?q=is%3Aissue+is%3Aopen) before creating new ones

## Version History

- `v1` - Initial release with core deployment features
- `v1.1.0` - Added healthcheck and backup retention
- `v1.2.0` - Enhanced error handling and logging

# SSH Setup Action

[![GitHub release](https://img.shields.io/github/release/relybytes/actions.svg)](https://github.com/relybytes/actions/releases)
[![GitHub marketplace](https://img.shields.io/badge/marketplace-ssh--setup-blue?logo=github)](https://github.com/marketplace/actions/ssh-setup)

A reusable GitHub Action for setting up SSH connections for deployment and automation workflows. Provides secure SSH key management, connection testing, and environment configuration.

## Features

- **Secure SSH key handling** with proper permissions
- **Automatic host key scanning** or manual known hosts
- **Connection testing** with timeout and error handling
- **Reusable SSH aliases** for subsequent steps
- **Environment variable setup** for easy access
- **Production-ready** with comprehensive validation

## Usage

### Basic Usage

```yaml
- name: Setup SSH
  uses: relybytes/actions/ssh-setup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

### Advanced Usage

```yaml
- name: Setup SSH Connection
  uses: relybytes/actions/ssh-setup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    port: "2222"
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    known_hosts: ${{ secrets.KNOWN_HOSTS }}
    connection_alias: "production_server"
    timeout: "30"
    ssh_options: "-o StrictHostKeyChecking=yes -o UserKnownHostsFile=/tmp/known_hosts"
```

### Using in Multi-Step Workflows

```yaml
- name: Setup SSH
  uses: relybytes/actions/ssh-setup@v1
  id: ssh
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}

- name: Deploy files
  run: |
    scp ./dist/* ${{ steps.ssh.outputs.connection_alias }}:/var/www/html/

- name: Run remote command
  run: |
    ssh ${{ steps.ssh.outputs.connection_alias }} "systemctl reload nginx"
```

## Inputs

| Input              | Description                        | Required | Default                                                       |
| ------------------ | ---------------------------------- | -------- | ------------------------------------------------------------- |
| `host`             | SSH host address                   | Yes      | -                                                             |
| `port`             | SSH port                           | No       | `22`                                                          |
| `username`         | SSH username                       | Yes      | -                                                             |
| `private_key`      | SSH private key                    | Yes      | -                                                             |
| `known_hosts`      | Known hosts content (optional)     | No       | -                                                             |
| `ssh_options`      | Additional SSH options             | No       | `-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null` |
| `connection_alias` | SSH connection alias for later use | No       | `deploy_target`                                               |
| `timeout`          | Connection timeout in seconds      | No       | `30`                                                          |

## Outputs

| Output             | Description                                      |
| ------------------ | ------------------------------------------------ |
| `connection_alias` | SSH connection alias for use in subsequent steps |
| `connection_test`  | Connection test result (`success`/`failed`)      |

## Setup

### 1. Generate SSH Key

```bash
ssh-keygen -t rsa -b 4096 -C "github-actions" -f ~/.ssh/relybytes_deploy
```

### 2. Add Public Key to Server

```bash
ssh-copy-id -i ~/.ssh/relybytes_deploy.pub user@your-server.com
```

Or manually add to `~/.ssh/authorized_keys`:

```bash
cat ~/.ssh/relybytes_deploy.pub | ssh user@your-server.com "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### 3. Configure GitHub Secrets

Add these secrets to your repository:

- `SSH_HOST`: Server hostname or IP address
- `SSH_USER`: SSH username
- `SSH_PRIVATE_KEY`: Content of private key file
- `KNOWN_HOSTS` (optional): Known hosts fingerprint

### 4. Optional: Get Known Hosts Fingerprint

For enhanced security, manually specify known hosts:

```bash
ssh-keyscan -p 22 your-server.com
```

Add the output to your `KNOWN_HOSTS` secret.

## Examples

### Simple Deployment Workflow

```yaml
name: Deploy Application

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup SSH
        uses: relybytes/actions/ssh-setup@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          private_key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Deploy files
        run: |
          rsync -avz ./dist/ deploy_target:/var/www/html/

      - name: Restart service
        run: |
          ssh deploy_target "sudo systemctl reload nginx"
```

### Multi-Server Deployment

```yaml
name: Deploy to Multiple Servers

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        server: [web1, web2, web3]
    steps:
      - uses: actions/checkout@v4

      - name: Setup SSH for ${{ matrix.server }}
        uses: relybytes/actions/ssh-setup@v1
        with:
          host: ${{ secrets[format('SSH_HOST_{0}', matrix.server)] }}
          username: ${{ secrets.SSH_USER }}
          private_key: ${{ secrets.SSH_PRIVATE_KEY }}
          connection_alias: ${{ matrix.server }}

      - name: Deploy to ${{ matrix.server }}
        run: |
          rsync -avz ./dist/ ${{ matrix.server }}:/var/www/html/

      - name: Health check on ${{ matrix.server }}
        run: |
          ssh ${{ matrix.server }} "curl -f http://localhost/health"
```

### Using with Custom SSH Options

```yaml
- name: Setup SSH with custom options
  uses: relybytes/actions/ssh-setup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    ssh_options: |
      -o StrictHostKeyChecking=yes
      -o UserKnownHostsFile=/tmp/known_hosts
      -o ConnectTimeout=60
      -o ServerAliveInterval=30
      -o ServerAliveCountMax=3
    known_hosts: ${{ secrets.KNOWN_HOSTS }}
```

### Using Environment Variables

After setup, you can use these environment variables in subsequent steps:

```yaml
- name: Setup SSH
  uses: relybytes/actions/ssh-setup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}

- name: Use SSH environment variables
  run: |
    echo "Connected to $SSH_HOST:$SSH_PORT as $SSH_USER"
    echo "Using alias: $SSH_ALIAS"
    scp ./app.tar.gz $SSH_ALIAS:/tmp/
```

## Security Best Practices

### SSH Key Management

- **Use dedicated keys** for GitHub Actions
- **Rotate keys regularly** (every 90 days)
- **Limit key permissions** to only necessary directories
- **Use read-only keys** for non-production environments
- **Monitor SSH access logs** on your servers

### Known Hosts

- **Always specify known hosts** for production environments
- **Use strict host key checking** for security
- **Verify fingerprints** before adding to secrets
- **Update known hosts** when servers change

### User Permissions

- **Create dedicated users** for deployments
- **Use sudo** only when necessary
- **Limit file system access** to required directories
- **Implement principle of least privilege**

## Troubleshooting

### Common Issues

#### Connection Timeout

- Verify host and port are correct
- Check network connectivity
- Increase timeout value if needed
- Verify firewall settings

#### Authentication Failed

- Check SSH key format (must be valid private key)
- Verify public key is properly authorized
- Check user permissions on server
- Ensure SSH service is running

#### Host Key Verification Failed

- Add known hosts fingerprint
- Use `StrictHostKeyChecking=no` for testing only
- Verify server hasn't changed
- Check for man-in-the-middle attacks

#### Permission Denied

- Verify user has SSH access
- Check directory permissions
- Ensure authorized_keys file permissions (600)
- Verify user account is not locked

### Debug Mode

Enable debug logging:

```yaml
- name: Enable debug
  run: |
    echo "ACTIONS_STEP_DEBUG=true" >> $GITHUB_ENV
    echo "ACTIONS_RUNNER_DEBUG=true" >> $GITHUB_ENV

- name: Setup SSH with debug
  uses: relybytes/actions/ssh-setup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

### Manual Testing

Test SSH connection locally before using in Actions:

```bash
# Test with private key
ssh -i ~/.ssh/relybytes_deploy user@your-server.com "echo 'Connection test successful'"

# Test with specific options
ssh -o StrictHostKeyChecking=no -o ConnectTimeout=30 \
    -i ~/.ssh/relybytes_deploy \
    user@your-server.com "echo 'Test passed'"
```

## Advanced Usage

### SSH Agent Forwarding

```yaml
- name: Setup SSH with agent forwarding
  uses: relybytes/actions/ssh-setup@v1
  with:
    host: ${{ secrets.BASTION_HOST }}
    username: ${{ secrets.BASTION_USER }}
    private_key: ${{ secrets.BASTION_KEY }}
    ssh_options: "-o ForwardAgent=yes"

- name: Connect through bastion
  run: |
    ssh ${{ secrets.TARGET_HOST }} "echo 'Connected through bastion'"
```

### Using with Bastion Host

```yaml
- name: Setup SSH to bastion
  uses: relybytes/actions/ssh-setup@v1
  with:
    host: ${{ secrets.BASTION_HOST }}
    username: ${{ secrets.BASTION_USER }}
    private_key: ${{ secrets.BASTION_KEY }}
    connection_alias: bastion

- name: Deploy through bastion
  run: |
    ssh bastion "scp -r ./dist/ ${{ secrets.TARGET_USER }}@${{ secrets.TARGET_HOST }}:/var/www/html/"
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

- `v1` - Initial release with basic SSH setup
- `v1.1.0` - Added connection alias and environment variables
- `v1.2.0` - Enhanced error handling and validation

# HestiaCP Backup Action

[![GitHub release](https://img.shields.io/github/release/relybytes/actions.svg)](https://github.com/relybytes/actions/releases)
[![GitHub marketplace](https://img.shields.io/badge/marketplace-hestia--backup-blue?logo=github)](https://github.com/marketplace/actions/hestiacp-backup)

A production-ready GitHub Action for creating remote backups of HestiaCP directories with compression, retention policies, and integrity verification.

## Features

- **Secure SSH-based backups** with key authentication
- **Multiple compression formats** (tar.gz, tar.bz2, zip)
- **Configurable retention policies** with automatic cleanup
- **Integrity verification** for compressed backups
- **Pre/post backup commands** for database dumps or service stops
- **Exclude patterns** for filtering unnecessary files
- **Detailed reporting** with backup statistics

## Usage

### Basic Backup

```yaml
- name: Backup Website
  uses: relybytes/actions/hestia-backup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    source_directory: /home/user/web/mysite.com/public_html/
```

### Advanced Backup

```yaml
- name: Comprehensive Backup
  uses: relybytes/actions/hestia-backup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    source_directory: /home/user/web/mysite.com/public_html/
    backup_directory: /home/user/backups
    backup_name: mysite_production
    compression: "tar.gz"
    retention: "14"
    exclude_patterns: "*.log,tmp/*,cache/*,node_modules/*,*.tmp"
    pre_backup_command: "mysqldump -u root -p${{ secrets.DB_PASSWORD }} mysite > /tmp/mysite.sql"
    post_backup_command: "rm -f /tmp/mysite.sql"
```

## Inputs

| Input                 | Description                            | Required | Default                                                       |
| --------------------- | -------------------------------------- | -------- | ------------------------------------------------------------- |
| `host`                | SSH host address                       | Yes      | -                                                             |
| `port`                | SSH port                               | No       | `22`                                                          |
| `username`            | SSH username                           | Yes      | -                                                             |
| `private_key`         | SSH private key                        | Yes      | -                                                             |
| `source_directory`    | Source directory to backup             | Yes      | -                                                             |
| `backup_directory`    | Remote backup directory                | No       | `~/backups`                                                   |
| `backup_name`         | Custom backup name (without extension) | No       | Auto-generated                                                |
| `compression`         | Compression type                       | No       | `tar.gz`                                                      |
| `retention`           | Number of backups to keep              | No       | `7`                                                           |
| `exclude_patterns`    | Exclude patterns (comma separated)     | No       | `*.log,tmp/*,cache/*,node_modules/*`                          |
| `pre_backup_command`  | Command to run before backup           | No       | -                                                             |
| `post_backup_command` | Command to run after backup            | No       | -                                                             |
| `ssh_options`         | Additional SSH options                 | No       | `-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null` |

## Outputs

| Output            | Description                   |
| ----------------- | ----------------------------- |
| `backup_path`     | Full path to created backup   |
| `backup_name`     | Name of backup file           |
| `backup_size`     | Size of backup in bytes       |
| `backup_duration` | Backup duration in seconds    |
| `backups_removed` | Number of old backups removed |

## Compression Formats

### tar.gz (Default)

- Good compression ratio
- Fast compression
- Widely supported
- Use for general purpose backups

### tar.bz2

- Better compression than tar.gz
- Slower compression
- Good for large files
- Use when storage is limited

### zip

- Cross-platform compatibility
- Moderate compression
- Windows-friendly
- Use for mixed environments

### none

- No compression
- Fastest backup
- Large file sizes
- Use for temporary backups

## Examples

### Simple Website Backup

```yaml
name: Daily Website Backup

on:
  schedule:
    - cron: "0 2 * * *" # Daily at 2 AM
  workflow_dispatch:

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Backup Website
        uses: relybytes/actions/hestia-backup@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          private_key: ${{ secrets.SSH_PRIVATE_KEY }}
          source_directory: /home/user/web/mysite.com/public_html/
          retention: "30" # Keep 30 days of backups
```

### Database and Files Backup

```yaml
- name: Backup Database and Files
  uses: relybytes/actions/hestia-backup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    source_directory: /home/user/web/mysite.com/
    backup_directory: /home/user/backups
    backup_name: mysite_full_backup
    compression: "tar.bz2"
    retention: "14"
    exclude_patterns: "cache/*,logs/*,tmp/*"
    pre_backup_command: |
      mysqldump -u root -p${{ secrets.DB_PASSWORD }} --single-transaction mysite > /home/user/web/mysite.com/mysite.sql &&
      tar -czf /home/user/web/mysite.com/config_backup.tar.gz /etc/nginx/sites-available/mysite.com
    post_backup_command: |
      rm -f /home/user/web/mysite.com/mysite.sql /home/user/web/mysite.com/config_backup.tar.gz
```

### Multi-Site Backup

```yaml
name: Backup All Sites

on:
  schedule:
    - cron: "0 3 * * 0" # Weekly on Sunday at 3 AM

jobs:
  backup:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        site: [site1, site2, site3]
    steps:
      - name: Backup ${{ matrix.site }}
        uses: relybytes/actions/hestia-backup@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          private_key: ${{ secrets.SSH_PRIVATE_KEY }}
          source_directory: /home/user/web/${{ matrix.site }}.com/public_html/
          backup_name: ${{ matrix.site }}_weekly
          compression: "tar.gz"
          retention: "4" # Keep 4 weeks
```

### Pre-Deployment Backup

```yaml
- name: Backup Before Deployment
  uses: relybytes/actions/hestia-backup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    source_directory: /home/user/web/mysite.com/public_html/
    backup_name: pre_deploy_${{ github.sha }}
    compression: "tar.gz"
    retention: "10"
    exclude_patterns: "cache/*,logs/*"

- name: Deploy
  uses: relybytes/actions/hestia-deploy@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    source: dist/
    target: /home/user/web/mysite.com/public_html/
    backup: "false" # Skip backup since we did it manually
```

### Large File Backup with Exclusions

```yaml
- name: Backup with Exclusions
  uses: relybytes/actions/hestia-backup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    source_directory: /home/user/web/mysite.com/
    compression: "tar.bz2"
    retention: "7"
    exclude_patterns: |
      *.log,
      tmp/*,
      cache/*,
      node_modules/*,
      vendor/*,
      storage/logs/*,
      storage/framework/cache/*,
      storage/framework/sessions/*,
      storage/framework/views/*
```

## Exclude Patterns

### File Patterns

```yaml
exclude_patterns: "*.log,*.tmp,*.cache"
```

### Directory Patterns

```yaml
exclude_patterns: "tmp/*,cache/*,logs/*"
```

### Mixed Patterns

```yaml
exclude_patterns: "*.log,tmp/*,cache/*,node_modules/*,storage/logs/*"
```

### Common Exclusions for Web Applications

```yaml
exclude_patterns: |
  *.log,
  tmp/*,
  cache/*,
  node_modules/*,
  vendor/*,
  storage/logs/*,
  storage/framework/cache/*,
  storage/framework/sessions/*,
  .git/*,
  .env*
```

## Pre and Post Backup Commands

### Database Dump

```yaml
pre_backup_command: "mysqldump -u root -p${{ secrets.DB_PASSWORD }} mysite > /tmp/mysite.sql"
post_backup_command: "rm -f /tmp/mysite.sql"
```

### Service Management

```yaml
pre_backup_command: "systemctl stop nginx"
post_backup_command: "systemctl start nginx"
```

### Application Maintenance Mode

```yaml
pre_backup_command: "php artisan down --maintenance=backup"
post_backup_command: "php artisan up"
```

### Complex Commands

```yaml
pre_backup_command: |
  # Create database dump
  mysqldump -u root -p${{ secrets.DB_PASSWORD }} --single-transaction mysite > /tmp/mysite.sql &&
  # Create config backup
  tar -czf /tmp/config.tar.gz /etc/nginx/sites-available/mysite.com &&
  # Stop services
  systemctl stop nginx php-fpm

post_backup_command: |
  # Cleanup temporary files
  rm -f /tmp/mysite.sql /tmp/config.tar.gz &&
  # Start services
  systemctl start nginx php-fpm
```

## Backup Naming

### Auto-Generated Names

Default format: `{directory_name}_{timestamp}`
Examples:

- `mysite_20240101_120000.tar.gz`
- `public_html_20240101_120000.tar.bz2`

### Custom Names

```yaml
backup_name: "production_backup"
# Result: production_backup_20240101_120000.tar.gz
```

### Environment-Specific Names

```yaml
backup_name: "mysite_${{ github.ref_name }}_backup"
# Result: mysite_main_backup_20240101_120000.tar.gz
```

## Best Practices

### Backup Scheduling

- **Daily backups** for production sites
- **Weekly backups** for development environments
- **Pre-deployment backups** before major updates
- **Monthly archives** for long-term storage

### Retention Policies

- **Production**: Keep 30-90 days
- **Staging**: Keep 14-30 days
- **Development**: Keep 7-14 days
- **Critical data**: Keep 1+ years

### Compression Choices

- **tar.gz**: Best balance of speed and size
- **tar.bz2**: Maximum compression for storage-constrained environments
- **zip**: Cross-platform compatibility
- **none**: Fastest for temporary backups

### Security

- Use **dedicated backup users** with limited permissions
- **Encrypt sensitive backups** separately if needed
- **Store backup credentials** in GitHub Secrets
- **Regularly test backup restoration**

## Troubleshooting

### Common Issues

#### Permission Denied

- Check SSH user permissions
- Verify directory ownership
- Ensure backup directory is writable

#### Disk Space Full

- Check available space on server
- Increase retention cleanup
- Use stronger compression
- Exclude unnecessary files

#### Backup Creation Failed

- Verify source directory exists
- Check exclude pattern syntax
- Ensure compression tools are installed
- Test commands manually

#### Integrity Check Failed

- Check disk space during backup
- Verify network stability
- Test compression format
- Check file system errors

### Debug Mode

Enable detailed logging:

```yaml
- name: Enable debug
  run: |
    echo "ACTIONS_STEP_DEBUG=true" >> $GITHUB_ENV

- name: Backup with debug
  uses: relybytes/actions/hestia-backup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    source_directory: /home/user/web/mysite.com/public_html/
```

### Manual Testing

Test backup commands locally:

```bash
# Test SSH connection
ssh -i ~/.ssh/backup_key user@server "echo 'Connected'"

# Test directory access
ssh -i ~/.ssh/backup_key user@server "ls -la /home/user/web/mysite.com/"

# Test backup command
ssh -i ~/.ssh/backup_key user@server "tar -czf /tmp/test.tar.gz -C /home/user/web/mysite.com public_html/"

# Test backup integrity
ssh -i ~/.ssh/backup_key user@server "tar -tzf /tmp/test.tar.gz | head"
```

## Advanced Usage

### Conditional Backups

```yaml
- name: Backup if changed
  if: contains(github.event.head_commit.modified, 'config/') || github.event_name == 'schedule'
  uses: relybytes/actions/hestia-backup@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    source_directory: /home/user/web/mysite.com/
```

### Multi-Directory Backup

```yaml
strategy:
  matrix:
    dir: [public_html, storage, config]
steps:
  - name: Backup ${{ matrix.dir }}
    uses: relybytes/actions/hestia-backup@v1
    with:
      host: ${{ secrets.SSH_HOST }}
      username: ${{ secrets.SSH_USER }}
      private_key: ${{ secrets.SSH_PRIVATE_KEY }}
      source_directory: /home/user/web/mysite.com/${{ matrix.dir }}/
      backup_name: mysite_${{ matrix.dir }}
```

### Backup Verification

```yaml
- name: Create backup
  uses: relybytes/actions/hestia-backup@v1
  id: backup
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    private_key: ${{ secrets.SSH_PRIVATE_KEY }}
    source_directory: /home/user/web/mysite.com/public_html/

- name: Verify backup size
  run: |
    SIZE=${{ steps.backup.outputs.backup_size }}
    if [[ $SIZE -lt 1000000 ]]; then
      echo "::error::Backup seems too small: $SIZE bytes"
      exit 1
    fi
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

- `v1` - Initial release with basic backup functionality
- `v1.1.0` - Added pre/post backup commands
- `v1.2.0` - Enhanced compression options and integrity checks

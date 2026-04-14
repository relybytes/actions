# HTTP Healthcheck Action

[![GitHub release](https://img.shields.io/github/release/relybytes/actions.svg)](https://github.com/relybytes/actions/releases)
[![GitHub marketplace](https://img.shields.io/badge/marketplace-http--healthcheck-blue?logo=github)](https://github.com/marketplace/actions/http-healthcheck)

A comprehensive GitHub Action for verifying HTTP endpoints are healthy after deployment. Supports multiple retry attempts, custom status codes, content validation, and detailed reporting.

## Features

- **Flexible HTTP methods** (GET, POST, PUT, DELETE, etc.)
- **Custom status code validation** with range support
- **Content validation** with regex patterns
- **Retry logic** with configurable delays
- **Custom headers** and request bodies
- **Response time monitoring**
- **Detailed reporting** with workflow summaries

## Usage

### Basic Healthcheck

```yaml
- name: Healthcheck
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://your-app.com/health
```

### Advanced Healthcheck

```yaml
- name: Comprehensive Healthcheck
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://your-app.com/api/health
    timeout: '60'
    retries: '5'
    delay: '10'
    expected_status: '200-299'
    expected_content: 'status.*ok'
    follow_redirects: 'true'
    headers: '{"Authorization": "Bearer ${{ secrets.API_TOKEN }}"}'
    method: 'GET'
    fail_on_error: 'true'
```

### POST Request Healthcheck

```yaml
- name: POST Healthcheck
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://your-app.com/api/validate
    method: 'POST'
    headers: '{"Content-Type": "application/json"}'
    body: '{"test": "healthcheck"}'
    expected_status: '200,201'
    expected_content: 'success'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `url` | URL to check | Yes | - |
| `timeout` | Request timeout in seconds | No | `30` |
| `retries` | Number of retry attempts | No | `3` |
| `delay` | Delay between retries in seconds | No | `5` |
| `expected_status` | Expected HTTP status code(s) | No | `200-299` |
| `expected_content` | Expected content in response body (regex) | No | - |
| `follow_redirects` | Follow HTTP redirects | No | `true` |
| `user_agent` | Custom User-Agent header | No | `RelyBytes-Healthcheck/1.0` |
| `headers` | Custom headers (JSON format) | No | `{}` |
| `method` | HTTP method | No | `GET` |
| `body` | Request body for POST/PUT requests | No | - |
| `fail_on_error` | Fail workflow on healthcheck failure | No | `true` |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | Healthcheck status (`success`/`failed`) |
| `http_code` | Final HTTP status code |
| `response_time` | Response time in milliseconds |
| `attempts` | Number of attempts made |
| `response_body` | Response body (first 1000 chars) |

## Status Code Formats

The `expected_status` parameter supports multiple formats:

### Single Status Code
```yaml
expected_status: '200'
```

### Multiple Status Codes
```yaml
expected_status: '200,201,204'
```

### Status Code Range
```yaml
expected_status: '200-299'
```

### Mixed Format
```yaml
expected_status: '200,201,300-399'
```

## Examples

### Simple Web Application

```yaml
name: Deploy and Healthcheck

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy
        run: echo "Deploying application..."
        
      - name: Healthcheck
        uses: relybytes/actions/http-healthcheck@v1
        with:
          url: https://myapp.com
          retries: '5'
          delay: '10'
```

### API Endpoint Validation

```yaml
- name: API Healthcheck
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://api.myapp.com/health
    method: 'GET'
    headers: '{"X-API-Key": "${{ secrets.API_KEY }}"}'
    expected_status: '200'
    expected_content: '"status":"healthy"'
    timeout: '15'
```

### Database Connection Check

```yaml
- name: Database Healthcheck
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://myapp.com/health/db
    method: 'POST'
    headers: '{"Content-Type": "application/json"}'
    body: '{"check": "database"}'
    expected_status: '200'
    expected_content: 'database.*connected'
    retries: '3'
```

### Multi-Environment Healthcheck

```yaml
- name: Healthcheck Staging
  if: github.ref == 'refs/heads/develop'
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://staging.myapp.com/health
    expected_status: '200-299'
    
- name: Healthcheck Production
  if: github.ref == 'refs/heads/main'
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://myapp.com/health
    expected_status: '200'
    expected_content: 'production.*ready'
    retries: '10'
    delay: '15'
```

### Service Dependency Healthcheck

```yaml
- name: Check Database
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://db.myapp.com/health
    expected_status: '200'
    fail_on_error: 'false'
    
- name: Check Redis
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://redis.myapp.com/health
    expected_status: '200'
    fail_on_error: 'false'
    
- name: Check Main Application
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://myapp.com/health
    expected_status: '200'
    expected_content: 'dependencies.*ok'
```

## Content Validation

### Basic Text Matching
```yaml
expected_content: 'Hello World'
```

### Regex Patterns
```yaml
expected_content: 'status.*ok'
expected_content: '"version":"[0-9]+\.[0-9]+\.[0-9]+"'
expected_content: '<title>My Application</title>'
```

### JSON Response Validation
```yaml
expected_content: '"status":"healthy"'
expected_content: '"database":"connected"'
expected_content: '"uptime":[0-9]+'
```

### HTML Response Validation
```yaml
expected_content: '<h1>Welcome</h1>'
expected_content: 'class="status.*active"'
```

## Headers Configuration

### Authentication Headers
```yaml
headers: '{"Authorization": "Bearer ${{ secrets.API_TOKEN }}"}'
headers: '{"X-API-Key": "${{ secrets.API_KEY }}"}'
```

### Content Type Headers
```yaml
headers: '{"Content-Type": "application/json"}'
headers: '{"Accept": "application/json"}'
```

### Custom Headers
```yaml
headers: '{"X-Client": "GitHub-Actions", "X-Environment": "production"}'
```

### Multiple Headers
```yaml
headers: |
  {
    "Authorization": "Bearer ${{ secrets.API_TOKEN }}",
    "Content-Type": "application/json",
    "X-Client": "GitHub-Actions"
  }
```

## Best Practices

### Healthcheck Endpoints

Design your healthcheck endpoints to return:

```json
{
  "status": "healthy",
  "timestamp": "2024-01-01T12:00:00Z",
  "version": "1.2.3",
  "dependencies": {
    "database": "connected",
    "redis": "connected",
    "external_api": "ok"
  }
}
```

### Retry Strategies

- **Quick services**: 3 retries, 5s delay
- **Database connections**: 5 retries, 10s delay
- **External APIs**: 10 retries, 30s delay
- **Cold starts**: 5 retries, 15s delay

### Status Code Expectations

- **Simple healthchecks**: `200`
- **API endpoints**: `200-299`
- **Created resources**: `201,202`
- **No content**: `204`
- **Redirects**: `300-399`

## Troubleshooting

### Common Issues

#### Connection Timeout
- Increase timeout value
- Check network connectivity
- Verify URL is accessible
- Check firewall settings

#### Status Code Mismatch
- Verify expected status codes
- Check API documentation
- Look for redirect responses
- Enable redirect following

#### Content Validation Failed
- Test regex patterns locally
- Check response body format
- Verify case sensitivity
- Escape special characters

#### Authentication Issues
- Verify token validity
- Check header format
- Ensure proper encoding
- Test with curl first

### Debug Mode

Enable detailed logging:

```yaml
- name: Enable debug
  run: |
    echo "ACTIONS_STEP_DEBUG=true" >> $GITHUB_ENV

- name: Healthcheck with debug
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://your-app.com/health
```

### Manual Testing

Test your healthcheck locally:

```bash
# Basic test
curl -f https://your-app.com/health

# With headers
curl -H "Authorization: Bearer $TOKEN" https://your-app.com/health

# With timeout
curl -m 30 -f https://your-app.com/health

# With POST data
curl -X POST -H "Content-Type: application/json" \
     -d '{"test": true}' \
     https://your-app.com/health
```

## Advanced Usage

### Conditional Healthchecks

```yaml
- name: Healthcheck if deployed
  if: steps.deploy.outputs.success == 'true'
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://your-app.com/health
```

### Parallel Healthchecks

```yaml
strategy:
  matrix:
    endpoint: [health, api/health, db/health]
steps:
  - name: Healthcheck ${{ matrix.endpoint }}
    uses: relybytes/actions/http-healthcheck@v1
    with:
      url: https://your-app.com/${{ matrix.endpoint }}
```

### Progressive Healthchecks

```yaml
- name: Basic healthcheck
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://your-app.com/health
    expected_status: '200'
    
- name: API healthcheck
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://your-app.com/api/health
    expected_status: '200'
    expected_content: 'api.*ready'
    
- name: Full integration test
  uses: relybytes/actions/http-healthcheck@v1
  with:
    url: https://your-app.com/api/test
    method: 'POST'
    body: '{"test": "integration"}'
    expected_status: '200'
    expected_content: 'test.*passed'
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

- `v1.0.0` - Initial release with basic healthchecking
- `v1.1.0` - Added content validation and custom headers
- `v1.2.0` - Enhanced retry logic and reporting

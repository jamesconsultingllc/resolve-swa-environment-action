# Resolve Azure SWA Environment Action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Resolve%20Azure%20SWA%20Environment-blue?logo=github)](https://github.com/marketplace/actions/resolve-azure-swa-environment)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A GitHub Action that intelligently resolves Azure Static Web App deployment environments based on branch names. Designed for GitFlow workflows but fully customizable for any branching strategy.

## Features

- 🌿 **GitFlow Ready** - Works out of the box with GitFlow branching (main, develop, feature/*, release/*, hotfix/*)
- 🎯 **Smart Resolution** - Automatically determines target deployment slot and source environment
- 🔧 **Fully Customizable** - Override default mappings with your own branch-to-environment rules
- 🔍 **Source Detection** - Identifies parent environment for settings inheritance
- 📝 **Branch Sanitization** - Generates SWA-compatible slot names from branch names

## Quick Start

```yaml
- name: Resolve Environment
  id: resolve-env
  uses: jamesconsultingllc/resolve-swa-environment-action@v1

- name: Deploy to Azure SWA
  uses: Azure/static-web-apps-deploy@v1
  with:
    deployment_environment: ${{ steps.resolve-env.outputs.target-slot }}
    # ... other inputs
```

## How It Works

The action maps branch names to Azure SWA environments:

| Branch | Target Environment | Source Environment | Is Preview |
|--------|-------------------|-------------------|------------|
| `main` | production (empty) | - | false |
| `master` | production (empty) | - | false |
| `develop` | development | - | false |
| `release/v1.0` | staging | development | true |
| `feature/my-feature` | featuremyfeature* | development | true |
| `hotfix/urgent-fix` | hotfixurgentfix* | production | true |

*Sanitized: alphanumeric, lowercase, max 20 characters

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `branch` | Branch name to resolve. Auto-detects if not provided. | No | Auto-detect |
| `branch-mappings` | JSON object mapping branch patterns to environments | No | GitFlow defaults |
| `parent-branch-mappings` | JSON object mapping branches to their parent environment | No | GitFlow defaults |
| `detect-source` | Whether to detect source environment for settings sync | No | `true` |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `branch` | The resolved branch name | `feature/my-feature` |
| `target-environment` | Target environment/slot name | `featuremyfeature` |
| `target-slot` | Alias for target-environment | `featuremyfeature` |
| `source-environment` | Source environment for settings | `development` |
| `is-preview` | Whether this is a preview environment | `true` |
| `is-production` | Whether this is production | `false` |
| `sanitized-branch` | Sanitized branch name | `featuremyfeature` |

## Examples

### Basic Usage (GitFlow)

```yaml
name: Deploy
on:
  push:
    branches: [main, develop, 'feature/**', 'release/**', 'hotfix/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Resolve Environment
        id: resolve-env
        uses: jamesconsultingllc/resolve-swa-environment-action@v1
      
      - name: Deploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_SWA_TOKEN }}
          deployment_environment: ${{ steps.resolve-env.outputs.target-slot }}
          app_location: "/"
          output_location: "dist"
```

### Conditional Steps Based on Environment

```yaml
- name: Resolve Environment
  id: resolve-env
  uses: jamesconsultingllc/resolve-swa-environment-action@v1

- name: Run E2E Tests (Preview Only)
  if: steps.resolve-env.outputs.is-preview == 'true'
  run: npm run test:e2e

- name: Notify Slack (Production Only)
  if: steps.resolve-env.outputs.is-production == 'true'
  run: |
    curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
      -d '{"text": "Production deployment complete!"}'
```

### Custom Branch Mappings

```yaml
- name: Resolve Environment
  id: resolve-env
  uses: jamesconsultingllc/resolve-swa-environment-action@v1
  with:
    branch-mappings: |
      {
        "main": "production",
        "staging": "staging",
        "dev": "development",
        "feature/*": "$sanitized",
        "bugfix/*": "$sanitized"
      }
    parent-branch-mappings: |
      {
        "feature/*": "development",
        "bugfix/*": "development"
      }
```

### Trunk-Based Development

```yaml
- name: Resolve Environment
  id: resolve-env
  uses: jamesconsultingllc/resolve-swa-environment-action@v1
  with:
    branch-mappings: |
      {
        "main": "production",
        "preview/*": "$sanitized"
      }
    parent-branch-mappings: |
      {
        "preview/*": "production"
      }
```

### Explicit Branch Override

```yaml
- name: Resolve Environment for PR
  id: resolve-env
  uses: jamesconsultingllc/resolve-swa-environment-action@v1
  with:
    branch: ${{ github.event.pull_request.head.ref }}
```

### Skip Source Detection

```yaml
- name: Resolve Target Only
  id: resolve-env
  uses: jamesconsultingllc/resolve-swa-environment-action@v1
  with:
    detect-source: 'false'
```

## Branch Sanitization

Azure SWA has specific requirements for environment names. This action sanitizes branch names to comply:

1. Removes all non-alphanumeric characters
2. Converts to lowercase
3. Truncates to 20 characters

| Original Branch | Sanitized Name |
|----------------|----------------|
| `feature/add-login` | `featureaddlogin` |
| `feature/UPPER-Case` | `featureuppercase` |
| `feature/very-long-branch-name-here` | `featureverylongbran` |
| `hotfix/fix-123` | `hotfixfix123` |

## Default Mappings

### Branch → Environment (branch-mappings)

```json
{
  "main": "production",
  "master": "production",
  "develop": "development",
  "release/*": "staging",
  "feature/*": "$sanitized",
  "hotfix/*": "$sanitized"
}
```

### Branch → Parent Environment (parent-branch-mappings)

```json
{
  "feature/*": "development",
  "hotfix/*": "production",
  "release/*": "development"
}
```

## Integration with Other Actions

This action is designed to work seamlessly with:

- [sync-swa-settings-action](https://github.com/jamesconsultingllc/sync-swa-settings-action) - Sync environment settings
- [manage-storage-cors-action](https://github.com/jamesconsultingllc/manage-storage-cors-action) - Manage CORS for preview environments

### Complete Workflow Example

```yaml
name: Azure SWA CI/CD

on:
  push:
    branches: [main, develop, 'feature/**', 'release/**', 'hotfix/**']
  pull_request:
    types: [closed]
    branches: [main, develop]

jobs:
  deploy:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Resolve Environment
        id: resolve-env
        uses: jamesconsultingllc/resolve-swa-environment-action@v1
      
      - name: Build and Deploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_SWA_TOKEN }}
          deployment_environment: ${{ steps.resolve-env.outputs.target-slot }}
          app_location: "/"
          output_location: "dist"
      
      - name: Azure Login
        if: steps.resolve-env.outputs.is-preview == 'true'
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Sync Settings from Parent
        if: steps.resolve-env.outputs.is-preview == 'true'
        uses: jamesconsultingllc/sync-swa-settings-action@v1
        with:
          swa-name: my-swa
          resource-group: my-rg
          source-environment: ${{ steps.resolve-env.outputs.source-environment }}
          target-environment: ${{ steps.resolve-env.outputs.target-environment }}
          overwrite: 'true'
```

## Requirements

- GitHub Actions runner with `bash` and `jq` installed (included in ubuntu-latest)
- For source detection via git ancestry: `fetch-depth: 0` in checkout action

## License

MIT License - see [LICENSE](LICENSE) for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Support

- 🐛 [Report Issues](https://github.com/jamesconsultingllc/resolve-swa-environment-action/issues)
- 💡 [Request Features](https://github.com/jamesconsultingllc/resolve-swa-environment-action/issues/new)
- 📖 [Documentation](https://github.com/jamesconsultingllc/resolve-swa-environment-action#readme)

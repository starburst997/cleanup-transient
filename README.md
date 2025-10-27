# Cleanup Transient Deployments

A GitHub composite action that automates the cleanup of transient deployment artifacts including Git tags, Docker images, Helm charts, and GitHub deployments.

## Features

- Deletes Git tags matching version patterns
- Removes Docker images from GitHub Container Registry (GHCR)
- Removes Helm charts from GHCR (with `charts/` prefix)
- Deletes GitHub deployments
- Optional Kubernetes/Helm cleanup (uninstall Helm releases from K8s clusters)
- Pure bash implementation using `gh` CLI (no JavaScript dependencies)
- Supports multiple pattern types (rc, pr-based, custom)

## Usage

### Basic Example

```yaml
name: Cleanup Transient Deployment
on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Cleanup PR environment
        uses: starburst997/cleanup-transient@v1
        with:
          pattern: pr-${{ github.event.pull_request.number }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

### Cleanup RC/Staging Environment

```yaml
name: Cleanup RC Environment
on:
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Cleanup RC environment
        uses: starburst997/cleanup-transient@v1
        with:
          pattern: rc
          token: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Input               | Description                                                                                                                                                | Required | Default                               |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------------------- |
| `pattern`           | Pattern suffix to clean (e.g., "rc", "pr-123")                                                                                                             | Yes      | -                                     |
| `versions`          | Space-separated list of versions to clean (e.g., "1.0.0-rc.1 1.0.0-rc.2"). If not specified, versions are auto-detected from git tags matching the pattern | No       | -                                     |
| `token`             | GitHub token with permissions to delete packages and environments                                                                                          | No       | `${{ github.token }}`                 |
| `registry`          | Container registry URL                                                                                                                                     | No       | `ghcr.io`                             |
| `username`          | Username or organization (defaults to repository owner)                                                                                                    | No       | `${{ github.repository_owner }}`      |
| `repo-owner`        | Repository owner (defaults to current repo owner)                                                                                                          | No       | `${{ github.repository_owner }}`      |
| `repo-name`         | Repository name (defaults to current repo name)                                                                                                            | No       | `${{ github.event.repository.name }}` |
| `docker-image-name` | Custom Docker image name (defaults to repo name in lowercase)                                                                                              | No       | -                                     |
| `helm-chart-name`   | Custom Helm chart name (defaults to charts/{repo-name} in lowercase)                                                                                       | No       | -                                     |
| `chart-prefix`      | Prefix for Helm chart names (combined with repo name if helm-chart-name is not provided)                                                                   | No       | `charts`                              |
| `environment-name`  | Custom environment name (defaults to 'staging' for rc, pattern for pr-)                                                                                    | No       | -                                     |
| `kube-config`       | Kubernetes config file content for kubectl access (enables K8s cleanup)                                                                                    | No       | -                                     |
| `namespace`         | Kubernetes namespace for Helm release                                                                                                                      | No       | `preview`                             |
| `helm`              | Helm release name to uninstall from Kubernetes cluster                                                                                                     | No       | -                                     |

## Outputs

| Output              | Description                                                    |
| ------------------- | -------------------------------------------------------------- |
| `versions`          | Space-separated list of versions that were cleaned up          |
| `versions-markdown` | Markdown-formatted bullet list of versions for GitHub comments |

## How It Works

### 1. Git Tag Cleanup

The action searches for and deletes Git tags matching the pattern `v*-{pattern}.*`:

- For pattern `rc`: matches tags like `v1.0.0-rc.1`, `v1.0.0-rc.2`
- For pattern `pr-123`: matches tags like `v1.0.0-pr-123.1`, `v1.0.0-pr-123.2`

### 2. Docker Image Cleanup

Removes Docker images from GHCR where:

- Package name matches the repository name (lowercase)
- Image tags match the versions extracted from Git tags

Example: For repo `starburst997/s3-mirror-sample-app`, it cleans `ghcr.io/starburst997/s3-mirror-sample-app:{version}`

### 3. Helm Chart Cleanup

Removes Helm charts from GHCR where:

- Package name is `{chart-prefix}/{repo-name}` (lowercase), defaults to `charts/dev/{repo-name}`
- Chart tags match the versions extracted from Git tags

Example: For repo `starburst997/s3-mirror-sample-app`, it cleans `ghcr.io/starburst997/charts/s3-mirror-sample-app:{version}`

The `chart-prefix` input can be customized to use a different prefix (e.g., `helm`, `charts`).

### 4. Kubernetes/Helm Cleanup (Optional)

If `kube-config` is provided, the action will:

- Set up `kubectl` and `helm` CLI tools
- Configure kubectl with the provided kube config
- Uninstall the specified Helm release from the Kubernetes namespace
- Default namespace is `preview`, but can be customized with `namespace` input

**This step is completely optional and will be skipped if `kube-config` is not provided.**

### 5. GitHub Deployment Cleanup

Deletes the GitHub deployment with the specified name:

- Finds the deployment by name (via environment field)
- Sets the deployment to `inactive` status (required before deletion)
- Deletes the deployment
- For `rc` pattern: deletes deployment named `staging`
- For `pr-XXX` pattern: deletes deployment named `pr-XXX`

## Requirements

- GitHub Actions runner with `gh` CLI (pre-installed on GitHub-hosted runners)
- Git installed and repository checked out with full history (`fetch-depth: 0`)
- GitHub token with appropriate permissions (see below)

## Permissions

The GitHub token needs the following permissions:

```yaml
permissions:
  contents: write # For deleting Git tags
  packages: write # For deleting Docker images and Helm charts
  deployments: write # For deleting environments
```

### Using a Personal Access Token (PAT)

For organization-level packages, you may need to use a PAT instead of `GITHUB_TOKEN`:

```yaml
- name: Cleanup
  uses: starburst997/cleanup-transient@v1
  with:
    pattern: pr-${{ github.event.pull_request.number }}
    token: ${{ secrets.PAT_WITH_PACKAGES_WRITE }}
```

The PAT should have these scopes:

- `repo` (for tags and environments)
- `write:packages` (for deleting packages)
- `delete:packages` (for deleting packages)

## Examples

### Cleanup on PR Close

```yaml
name: Cleanup PR Environment
on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
      deployments: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Cleanup PR artifacts
        id: cleanup
        uses: starburst997/cleanup-transient@v1
        with:
          pattern: pr-${{ github.event.pull_request.number }}
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Display cleaned versions
        run: echo "Cleaned versions: ${{ steps.cleanup.outputs.versions }}"

      - name: Comment on PR
        if: steps.cleanup.outputs.versions != ''
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🗑️ Cleanup Complete\n\nThe following versions were cleaned up:\n\n${{ steps.cleanup.outputs.versions-markdown }}`
            })
```

### Manual Cleanup with Workflow Dispatch

```yaml
name: Manual Cleanup
on:
  workflow_dispatch:
    inputs:
      pattern:
        description: "Pattern to clean (e.g., rc, pr-123)"
        required: true
        type: string
      versions:
        description: "Optional: space-separated versions to clean (e.g., '1.0.0-rc.1 1.0.0-rc.2')"
        required: false
        type: string

jobs:
  cleanup:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
      deployments: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Cleanup pattern
        uses: starburst997/cleanup-transient@v1
        with:
          pattern: ${{ inputs.pattern }}
          versions: ${{ inputs.versions }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

### Cleanup Multiple Patterns

```yaml
name: Cleanup Multiple Patterns
on:
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
      deployments: write
    strategy:
      matrix:
        pattern: [pr-100, pr-101, pr-102]
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Cleanup ${{ matrix.pattern }}
        uses: starburst997/cleanup-transient@v1
        with:
          pattern: ${{ matrix.pattern }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

### Cleanup with Kubernetes/Helm

```yaml
name: Cleanup PR with Kubernetes
on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
      deployments: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Cleanup PR artifacts and Kubernetes resources
        uses: starburst997/cleanup-transient@v1
        with:
          pattern: pr-${{ github.event.pull_request.number }}
          token: ${{ secrets.GITHUB_TOKEN }}
          kube-config: ${{ secrets.KUBE_CONFIG }}
          namespace: preview
          helm: pr-${{ github.event.pull_request.number }}
```

> **Note:** If `kube-config` is not provided, the Kubernetes/Helm cleanup steps will be skipped automatically.

## Troubleshooting

### Package Not Found

If you see "Version X not found in registry", this is normal and means:

- The Docker image or Helm chart was never published for that version
- It was already deleted in a previous cleanup

### Permission Denied

If you get permission errors:

1. Check that your workflow has the correct `permissions` block
2. For organization packages, use a PAT with `write:packages` and `delete:packages` scopes
3. Ensure the token owner has admin access to the repository

### Environment Deletion Fails

GitHub environments may fail to delete if:

- They are protected by branch protection rules
- They have active deployments
- The environment doesn't exist (this is logged but not treated as an error)

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

If you encounter any issues or have questions, please [open an issue](../../issues) on GitHub.

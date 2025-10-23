# Instructions for Claude

This document contains instructions for AI assistants (like Claude) working on this project.

## Project Overview

This is a GitHub composite action that automates the cleanup of transient deployment artifacts including:
- Git tags
- Docker images (GHCR)
- Helm charts (GHCR)
- GitHub deployment environments

## Documentation Consistency Requirements

**CRITICAL:** When adding, modifying, or removing any inputs or outputs from `action.yml`, you MUST update the following files to maintain documentation consistency:

### 1. action.yml
- Main action definition
- Contains all input and output definitions
- Location: `/action.yml`

### 2. README.md
- Primary documentation file
- Must document all inputs in the "Inputs" table
- Must document all outputs in the "Outputs" table
- Must include usage examples showing new features
- Location: `/README.md`

### 3. docs/index.html
- GitHub Pages landing page
- Should mention key features and outputs
- Update the outputs description section if outputs change significantly
- Location: `/docs/index.html`

## Current Inputs

| Input | Description | Type | Required |
|-------|-------------|------|----------|
| `pattern` | Pattern suffix to clean (e.g., "rc", "pr-123") | string | Yes |
| `versions` | Space-separated list of versions to clean. If not specified, versions are auto-detected from git tags | string | No |
| `token` | GitHub token with permissions | string | No (default: `${{ github.token }}`) |
| `registry` | Container registry URL | string | No (default: `ghcr.io`) |
| `username` | Username or organization | string | No (default: `${{ github.repository_owner }}`) |
| `repo-owner` | Repository owner | string | No (default: `${{ github.repository_owner }}`) |
| `repo-name` | Repository name | string | No (default: `${{ github.event.repository.name }}`) |
| `docker-image-name` | Custom Docker image name | string | No |
| `helm-chart-name` | Custom Helm chart name | string | No |
| `environment-name` | Custom environment name | string | No |
| `kube-config` | Kubernetes config for kubectl (enables K8s cleanup) | string | No |
| `namespace` | Kubernetes namespace | string | No (default: `preview`) |
| `helm` | Helm release name to uninstall | string | No |

## Current Outputs

| Output | Description | Format |
|--------|-------------|--------|
| `versions` | Space-separated list of cleaned versions | `1.0.0-rc.1 1.0.0-rc.2` |
| `versions-markdown` | Markdown bullet list of cleaned versions | `- 1.0.0-rc.1\n- 1.0.0-rc.2` |

## Workflow for Adding New Inputs

1. **Add to action.yml**
   - Add the input definition under `inputs:` section
   - Include description, required flag, and default value
   - Update the relevant step(s) to use the new input

2. **Update README.md**
   - Add a row to the "Inputs" table
   - Add usage examples demonstrating the new input
   - Update "How It Works" section if the input changes behavior

3. **Update docs/index.html** (if significant)
   - Mention the new feature in the features section if it's user-facing
   - Update code examples if necessary

## Workflow for Adding New Outputs

1. **Add to action.yml**
   - Add the output definition under `outputs:` section
   - Include description
   - Ensure the step that generates the output sets it using `$GITHUB_OUTPUT`

2. **Update README.md**
   - Add a row to the "Outputs" table
   - Add usage examples showing how to use the output
   - Common uses: PR comments, workflow summaries, conditional steps

3. **Update docs/index.html**
   - Update the outputs description section
   - Mention key use cases for the output

## Code Style Guidelines

### Bash Scripts
- Use lowercase variable names for local variables
- Use UPPERCASE for environment variables and exported values
- Always quote variables: `"$VARIABLE"`
- Use conditional checks for empty values: `[ -z "$VAR" ]`
- Provide fallback logic for optional inputs

### YAML
- Use double quotes for strings with special characters
- Use proper indentation (2 spaces)
- Add comments for complex logic

### Documentation
- Use tables for structured data (inputs, outputs)
- Include code examples for all features
- Keep examples realistic and practical
- Use emoji sparingly and appropriately

## Testing Considerations

When making changes:
1. Test with both default and custom input values
2. Test with empty/missing optional inputs
3. Test outputs in different scenarios (empty list, single item, multiple items)
4. Verify markdown formatting works correctly in GitHub comments

## Design Philosophy

1. **Backward Compatibility:** New inputs should have empty defaults that preserve existing behavior
2. **Flexibility:** Allow customization while providing sensible defaults
3. **Clear Outputs:** Provide both machine-readable and human-readable output formats
4. **Error Handling:** Fail gracefully and provide clear error messages
5. **Documentation:** Every feature must be documented in all three locations

## Common Pitfalls

- ❌ Adding an input without updating README.md
- ❌ Changing output format without updating documentation
- ❌ Not testing with empty/default values
- ❌ Forgetting to use proper multiline output format in bash (`<<EOF`)
- ❌ Not URL-encoding special characters in package names (e.g., `charts/` → `charts%2F`)

## GitHub Pages

The `docs/index.html` file is served as GitHub Pages. To enable:
1. Go to repository Settings → Pages
2. Set Source to "Deploy from a branch"
3. Select branch: `main` and folder: `/docs`
4. Save

The site will be available at: `https://[username].github.io/cleanup-transient/`

## Marketplace Categories

When publishing to GitHub Marketplace, use these categories:
1. **Deployment** (primary)
2. **Utilities** (secondary)

## Additional Notes

- The action uses composite run steps (bash + gh CLI)
- No JavaScript/TypeScript dependencies required
- Designed for cleanup of transient environments (PR-based, RC, etc.)
- Supports both user and organization packages in GHCR
- Optional Kubernetes/Helm cleanup (activated when `kube-config` input is provided)

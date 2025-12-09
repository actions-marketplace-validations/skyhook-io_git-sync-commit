# Git Sync and Commit Action

[![Release](https://github.com/skyhook-io/git-sync-commit/actions/workflows/release.yml/badge.svg)](https://github.com/skyhook-io/git-sync-commit/actions/workflows/release.yml)

A GitHub Action that safely syncs, commits, and pushes changes with automatic rebase and retry logic to handle concurrent modifications.

## Why This Action?
 
1. **Race condition handling**: Automatically retries push operations when concurrent changes occur
2. **Rebase-first workflow**: Always rebases before pushing to maintain linear history
3. **Safe stashing**: Preserves uncommitted changes during sync operations
4. **Smart commit detection**: Only commits when there are actual changes to push
5. **Configurable retry logic**: Handles concurrent GitOps workflows gracefully
6. **Clear outputs**: Returns commit status and SHA for downstream steps

## Use Cases

- **GitOps deployments**: Automatically commit manifest updates and handle concurrent deployments
- **Automated updates**: Commit dependency updates, generated files, or version bumps
- **Configuration management**: Sync configuration changes across environments
- **Release automation**: Update version files and changelogs during releases

## Usage

### Basic Example

```yaml
- uses: skyhook-io/git-sync-commit@v1
  with:
    commit_message: 'Update deployment manifests [skip ci]'
    file_pattern: 'manifests/**/*'
```

### GitOps Deployment Example

```yaml
- name: Update kustomize manifests
  uses: skyhook-io/kustomize-edit@v1
  with:
    overlay_dir: overlays/production
    image: myapp
    tag: v1.2.3

- name: Commit and push changes
  uses: skyhook-io/git-sync-commit@v1
  with:
    commit_message: |
      Deploy myapp v1.2.3 to production [skip ci]

      Automated deployment triggered by ${{ github.actor }}
    file_pattern: 'overlays/production/*'
    max_retries: 5
```

### Using Commit SHA in Downstream Steps

```yaml
- id: commit
  uses: skyhook-io/git-sync-commit@v1
  with:
    commit_message: 'Update version to ${{ github.ref_name }}'
    file_pattern: 'VERSION package.json'

- name: Create deployment
  if: steps.commit.outputs.committed == 'true'
  run: |
    echo "Deploying commit ${{ steps.commit.outputs.commit_sha }}"
    kubectl set image deployment/app app=myapp:${{ steps.commit.outputs.commit_sha }}
```

### Conditional Commit Based on Changes

```yaml
- name: Generate documentation
  run: npm run docs:generate

- id: commit
  uses: skyhook-io/git-sync-commit@v1
  with:
    commit_message: 'Update generated documentation [skip ci]'
    file_pattern: 'docs/**/*'

- name: Notify on changes
  if: steps.commit.outputs.committed == 'true'
  run: |
    echo "Documentation updated in commit ${{ steps.commit.outputs.commit_sha }}"
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `commit_message` | Commit message | Yes | - |
| `file_pattern` | Files to stage for commit (e.g., ".", "*.yml", "path/to/files/*") | No | `.` |
| `path` | Working directory for git operations | No | `.` |
| `commit_user_name` | Git committer name | No | `github-actions[bot]` |
| `commit_user_email` | Git committer email | No | `41898282+github-actions[bot]@users.noreply.github.com` |
| `max_retries` | Maximum number of push retry attempts (handles concurrent pushes) | No | `3` |

## Outputs

| Output | Description |
|--------|-------------|
| `committed` | Whether a commit was created (`true`/`false`) |
| `commit_sha` | SHA of the created commit (empty if no commit) |

## How It Works

The action follows this workflow:

1. **Validate inputs** - Ensures working directory exists and inputs are valid
2. **Stash changes** - Safely stashes any uncommitted changes (including untracked files)
3. **Sync with upstream** - Pulls latest changes using rebase to maintain linear history
4. **Restore changes** - Pops the stash to restore your changes
   - **Automatic conflict resolution**: If conflicts occur between stashed changes and pulled changes, the action automatically resolves them by accepting your stashed changes (the newer modifications from this workflow)
   - This ensures concurrent workflow changes are properly merged without leaving conflict markers
5. **Stage files** - Adds files matching the specified pattern
6. **Safety check** - Verifies no conflict markers are present in staged files (additional safeguard)
7. **Commit** - Creates a commit if there are staged changes
8. **Push with retry** - Attempts to push with automatic retry on failure:
   - On push failure, rebases again and retries
   - Continues up to `max_retries` attempts
   - Handles race conditions from concurrent workflows

## Examples

### Multi-Environment GitOps

```yaml
name: Deploy to Environments

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, production]

    steps:
      - uses: actions/checkout@v4

      - name: Update manifests
        uses: skyhook-io/kustomize-edit@v1
        with:
          overlay_dir: overlays/${{ matrix.environment }}
          image: myapp
          tag: ${{ github.ref_name }}

      - name: Commit changes
        uses: skyhook-io/git-sync-commit@v1
        with:
          commit_message: Deploy ${{ github.ref_name }} to ${{ matrix.environment }} [skip ci]
          file_pattern: overlays/${{ matrix.environment }}/*
          max_retries: 10  # Higher retries for parallel deployments
```

### Automated Dependency Updates

```yaml
name: Update Dependencies

on:
  schedule:
    - cron: '0 2 * * 1'  # Weekly on Monday

jobs:
  update:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Update dependencies
        run: npm update

      - id: commit
        uses: skyhook-io/git-sync-commit@v1
        with:
          commit_message: |
            chore: update npm dependencies [skip ci]

            Automated weekly dependency update
          file_pattern: 'package.json package-lock.json'

      - name: Create PR
        if: steps.commit.outputs.committed == 'true'
        run: |
          gh pr create \
            --title "Weekly dependency update" \
            --body "Automated dependency update from commit ${{ steps.commit.outputs.commit_sha }}"
```

### Custom Working Directory

```yaml
- name: Clone deployment repo
  uses: actions/checkout@v4
  with:
    repository: my-org/deployment-configs
    path: deployment

- name: Update configs
  run: |
    cd deployment
    ./scripts/update-config.sh ${{ github.ref_name }}

- uses: skyhook-io/git-sync-commit@v1
  with:
    path: deployment
    commit_message: Update configuration for ${{ github.ref_name }}
    file_pattern: 'configs/**/*.yaml'
```

### Skip CI in Commit Message

To prevent infinite loops in CI/CD pipelines, add `[skip ci]` to your commit message:

```yaml
- uses: skyhook-io/git-sync-commit@v1
  with:
    commit_message: |
      Deploy version ${{ github.ref_name }} [skip ci]

      This prevents the push from triggering the workflow again
    file_pattern: 'deployment/*'
```

## Migration from hisco/safe-rebase-commit

This action is a direct replacement for `hisco/safe-rebase-commit` with enhanced features and Skyhook standards.

### Key Improvements

1. **Composite action**: Embedded logic (no external script dependencies)
2. **Better validation**: Input validation before operations
3. **Enhanced outputs**: Returns commit status and SHA
4. **GitHub summaries**: Visual summaries in Actions UI
5. **Skyhook branding**: Consistent with other Skyhook actions
6. **Better error handling**: Clear error messages and validation

### Migration Guide

Simply update your action reference:

```yaml
# Before (hisco/safe-rebase-commit)
- uses: hisco/safe-rebase-commit@v1.0.1
  with:
    path: deployment
    commit_message: Update env 'production' to version 'v1.2.3' [skip ci]
    file_pattern: overlays/production/*

# After (skyhook-io/git-sync-commit)
- uses: skyhook-io/git-sync-commit@v1
  with:
    path: deployment
    commit_message: Update env 'production' to version 'v1.2.3' [skip ci]
    file_pattern: overlays/production/*
```

All inputs are compatible, no changes needed!

## Notes

- **Authentication**: Ensure your workflow has write permissions to the repository
  ```yaml
  permissions:
    contents: write
  ```
- **Branch protection**: The action respects branch protection rules
- **Concurrent workflows**: Use higher `max_retries` values when multiple workflows may push simultaneously
- **Linear history**: Uses rebase instead of merge to maintain clean history
- **Stash safety**: Uncommitted changes are preserved during sync operations
- **No changes = no commit**: The action skips commit creation when there are no staged changes

## Permissions

This action requires the following permissions:

```yaml
permissions:
  contents: write  # Required to push commits
```

## Troubleshooting

### Push fails with "non-fast-forward" error

This usually means concurrent changes occurred. The action automatically handles this with retries. If you see this repeatedly, increase `max_retries`:

```yaml
- uses: skyhook-io/git-sync-commit@v1
  with:
    commit_message: My commit
    max_retries: 10  # Increase for high-concurrency scenarios
```

### Conflict markers in committed files

The action now includes automatic conflict resolution and a safety check to prevent committing conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`). If you see the error "Conflict markers detected in staged files!", this means:
1. A merge conflict occurred that couldn't be auto-resolved
2. The action prevented committing corrupted files
3. Review the workflow logs to understand the conflict
4. This is a safety feature to protect your repository from corrupted files

### No commit created but expected changes

Check that:
1. Your `file_pattern` matches the changed files
2. The files are actually modified
3. The action has the right `path` if using a subdirectory

### "Permission denied" errors

Ensure your workflow has write permissions:

```yaml
permissions:
  contents: write
```

For organization repositories, also check:
- Repository settings allow GitHub Actions to create and approve pull requests
- Branch protection rules allow the github-actions bot to push

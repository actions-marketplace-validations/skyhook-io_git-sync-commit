# Changelog - Migration from hisco/safe-rebase-commit

## Version 1.0.0 - Initial Skyhook Release

This is the first release of `skyhook-io/git-sync-commit`, migrated from `hisco/safe-rebase-commit@v1.0.1`.

### 🎯 Migration Summary

This action has been migrated to the Skyhook organization with improved standards, better user experience, and enhanced features while maintaining full backward compatibility with the original `hisco/safe-rebase-commit` action.

### ✨ Improvements & Changes

#### 1. **Composite Action Architecture** (Standard Alignment)
- **Before**: Used external `entrypoint.sh` script
- **After**: Fully embedded composite action with inline shell steps
- **Why**: Aligns with Skyhook standard for composite actions (see `docker-build-push-action`, `kustomize-deploy`)
- **Benefit**: Easier to maintain, better error handling, no external script dependencies

#### 2. **Input Validation** (New Feature)
- **Added**: Pre-flight validation step that checks:
  - Working directory exists
  - Commit message is provided
  - `max_retries` is a valid positive integer
- **Before**: Validation happened during execution
- **Benefit**: Fail fast with clear error messages before git operations begin

#### 3. **Enhanced Outputs** (New Feature)
- **Added**: Two new outputs:
  - `committed`: Boolean indicating if a commit was created
  - `commit_sha`: SHA of the created commit
- **Before**: No outputs provided
- **Benefit**: Enables conditional downstream steps and better workflow orchestration

#### 4. **GitHub Step Summaries** (Standard Alignment)
- **Added**: Automatic summary generation in GitHub Actions UI showing:
  - Commit status (success/failure/no changes)
  - Commit SHA and details
  - Author information
  - File patterns
- **Before**: Only console logs
- **Why**: Aligns with Skyhook standard (see `docker-build-push-action` summary generation)
- **Benefit**: Better visibility in GitHub Actions UI

#### 5. **Improved Error Handling** (Enhancement)
- **Added**: Better error messages with `::error::` annotations
- **Added**: Graceful handling of "no changes to commit" scenario
- **Before**: Generic error output
- **Benefit**: Clearer debugging and better CI/CD workflow design

#### 6. **Skyhook Branding** (Standard Alignment)
- **Updated**: Action metadata to match Skyhook organization standards:
  - Name: `Skyhook Git Sync and Commit`
  - Author: `Skyhook`
  - Branding: Purple color, git-commit icon
- **Why**: Consistency with other Skyhook actions
- **Benefit**: Professional appearance on GitHub Marketplace

#### 7. **Default Email Address** (Enhancement)
- **Before**: Custom email `actions@github.com`
- **After**: GitHub's official Actions bot email `41898282+github-actions[bot]@users.noreply.github.com`
- **Why**: Uses GitHub's recommended format for bot commits
- **Benefit**: Commits appear with proper GitHub Actions bot avatar

#### 8. **Documentation** (Improvement)
- **Added**: Comprehensive README with:
  - Multiple real-world examples
  - Migration guide from `hisco/safe-rebase-commit`
  - Troubleshooting section
  - Clear use case descriptions
  - GitOps-focused examples
- **Before**: Basic README
- **Benefit**: Easier onboarding and better developer experience

#### 9. **No-Changes Detection** (Enhancement)
- **Added**: Explicit check for staged changes before committing
- **Added**: Output `committed=false` when no changes exist
- **Before**: Would attempt commit (which would fail)
- **Benefit**: Cleaner workflow execution, no unnecessary error states

### 🔄 Backward Compatibility

**All inputs from `hisco/safe-rebase-commit@v1.0.1` are fully supported:**

| Input | Compatible | Notes |
|-------|------------|-------|
| `path` | ✅ Yes | Same behavior |
| `commit_message` | ✅ Yes | Same behavior |
| `commit_user_name` | ✅ Yes | Same default updated to `github-actions[bot]` |
| `commit_user_email` | ✅ Yes | Default updated to official GitHub Actions bot email |
| `file_pattern` | ✅ Yes | Same behavior |
| `max_retries` | ✅ Yes | Same behavior with added validation |

**Migration is a simple find-and-replace:**
```yaml
# Before
uses: hisco/safe-rebase-commit@v1.0.1

# After
uses: skyhook-io/git-sync-commit@v1
```

### 📋 Implementation Differences

#### Original Implementation (hisco/safe-rebase-commit)
```yaml
runs:
  using: 'composite'
  steps:
    - run: chmod +x ${{ github.action_path }}/entrypoint.sh
      shell: bash
    - run: ${{ github.action_path }}/entrypoint.sh
      shell: bash
      working-directory: ${{ inputs.path }}
      env:
        COMMIT_MESSAGE: ${{ inputs.commit_message }}
        COMMIT_USER_NAME: ${{ inputs.commit_user_name }}
        COMMIT_USER_EMAIL: ${{ inputs.commit_user_email }}
        FILE_PATTERN: ${{ inputs.file_pattern }}
        MAX_RETRIES: ${{ inputs.max_retries }}
```

#### New Implementation (skyhook-io/git-sync-commit)
```yaml
runs:
  using: 'composite'
  steps:
    - name: Validate inputs          # NEW: Pre-flight validation
      shell: bash
      run: |
        # Validation logic...

    - name: Sync and commit changes  # ENHANCED: Inline implementation
      id: commit                     # NEW: Step ID for outputs
      shell: bash
      working-directory: ${{ inputs.path }}
      env:
        # Same environment variables...
      run: |
        # Embedded logic with outputs...
        echo "committed=true" >> $GITHUB_OUTPUT      # NEW
        echo "commit_sha=$COMMIT_SHA" >> $GITHUB_OUTPUT  # NEW

    - name: Generate summary         # NEW: GitHub Actions summary
      if: always()
      shell: bash
      run: |
        # Summary generation...
```

### 🎨 Naming Rationale

**Why `git-sync-commit` instead of `safe-rebase-commit`?**

1. **Shorter and clearer**: "sync" captures the stash→rebase→commit→push workflow
2. **Action-focused**: Emphasizes what it does (syncs then commits)
3. **Skyhook pattern**: Matches naming of other actions (`kustomize-edit`, `kustomize-apply`)
4. **User perspective**: Users care about "syncing and committing", not implementation details
5. **Professional**: "safe" is implied in production-grade actions

### 🔧 Technical Changes

#### File Structure
```
hisco/safe-rebase-commit/          skyhook-io/git-sync-commit/
├── action.yml                      ├── action.yml (enhanced)
├── entrypoint.sh          →        ├── README.md (comprehensive)
├── README.md                       ├── CHANGELOG.md (this file)
└── LICENSE                         ├── LICENSE
                                    ├── .gitignore
                                    └── .github/
                                        └── workflows/
                                            ├── release.yml
                                            └── test.yml
```

### 📚 References

**Skyhook Standards Applied:**
- Composite action pattern: [docker-build-push-action](https://github.com/skyhook-io/docker-build-push-action)
- Validation approach: [docker-build-push-action/action.yml:195-232](https://github.com/skyhook-io/docker-build-push-action/blob/main/action.yml#L195-L232)
- Summary generation: [docker-build-push-action/action.yml:362-405](https://github.com/skyhook-io/docker-build-push-action/blob/main/action.yml#L362-L405)
- Branding standards: All Skyhook actions use purple color with relevant icons

### 🚀 Future Enhancements (Potential)

Features that could be added in future versions:

1. **GPG Signing**: Optional GPG signing of commits
2. **Branch Creation**: Automatic branch creation if it doesn't exist
3. **PR Creation**: Optional PR creation instead of direct push
4. **Conflict Resolution**: Configurable conflict resolution strategies
5. **Metrics**: Output push attempt count and timing information

---

## Original Credits

Original implementation by [@hisco](https://github.com/hisco) at [hisco/safe-rebase-commit](https://github.com/hisco/safe-rebase-commit).

This action builds upon that foundation with Skyhook standards and enhancements.

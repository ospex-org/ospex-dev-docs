# Git Workflow for Milestone-Driven Development

## Philosophy
Each milestone gets its own branch. Small, frequent commits. Clear commit messages that explain the "why" not just the "what".

## Branch Strategy

### For each repo, maintain:
- `main` - Stable, working code (or in some cases, branch name is `master`)
- `milestone-XXX-description` - Feature branches for each milestone
- `hotfix-*` - Emergency fixes (rare)

## Daily Workflow

### Starting a New Milestone
```bash
# In the relevant repo (e.g., ospex-foundry-matched-pairs)
git checkout main
git pull origin main
git checkout -b milestone-001-contract-redeployment
git push -u origin milestone-001-contract-redeployment
```

### During Development (Make commits frequently)
```bash
# After each small working change
git add .
git status  # Double-check what you're committing
git commit -m "Fix: core contract fee processing for leaderboard entry"

# Push regularly (at least daily)
git push
```

### Commit Message Format
```bash
# Good commit messages:
git commit -m "Fix: leaderboard fee processing bug in OspexCore"
git commit -m "Add: validation for position display in frontend" 
git commit -m "Update: contract addresses after redeployment"
git commit -m "Test: add odds conversion verification script"

# Bad commit messages (avoid these):
git commit -m "fix bug"
git commit -m "wip"  
git commit -m "updates"
```

## Commit Types
- **Fix**: Bug repairs
- **Add**: New features or files
- **Update**: Changes to existing functionality
- **Remove**: Deleting code/features
- **Test**: Adding or updating tests
- **Docs**: Documentation changes
- **Refactor**: Code reorganization without functionality changes

## When to Commit

### Commit After Each:
- Bug fix that works
- New function or component that compiles
- Configuration change that works
- Test that passes
- Documentation update

### DON'T Commit:
- Broken/non-compiling code (unless WIP is clearly marked)
- Debugging console.log statements
- Temporary files or secrets
- Multiple unrelated changes in one commit

## Completing a Milestone

### When milestone is done and tested:
```bash
# Switch to main and merge
git checkout main
git pull origin main
git merge milestone-001-contract-redeployment
git push origin main

# Tag the completion
git tag milestone-001-complete
git push origin milestone-001-complete

# Clean up (optional, keep for reference initially)
git branch -d milestone-001-contract-redeployment
git push origin --delete milestone-001-contract-redeployment
```

## Multi-Repo Coordination

### When a milestone touches multiple repos:
1. Create milestone branch in each affected repo
2. Update documentation repo with progress tracking
3. Commit to documentation repo after each major step

```bash
# In ospex-documentation repo
git add milestones/milestone-001-contract-redeployment.md
git commit -m "Update: milestone 001 progress - contracts deployed"
git push
```

## Emergency Rollback

### If something breaks in main:
```bash
# Quick revert of last commit
git revert HEAD
git push origin main

# Or rollback to specific commit
git reset --hard [commit-hash]
git push --force origin main  # Use with caution!
```

## Branch Naming Convention

### For milestones:
- `milestone-001-contract-redeployment`
- `milestone-002-data-consistency-fix`
- `milestone-003-mvp-definition`

### For hotfixes:
- `hotfix-oracle-connection`
- `hotfix-frontend-crash`

### For experiments (that might not work):
- `experiment-new-scoring-method`
- `spike-performance-optimization`

## Tracking Progress Across Repos

### In ospex-documentation, maintain:
```
/milestones/
  milestone-001-contract-redeployment.md
    - Links to commits in ospex-foundry repo
    - Links to commits in ospex-frontend repo
    - Current status and blockers
```

## Tools & Commands

### Useful git commands:
```bash
# See what you've changed
git diff

# See commit history
git log --oneline

# See which files changed in last few commits  
git log --name-only -5

# Check which branch you're on
git branch

# See all remote branches
git branch -a

# Undo local changes (before commit)
git checkout -- filename.js
```

### VS Code / Cursor Integration
- Use built-in git panel for staging changes
- Review diffs before committing
- Use commit message templates

## AI Session Integration

### Save AI-generated code properly:
1. Get code from AI session
2. Test it works
3. Commit with clear message: `"Add: position validation logic from AI session 2025-01-03"`
4. Save AI session notes to `/ai-sessions/` folder
5. Link session notes in commit if complex

## Red Flags (Things to Avoid)

### Branch Problems:
- Working directly on `main` branch
- Branches that live for weeks without merging
- Multiple unrelated features in one branch

### Commit Problems:
- Commits with 50+ files changed
- Commit messages that don't explain the change
- Committing broken code to main

### Process Problems:
- Not committing for days at a time
- Forgetting which branch you're working on
- Not documenting what the milestone accomplished

---

*Remember: Git is your safety net. Commit early, commit often, and your future self will thank you when you need to track down when something broke.*
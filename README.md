# Ospex Development Documentation

Internal development documentation for the Ospex peer-to-peer sports betting platform.

## What's This Repo For?

This repository contains architecture documentation, development milestones, workflows, and internal processes for the Ospex development team. This is **not** user-facing documentation - that lives in a separate repo and is deployed as a public website.

## Repository Structure

```
/architecture/          - System overviews, component docs, technical specs
/milestones/            - Milestone tracking files with completion status  
/workflows/             - Git workflow, testing procedures, deployment guides
/ai-sessions/           - Saved AI conversation notes and code sessions
```

## How to Use This Repo

### For Active Development
- Create new milestones in `/milestones/` before starting work
- Update milestone status as you complete tasks: `🟡 Planning → 🟠 In Progress → ✅ Complete`
- Document architectural decisions in `/architecture/`
- Save important AI sessions that generate code or solve problems

### For Reference
- Check `/workflows/git-workflow.md` for branching and commit procedures
- Review completed milestones to understand past decisions
- Reference system overview for current architecture state

## Current System Status
- **Version**: v2 in development (orderbook matching)
- **Network**: Polygon Amoy testnet
- **Status**: Testing phase, debugging data consistency issues

## Quick Links
- [System Overview](architecture/system-overview.md) - Complete architecture documentation
- [Git Workflow](workflows/git-workflow.md) - Development procedures
- [Current Milestones](milestones/) - Active and completed work

## User-Facing Documentation
For public documentation about how to use Ospex, see the separate `ospex-docs` repository (deployed to Heroku).
This currently shows documentation for Ospex v1.

---
*This repo supports milestone-driven development with clear documentation and progress tracking.*
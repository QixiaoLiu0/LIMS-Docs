# GearCode Team: Git Conventions & Workflow

This document outlines the standard Git workflow for our LIMS project. All team members must follow these conventions to ensure a clean codebase and prevent merge conflicts.

## 1. Branching Strategy

We use a simplified Gitflow approach.

- **`main`** : The production-ready branch. **(NEVER PUSH DIRECTLY HERE!)**
- **`develop`** : The active integration branch. All completed features are merged here for Sprint testing.
- **Feature Branches** : Created for daily work.
- **Naming Convention:** `feature/<Jira-Ticket>-<short-description>`
- **Example:** `feature/SCRUM-34-test-type-api` or `feature/SCRUM-40-frontend-ui`

## 2. Commit Message Format

We follow the **Conventional Commits** standard. Every commit must clearly state _what_ was done and _why_ . Always include the Jira ticket number.

**Structure:** `<type>: [JIRA-ID] <short summary>`

**Allowed Types:**

- `feat`: A new feature or API endpoint.
- `fix`: A bug fix.
- `docs`: Documentation changes (e.g., updating README or API docs).
- `refactor`: Code changes that neither fix a bug nor add a feature.
- `chore`: Updating build tasks, dependencies, or project skeleton.

**Good Examples:**

- `feat: [SCRUM-34] implement POST /api/test-types endpoint`
- `fix: [SCRUM-40] resolve UI rendering issue on Test Type cards`
- `docs: [SCRUM-27] add Git conventions to wiki`

## 3. Pull Request (PR) & Merge Workflow

To get your code into `develop`, you must follow these steps:

1. Make sure your local branch is up to date: `git pull origin develop`.
2. Push your feature branch to GitHub: `git push origin feature/SCRUM-XX-name`.
3. Open a Pull Request (PR) on GitHub targeting the `develop` branch.
4. **Mandatory:** Link the PR to the corresponding Jira ticket.
5. **Code Review:** At least **1 team member** must approve the PR before it can be merged.
6. Once approved, use "Squash and Merge" to keep the `develop` commit history clean.

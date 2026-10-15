# Copyright 2026 World Wide Technology (WWT)

# Contributing to Baton

Before you start, please ensure you understand the details around code of conduct and licensing mentioned in [README.md](./README.md)


If you want to help improve Baton, you can do so in three easy steps!

Step 1: Fork the repository

Step 2: Make the changes on your fork and add meaningful commit messages

Step 3: Open a pull request to merge the changes with this repository


### Code Formatting

Ensure that you run `gofmt -w file.go` against any file which you have modified to ensure consistent formatting across the project.


## Developer Rights

Access to this repository is organised into the following roles. Each role
maps to a GitHub repository permission level.

| Role        | GitHub permission | Rights                                                                                                   |
|-------------|-------------------|----------------------------------------------------------------------------------------------------------|
| Contributor | Read (fork)       | Fork the repository, open issues and pull requests from a fork.                                           |
| Developer   | Write             | Everything a contributor can do, plus: push feature branches to this repository, review and approve pull requests, label and triage issues, re-run CI. |
| Maintainer  | Maintain / Admin  | Everything a developer can do, plus: merge to `main`, manage releases, change branch protection, grant or revoke developer rights. |

### What developers may and may not do

Developers **may**:
- Create branches in this repository (please use a prefix such as `feature/`, `fix/` or `docs/`).
- Review and approve pull requests for the areas they are listed for in [`.github/CODEOWNERS`](./.github/CODEOWNERS).
- Triage issues and pull requests.

Developers **may not**:
- Push directly to `main`; all changes go through a pull request with at least one approving review.
- Change repository settings, branch protection rules or `.github/CODEOWNERS` without maintainer review.

### Requesting developer rights

1. Have at least one merged pull request in this repository.
2. Open an issue titled `Developer rights request: <your GitHub username>` describing the areas you would like to work on.
3. A maintainer grants the **Write** permission under *Settings → Collaborators and teams* and, where appropriate,
   adds you to the relevant paths in [`.github/CODEOWNERS`](./.github/CODEOWNERS) via a pull request.

Rights can be revoked by a maintainer at any time, for example after prolonged inactivity or a breach of the
[Code of Conduct](./CODE_OF_CONDUCT.md).

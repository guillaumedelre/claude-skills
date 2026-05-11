When contributing to an open source project (fixing an issue, submitting a PR):

## Reading the issue

- Read the issue in full before starting: description, every comment, and every linked resource
- Do not start implementing until you have understood the complete context; a comment thread often narrows, changes, or cancels the original request
- If the issue references related issues or PRs, read those too

## Contribution guidelines

- Look for contribution rules before touching any code: `CONTRIBUTING.md`, `CONTRIBUTING`, `.github/CONTRIBUTING.md`, relevant sections in `README.md`
- Respect coding standards, branch naming, commit message format, and PR template defined by the project
- If no guidelines exist, mirror the style of recent commits and PRs in the repository

## Documentation

- Check whether the project has user-facing documentation (docs/, wiki, external site)
- If the change affects behavior, CLI output, configuration, or public API, update the relevant documentation in the same PR
- Do not leave documentation out of sync with the implementation

## Branch naming

- Before creating a branch, inspect recent branch names in the repository (`git branch -r`) and existing PR titles to infer the project convention
- Propose a branch name that follows that convention; if no pattern is detectable, default to `type/short-description` (e.g. `fix/null-pointer-on-empty-input`)
- Ask for confirmation before creating the branch if the convention is ambiguous

## Quality checks

- Before opening a PR, run the full test suite and every quality tool present in the project
- Common tools to look for and run if present: `phpunit`, `psalm`, `phpstan`, `php-cs-fixer`, `rector`, `eslint`, `prettier`, `mypy`, `ruff`, etc.
- Detect available tools from `composer.json` (scripts section), `Makefile`, `justfile`, CI configuration (`.github/workflows/`, `.gitlab-ci.yml`), or `pyproject.toml`
- Do not open a PR with a failing test or quality violation unless the failure pre-existed and is documented

## Pull request target

- Always open the PR against the **upstream repository** (`gh pr create --repo upstream/repo --head fork:branch`), never against the fork
- Verify with `gh pr list --repo upstream/repo --author <handle>` after creation to confirm the PR landed in the right place
- If a PR was accidentally opened on the fork, close it with a comment pointing to the upstream PR, then open the correct one

## Notifying the maintainer

- After the PR is created, post a comment on the original issue to notify the maintainer: mention the PR number and a one-sentence summary of what was done
- Keep the comment factual and concise; do not repeat the full PR description

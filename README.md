# workflow_layout

Example of a two-phase release process using GitHub Actions workflows.

## Why two workflows?

A single release workflow that pushes a tag and fires downstream jobs has several problems:

- It reports success before downstream workflows complete
- Tests and artifact builds run in parallel, so images can be published before tests pass
- Downstream failures (helm chart sync, image builds) are invisible to the person who triggered the release
- Code changes during release violate PR-based branch protection policies

The two-workflow model fixes this by separating **code changes** (pre-release) from **artifact publishing** (release).

## Workflows

### Pre-release (`pre-release.yaml`)

Prepares a release by making code changes and opening a pull request.

**Trigger:** Manual dispatch
**Inputs:**
- `version` (required) -- semver version to release (e.g. `1.5.0`)
- `operand-versions` (optional) -- JSON object of operand versions for operator repos

**What it does:**
1. Parses the version to derive the release branch name (`1.5.0` -> `release-1.5`)
2. Creates the release branch from `main` if it doesn't exist
3. Creates a `pre-release-v{version}` branch
4. Updates `release.yaml` with the target version
5. Runs component-specific pre-release steps (PLACEHOLDER)
6. Opens a PR to the release branch

**Output:** A pull request against the release branch.

### Release (`release.yaml`)

Runs after the pre-release PR is merged. Builds artifacts and creates the GitHub release.

**Trigger:** Manual dispatch
**Inputs:**
- `release-branch` (required) -- the release branch to release from (e.g. `release-1.5`)

**What it does:**

```
read-version
     |
smoke-tests
     |
    tag ──────────────────────────────
     |              |                |
build-operator   build-helm      build-binaries
     |
build-bundle
     |
build-catalog
     |              |                |
     └──────── create-release ───────┘
```

The job dependency graph enforces:
- Tests must pass before any artifacts are built
- Images build in dependency order: operator -> bundle -> catalog
- Helm charts and binaries run in parallel with images (both only need the tag)
- The GitHub release is created **last**, only after all artifact jobs succeed
- No code changes, no commits -- only a tag is created

**Output:** A GitHub release with all artifact references.

### Version Gate (`version-gate.yaml`)

CI check that runs on any PR modifying `release.yaml`.

**Trigger:** Pull request (on path `release.yaml`)

**What it checks:**
- On non-main branches, the version must not be `0.0.0`
- For each dependency targeting a specific version, the corresponding GitHub release must exist

This prevents operators from releasing before their operands are available.

## release.yaml

Each component repo contains a `release.yaml` that tracks the component version and its dependencies:

```yaml
workflow_layout:
  version: "0.1.0"

dependencies:
  some_repo: "0.2.0"
```

Dependencies can be in two states:
- `"0.0.0"` -- targeting latest (used on the `main` branch)
- A specific version -- targeting a released version (used on release branches)

## Adapting to your component

Search for `PLACEHOLDER` in the workflow files. These are the sections that need component-specific logic:

- **Pre-release steps** (`pre-release.yaml`): Add your code modifications here (e.g. `make bundle`, `cargo update`, manifest regeneration)
- **Smoke tests** (`release.yaml`): Add your lint, format, and validation checks
- **Image builds** (`release.yaml`): Replace with real `docker build` / `podman build` and push commands
- **Helm chart packaging** (`release.yaml`): Add real `helm package` and cross-repo PR creation
- **Binary builds** (`release.yaml`): Add real `go build` or equivalent commands
- **Release asset uploads** (`release.yaml`): Attach built artifacts to the GitHub release

## Presentation

A slide deck walking through the two-workflow model and demo runs is in the [slides/](slides/) directory. See its [README](slides/README.md) for setup instructions.

## Shared scripts

- `.github/scripts/parse-version.sh` -- Reads `release.yaml`, validates semver, outputs version components and release branch name
- `.github/scripts/validate-release-yaml.sh` -- Validates version gating rules for the version-gate CI check

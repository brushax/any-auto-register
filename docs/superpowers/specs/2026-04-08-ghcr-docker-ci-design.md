# GHCR Docker CI Design

## Goal

When the fork receives new commits on `main`, GitHub Actions should automatically build the repository Docker image and publish it to GitHub Container Registry under `ghcr.io/brushax/any-auto-register`.

## Scope

- Add a dedicated workflow for Docker build and GHCR publish.
- Validate Docker builds on pull requests without pushing packages.
- Document the workflow behavior in the repository README.

## Design

### Workflow boundaries

Use a standalone workflow file so Docker publishing stays isolated from any future desktop release or tag-based workflows.

### Triggers

- `push` to `main`: build and push to GHCR
- `pull_request` targeting `main`: build only
- `workflow_dispatch`: manual rebuild/publish

### Registry and tags

Publish to `ghcr.io/${{ github.repository }}`. Generate:

- `latest` on the default branch
- branch tag such as `main`
- immutable `sha-<commit>` tag
- PR tags for validation builds

### Permissions

Grant `contents: read` and `packages: write`. Authenticate to GHCR with the built-in `GITHUB_TOKEN` only for non-PR runs.

### Validation

This change is configuration-only, so no new test suite is required. Validation will rely on:

- workflow file inspection
- README diff inspection
- `git diff --check`
- local Docker build preflight when environment networking allows it

## Files

- Create `.github/workflows/docker-build.yml`
- Update `README.md`

## Risks

- First publish can fail if repository Actions permissions do not allow package writes.
- Local Docker verification can be blocked by network timeouts while pulling base images.

# m1.2-split-images

Split the current monolithic Dockerfile into base + claude images.

## Goal

Create a layered image structure:
- Base image with common tools, firewall infrastructure, shell setup
- Claude image extends base with only Claude Code and aliases

## Current State

Single Dockerfile in `.devcontainer/Dockerfile` containing everything.

## Plan

### Base image (`images/base/Dockerfile`)

From debian:bookworm, include:
- System packages: less, git, procps, sudo, fzf, zsh, man-db, unzip, gnupg2, gh, iptables, ipset, iproute2, dnsutils, aggregate, jq, nano, vim, curl, wget, ca-certificates
- yq (new - for policy parsing in m1.4)
- Non-root user setup (dev, uid/gid 500)
- Command history persistence
- Workspace directory setup
- git-delta
- zsh with powerline10k
- Firewall script and sudoers setup

### Claude image (`images/agents/claude/Dockerfile`)

```dockerfile
ARG BASE_IMAGE=agent-sandbox-base:local
FROM ${BASE_IMAGE}

ARG CLAUDE_CODE_VERSION=latest
RUN curl -fsSL https://claude.ai/install.sh | bash -s $CLAUDE_CODE_VERSION

RUN echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
RUN echo "alias yolo-claude='cd /workspace && claude --dangerously-skip-permissions'" >> ~/.zshrc
RUN echo "alias yc='yolo-claude'" >> ~/.zshrc
```

### Update devcontainer

Update `.devcontainer/Dockerfile` to build from local images or use a multi-stage build that references the new structure.

## Tasks

- [x] Create `images/base/Dockerfile`
- [x] Create `images/agents/claude/Dockerfile`
- [x] Add yq to base image
- [x] Create `images/build.sh` for local builds
- [x] Add `entrypoint.sh` for compose support
- [x] Update `.devcontainer/` to use new structure
- [x] Add `docker-compose.yml`
- [x] Update README with build step and compose instructions
- [x] Verify base image builds - PASSED
- [x] Verify claude image builds - PASSED
- [x] Test compose workflow - PASSED (firewall works, claude auth works, credentials persist)
- [x] Test devcontainer still works - PASSED
- [x] Remove redundant `.devcontainer/init-firewall.sh`
- [x] Add environment variables to docker-compose.yml
- [x] Add CLAUDE.md and settings.json mounts to docker-compose.yml

## Current State (for session resume)

**COMPLETE** - Ready for PR.

Findings:
- Devcontainer bypasses entrypoint, so `postStartCommand` is still required for firewall init
- Entrypoint handles firewall for standalone docker-compose usage
- Both paths are idempotent (check for existing REJECT rule before running)

## Out of Scope

- Publishing images to GHCR (m2)
- Policy file reading (m1.4)
- Other agents (m4)

## Notes

- Firewall script stays in base with hardcoded domains for now; m1.4 will make it read from policy.yaml
- The BASE_IMAGE arg allows local builds or pulling from registry later

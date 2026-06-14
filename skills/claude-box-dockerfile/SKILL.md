---
name: claude-box-dockerfile
description: Write and maintain a project's .claude/claude-box/Dockerfile for the claude-box dev sandbox. Use when setting up claude-box for a repo, and whenever you add/remove a dependency or change a service port — the Dockerfile must be updated in tandem. Covers inheriting the base image, installing all project dependencies, and exposing host ports.
---

# Authoring a claude-box Dockerfile

[claude-box](https://github.com/ksmithbaylor/claude-box) runs Claude Code in a per-project Docker
sandbox. A repo opts in with **one** file — `.claude/claude-box/Dockerfile` — that defines the
container's toolchain. This skill is how you create and maintain it.

## When to use this skill

- Setting up claude-box in a repo for the first time (`claude-box --init` scaffolds the file).
- **Any time you add or remove a project dependency** — a language runtime, package manager, system
  library, or CLI the project needs to build, test, or run.
- **Any time the project starts using, or changes, a host port** — a dev server, API, database, etc.

Treat the Dockerfile like a lockfile for the dev environment: if the environment changes, the
Dockerfile changes in the **same** change.

## Rules

### 1. Inherit from the claude-box base image

Always start with exactly:

```dockerfile
ARG BASE_IMAGE=claude-box-base:latest
FROM ${BASE_IMAGE}
```

The base already provides Claude Code, claude-box's privilege-dropping entrypoint, and common
tooling — **Node 22, git, jq, ripgrep, curl, unzip, build-essential, python3**. Never `FROM` a
different base (you'd lose all of that and break the box), and don't reinstall what the base already
includes.

### 2. Install ALL project dependencies

Add everything the project needs that isn't in the base, so a fresh build can run the project's full
`build` / `test` / `lint` / `dev` flow with no "command not found". Inspect the repo's manifests to
know what's required — e.g. `package.json`, `deno.json`, `pyproject.toml` / `requirements.txt`,
`go.mod`, `Gemfile`, `Cargo.toml`, `.tool-versions`, `Makefile`, CI config. **Pin versions** for
reproducibility.

```dockerfile
# system packages
RUN apt-get update && apt-get install -y --no-install-recommends \
      libpq-dev postgresql-client \
 && rm -rf /var/lib/apt/lists/*

# a language toolchain, pinned, installed system-wide so it's on PATH
ARG DENO_VERSION=2.4.2
RUN curl -fsSL https://deno.land/install.sh | DENO_INSTALL=/usr/local sh -s "v${DENO_VERSION}"
```

### 3. Expose any host ports the project needs

Declare each port a service in the box listens on with `EXPOSE`. claude-box publishes EXPOSEd ports
to the host on `127.0.0.1`, letting you reach a dev server / API / db from the host. Only expose what
you actually use.

```dockerfile
EXPOSE 3000      # web dev server
EXPOSE 5432      # database
```

The service must listen on **`0.0.0.0`** inside the box (not `127.0.0.1`) for the published port to
reach it — most dev servers need an explicit host flag (e.g. `vite --host 0.0.0.0`,
`next dev -H 0.0.0.0`, or `HOST=0.0.0.0`).

### 4. Don't set a USER

Leave the image's default user as root. claude-box's entrypoint reproduces your host user and drops
privileges itself, so a `USER` instruction is unnecessary and can interfere with it. End the file
with your dependency installs / `EXPOSE` lines.

### 5. Keep it in tandem (most important)

The Dockerfile must always reflect the project's real dependencies and ports. Whenever you:

- add, upgrade, or remove a tool / runtime / system package → update the install steps;
- add, remove, or change a port a service uses → update the `EXPOSE` lines;

do it **in the same change**, not "later." Introducing a dependency or port in the code without
updating this Dockerfile silently breaks the box for the next person (including future you). After
making such a change, re-read this Dockerfile and reconcile it before you consider the work done.

## Example

```dockerfile
ARG BASE_IMAGE=claude-box-base:latest
FROM ${BASE_IMAGE}

# System libraries
RUN apt-get update && apt-get install -y --no-install-recommends \
      libpq-dev \
 && rm -rf /var/lib/apt/lists/*

# Deno, pinned to match the project toolchain
ARG DENO_VERSION=2.4.2
RUN curl -fsSL https://deno.land/install.sh | DENO_INSTALL=/usr/local sh -s "v${DENO_VERSION}"

# Ports the project's services listen on
EXPOSE 7777      # API server
EXPOSE 5173      # frontend dev server
```

## Verify

- First time: `claude-box --init`, then edit `.claude/claude-box/Dockerfile`.
- After changes: `claude-box --rebuild` to rebuild the image.
- Sanity: inside the box every tool the project needs is on `PATH`, the build/test/dev commands run,
  and services on `EXPOSE`d ports are reachable from the host.

## Checklist before finishing a change

- [ ] `FROM ${BASE_IMAGE}` (the claude-box base), not a different base.
- [ ] Every dependency the change introduced is installed here, pinned.
- [ ] Nothing the base already provides is reinstalled.
- [ ] Every host port a service needs is `EXPOSE`d (and stale ones removed).
- [ ] No `USER` instruction.
- [ ] The Dockerfile was updated in the **same** change as the code that needed it.

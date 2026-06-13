# claude-box

Run [Claude Code](https://claude.com/claude-code) in a per-project Docker sandbox, so you can use
**auto mode** (`--dangerously-skip-permissions`) without it being able to mess up your host.

Your repo and your global `~/.claude` are bind-mounted in, so editing, committing/pushing, skills,
memory, and settings all work exactly as they do outside the box. Project dependencies live in the
container. macOS notification sounds are forwarded back to the host. It's a single self-contained
bash script — the container image is defined inline and built on first use.

> Targets a **macOS host with Docker Desktop**. The audio bits are macOS-specific (and degrade
> gracefully elsewhere); the rest is portable.

## Why

Claude Code's auto mode is great for flow, but letting an agent run commands unsupervised on your
machine is risky. A container gives you a blast radius: the agent gets a real, full toolchain and
your Claude config, but can only touch what you explicitly mount — the current repo and `~/.claude`.

## Requirements

- macOS with [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
  (`brew install --cask docker`).
- `bash` (the script uses `#!/usr/bin/env bash`).
- Claude Code installed/used on the host already (so you have a `~/.claude`).

## Install

It's just one script — put it somewhere on your `PATH` and make it executable:

```sh
git clone https://github.com/ksmithbaylor/claude-box.git
install -m 0755 claude-box/claude-box /usr/local/bin
```

Or copy `claude-box` anywhere on your `PATH` by hand and `chmod +x` it. No other files are needed at
runtime.

## Quick start

```sh
cd your-project
claude-box            # first run builds the image, then launches Claude in auto mode
```

- **First run** builds the base image (a few minutes) and drops you into Claude. Authenticate with a
  host-minted token first — the box can't run interactive login (see [Authentication](#authentication)).
- **No dependencies needed?** It just works in any directory — even an empty one — using the base
  image (Node, git, jq, ripgrep, curl, unzip, build tools).
- **Need project deps?** Scaffold a tiny Dockerfile and add them:

  ```sh
  claude-box --init     # writes .claude/claude-box/Dockerfile
  # edit it to install your toolchain, then:
  claude-box
  ```

## How it works

Three layers, generic → specific:

1. **Base image** (`claude-box-base`) — Node 22 + Claude Code + common CLI tools. Defined inline in
   the script and built from stdin, tagged by a content hash so it only rebuilds when the definition
   changes. Shared across all your projects.
2. **Project image** (optional) — `<repo>/.claude/claude-box/Dockerfile`, which does `FROM` the base
   and installs that project's dependencies. With no such file, the base image is used directly.
3. **The launcher** (`claude-box`) — builds the above as needed and runs the container with the
   mounts/env below, then starts `claude`.

### What gets mounted

| Host                     | Container                   | Why                                                                                                                                                                          |
| ------------------------ | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `$PWD`                   | `$PWD` (same absolute path) | So Claude's per-project history/memory keys match the host. Edit/commit/push from your host as usual.                                                                        |
| `~/.claude`              | `~/.claude`                 | Read-write: skills, `settings.json`/statusline edits, memories, and credentials written in the box persist to the host. Also mounted at the host home path so absolute `~`-style config paths (e.g. a statusLine command) resolve.                                                                                                                     |
| `~/.claude.json`         | regenerated in a home volume | Top-level config. Bind-mounting the file directly breaks Claude's atomic-rename writes, so each run it's regenerated into the container home from yours **with `.oauthAccount` stripped** — the box authenticates via `CLAUDE_CODE_OAUTH_TOKEN`, not the host account (whose creds are in the Keychain). Container-local otherwise. |
| `~/.gitconfig`           | `~/.gitconfig` (ro)         | Commit identity (if present).                                                                                                                                                |
| a named volume           | container `~`               | Persists caches (e.g. language package caches) between runs.                                                                                                                 |
| `~/.agents` (if present) | container `~/.agents` (ro)  | For tools that symlink into `~/.claude` with relative links. Read-only — the box reads these but shouldn't rewrite them on the host. Configurable — see [Configuration](#configuration).                                                                                                                                            |

The container runs as the non-root `node` user with `IS_SANDBOX=1` so auto mode is allowed, and
`git`'s `safe.directory` is set so in-container git works on the bind-mounted repo.

### Per-project Dockerfile

`claude-box --init` scaffolds `.claude/claude-box/Dockerfile`:

```dockerfile
ARG BASE_IMAGE=claude-box-base:latest
FROM ${BASE_IMAGE}

# Example — Deno:
ARG DENO_VERSION=2.4.2
RUN curl -fsSL https://deno.land/install.sh | DENO_INSTALL=/usr/local sh -s "v${DENO_VERSION}"

USER node
```

It lives under the repo's `.claude/` directory (next to `commands/`, `hooks/`, `settings.json`), so
it's naturally tracked with your project config. See [`examples/`](examples/) for a filled-in one.

## Authentication

**Authenticate once on the host with a token — the box can't run interactive login.** Claude on
macOS keeps credentials in the **Keychain** (which the Linux box can't read), and an in-box `/login`
can't finish: it uses a loopback OAuth callback on a **random container-internal `localhost` port**
the host can't reach (Docker `-p` can't target a container-loopback listener, and Docker Desktop
host networking doesn't bridge it), and there's no flag/env to force the paste-a-code flow on the
interactive client. So mint a long-lived token instead:

```sh
claude setup-token                 # on your Mac — host browser + loopback work here
export CLAUDE_CODE_OAUTH_TOKEN=…   # add to your shell profile
```

claude-box forwards `CLAUDE_CODE_OAUTH_TOKEN` into the box, so it's authenticated on startup — no
login prompt, no browser, no localhost. `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN` /
`ANTHROPIC_BASE_URL` also work if set. (No host machine? `claude setup-token --no-browser` works
anywhere — including inside the box — and prints a token via a paste-a-code flow.)

## Sounds

The container has no audio device, so hooks that call macOS `afplay` (e.g. a `Stop` hook running
`afplay /System/Library/Sounds/Glass.aiff`) would fail. The base image ships an `afplay` shim that
drops its args into a shared spool dir, and a small background watcher in the launcher plays them
with your host player. No hook changes needed. Disable with `CLAUDE_BOX_NO_AUDIO=1`; override the
player with `CLAUDE_BOX_PLAYER`. macOS host only; skipped if the player isn't found.

## Security

**Auto mode is the whole point, so think about the trust boundary.** The container can run anything,
but can only reach what you mount. Two things to know:

- **Read-write mounts are reachable.** `~/.claude`, the repo, and any `CLAUDE_BOX_MOUNTS` paths are
  mounted read-write, so a misbehaving agent can modify _those host paths_ directly (e.g. deleting
  files under `~/.claude`). `CLAUDE_BOX_HOME_MOUNTS` (default `.agents`) are mounted read-only.
  Everything else on your machine is protected by the container. Mount only what you're comfortable
  exposing.
- **The sound channel is not an injection vector.** The watcher reads each request into an array and
  passes it to the player as quoted argv — there's no `eval` / `sh -c`, so shell metacharacters in a
  request (`;`, `$()`, backticks, …) are inert literal filenames. As an extra safeguard, the watcher
  only plays requests whose file argument is an **existing regular file with an audio extension**,
  and ignores symlinked requests. The worst a request can do is play a real sound file.

## Configuration

All optional environment variables:

| Variable                                                            | Default   | Purpose                                                                                                                                                                                                    |
| ------------------------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CLAUDE_BOX_MOUNTS`                                                 | —         | Space-separated host paths to bind-mount at the **same absolute path** in the box (e.g. absolute symlink targets, shared dirs outside the repo).                                                           |
| `CLAUDE_BOX_HOME_MOUNTS`                                            | `.agents` | Space-separated names under `$HOME` mounted **read-only** at the same name under the container home — for tools that symlink into `~/.claude` with **relative** links (the target must sit beside `.claude` in `$HOME`). |
| `CLAUDE_BOX_PLAYER`                                                 | `afplay`  | Host command used to play forwarded sounds.                                                                                                                                                                |
| `CLAUDE_BOX_NO_AUDIO`                                               | —         | Set to disable sound forwarding.                                                                                                                                                                           |
| `CLAUDE_CODE_OAUTH_TOKEN`                                           | —         | Host-minted token (`claude setup-token`); forwarded so the box authenticates without an in-box login.                                                                                                      |
| `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_BASE_URL` | —         | Forwarded into the container if set.                                                                                                                                                                       |

### Options

```
claude-box [options] [-- claude args...]
  --init      Scaffold .claude/claude-box/Dockerfile in the current repo.
  --rebuild   Force rebuild of the base and project images (no cache).
  --no-auto   Don't pass --dangerously-skip-permissions to Claude.
  -h, --help  Show help.
  --version   Print version.
```

Anything after the options (or a literal `--`) is passed to `claude`, e.g. `claude-box -- --resume`.

## Caveats

- **macOS-focused.** Auth notes and audio assume macOS + Docker Desktop. On a Linux host, audio is
  skipped unless you set `CLAUDE_BOX_PLAYER` to something present (e.g. `paplay`).
- **`~/.claude.json` is container-local** after the first run (seeded from yours). Everything under
  `~/.claude/` stays host-shared, so settings/skills/credentials are identical.
- **First build takes a few minutes** (downloads base packages + Claude Code). Subsequent runs are
  fast; deps are cached in the home volume. Changing the base definition (e.g. upgrading the script)
  triggers one rebuild and leaves the old base image as an orphan you can `docker image prune`.

## Uninstall / cleanup

```sh
rm ~/bin/claude-box                                  # remove the script
docker image rm $(docker image ls -q 'claude-box-*') # remove built images
docker volume rm $(docker volume ls -q -f name=claude-box-home)  # remove home volumes
```

## License

MIT — see [LICENSE](LICENSE).

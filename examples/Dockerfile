# Example per-project Dockerfile for claude-box.
#
# Copy to <your-repo>/.claude/claude-box/Dockerfile (or run `claude-box --init`
# and edit), then add your project's toolchain. It FROMs the shared base image,
# which already has Node, git, jq, ripgrep, curl, unzip, and build tools.

ARG BASE_IMAGE=claude-box-base:latest
FROM ${BASE_IMAGE}

# Example: install Deno, pinned to a specific version.
ARG DENO_VERSION=2.4.2
RUN curl -fsSL https://deno.land/install.sh | DENO_INSTALL=/usr/local sh -s "v${DENO_VERSION}" \
 && deno --version

# Add more project dependencies here (apt packages, language runtimes, CLIs...).

USER node

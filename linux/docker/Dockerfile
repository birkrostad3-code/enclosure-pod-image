# syntax=docker/dockerfile:1
# linux/docker/Dockerfile: pre-built RunPod pod image for the enclosure-engine neural loop.
#
# Bakes in what linux/setup-pod.sh otherwise installs on every new pod (10-15 min), at fixed paths
# OUTSIDE /workspace (RunPod mounts the pod volume over /workspace at run time):
#   /opt/enclosure/opt/cargo, /opt/enclosure/opt/rustup   rustup + toolchain $RUST_TOOLCHAIN (minimal)
#                                                          + the crates.io crates of the three Rust
#                                                          crates (cargo fetch --locked)
#   /opt/enclosure/opt/node -> node-vX.Y.Z                 Node.js $NODE_MAJOR LTS (SHA-256 checked)
#   /opt/enclosure/venv                                    venv + pip install -r linux/requirements.txt
#                                                          (torch 2.10.0+cu128)
#   /opt/enclosure/IMAGE-INFO.env                          versions and input hashes (written last;
#                                                          linux/pod-env.sh uses the image only if it exists)
# The apt packages are setup-pod.sh's APT_PKGS. linux/pod-env.sh and setup-pod.sh detect the image
# and skip the apt / rustup / Node / pip steps; without the image they behave as before.
#
# NOT in the image: project sources, checkpoints, data, Rust binaries, and no credentials of any
# kind. The Rust binaries are built at run time from the uploaded sources (linux/build-rust.sh,
# offline from the cached crates), so botSourceHash always matches the uploaded code.
# The build context is filtered by Dockerfile.dockerignore (a whitelist: linux/requirements.txt and
# the Cargo manifests/lockfiles only); they are bind-mounted during the build, never copied into a
# layer.
#
# Build (from the project root, or from the directory written by export-build-context.ps1):
#   docker build -f linux/docker/Dockerfile -t <registry>/<name>:<tag> .
# Build args: BASE_IMAGE, RUST_TOOLCHAIN (1.98.1 = the PC's rustc), NODE_MAJOR (24), NODE_VERSION
# (empty = newest NODE_MAJOR LTS at build time), BASE_PYTHON (python3, as setup-pod.sh).
# See linux/docker/README.md.

ARG BASE_IMAGE=runpod/pytorch:2.8.0-py3.11-cuda12.8.1-cudnn-devel-ubuntu22.04
FROM ${BASE_IMAGE}

ARG BASE_IMAGE
ARG RUST_TOOLCHAIN=1.98.1
ARG NODE_MAJOR=24
ARG NODE_VERSION=
ARG BASE_PYTHON=python3
ARG DEBIAN_FRONTEND=noninteractive
ARG IMG=/opt/enclosure

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

# 1. apt packages: the same list as setup-pod.sh step 3 (it then finds them installed and skips).
RUN apt-get update -y; \
    apt-get install -y --no-install-recommends build-essential pkg-config git tmux rsync curl \
      ca-certificates xz-utils zstd pigz jq procps psmisc htop unzip lsof; \
    rm -rf /var/lib/apt/lists/*

# 2. Python venv + linux/requirements.txt (the largest layer, ~7 GB: torch and its CUDA wheels).
#    Same commands as setup-pod.sh step 6; the .requirements-sha256 marker has the same format, so
#    setup-pod.sh skips pip when the pod's linux/requirements.txt is byte-identical.
RUN --mount=type=bind,source=linux/requirements.txt,target=/tmp/enclosure-requirements.txt \
    venv="$IMG/venv"; mkdir -p "$IMG"; \
    if ! "$BASE_PYTHON" -m venv "$venv"; then \
      pyv="$("$BASE_PYTHON" -c 'import sys; print("%d.%d" % sys.version_info[:2])')"; \
      apt-get update -y; apt-get install -y --no-install-recommends "python$pyv-venv"; \
      rm -rf /var/lib/apt/lists/* "$venv"; \
      "$BASE_PYTHON" -m venv "$venv"; \
    fi; \
    "$venv/bin/python" -m pip install --no-cache-dir --upgrade pip wheel; \
    "$venv/bin/python" -m pip install --no-cache-dir -r /tmp/enclosure-requirements.txt \
      --extra-index-url https://download.pytorch.org/whl/cu128; \
    sha256sum /tmp/enclosure-requirements.txt | cut -d' ' -f1 > "$venv/.requirements-sha256"; \
    "$venv/bin/python" -c 'import platform, torch, numpy, psutil; \
assert torch.version.cuda, "torch is a CPU-only build"; \
print("python", platform.python_version(), "torch", torch.__version__, "cuda", torch.version.cuda, "numpy", numpy.__version__)'

# 3. Node.js LTS (work/external/download-dataset.cjs), checked against nodejs.org SHASUMS256.txt.
#    Same layout as setup-pod.sh step 5: $OPT/node-vX.Y.Z and the symlink $OPT/node.
RUN opt="$IMG/opt"; mkdir -p "$opt" /tmp/enclosure-node; cd /tmp/enclosure-node; \
    ver="${NODE_VERSION:-}"; \
    if [ -z "$ver" ]; then \
      ver="$(curl -fsSL https://nodejs.org/dist/index.json | jq -r --arg m "v$NODE_MAJOR." \
        '[.[] | select((.version | startswith($m)) and ((.lts | type) == "string"))][0].version')"; \
    fi; \
    [[ $ver == "v$NODE_MAJOR."* ]] || { echo "no Node.js $NODE_MAJOR LTS release found (got '$ver')" >&2; exit 1; }; \
    tarball="node-$ver-linux-x64.tar.xz"; \
    curl --proto '=https' --tlsv1.2 -fsSLO "https://nodejs.org/dist/$ver/$tarball"; \
    curl --proto '=https' --tlsv1.2 -fsSLo SHASUMS256.txt "https://nodejs.org/dist/$ver/SHASUMS256.txt"; \
    line="$(grep -E "  $tarball\$" SHASUMS256.txt)" || { echo "no checksum for $tarball" >&2; exit 1; }; \
    echo "$line" | sha256sum -c --strict -; \
    rm -rf "$opt/node-$ver" "$opt/node-$ver-linux-x64"; \
    tar --no-same-owner -xJf "$tarball" -C "$opt"; \
    mv "$opt/node-$ver-linux-x64" "$opt/node-$ver"; \
    ln -sfn "$opt/node-$ver" "$opt/node"; \
    cd /; rm -rf /tmp/enclosure-node; \
    "$opt/node/bin/node" --version

# 4. Rust: rustup (minimal profile) + the pinned toolchain, as setup-pod.sh step 4 but pinned and
#    with rustup-init's published SHA-256 checked.
RUN export CARGO_HOME="$IMG/opt/cargo" RUSTUP_HOME="$IMG/opt/rustup"; \
    mkdir -p /tmp/enclosure-rustup; cd /tmp/enclosure-rustup; \
    base=https://static.rust-lang.org/rustup/dist/x86_64-unknown-linux-gnu; \
    curl --proto '=https' --tlsv1.2 -fsSLO "$base/rustup-init"; \
    curl --proto '=https' --tlsv1.2 -fsSLO "$base/rustup-init.sha256"; \
    echo "$(cut -d' ' -f1 rustup-init.sha256)  rustup-init" | sha256sum -c --strict -; \
    chmod +x rustup-init; \
    ./rustup-init -y --profile minimal --default-toolchain "$RUST_TOOLCHAIN" --no-modify-path; \
    cd /; rm -rf /tmp/enclosure-rustup; \
    "$CARGO_HOME/bin/rustup" default "$RUST_TOOLCHAIN"; \
    "$CARGO_HOME/bin/rustc" --version; "$CARGO_HOME/bin/cargo" --version

# 5. Pre-fetch the crates.io dependencies of enclosure-bot, game-import and extconv into
#    $CARGO_HOME/registry, so that linux/build-rust.sh can build with OFFLINE=1. Only the
#    manifests/lockfiles are used (bind mount), with empty stub sources in a temporary tree that is
#    deleted in the same step: no source code ends up in the image.
RUN --mount=type=bind,target=/tmp/enclosure-ctx \
    export CARGO_HOME="$IMG/opt/cargo" RUSTUP_HOME="$IMG/opt/rustup"; \
    ctx=/tmp/enclosure-ctx; t=/tmp/enclosure-cargo-fetch; rm -rf "$t"; \
    for c in rust/enclosure-bot rust/game-import work/external/convert/extconv \
             work/constellation/meridian-bot-kit/engine work/constellation/meridian-bot-kit/scout; do \
      mkdir -p "$t/$c/src"; cp "$ctx/$c/Cargo.toml" "$t/$c/"; \
      if [ -f "$ctx/$c/Cargo.lock" ]; then cp "$ctx/$c/Cargo.lock" "$t/$c/"; fi; \
      : > "$t/$c/src/lib.rs"; echo 'fn main() {}' > "$t/$c/src/main.rs"; \
    done; \
    for c in rust/enclosure-bot rust/game-import work/external/convert/extconv; do \
      "$CARGO_HOME/bin/cargo" fetch --locked --manifest-path "$t/$c/Cargo.toml"; \
    done; \
    rm -rf "$t"; \
    du -sh "$CARGO_HOME/registry"

# 6. IMAGE-INFO.env (last: linux/pod-env.sh only switches to the image paths when it exists).
#    CARGO_LOCKS_SHA256 = sha256 of the three Cargo.lock files concatenated in this order with CRs
#    removed; setup-pod.sh computes the same and builds offline only when they match.
RUN --mount=type=bind,target=/tmp/enclosure-ctx \
    export CARGO_HOME="$IMG/opt/cargo" RUSTUP_HOME="$IMG/opt/rustup"; \
    ctx=/tmp/enclosure-ctx; \
    locks="$(cat "$ctx/rust/enclosure-bot/Cargo.lock" "$ctx/rust/game-import/Cargo.lock" \
      "$ctx/work/external/convert/extconv/Cargo.lock" | tr -d '\r' | sha256sum | cut -c1-64)"; \
    { echo "IMAGE_SCHEMA=1"; \
      echo "IMAGE_BUILT_UTC=$(date -u +%Y-%m-%dT%H:%M:%SZ)"; \
      echo "BASE_IMAGE=${BASE_IMAGE:-unknown}"; \
      echo "RUST_TOOLCHAIN=$RUST_TOOLCHAIN"; \
      echo "RUSTC_VERSION=$("$CARGO_HOME/bin/rustc" --version)"; \
      echo "NODE_VERSION=$("$IMG/opt/node/bin/node" --version)"; \
      echo "PYTHON_VERSION=$("$IMG/venv/bin/python" -c 'import platform; print(platform.python_version())')"; \
      echo "TORCH_VERSION=$("$IMG/venv/bin/python" -c 'import torch; print(torch.__version__)')"; \
      echo "REQUIREMENTS_SHA256=$(cat "$IMG/venv/.requirements-sha256")"; \
      echo "CARGO_LOCKS_SHA256=$locks"; \
    } > "$IMG/IMAGE-INFO.env"; \
    cat "$IMG/IMAGE-INFO.env"

LABEL org.opencontainers.image.title="enclosure-pod" \
      org.opencontainers.image.description="RunPod base + toolchains for enclosure-engine (Rust, Node, torch venv). No project code, data or credentials."
# CMD/ENTRYPOINT are inherited from the RunPod base image (/start.sh: SSH with PUBLIC_KEY, Jupyter).

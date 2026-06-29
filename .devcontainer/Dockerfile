# devcontainer Dockerfile for runtimed

FROM quay.io/jupyter/base-notebook:ubuntu-24.04

USER root

# Install system libraries for jupyter-zmq-client and useful CLI tools
RUN --mount=type=cache,target=/var/cache/apt,sharing=shared \
    --mount=type=cache,target=/var/lib/apt,sharing=shared \
    apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        llvm clang libclang-dev \
        libzmq3-dev \
        pkg-config \
        ca-certificates \
        git git-man \
        gh \
        curl \
        bash-completion \
        less \
        ripgrep \
        fd-find \
        bat \
        jq


# Grant jovyan passwordless sudo (no start.sh hacks required)
RUN echo "jovyan ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/jovyan-nopasswd \
    && chmod 0440 /etc/sudoers.d/jovyan-nopasswd

USER ${NB_USER}

# Install Python dependencies and register Jupyter kernels
COPY .github/requirements.txt /tmp/requirements.txt
RUN --mount=type=cache,target=/home/jovyan/.cache/pip,uid=1000 \
    pip install -r /tmp/requirements.txt \
    && python -m ipykernel install --user --name=python \
    && python -m ipykernel install --user --name=python3

# Install Rust (minimal profile) and cargo tools.
# rust-toolchain.toml will switch to the pinned toolchain on first `cargo` invocation.
RUN --mount=type=cache,target=/home/jovyan/.rustup,uid=1000 \
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \
        | sh -s -- -y --profile minimal \
    && . "${HOME}/.cargo/env" \
    && rustup component add llvm-tools-preview \
    && cargo install cargo-llvm-cov --locked \
    && cargo install cargo-insta --locked

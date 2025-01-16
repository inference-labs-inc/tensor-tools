# Based off of subtensor builder from https://github.com/opentensor/subtensor/blob/main/Dockerfile
FROM ubuntu:24.04 AS builder
SHELL ["/bin/bash", "-c"]

# Set noninteractive mode for apt-get
ARG DEBIAN_FRONTEND=noninteractive

# Set up Rust environment
ENV RUST_BACKTRACE=1
RUN apt-get update && \
  apt-get install -y curl build-essential protobuf-compiler clang git pkg-config libssl-dev && \
  rm -rf /var/lib/apt/lists/*

RUN set -o pipefail && curl https://sh.rustup.rs -sSf | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"
RUN rustup update stable
RUN rustup target add wasm32-unknown-unknown --toolchain stable

# Copy entire repository
RUN git clone https://github.com/opentensor/subtensor.git /build
WORKDIR /build

# Build the project
RUN cargo build --workspace --profile release --features pow-faucet --manifest-path "Cargo.toml" --locked

# Verify the binary was produced
RUN test -e /build/target/release/node-subtensor

# Final devcontainer
FROM mcr.microsoft.com/devcontainers/base:noble

# Install dependencies and some nice to have tools
RUN apt update && \
    apt install -y \
    python3-dev \
    python3-venv \
    pipx \
    build-essential \
    jq \
    git \
    aria2 \
    curl \
    make \
    clang \
    pkg-config \
    libssl-dev \
    llvm \
    libudev-dev \
    protobuf-compiler \
    byobu \
    fish \
    wget \
    && apt clean && rm -rf /var/lib/apt/lists/*

# Microsoft devcontainers have a default user vscode (1000:1000) already created
USER vscode
WORKDIR /home/vscode

# Install Rust
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/home/vscode/.cargo/bin:${PATH}"

# Install subtensor
RUN git clone https://github.com/opentensor/subtensor.git
    # cd subtensor && \
    # cargo build --workspace --profile release --features pow-faucet --manifest-path "Cargo.toml" --locked
# Copy subtensor binary
COPY --from=builder /build/target/release/node-subtensor /home/vscode/subtensor/target/release/node-subtensor

# # Install Jolt
# ENV RUST_TOOLCHAIN=nightly-2024-09-30
# RUN rustup toolchain install ${RUST_TOOLCHAIN} && \
#     cargo +${RUST_TOOLCHAIN} install --git https://github.com/a16z/jolt --force --bins jolt

# Install node et al.
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash && \
    export NVM_DIR="/home/vscode/.nvm" && \
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" && \
    [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion" && \
    nvm install 20 && \
    npm install --prefix /home/vscode/.snarkjs snarkjs@0.7.4 && \
    mkdir -p /home/vscode/.local/bin && \
    ln -s $(which node) /home/vscode/.local/bin/node && \
    ln -s $(which npm) /home/vscode/.local/bin/npm
USER root
RUN install -m 755 /home/vscode/.local/bin/node /usr/bin/node && \
    install -m 755 /home/vscode/.local/bin/npm /usr/bin/npm
USER vscode

# Install uv then use uv to install torch and bittensor-cli to ~/.venv
RUN pipx install uv && \
    /home/vscode/.local/bin/uv venv && \
    /home/vscode/.local/bin/uv pip install torch --index-url https://download.pytorch.org/whl/cpu && \
    /home/vscode/.local/bin/uv pip install bittensor-cli && \
    /home/vscode/.local/bin/uv cache clean
ENV PATH="/home/vscode/.local/bin:${PATH}"
ENV PATH="/home/vscode/.venv/bin:${PATH}"

# Set wallet path so btcli doesn't prompt for it (still does even with config set)
ENV WPATH="--wallet.path /home/vscode/.bittensor/wallets"

# Set btcli config.yml and create wallets
RUN yes | btcli config set --network ws://127.0.0.1:9944    $WPATH && \
    btcli wallet new_coldkey --no-use-password --n-words 15 $WPATH --wallet.name owner && \
    btcli wallet new_hotkey  --no-use-password --n-words 15 $WPATH --wallet.name owner --wallet.hotkey default && \
    btcli wallet new_coldkey --no-use-password --n-words 15 $WPATH --wallet.name miner && \
    btcli wallet new_hotkey  --no-use-password --n-words 15 $WPATH --wallet.name miner --wallet.hotkey default && \
    btcli wallet new_coldkey --no-use-password --n-words 15 $WPATH --wallet.name validator && \
    btcli wallet new_hotkey  --no-use-password --n-words 15 $WPATH --wallet.name validator --wallet.hotkey default

# Mint some tokens
RUN cd subtensor && \
    BUILD_BINARY=0 ./scripts/localnet.sh > /dev/null 2>&1 & \
    sleep 5 && \
    yes | btcli wallet faucet --wallet.name owner & \
    sleep 180 && \
    pkill -2 localnet ; \
    pkill -2 subtensor ; \
    sleep 1

RUN cd subtensor && \
    BUILD_BINARY=0 ./scripts/localnet.sh --no-purge > /dev/null 2>&1 & \
    sleep 5 && \
    yes | btcli wallet faucet --wallet.name miner & \
    sleep 180 && \
    pkill -2 localnet ; \
    pkill -2 subtensor ; \
    sleep 1

RUN cd subtensor && \
    BUILD_BINARY=0 ./scripts/localnet.sh --no-purge > /dev/null 2>&1 & \
    sleep 5 && \
    yes | btcli wallet faucet --wallet.name validator & \
    sleep 180 && \
    pkill -2 localnet ; \
    pkill -2 subtensor ; \
    sleep 1

# Register the subnet
RUN cd subtensor && \
    BUILD_BINARY=0 ./scripts/localnet.sh --no-purge > /dev/null 2>&1 & \
    sleep 5 && \
    btcli subnet create   --wallet.name owner     --no-prompt                                      && \
    btcli subnet register --wallet.name miner     --no-prompt --netuid 1   --wallet.hotkey default && \
    btcli subnet register --wallet.name validator --no-prompt --netuid 1   --wallet.hotkey default && \
    btcli root nominate   --wallet.name validator --no-prompt              --wallet.hotkey default && \
    btcli stake add       --wallet.name validator --no-prompt --amount 100 --wallet.hotkey default && \
    pkill -2 localnet ; \
    pkill -2 subtensor ; \
    sleep 1

# Add scripts to start and stop the localnet
RUN echo "#!/usr/bin/env bash" > ~/.local/bin/start_localnet.sh && \
    echo "cd ~/subtensor" >> ~/.local/bin/start_localnet.sh && \
    echo "BUILD_BINARY=0 ./scripts/localnet.sh --no-purge > /dev/null 2>&1 &" >> ~/.local/bin/start_localnet.sh && \
    chmod +x ~/.local/bin/start_localnet.sh && \
    echo "#!/usr/bin/env bash" > ~/.local/bin/start_localnet_foreground.sh && \
    echo "cd ~/subtensor" >> ~/.local/bin/start_localnet_foreground.sh && \
    echo "BUILD_BINARY=0 ./scripts/localnet.sh --no-purge" >> ~/.local/bin/start_localnet_foreground.sh && \
    chmod +x ~/.local/bin/start_localnet_foreground.sh && \
    echo "#!/usr/bin/env bash" > ~/.local/bin/stop_localnet.sh && \
    echo "pkill -2 localnet" >> ~/.local/bin/stop_localnet.sh && \
    echo "pkill -2 subtensor" >> ~/.local/bin/stop_localnet.sh && \
    chmod +x ~/.local/bin/stop_localnet.sh

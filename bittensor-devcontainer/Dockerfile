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

# Install Rust
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

# Install subtensor
RUN git clone https://github.com/opentensor/subtensor.git
    # cd subtensor && \
    # cargo build --workspace --profile release --features pow-faucet --manifest-path "Cargo.toml" --locked
# Copy subtensor binary
COPY --from=builder /build/target/release/node-subtensor /subtensor/target/release/node-subtensor

# # Install Jolt
# ENV RUST_TOOLCHAIN=nightly-2024-09-30
# RUN rustup toolchain install ${RUST_TOOLCHAIN} && \
#     cargo +${RUST_TOOLCHAIN} install --git https://github.com/a16z/jolt --force --bins jolt

# Install node et al.
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash && \
    export NVM_DIR="/root/.nvm" && \
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" && \
    [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion" && \
    nvm install 20 && \
    npm install --prefix /root/.snarkjs snarkjs@0.7.4 && \
    ln -s $(which node) /usr/bin/node && \
    ln -s $(which npm) /usr/bin/npm

# Install uv then use uv to install torch and bittensor-cli to /opt/.venv
RUN pipx install uv && \
    cd /opt && \
    /root/.local/bin/uv venv && \
    /root/.local/bin/uv pip install torch --index-url https://download.pytorch.org/whl/cpu && \
    /root/.local/bin/uv pip install bittensor-cli && \
    /root/.local/bin/uv cache clean
ENV PATH="/root/.local/bin:${PATH}"
ENV PATH="/opt/.venv/bin:${PATH}"

# Set flags for local subtensor
ENV WPATH="--wallet.path /root/.bittensor/wallets/"
ENV LOCALNET="--subtensor.chain_endpoint ws://127.0.0.1:9944"

# Create wallets
RUN btcli wallet new_coldkey --no-use-password --n-words 15 $WPATH --wallet.name owner && \
    btcli wallet new_hotkey  --no-use-password --n-words 15 $WPATH --wallet.name owner --wallet.hotkey default && \
    btcli wallet new_coldkey --no-use-password --n-words 15 $WPATH --wallet.name miner && \
    btcli wallet new_hotkey  --no-use-password --n-words 15 $WPATH --wallet.name miner --wallet.hotkey default && \
    btcli wallet new_coldkey --no-use-password --n-words 15 $WPATH --wallet.name validator && \
    btcli wallet new_hotkey  --no-use-password --n-words 15 $WPATH --wallet.name validator --wallet.hotkey default

# Mint some tokens
RUN cd subtensor && \
    BUILD_BINARY=0 ./scripts/localnet.sh > /dev/null 2>&1 & \
    sleep 5 && \
    yes | btcli wallet faucet --wallet.name owner $WPATH $LOCALNET & \
    sleep 180 && \
    pkill -2 localnet ; \
    pkill -2 subtensor ; \
    sleep 1

RUN cd subtensor && \
    BUILD_BINARY=0 ./scripts/localnet.sh --no-purge > /dev/null 2>&1 & \
    sleep 5 && \
    yes | btcli wallet faucet --wallet.name miner $WPATH $LOCALNET & \
    sleep 180 && \
    pkill -2 localnet ; \
    pkill -2 subtensor ; \
    sleep 1

RUN cd subtensor && \
    BUILD_BINARY=0 ./scripts/localnet.sh --no-purge > /dev/null 2>&1 & \
    sleep 5 && \
    yes | btcli wallet faucet --wallet.name validator $WPATH $LOCALNET & \
    sleep 180 && \
    pkill -2 localnet ; \
    pkill -2 subtensor ; \
    sleep 1

# Register the subnet
RUN cd subtensor && \
    BUILD_BINARY=0 ./scripts/localnet.sh --no-purge > /dev/null 2>&1 & \
    sleep 5 && \
    btcli subnet create   --wallet.name owner     --no-prompt                                      $WPATH $LOCALNET && \
    btcli subnet register --wallet.name miner     --no-prompt --netuid 1   --wallet.hotkey default $WPATH $LOCALNET && \
    btcli subnet register --wallet.name validator --no-prompt --netuid 1   --wallet.hotkey default $WPATH $LOCALNET && \
    btcli root nominate   --wallet.name validator --no-prompt              --wallet.hotkey default $WPATH $LOCALNET && \
    btcli stake add       --wallet.name validator --no-prompt --amount 100 --wallet.hotkey default $WPATH $LOCALNET && \
    pkill -2 localnet ; \
    pkill -2 subtensor ; \
    sleep 1

# Add scripts to start and stop the localnet, and aliases for common btcli commands
RUN echo "#!/usr/bin/env bash" > /start_localnet.sh && \
    echo "cd /subtensor" >> /start_localnet.sh && \
    echo "BUILD_BINARY=0 ./scripts/localnet.sh --no-purge > /dev/null 2>&1 &" >> /start_localnet.sh && \
    chmod +x /start_localnet.sh && \
    echo "#!/usr/bin/env bash" > /stop_localnet.sh && \
    echo "pkill -2 localnet" >> /stop_localnet.sh && \
    echo "pkill -2 subtensor" >> /stop_localnet.sh && \
    chmod +x /stop_localnet.sh && \
    echo 'alias btcliwo="btcli wallet overview '"$WPATH"' '"$LOCALNET"'"' >> ~/.bashrc && \
    fish -c 'alias --save btcliwo="btcli wallet overview '"$WPATH"' '"$LOCALNET"'"' && \
    echo 'alias btclisl="btcli subnet list '"$LOCALNET"'"' >> ~/.bashrc && \
    fish -c 'alias --save btclisl="btcli subnet list '"$LOCALNET"'"' && \
    echo "source /opt/venv/bin/activate" >> ~/.bashrc && \
    echo "source /opt/venv/bin/activate.fish" >> ~/.config/fish/config.fish

FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# polkit + systemd + dbus
RUN apt-get update && apt-get install -y \
    policykit-1 \
    dbus \
    systemd \
    systemd-sysv \
    util-linux \
    && rm -rf /var/lib/apt/lists/*

# Testing tools
RUN apt-get update && apt-get install -y \
    expect \
    passwd \
    procps \
    strace \
    dbus-x11 \
    && rm -rf /var/lib/apt/lists/*

# Build dependencies (for building polkit from source if needed)
RUN apt-get update && apt-get install -y \
    build-essential \
    meson \
    ninja-build \
    pkg-config \
    libglib2.0-dev \
    libduktape207 \
    libsystemd-dev \
    libdbus-1-dev \
    libexpat1-dev \
    libpam0g-dev \
    libgirepository1.0-dev \
    && rm -rf /var/lib/apt/lists/*

# General tools + Node.js 20+
RUN apt-get update && apt-get install -y \
    git \
    curl \
    wget \
    jq \
    ca-certificates \
    gnupg \
    && mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
       | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" \
       > /etc/apt/sources.list.d/nodesource.list \
    && apt-get update && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/*

# GitHub CLI
RUN (type -p wget >/dev/null || apt-get install -y wget) \
    && mkdir -p -m 755 /etc/apt/keyrings \
    && wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg \
       | tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
    && chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
       | tee /etc/apt/sources.list.d/github-cli-stable.list > /dev/null \
    && apt-get update && apt-get install -y gh \
    && rm -rf /var/lib/apt/lists/*

# Gemini CLI
RUN npm install -g @google/gemini-cli

# Non-root test user with known password
RUN useradd -m testuser && echo "testuser:testpass" | chpasswd

# Workspace directories
RUN mkdir -p /workspace/output/skills /workspace/relevant-skills /workspace/polkit-src

# D-Bus runtime directory
RUN mkdir -p /run/dbus

# systemd as PID 1
CMD ["/sbin/init"]

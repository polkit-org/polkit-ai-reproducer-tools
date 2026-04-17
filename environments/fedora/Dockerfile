FROM fedora:latest

# polkit + systemd + dbus
RUN dnf install -y \
    polkit \
    dbus-daemon \
    systemd \
    util-linux \
    && dnf clean all

# Testing tools
RUN dnf install -y \
    expect \
    passwd \
    procps-ng \
    strace \
    dbus-tools \
    && dnf clean all

# Build dependencies (for building polkit from source if needed)
RUN dnf install -y \
    gcc \
    meson \
    ninja-build \
    pkgconf \
    glib2-devel \
    duktape-devel \
    systemd-devel \
    dbus-devel \
    expat-devel \
    pam-devel \
    gobject-introspection-devel \
    && dnf clean all

# General tools
RUN dnf install -y \
    git \
    curl \
    jq \
    nodejs \
    && dnf clean all

# GitHub CLI
RUN dnf install -y 'dnf-command(config-manager)' && \
    dnf config-manager addrepo --from-repofile=https://cli.github.com/packages/rpm/gh-cli.repo && \
    dnf install -y gh && \
    dnf clean all

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

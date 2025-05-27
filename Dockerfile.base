# Step 1: Base Image
FROM debian:bookworm-slim

# Step 2: Set environment variables for non-interactive apt and locale
USER root
ENV DEBIAN_FRONTEND=noninteractive
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US.UTF-8
ENV LC_ALL=en_US.UTF-8

# Step 3: Install essential system packages, including 'locales'
RUN apt-get update && \
    apt-get upgrade -y --no-install-recommends --no-install-suggests && \
    apt-get install -y --no-install-recommends --no-install-suggests \
        bash \
        ca-certificates \
        curl \
        git \
        gnupg \
        locales \
        procps \
        sudo \
        vim \
        xz-utils && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Step 4: Configure and generate the en_US.UTF-8 locale
RUN echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen en_US.UTF-8 && \
    update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 LANGUAGE=en_US.UTF-8

# Step 5: Install Docker Client and Plugins (as root)
RUN install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
    chmod a+r /etc/apt/keyrings/docker.gpg && \
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    tee /etc/apt/sources.list.d/docker.list > /dev/null
RUN apt-get update && \
    apt-get install -y --no-install-recommends --no-install-suggests \
        docker-ce-cli \
        docker-buildx-plugin \
        docker-compose-plugin && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# Optional: symlink docker-compose for convenience if scripts expect /usr/bin/docker-compose
RUN ln -s /usr/libexec/docker/cli-plugins/docker-compose /usr/local/bin/docker-compose || true

# Step 6: Create 'coder' user, grant passwordless sudo, add to 'docker' and 'sudo' groups
ARG USERNAME=coder
ARG USER_UID=1000
ARG USER_GID=$USER_UID

RUN groupadd --gid $USER_GID $USERNAME || true # Allow if group already exists
RUN useradd --uid $USER_UID --gid $USER_GID --create-home -m $USERNAME -s /bin/bash || true # Allow if user already exists
RUN usermod -aG sudo $USERNAME
# Add user to 'docker' group to run Docker commands without sudo (if Docker socket is mounted)
# Note: Docker daemon itself is expected to run separately (e.g., DinD or host Docker)
RUN groupadd docker || true # Ensure docker group exists
RUN usermod -aG docker $USERNAME
RUN echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" > "/etc/sudoers.d/$USERNAME-nopasswd" && \
    chmod 0440 "/etc/sudoers.d/$USERNAME-nopasswd"

# Step 7: Prepare /nix directory for coder's Nix installation (as root)
# This ensures the 'coder' user can install Nix here, especially if /nix is a volume.
RUN mkdir -p /nix && \
    chown $USERNAME:$USERNAME /nix

# Step 8: Set default shell for subsequent RUN commands to bash
SHELL ["/bin/bash", "-c"]

# Step 9: Switch to 'coder' user for Nix installation and user-specific setup
USER $USERNAME
WORKDIR /home/$USERNAME

# Step 10: Install Nix (single-user installation by 'coder')
RUN curl -L https://nixos.org/nix/install | sh -s -- --no-daemon

# Step 11: Define environment variables for Nix setup
ENV NIX_ENV_SCRIPT="/home/$USERNAME/.nix-profile/etc/profile.d/nix.sh"
# Update PATH to include user's Nix profile and system Nix profile (for Nix binaries)
ENV PATH="/home/$USERNAME/.nix-profile/bin:/nix/var/nix/profiles/default/bin:$PATH"

# Step 12: Configure Nix (user-specific) for Flakes and set nix-path for channels
RUN mkdir -p "/home/$USERNAME/.config/nix" && \
    echo "experimental-features = nix-command flakes" >> "/home/$USERNAME/.config/nix/nix.conf" && \
    # $HOME will expand to /home/$USERNAME when sourced or used by Nix tools run as $USERNAME
    echo "nix-path = nixpkgs=\$HOME/.nix-defexpr/channels/nixpkgs" >> "/home/$USERNAME/.config/nix/nix.conf"

# Step 13: Add Nix channels and update (as 'coder')
# This runs after Nix is installed and nix.conf is set up.
# Needs to source nix.sh for nix-channel to be in PATH and for nix.conf to be effective.
RUN . "$NIX_ENV_SCRIPT" && \
    nix-channel --add https://nixos.org/channels/nixpkgs-unstable nixpkgs && \
    nix-channel --update

# Step 14: Install direnv and nix-direnv using Nix profiles (as 'coder')
# Needs to source nix.sh for 'nix profile' command and for nix.conf (with nix-path) to be respected.
RUN . "$NIX_ENV_SCRIPT" && \
    nix profile install nixpkgs#direnv nixpkgs#nix-direnv

# Step 15: Modify .bashrc for interactive shells for 'coder'
RUN { \
        echo -e "\n# Source Nix environment script if it exists"; \
        echo "if [ -f \"\$NIX_ENV_SCRIPT\" ]; then . \"\$NIX_ENV_SCRIPT\"; fi"; \
        echo -e "\n# Initialize direnv hook"; \
        echo 'eval "$(direnv hook bash)"'; \
    } >> "/home/$USERNAME/.bashrc"

# Step 16: Verify Nix installation (optional, build-time check)
RUN . "$NIX_ENV_SCRIPT" && nix --version
RUN . "$NIX_ENV_SCRIPT" && nix flake --version
RUN . "$NIX_ENV_SCRIPT" && direnv version # Verify direnv

# Step 17: Default command to keep the container running
CMD ["/bin/bash"]

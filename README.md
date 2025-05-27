# Universal Development Environment (Nix, Docker, Direnv)

This repository defines a reproducible and feature-rich development environment using Docker, Nix (with Flakes and channel support), Docker-in-Docker, and Direnv. It's designed to be used with VS Code Dev Containers and as a base for Coder (coder.com) workspaces.

## Purpose

The goal of this environment is to provide a consistent and powerful development setup that includes:
- A Debian Bookworm base.
- Nix for package management and reproducible shells.
- Full Docker capabilities (via Docker-in-Docker for local use, or host Docker socket for Coder).
- Direnv for automatic environment loading per project.
- Essential command-line tools and `vim`.

## Prerequisites

To use this development environment locally, you'll need:
- **Docker Desktop** (or Docker Engine with Docker Compose) installed and running.
- **Git** installed.
- **VS Code** with the **Dev Containers** extension (or a compatible editor like Cursor).

## How to Use (Local Development with VS Code/Cursor)

1.  **Clone this Repository (Optional - for testing the definition itself):**
    While this repository *defines* the environment, you typically wouldn't clone this repo to do your *actual project work* inside it. Instead, your project would have its own `.devcontainer/devcontainer.json` (or use this one if it's a shared base definition). However, to test changes to this environment definition itself:
    ```bash
    git clone <URL_of_this_devenvs_repo>
    cd devenvs 
    ```

2.  **Open in VS Code/Cursor:**
    * Open the `devenvs` folder (or your project folder that uses this dev container definition) in VS Code or Cursor.
    * If prompted, click **"Reopen in Container"**.
    * Alternatively, open the Command Palette (Ctrl+Shift+P or Cmd+Shift+P) and select "Dev Containers: Reopen in Container" or "Dev Containers: Rebuild and Reopen in Container".

3.  **First-Time Setup (Inside the Container):**
    * The `postCreateCommand` in `.devcontainer/devcontainer.json` (and the `Dockerfile`) attempts to set up Nix channels. If you need to manually refresh or add a different channel:
      ```bash
      # Inside the dev container terminal
      nix-channel --add [https://nixos.org/channels/nixpkgs-unstable](https://nixos.org/channels/nixpkgs-unstable) nixpkgs # Or your preferred channel
      nix-channel --update
      ```

## Key Features & Included Tools

* **Base OS:** Debian 12 (Bookworm) Slim
* **User:** `coder` (UID/GID 1000) with passwordless `sudo` access.
* **Nix:**
    * Single-user installation.
    * Flakes and `nix-command` enabled by default.
    * `nix-path` configured in `~/.config/nix/nix.conf` to find `<nixpkgs>` via channels.
* **Docker:**
    * Docker client, Docker Buildx, and Docker Compose plugin installed in the dev environment.
    * When used with the provided `docker-compose.yml` locally, a Docker-in-Docker (`dind`) service provides a Docker daemon. The `dev-env` service is configured with `DOCKER_HOST=tcp://dind:2376`.
* **Direnv:**
    * Installed via Nix.
    * Hooked into Bash (`~/.bashrc`). Create `.envrc` files in your projects with `use nix` or `use flake` to automatically load Nix environments.
* **Other Tools:**
    * `git`
    * `curl`, `xz-utils`, `gnupg`
    * `vim`
    * `locales` (en_US.UTF-8 configured)
    * Standard Debian utilities.
* **Persistence (when used with `docker-compose.yml`):**
    * `/home/coder`: The entire user home directory is persisted using the `coder_home` named volume. This includes shell history, Nix user profile, Nix channels data, local configurations, etc.
    * `/nix`: The Nix store is persisted using the `nix_store` named volume.
    * `/workspace`: Your project files (from the directory containing `docker-compose.yml`) are mounted here.

## How to Use with Coder

This repository (specifically its `.devcontainer/devcontainer.json`, `docker-compose.yml`, and `Dockerfile.base`) can be used as the source for a Coder template:

1.  Push this `devenvs` repository (containing these three files) to a Git provider (e.g., GitHub, GitLab).
2.  In your Coder deployment, create a new template.
3.  In the template's Terraform configuration (`main.tf`), use the `coder_parameter` to ask for the Git repository URL and point the `data "coder_envbuilder_image"` resource to this repository and its `.devcontainer/devcontainer.json` file. (See example `main.tf` discussed previously).
4.  Coder's Envbuilder will use these files to build the workspace image.
5.  **Docker Access in Coder:** The `coder_docker_workspace` resource in the `main.tf` typically mounts the host's Docker socket, providing Docker access. The `dind` service from the `docker-compose.yml` is primarily for local testing and might not be used by the Coder runtime if the host socket is mounted.

## Customization

* **System Packages:** Add to the `apt-get install` list in `Dockerfile.base`.
* **Nix Packages (Global for User):** Use `nix profile install nixpkgs#yourpackage` inside the container (will be persisted in `~/.nix-profile` via the `coder_home` volume).
* **Project-Specific Environments:** Use `shell.nix` or `flake.nix` within your actual project directories (cloned into `/workspace`) and leverage `direnv` with `use nix` or `use flake` in an `.envrc` file for automatic activation.

## .gitignore

A `.gitignore` file is included to exclude common OS-specific files, editor temporary files, and local Docker Compose overrides from being committed to *this* repository.

---

This `README.md` should give a good overview. You can, of course, tailor it further with more specific details about your intended use cases or any advanced configurations you add!
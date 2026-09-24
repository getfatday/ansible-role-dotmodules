# ansible-role-dotmodules

A modular Ansible role for managing dotfiles and macOS configuration. This role provides a flexible system for organizing dotfiles into modules, with support for both traditional GNU Stow deployment and intelligent file merging for shared configuration files.

## Features

- **Modular Dotfile Management**: Organize dotfiles into logical modules (shell, git, dev-tools, etc.)
- **GNU Stow Integration**: Leverages GNU Stow for clean symlink-based dotfile deployment
- **File Merging Support**: Intelligent merging of shared configuration files (e.g., `.zshrc`, `.bashrc`)
- **Conflict Resolution**: Automatic detection and resolution of file strategy conflicts
- **Homebrew Integration**: Seamless package management via `community.general` modules
- **Mac App Store Integration**: App installation via `geerlingguy.mac.mas`
- **Ansible Best Practices**: Follows Ansible conventions and uses recommended modules

## File Merging Strategy

When multiple modules need to contribute to the same file (e.g., `.zshrc`), the role uses an intelligent merging strategy:

```yaml
# Module A (shell-zsh/config.yml)
mergeable_files:
  - ".zshrc"

# Module B (dev-tools-zsh/config.yml)  
mergeable_files:
  - ".zshrc"
```

**Result**: A merged `.zshrc` file with clear attribution:
```bash
# =============================================================================
# SHELL-ZSH MODULE CONTRIBUTION
# =============================================================================
eval "$(starship init zsh)"
setopt AUTO_CD

# =============================================================================
# DEV-TOOLS-ZSH MODULE CONTRIBUTION
# =============================================================================
source /opt/homebrew/share/zsh-autosuggestions/zsh-autosuggestions.zsh
alias grep='grep --color=auto'
```

**Conflict Resolution**: The role automatically detects conflicts between merge and stow strategies, providing clear error messages with resolution options.

## Requirements

- **Operating System:** macOS
- **Ansible:** Version 2.9 or higher is recommended.
- **Homebrew:** Must be installed on the target machine (typically installed by the [ansible-control-bootstrap](https://github.com/getfatday/ansible-control-bootstrap) script before this role runs).
- **Dependencies:** This role requires the `community.general` Ansible collection, which provides the Homebrew modules (`homebrew`, `homebrew_cask`, `homebrew_tap`) used to manage Homebrew packages.

## Role Variables

The following variables can be set to customize the behavior of this role:

### Core Configuration

- **`dotmodules.repo`**
  URL or path to the dotmodules repository.
  *Default:* `"https://github.com/getfatday/dotmodules.git"`

- **`dotmodules.dest`**
  Destination directory for the cloned repository.
  *Default:* `"{{ ansible_env.HOME }}/.dotmodules"`

- **`dotmodules.install`**
  List of modules to install and configure.
  *Default:* `[]`

- **`dotm_platform`**
  The machine's platform as a capability list. `brew` in the list enables the Homebrew tap,
  package and cask tasks; `apt` enables the apt task (which runs with `become`).
  *Default:* `['macos', 'brew', 'gui']` (a Mac with Homebrew; unchanged behavior when unset)

### Module Configuration

Each module can specify the following variables in its `config.yml`:

- **`homebrew_packages`**: List of Homebrew packages to install
- **`homebrew_taps`**: List of Homebrew taps to add
- **`homebrew_casks`**: List of Homebrew casks to install
- **`apt_packages`**: List of Debian/Ubuntu packages to install where `dotm_platform` includes `apt`
- **`mas_installed_apps`**: List of Mac App Store apps to install
- **`stow_dirs`**: List of directories to deploy via GNU Stow
- **`mergeable_files`**: List of files to merge with other modules
- **`github_releases`**: List of pinned GitHub release assets to install without a package manager (see [Pinned GitHub releases](#pinned-github-releases))

### Example Module Configuration

```yaml
# modules/shell-zsh/config.yml
homebrew_packages:
  - zsh
  - starship
  - fzf

mergeable_files:
  - ".zshrc"
  - ".bashrc"

stow_dirs:
  - shell-zsh
```

### Pinned GitHub releases

`github_releases` installs a tool from a GitHub release asset pinned by tag and sha256, so a
module does not need a third-party Homebrew tap for it. The task set is platform-independent:
it runs wherever the reduced configuration carries at least one entry.

Each entry:

```yaml
github_releases:
  - repo: owner/name            # GitHub repository; redirects are followed
    tag: "1.2.3"                # release tag, pinned on purpose
    asset: name.tar.gz          # gzip tarball attached to that release
    sha256: <64 hex chars>      # checksum of the asset
    install:
      app: name.app             # optional: bundle copied into ~/Applications
      bin:                      # optional: paths inside the extracted tree linked into ~/.local/bin
        - name.app/Contents/MacOS/name
    strip_components: 0         # optional, default 0; passed to tar --strip-components
```

How an entry is applied (`tasks/github_releases.yml`):

1. The asset is cached under `~/.local/share/dotm/releases/<repo>/<tag>/`.
2. `get_url` downloads it with `checksum: sha256:<sha256>`. The download fails closed: on a
   mismatch the task fails, nothing is written to the tag directory and no later task runs for
   that entry (no extraction, no link, no `.app`, no stamp).
3. The tarball is extracted with the tar the OS ships (`/usr/bin/tar`, bsdtar on macOS), then
   `install.app` is copied into `~/Applications` and each `install.bin` path is linked into
   `~/.local/bin`.
4. A stamp file `<tag dir>/.dotm-installed` records the tag and sha256.

Idempotence: entries whose stamp exists are filtered out before anything else runs, and the tar
command carries `creates:` on the stamp, so a second run reports `changed=0` for every installed
entry. Changing the tag or the sha256 selects a new tag directory and installs again.

Check mode: the download is skipped under `--check`, and the tasks that depend on the asset being
on disk skip with it, so a dry run plans the whole task list without network access.

---

## Dependencies

This role depends on:

- **community.general** collection
  Provides the Homebrew modules (`homebrew`, `homebrew_cask`, `homebrew_tap`) used to manage Homebrew packages, taps, and casks. Install via `ansible-galaxy collection install community.general`.

- **geerlingguy.mac** collection (optional)
  Only required if using Mac App Store (MAS) functionality. Provides the `geerlingguy.mac.mas` role for installing Mac App Store applications.

---

## Example Playbook

Below is an example playbook that demonstrates how to use this role:

```yaml
---
- name: Deploy dotfiles with ansible-role-dotmodules
  hosts: localhost
  vars:
    dotmodules:
      repo: "https://github.com/your-org/dotfiles.git"
      dest: "{{ ansible_env.HOME }}/.dotmodules"
      install:
        - shell-zsh      # Shell configuration with merging
        - dev-tools-zsh  # Development tools with merging
        - git           # Git configuration (stow only)
        - editor        # Editor configuration (stow only)
  roles:
    - ansible-role-dotmodules
```

## Module Structure

Create modules in your dotfiles repository with the following structure:

```
dotfiles/
├── modules/
│   ├── shell-zsh/
│   │   ├── config.yml
│   │   └── files/
│   │       └── .zshrc
│   ├── dev-tools-zsh/
│   │   ├── config.yml
│   │   └── files/
│   │       └── .zshrc
│   └── git/
│       ├── config.yml
│       └── files/
│           └── .gitconfig
```

## Testing

The role includes comprehensive tests:

```bash
# Test file merging functionality
ansible-playbook tests/test-merge.yml

# Test conflict detection
ansible-playbook tests/test-conflict.yml

# Test with dependencies
ansible-playbook tests/test-with-deps.yml
```

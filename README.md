# Synchronizing Two macOS Machines with Nix, nix-darwin, and Home Manager

## 1. Overview

macOS does not run NixOS as its operating system. Instead, the same declarative
configuration model can be achieved through a combination of three components:

- **Nix** -- the package manager, providing reproducible builds and a unified
  package set.
- **nix-darwin** -- a module system analogous to NixOS modules, targeting macOS.
  It manages system-level configuration: packages, `launchd` services, shell
  registration, and `defaults write` settings.
- **Home Manager** -- a module system for per-user configuration: dotfiles,
  shell environments, editor setups, and user-level packages. It can be
  embedded directly into nix-darwin as a submodule.

All three are composed through a single **Nix flake** -- a `flake.nix` file
checked into a Git repository. The flake defines one output per machine. Both
machines build from the same source; only a small host-specific file differs
between them.

> **Note.** Flakes remain an experimental feature in the Nix CLI. The
> configuration shown below explicitly enables the `nix-command` and `flakes`
> feature flags.

## 2. Architecture

The dependency graph is as follows:

```text
flake.nix  (entry point, pins all inputs via flake.lock)
  ├── nixpkgs           (package collection)
  ├── nix-darwin        (system-level macOS configuration)
  │     └── modules: common.nix, <host>.nix
  ├── home-manager      (user-level configuration)
  │     └── modules: common.nix
  └── nix-homebrew      (optional: declarative Homebrew installation)
```

Each layer has a well-defined responsibility:

| Layer | Scope | Examples |
|---|---|---|
| nix-darwin | System-wide packages and settings | `environment.systemPackages`, `system.defaults.*`, `launchd` daemons, login shell |
| Home Manager | Per-user packages and dotfiles | `home.packages`, `programs.git`, `programs.zsh`, SSH config |
| Homebrew (via nix-darwin) | GUI applications unavailable in nixpkgs | `homebrew.casks`, `homebrew.brews`, Mac App Store apps |
| nix-homebrew | Homebrew installation itself | Bootstrap and migration of the Homebrew prefix |

## 3. Repository Layout

```text
~/.config/nix-config/
├── flake.nix
├── flake.lock
├── hosts/
│   ├── common.nix          # shared system configuration
│   ├── macbook-pro.nix     # host-specific overrides
│   └── macbook-air.nix     # host-specific overrides
└── home/
    └── common.nix          # shared user configuration
```

The `hosts/common.nix` module contains everything shared between machines:
packages, shell settings, macOS defaults. Each host-specific file contains only
what differs -- at minimum, `nixpkgs.hostPlatform`.

## 4. Prerequisites and Constraints

Several requirements apply to current versions of nix-darwin and must be
addressed before the first build.

**Root activation.** System activation (`switch`, `activate`, `rollback`,
`check`) runs as root. All `darwin-rebuild` invocations require `sudo`.

**`system.primaryUser`.** nix-darwin requires an explicit declaration of the
primary user account. Without it, certain activation scripts will fail.

**`nixpkgs.hostPlatform`.** Each host must declare its platform:
`aarch64-darwin` for Apple Silicon, `x86_64-darwin` for Intel.

**`system.stateVersion`.** Set to `6` for new nix-darwin installations. Do not
increment without reviewing the release notes.

**`users.users.<name>.home`.** When Home Manager is embedded as a nix-darwin
module, it reads `home.homeDirectory` from this value. If it is not set, Home
Manager will fail to locate the user's home directory.

**`home.stateVersion`.** Used by Home Manager as a version gate for
backward-compatible migrations. Set it to the release you started with and do
not raise it without reviewing the changelog.

> **Version note.** All examples in this document reference the **25.11**
> release branches. At the time of writing (early 2026), 25.11 is the current
> stable release. If you are reading this at a later date, substitute the
> current stable branch. Verify availability at the nix-darwin, nixpkgs, and
> Home Manager repositories before pinning.

## 5. Configuration

### 5.1 flake.nix

The flake defines inputs (pinned to stable release branches) and two
`darwinConfigurations` outputs -- one per machine.

```nix
{
  description = "Shared macOS configuration for two personal MacBooks";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-25.11-darwin";

    nix-darwin.url = "github:nix-darwin/nix-darwin/nix-darwin-25.11";
    nix-darwin.inputs.nixpkgs.follows = "nixpkgs";

    home-manager.url = "github:nix-community/home-manager/release-25.11";
    home-manager.inputs.nixpkgs.follows = "nixpkgs";
  };

  outputs = inputs@{ self, nix-darwin, home-manager, ... }: {
    darwinConfigurations = {

      "macbook-pro" = nix-darwin.lib.darwinSystem {
        specialArgs = { inherit inputs; };
        modules = [
          ./hosts/common.nix
          ./hosts/macbook-pro.nix
          home-manager.darwinModules.home-manager
          {
            home-manager.useGlobalPkgs = true;
            home-manager.users.ivan = import ./home/common.nix;
          }
        ];
      };

      "macbook-air" = nix-darwin.lib.darwinSystem {
        specialArgs = { inherit inputs; };
        modules = [
          ./hosts/common.nix
          ./hosts/macbook-air.nix
          home-manager.darwinModules.home-manager
          {
            home-manager.useGlobalPkgs = true;
            home-manager.users.ivan = import ./home/common.nix;
          }
        ];
      };

    };
  };
}
```

The `follows` declarations ensure that nix-darwin and Home Manager resolve
against the same `nixpkgs` revision. The `specialArgs` attrset passes inputs to
all modules, which is useful if a module needs to reference a flake input
directly.

Pinning to release branches (rather than `nixpkgs-unstable`) provides more
predictable behavior when the goal is state synchronization between two
machines.

### 5.2 hosts/common.nix

This module is evaluated on every host. It carries the shared system-level
configuration.

```nix
{ pkgs, ... }:

{
  system.stateVersion = 6;
  system.primaryUser = "ivan";

  users.users.ivan = {
    home = "/Users/ivan";
  };

  nix.settings.experimental-features = [ "nix-command" "flakes" ];

  environment.systemPackages = with pkgs; [
    git
    tmux
    neovim
    ripgrep
    fd
  ];

  programs.zsh.enable = true;

  system.defaults.dock.autohide = true;
}
```

`environment.systemPackages` makes packages available to all users on the
system. `system.defaults.*` maps directly to `defaults write` domains --
`system.defaults.dock.autohide`, for example, sets `com.apple.dock autohide
-bool true`.

### 5.3 Host-Specific Modules

Each host file contains only machine-specific overrides. For two Apple Silicon
MacBooks, the files are identical in content but exist as separate units for
future divergence (external displays, additional hardware, host-specific
services).

**hosts/macbook-pro.nix**

```nix
{ ... }:

{
  nixpkgs.hostPlatform = "aarch64-darwin";
}
```

**hosts/macbook-air.nix**

```nix
{ ... }:

{
  nixpkgs.hostPlatform = "aarch64-darwin";
}
```

If one machine runs on Intel, substitute `"x86_64-darwin"`.

### 5.4 home/common.nix

The shared Home Manager module. All user-level packages, dotfiles, and program
configurations are declared here.

```nix
{ pkgs, ... }:

{
  home.stateVersion = "25.11";

  home.packages = with pkgs; [
    jq
    yq
    bat
  ];

  programs.git = {
    enable = true;
    userName = "Ivan";
    userEmail = "ivan@example.com";
  };

  programs.zsh.enable = true;
  programs.tmux.enable = true;
  programs.neovim.enable = true;
}
```

When `home-manager.useGlobalPkgs = true` is set in the flake, Home Manager
shares the same `pkgs` instance as the system configuration. This avoids
evaluating nixpkgs twice and guarantees version consistency between system and
user packages.

## 6. Homebrew Integration

Two distinct concerns are often conflated:

1. **Managing Homebrew packages declaratively** -- handled by `homebrew.*`
   options in nix-darwin. This generates a Brewfile and runs `brew bundle`
   during system activation. It does not install Homebrew itself.
2. **Installing Homebrew declaratively** -- handled by the separate
   `nix-homebrew` flake. It manages the Homebrew prefix, taps, and can
   auto-migrate an existing installation.

### 6.1 Package Management Only

If Homebrew is already installed on the machine, add the following to
`hosts/common.nix`:

```nix
{
  homebrew = {
    enable = true;
    casks = [
      "raycast"
      "iina"
    ];
  };
}
```

### 6.2 Declarative Homebrew Installation

To also manage the Homebrew installation itself, add `nix-homebrew` as a flake
input:

```nix
# in flake.nix inputs:
nix-homebrew.url = "github:zhaofengli/nix-homebrew";

# in each darwinConfiguration modules list:
nix-homebrew.darwinModules.nix-homebrew
{
  nix-homebrew = {
    enable = true;
    user = "ivan";
    autoMigrate = true;
  };
}
```

## 7. Bootstrap

Nix on macOS requires a multi-user installation; single-user mode is not
supported on Mac. The bootstrap sequence for a clean machine is as follows:

```bash
# 1. Install Nix (multi-user daemon mode)
curl -L https://nixos.org/nix/install | sh -s -- --daemon

# 2. Clone the configuration repository
git clone git@github.com:you/nix-config.git ~/.config/nix-config
cd ~/.config/nix-config

# 3. Initial nix-darwin bootstrap
#    The --extra-experimental-features flag is required because
#    flakes are not yet enabled in the default Nix configuration.
sudo nix --extra-experimental-features "nix-command flakes" \
  run nix-darwin/nix-darwin-25.11#darwin-rebuild -- \
  switch --flake .#macbook-pro

# 4. Subsequent rebuilds (flakes are now enabled via nix.settings)
sudo darwin-rebuild switch --flake .#macbook-pro
```

On the second machine, the procedure is identical -- only the output name
changes:

```bash
sudo darwin-rebuild switch --flake ~/.config/nix-config#macbook-air
```

`darwin-rebuild` can infer the configuration name from `LocalHostName` if no
`#name` is specified. The examples above always pass the name explicitly to
avoid ambiguity.

## 8. Update Workflow

The `flake.lock` file pins exact revisions of every input. Running
`nix flake update` resolves the latest commits on each pinned branch and
rewrites the lock file. Both machines achieve identical builds by sharing the
same `flake.lock` through Git.

The correct update sequence involves a single source of truth:

```bash
# On the primary machine:
cd ~/.config/nix-config
nix flake update
sudo darwin-rebuild switch --flake .#macbook-pro
git add flake.lock
git commit -m "flake.lock: update inputs"
git push

# On the secondary machine:
cd ~/.config/nix-config
git pull
sudo darwin-rebuild switch --flake .#macbook-air
```

> **Important.** Do not run `nix flake update` on both machines independently.
> This produces divergent `flake.lock` files and results in merge conflicts.
> Designate one machine as the update source; the other pulls the committed lock
> file and rebuilds.

## 9. Scope and Limitations

### What This Configuration Synchronizes

- System-level packages (`environment.systemPackages`)
- User-level packages (`home.packages`)
- Shell configuration (Zsh, Bash, Fish)
- Dotfiles (Git, SSH, tmux, Neovim, and any other program supported by Home
  Manager)
- macOS system preferences (`system.defaults.*`)
- Homebrew formulae and casks (declarative list)
- `launchd` agents and daemons

### What Remains Outside This Configuration

Nix manages **configuration**, not **data**. The following require separate
mechanisms:

| Concern | Typical solution |
|---|---|
| User files and documents | iCloud Drive, Syncthing |
| Secrets and credentials | `agenix` or `sops-nix` (encrypted in the same repository) |
| Keychain entries | macOS Keychain sync via iCloud, or manual export |
| Application state and databases | Application-specific sync (e.g., browser profiles) |
| Xcode signing identities | Apple Developer portal, manual provisioning |
| Apple ID and iCloud settings | Manual setup per machine |

## 10. Summary

For two personal MacBooks, the most effective synchronization strategy is a
single Git repository containing a Nix flake with shared modules for both
nix-darwin (system layer) and Home Manager (user layer). Host-specific files
isolate the minimal per-machine differences. Homebrew GUI applications are
declared through `homebrew.*` in nix-darwin, with optional declarative
installation via `nix-homebrew`. Updates propagate through a committed
`flake.lock`: one machine updates, commits, and pushes; the other pulls and
rebuilds. This approach brings approximately 90% of a developer's working
environment under version control and makes it reproducible from a single
`darwin-rebuild switch` invocation.

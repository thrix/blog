---
title: "HostConfig — Home Manager Module for Fedora Atomic Host Configuration"
date: 2026-03-14
draft: false
tags: ["nix", "home-manager", "fedora-atomic", "silverblue", "toolbox"]
summary: "How I extracted a reusable Home Manager module that materializes Nix store symlinks as real files — so host-side programs on Fedora Silverblue can actually read them."
---

I run [Home Manager](https://github.com/nix-community/home-manager) inside a
[nix-toolbox](https://thrix.github.io/nix-toolbox) container on Fedora
Silverblue (Bootable Containers). This setup works great — Nix manages my
dotfiles, packages, and program configs — but it has one fundamental problem:
**the host can't follow Nix store symlinks**.

Home Manager creates symlinks like:

```text
~/.config/sway/config -> /nix/store/abc123-home-manager-files/.config/sway/config
```

The Nix store data lives on the host filesystem — nix-toolbox stores it under
`$XDG_DATA_HOME/nix` and bind-mounts it to `/nix` inside the container. But
that `/nix` mount only exists inside the toolbox. Host-side programs like sway,
waybar, foot, or firefox see symlinks pointing to `/nix/store/...` paths that
don't resolve — the config files are effectively invisible to the host.

## The workaround — inline activation scripts

My original solution was two activation scripts baked directly into `home.nix`.
One ran before Home Manager's `checkLinkTargets` to restore symlinks from
backups, the other ran after `linkGeneration` to copy symlinks into real files:

```nix
home.activation.restoreNixLinks = lib.hm.dag.entryBefore ["checkLinkTargets"] ''
  files="
    $HOME/.config/sway/config
    $HOME/.config/waybar/config
    # ... 9 paths listed twice across two scripts
  "
  for file in $files; do
      [ ! -L "$file".lnk ] && continue
      mv "$file".lnk "$file"
  done
'';

home.activation.createHostConfig = lib.hm.dag.entryAfter ["linkGeneration"] ''
  # same file list again...
  for file in $files; do
      [ ! -L "$file" ] && continue
      target=$(readlink -f "$file")
      mv "$file" "$file".lnk
      cp "$target" "$file"
      chmod 644 "$file"
  done

  # plus a hardcoded desktop entry copy loop
  desktop_entries="1password discord"
  for entry in $desktop_entries; do
    cp -f $HOME/.nix-profile/share/applications/$entry.desktop \
      $HOME/.local/share/applications
  done
'';
```

This worked but had several problems:

- **The file list was duplicated** across both scripts — easy to forget one
- **Desktop entries were hardcoded** separately from `xdg.desktopEntries`
  (and `dropbox` was missing entirely)
- **The copy wasn't atomic** — if something failed mid-way, you could end up
  with no file at the path
- **Not reusable** — anyone with a similar toolbox-on-Silverblue setup would
  have to copy-paste the scripts

## The module — `hostConfig`

I extracted the logic into a proper Home Manager module with three options:

```nix
# modules/host-config.nix
options.hostConfig = {
  enable = lib.mkEnableOption "host config file materialization";

  files = lib.mkOption {
    type = lib.types.listOf lib.types.str;
    default = [];
    description = "Home-relative paths to materialize as real files.";
  };

  xdgDesktopEntries = lib.mkOption {
    type = lib.types.bool;
    default = false;
    description = "Auto-materialize all xdg.desktopEntries.";
  };
};
```

When `xdgDesktopEntries = true`, the module derives desktop entry paths
directly from `config.xdg.desktopEntries` — add a new entry there and it's
automatically materialized, no second list to maintain:

```nix
desktopFiles =
  lib.optionals cfg.xdgDesktopEntries
  (map (name: ".local/share/applications/${name}.desktop")
    (lib.attrNames config.xdg.desktopEntries));
```

This is how I share desktop applications installed via Nix inside the toolbox
with the host's desktop environment. I actively use this for 1Password,
Discord, and Dropbox — all three are Nix packages that run inside the toolbox
but need their `.desktop` files visible on the host so they appear in the
application launcher and can be started from sway.

Using it in `home.nix` is now clean:

```nix
hostConfig = {
  enable = true;
  xdgDesktopEntries = true;
  files = [
    ".config/foot/foot.ini"
    ".config/sway/config"
    ".config/waybar/config"
    ".config/waybar/style.css"
    ".local/share/applications/mimeapps.list"
    ".mozilla/firefox/profiles.ini"
    ".mozilla/firefox/${username}/containers.json"
    ".mozilla/firefox/${username}/search.json.mozlz4"
    ".mozilla/firefox/${username}/user.js"
  ];
};
```

This replaced 65 lines of inline shell with 15 lines of declarative config.

## Atomic copy-before-move

The original script did `mv` then `cp` — if the copy failed, the symlink was
already gone and you'd end up with no config file at the path. The new module
uses an atomic copy-before-move pattern:

```bash
_hc_create_file() {
  local _hc_file="$1"
  [ -L "$_hc_file" ] || return 0
  local _hc_target
  _hc_target=$(readlink -f "$_hc_file")
  cp "$_hc_target" "$_hc_file.new" || return 1
  mv "$_hc_file" "$_hc_file.lnk"
  mv "$_hc_file.new" "$_hc_file"
  chmod 644 "$_hc_file"
}
```

If `cp` fails (disk full, permissions), the original symlink stays intact. The
`_hc_restore_file` function also cleans up any leftover `.new` temp files from
previous partial runs before restoring the `.lnk` symlink backups.

## Testing and CI

This project had no tests until now. Extracting the shell logic into a proper
module felt like a good opportunity to change that.

The shell functions live in a single file — `modules/host-config-lib.sh` — that
serves as the single source of truth. The Nix module sources it at activation
time (`source ${./host-config-lib.sh}`), and the
[bats](https://github.com/bats-core/bats-core) test suite sources the same
file directly:

```bash
# tests/host-config.bats
source "${BATS_TEST_DIRNAME}/../modules/host-config-lib.sh"
```

No duplicated test helpers that can silently diverge from the real code. The
tests use a temp directory with fake symlinks — no Nix store or Home Manager
needed:

- **Symlink materialization** — verifies the symlink becomes a real file with
  correct content and mode 644, with the original symlink backed up as `.lnk`
- **Idempotency** — running `_hc_create_file` twice on an already-materialized
  file leaves it untouched (mtime unchanged)
- **Failure safety** — when `cp` can't write (directory made non-writable),
  the original symlink stays intact and no `.lnk` backup is created
- **Restore** — `_hc_restore_file` correctly moves the `.lnk` symlink backup
  back to the original path
- **Cleanup** — leftover `.new` temp files from partial runs are removed

I also added GitHub Actions CI with three parallel jobs: `nix build` for both
user configurations, bats tests, and pre-commit hooks (alejandra, deadnix,
gitleaks). Dependabot keeps the GitHub Actions versions up to date.

All of this is wired through `make check` targets:

```makefile
check/nix:  ## Build Nix config for all users without creating result links
    nix build .#homeConfigurations.thrix.activationPackage --no-link
    nix build .#homeConfigurations.mvadkert.activationPackage --no-link

check/bats:  ## Run bats unit tests
    bats tests/

check: check/nix check/bats  ## Run all checks
```

## Reusable via flake

The module is exposed as a flake output, so anyone with a similar
toolbox-on-Silverblue setup can consume it directly:

```nix
# In your flake.nix
inputs.nix-config.url = "github:thrix/nix-config";

homeConfigurations."alice" = home-manager.lib.homeManagerConfiguration {
  pkgs = nixpkgs.legacyPackages.x86_64-linux;
  modules = [
    inputs.nix-config.homeManagerModules.hostConfig
    ./home.nix
  ];
};
```

## What's next

The PR is at [thrix/nix-config#1](https://github.com/thrix/nix-config/pull/1).

I've been using this Nix + Home Manager + nix-toolbox setup to manage my Fedora
Atomic Desktop laptops for 2 years now. I'm preparing a talk for this year's
[DevConf.CZ](https://www.devconf.info/cz/) — if accepted, I'll go into much
more detail about the full setup and how it all fits together.

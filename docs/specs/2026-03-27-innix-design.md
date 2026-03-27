# innix — macOS Setup Framework & TUI

**Date:** 2026-03-27
**Status:** Draft
**Repo:** github.com/will-corrigan/innix

## Problem

Setting up a new Mac with a reproducible, declarative configuration is powerful but hard. nix-darwin + home-manager + Stylix can do it, but the learning curve is steep — you need to understand Nix, know which options exist, wire up modules manually, and maintain it all yourself.

innix is a single Go binary that scaffolds a complete nix-darwin configuration repo through a guided TUI, then helps you manage it over time.

## Design Principles

1. **One command** — just `innix`. It figures out what to do from context.
2. **Stateless** — no JSON state files. innix scans the filesystem every time.
3. **Generate once, user owns** — innix creates files, then they belong to you. Edit freely.
4. **Graceful degradation** — curated configs adapt to what's installed. Missing bat? fzf still works, just no preview.
5. **No opinions without consent** — macOS stays at system defaults unless you explicitly choose otherwise. Curated configs are opt-in per program.
6. **Every side effect confirmed** — installing dependencies, generating keys, writing files. All skippable.
7. **Verify, don't trust** — when the user says they've done a manual step, check before continuing.

## Architecture

### What innix is

A single Go binary built with Charm Huh v2 (forms), BubbleTea v2 (custom screens), and Lipgloss v2 (styling). Templates are embedded via `go:embed`. Zero runtime dependencies.

### How it detects context

```
innix (in empty dir)           -> scaffold wizard
innix (in innix repo)          -> add/edit/remove programs, add/edit hosts
innix (in unknown dir)         -> "not an innix repo"
```

An innix repo is identified by filesystem structure: `dotfiles/curated/`, `dotfiles/minimal/`, `hosts/`, `modules/`, and `.innix-version`.

### State management

Stateless. Every time innix runs:
- Available programs -> scan `dotfiles/curated/` and `dotfiles/minimal/` for `.nix` files
- Enabled programs per host -> read the host TOML
- innix version -> `.innix-version` file (just the version string, e.g., `1.0.0`)

No JSON state file to corrupt. The filesystem is the source of truth.

### Distribution

| Method | Use case |
|---|---|
| `brew install innix` | Primary — works before Nix is installed |
| `nix run github:will-corrigan/innix` | For existing Nix users re-scaffolding |
| GitHub release binaries | `curl -fsSL <url> \| tar xz && ./innix` |

`nix run` requires Nix to already be installed. Document this explicitly — it's for existing users, not first-time setup.

---

## What innix generates

```
my-mac-config/
├── .innix-version              # "1.0.0" — gates future migrations
├── flake.nix                   # Auto-discovers hosts/*.toml via readDir
├── flake.lock
├── fonts.nix                   # 30 font catalog (flat keys)
├── schema.json                 # Generated — regenerated on add/remove
├── wallpapers/
│   └── default.avif
├── modules/
│   ├── common.nix              # Only writes macOS settings present in host TOML
│   ├── packages.nix            # Base (nh) + host extra_packages.cli
│   ├── homebrew.nix            # Integration deps + host extra_packages
│   └── home.nix                # Conditional dotfile imports
├── hosts/
│   └── <hostname>.toml         # Generated from wizard
└── dotfiles/
    ├── curated/                # Programs user picked "recommended"
    └── minimal/                # Programs user picked "start fresh"
```

Only the programs the user selected get dotfiles generated. If they pick 4 programs, 4 pairs of files appear.

---

## TOML Host Format

Programs are tables (not flat strings) to support per-program overrides:

```toml
#:schema ../schema.json

[machine]
computer_name = "My-Mac"
type          = "apple-silicon"
username      = "alice"

[user]
name  = "Alice Smith"
email = "alice@example.com"

[appearance]
font      = "fira-code"
theme     = "catppuccin-mocha"
wallpaper = "default"

[preferences]
vim_keybindings = true

[programs.zsh]
config = "curated"

[programs.starship]
config = "curated"

[programs.ghostty]
config = "curated"

[programs.zed]
config = "curated"
vim    = false              # override: no vim for zed

[programs.git]
config = "curated"

[programs.fzf]
config = "curated"

[programs.bat]
config = "minimal"

[integrations]
ssh_signing = "1password"

[extra_packages]
cli   = ["ripgrep", "fd", "jq"]
brews = ["awscli"]
casks = ["slack", "discord"]

[macos.keyboard]
caps_lock    = "escape"
key_repeat   = "fast"
repeat_delay = "short"
press_and_hold = false

[macos.dock]
position        = "bottom"
icon_size       = "medium"
auto_hide       = true
minimize_to_app = true
mru_spaces      = false
autohide_delay  = "instant"
launch_animation = false

[macos.finder]
default_view            = "list"
show_extensions         = true
show_path_bar           = true
show_hidden             = false
extension_change_warning = false
quit_menu_item          = true
default_search_scope    = "current-folder"

[macos.trackpad]
tap_to_click     = true
natural_scroll   = true
three_finger_drag = true

[macos.developer]
expand_save_panels    = true
expand_print_panels   = true
full_keyboard_access  = true
disable_autocorrect   = true
disable_autocapitalize = true
disable_smart_quotes  = true
disable_smart_dashes  = true
disable_smart_periods = true

[macos.privacy]
disable_guest_account     = true
disable_personalized_ads  = true
no_ds_store_on_network    = true
no_ds_store_on_usb        = true

[macos.screenshots]
location       = "~/Desktop"
format         = "png"
disable_shadow = true

[macos.security]
touch_id_sudo            = true
screensaver_require_password = true
screensaver_password_delay = 0

[macos.clock]
twenty_four_hour = true
show_seconds     = false

[macos.battery]
show_percentage = true

[macos.startup]
disable_chime = false
```

### Nix consumption of table-format programs

The innix `home.nix` template handles the table format:

```nix
{ lib, host, ... }:
let
  username = host.machine.username;
  programs = host.programs or {};
  preferences = host.preferences or {};

  programImport = name: value:
    let
      level = if builtins.isString value then value else value.config or "minimal";
    in
    if level == "curated"
    then ../dotfiles/curated/${name}.nix
    else ../dotfiles/minimal/${name}.nix;

  enabledImports = lib.mapAttrsToList programImport programs;

  integrations = host.integrations or {};
  sessionVars =
    if (integrations.ssh_signing or null) == "1password"
    then { SSH_AUTH_SOCK = "/Users/${username}/Library/Group Containers/2BUA8C4S2C.com.1password/t/agent.sock"; }
    else {};
in
{
  home-manager.useGlobalPkgs = true;
  home-manager.useUserPackages = true;
  home-manager.backupFileExtension = "backup";
  home-manager.extraSpecialArgs = { inherit host; };

  home-manager.users.${username} = {
    home.username = username;
    home.homeDirectory = lib.mkForce "/Users/${username}";
    home.stateVersion = "24.11";
    home.sessionVariables = sessionVars;
    imports = enabledImports;
  };
}
```

### Vim keybinding resolution

Dotfiles that support vim read from `host` with this resolution order:

1. Per-program `vim` override (e.g., `host.programs.zed.vim`)
2. Global preference (`host.preferences.vim_keybindings`)
3. Default: `false`

Example in a curated dotfile:

```nix
{ host, ... }:
let
  preferences = host.preferences or {};
  programConfig = host.programs.zed or {};
  useVim =
    if programConfig ? vim then programConfig.vim
    else preferences.vim_keybindings or false;
in
{
  programs.zed-editor = {
    enable = true;
    package = null;
    userSettings = {
      vim_mode = useVim;
      relative_line_numbers = useVim;
      # ... rest of curated config
    };
  };
}
```

Programs that support vim toggling: **zsh** (bindkey -v), **btop** (vim_keys), **zed** (vim_mode), **tmux** (keyMode vi, copy-mode-vi bindings).

### Graceful degradation

Curated dotfiles check what other programs are available at Nix evaluation time:

```nix
{ host, ... }:
let
  programs = host.programs or {};
  extraCli = (host.extra_packages or {}).cli or [];
  hasBat = programs ? bat || builtins.elem "bat" extraCli;
  hasFd  = programs ? fd  || builtins.elem "fd"  extraCli;
in
{
  programs.fzf = {
    enable = true;
    defaultCommand =
      if hasFd then "fd --hidden --strip-cwd-prefix --exclude .git"
      else null;
    defaultOptions = [
      "--height=60%"
      "--layout=reverse"
      "--border=rounded"
      "--multi"
    ] ++ lib.optionals hasBat [
      "--preview='bat --color=always --style=numbers --line-range=:500 {}'"
      "--preview-window=right,50%,border-left"
    ];
  };
}
```

Same pattern for zsh (eza aliases only if eza available), ghostty (shell-integration only if zsh enabled), etc.

---

## Curated Configs

Curated configs shipped with innix are genuinely universal. They are NOT copied from the personal repo — they are written from scratch based on official docs and community consensus.

### What curated means per program

| Program | Curated includes | Vim additions |
|---|---|---|
| **zsh** | History persistence (10k, dedup, share, extended), autosuggestion, syntax highlighting, completion, history substring search, safety aliases (rm -i, cp -i, mv -i), conditional eza aliases, conditional mise activation | `bindkey -v`, `KEYTIMEOUT=1`, j/k history navigation |
| **starship** | Full nerd font prompt with git/language/cloud modules, two-line layout, right-aligned duration+time, enabled: git_metrics, kubernetes, time, memory_usage, os, sudo, status | Vim mode character symbols (auto when zsh vi mode active) |
| **fzf** | Conditional fd for commands, conditional bat preview, 60% height, reverse layout, rounded border, multi-select, ctrl-d/u scroll, clipboard yank | N/A (already has vim-like bindings) |
| **bat** | Style: numbers+changes+header+grid, italic text, less -FR pager | N/A |
| **btop** | Transparent theme background | `vim_keys = true` |
| **ghostty** | Font thicken, cell height +2, 92% opacity, blur, bar cursor, option-as-alt, window padding, copy-on-select, split keybindings, conditional shell-integration | N/A |
| **zed** | Telemetry off, tab size 2, soft wrap, indent guides, git inline blame, inlay hints, format on save, language configs (TS/Go/Nix/Markdown), LSP configs (gopls, tailwindcss) | `vim_mode`, relative line numbers, smartcase find, system clipboard, ctrl-h/j/k/l pane navigation |
| **tmux** | C-a prefix, mouse, escape-time 0, base-index 1, 50k history, true color, clipboard integration, split bindings with current path, pane navigation | `keyMode = "vi"`, vi copy-mode bindings, h/j/k/l pane nav, H/J/K/L pane resize |
| **git** | Sensible defaults (rebase on pull, auto-setup remote, main branch), histogram diff, zdiff3 merge conflicts, rerere, fetch prune, rebase autoSquash/autoStash/updateRefs, fsckobjects, conditional SSH/GPG signing, gh CLI, conditional SSH config | N/A |
| **direnv** | Zsh integration, nix-direnv, silent mode, warn_timeout 10s, hide_env_diff | N/A |
| **zoxide** | Zsh integration, `--cmd cd` (replaces cd with smart jump) | N/A |
| **mise** | package=null (Homebrew), all shell integrations disabled (manual activation in zsh), auto_install explicit | N/A |

### What curated does NOT include

- Personal aliases (company k8s contexts, tool-specific shortcuts)
- Hardcoded paths (rancher, specific config locations)
- Plugin references that assume a specific theme (catppuccin tmux status)
- Tool assumptions without graceful degradation
- Global language installations (no `node = "lts"` in mise)

---

## macOS Defaults

Only settings the user explicitly chooses get written to `common.nix`. Everything else stays at macOS system defaults. No hardcoded opinions.

The TUI groups settings into digestible categories:

### Developer defaults (recommended for most developers)
- Fast key repeat + disable press-and-hold
- Disable autocorrect, autocapitalize, smart quotes/dashes/periods
- Expand save/print dialogs by default
- Full keyboard access (tab through all controls)

### Dock
- Position (left/bottom/right)
- Icon size (small/medium/large)
- Auto-hide + delay + speed
- Don't rearrange Spaces by recent use (`mru_spaces = false`)
- Disable launch animation
- Minimize effect (genie/scale)
- Show hidden app indicators
- Hot corners (optional, power user)

### Finder
- Default view (list/columns/icons/gallery)
- Show extensions, path bar, status bar, hidden files
- Disable extension change warning
- Allow Finder quit (Cmd+Q)
- Search scope: current folder
- Sort folders first

### Trackpad
- Tap to click
- Natural scrolling
- Three-finger drag

### Privacy & Security
- Touch ID for sudo
- Disable guest account
- Screensaver password (immediate)
- Disable personalized ads
- Don't write .DS_Store on network/USB drives

### Screenshots
- Save location (Desktop/Downloads/custom)
- Format (png/jpg)
- Disable shadow

### Clock & Battery
- 24-hour time
- Show seconds
- Battery percentage

### Startup
- Disable boot chime

---

## TUI Wizard Flow

### Step 1: Bootstrap dependencies

Detect Nix (`nix --version`), nix-darwin (`darwin-rebuild --version`), Homebrew (`brew --version`). For each missing:

```
Nix is not installed.
Nix is the package manager that makes this work.

  [Install Nix]  [Skip]
```

**Shell restart handling:** If Nix installation requires a shell restart (it modifies `/etc/zshrc`), the wizard saves progress to a temporary file, tells the user to restart their terminal, and on next `innix` run detects the partial state and resumes from where it left off.

### Step 2: Your machine

Auto-detect from `scutil --get LocalHostName`, `uname -m`, `$USER`. Confirm with user. Friendly wording — no "hostname" or "platform".

### Step 3: Appearance

- **Theme**: Select from popular ~20, with live palette preview (TUI theme matches selection). "Search all themes" option shells out to list base16-schemes.
- **Font**: Select from embedded fonts.nix catalog (30 fonts) with sample text.
- **Wallpaper**: Select from `wallpapers/` with inline thumbnail via go-termimg (Kitty protocol on Ghostty, half-block fallback). "Use my own" option for custom path.

Font preview limitation: the TUI can show the font name and a description, but cannot render the actual font in a terminal that doesn't have it installed yet. Note this honestly.

### Step 4: Programs

Multi-select grouped by concern:

```
Shell & Terminal
  [ ] zsh         Shell with history, completions, and syntax highlighting
  [ ] starship    Prompt with git, language, and cloud indicators
  [ ] ghostty     GPU-accelerated terminal emulator
  [ ] tmux        Terminal multiplexer with pane splitting

Editors
  [ ] zed         Modern editor with LSP and format-on-save

CLI Tools
  [ ] fzf         Fuzzy finder for files and history
  [ ] bat         Syntax-highlighted file viewer (replaces cat)
  [ ] btop        Visual system monitor
  [ ] zoxide      Smart directory jumping (replaces cd)

Development
  [ ] git         Git + GitHub CLI + SSH config
  [ ] direnv      Per-project environments (with nix-direnv)
  [ ] mise        Language version manager (Node, Python, Go, etc.)
                   Learn more: https://mise.jdx.dev
```

Global vim toggle at the top of the Development section:

```
  [x] Vim keybindings (applies to zsh, btop, zed, tmux)
```

For each selected program:

```
Use our recommended config for zed?
Includes: format-on-save, indent guides, git blame, LSP,
language configs for TypeScript, Go, Nix, Markdown

  > Recommended config
    Start fresh (just enable it)
```

Per-program vim override shown when relevant:

```
  [x] zed    Modern editor with LSP
              Vim keybindings: ON (change)
```

**Contextual follow-ups:**
- git selected -> "Name for commits?" / "Email?"
- git selected -> "Sign commits with SSH keys?" -> integrations step

**Dependency hints:**
- fzf selected -> "fzf works best with fd (file finder) and bat (previewer). Add them too?" (non-blocking, adds to extra_packages)
- zsh eza aliases -> "The recommended zsh config includes pretty ls aliases via eza. Add eza?" (non-blocking)

### Step 5: Integrations

Only shown if git selected.

```
SSH signing proves commits came from you.

  > 1Password   (manages SSH keys in your vault)
    GPG          (traditional key-based signing)
    None         (skip signing)
```

1Password flow: install -> guided manual toggle -> **verify** (check agent socket, op --version, op account list) -> key generation with confirmation -> GitHub key setup.

### Step 6: Extra packages

```
Extra CLI tools? (comma-separated, enter to skip)
  Browse: https://search.nixos.org/packages

Extra desktop apps? (comma-separated, enter to skip)
  Browse: https://formulae.brew.sh/cask/

Extra Homebrew formulae? (comma-separated, enter to skip)
  Browse: https://formulae.brew.sh/formula/
```

Homebrew names validated cheaply via `brew info`. Nixpkgs names validated at build time with clear error message.

### Step 7: macOS preferences

Grouped categories (see macOS Defaults section above). Each with sensible defaults pre-selected. User can accept defaults or customize per-setting.

### Step 8: Summary & confirm

Show full config. All answers collected up to this point — nothing written yet.

```
  [Save & build]    [Edit]    [Cancel]
```

**Atomic write:** All files written at once. If interrupted during write, next `innix` run detects partial state (has `.innix-version` but missing key files) and offers to re-scaffold.

Save: writes all files, `git init`, initial commit, offers first build.

### Step 9: Post-build notices

Programs added as extra packages but not managed by innix:

```
Installed but configured outside this framework:
  ripgrep     https://github.com/BurntSushi/ripgrep
  kubectl     https://kubernetes.io/docs/reference/kubectl/
```

### Returning use (existing repo)

```
innix detected an existing configuration.

  > Add a program
    Edit a host
    Remove a program
    Add a new host
```

**Add program:** shows programs not yet installed (scans `dotfiles/` vs full catalog), user picks, curated vs minimal, writes dotfile pair, regenerates schema.json, offers to add to host TOML.

**Edit host:** lists hosts from `hosts/*.toml`, shows current values as defaults, modify any section.

**Remove program:** shows installed programs, user picks, removes from host TOML. Dotfiles only deleted if no other host references the program (scans all TOMLs).

**Add host:** runs steps 2-8 of wizard, generates new TOML.

---

## Program Management

### Adding a program

1. User selects program from innix's embedded catalog
2. innix writes `dotfiles/curated/<name>.nix` and `dotfiles/minimal/<name>.nix` from embedded templates
3. innix regenerates `schema.json` (embedded metadata + filesystem scan)
4. innix offers to add `[programs.<name>]` to selected host TOML
5. User rebuilds

### Removing a program

1. User selects installed program
2. innix removes `[programs.<name>]` from selected host TOML
3. innix scans all other host TOMLs — if no host references this program, deletes the dotfile pair
4. innix regenerates `schema.json`
5. User rebuilds

### Updating curated configs

Not managed in v1. Dotfiles belong to the user after creation.

To get fresh defaults: delete the curated dotfile, run `innix` -> add the program again.

Future (v2): `innix diff <program>` shows what changed between user's version and latest template.

---

## Schema Regeneration

`schema.json` is a generated artifact, not user-authored. innix regenerates it:

- **When:** on add program, remove program, or explicit request
- **Source of truth:** innix's embedded program catalog (has names, descriptions, vim support metadata) merged with filesystem scan of `dotfiles/` (catches manually added programs)
- **For custom programs:** if a user manually adds `dotfiles/curated/neovim.nix`, innix detects it on next scan and adds `neovim` to the schema with a generic description. The user can hand-edit the schema for a better description — innix preserves custom entries on regeneration.

---

## .innix-version

Contains just a version string (e.g., `1.0.0`). Purpose:

- **Repo detection:** presence confirms this is an innix repo
- **Future migrations:** if innix v2 changes TOML format, common.nix structure, or dotfile conventions, it reads the version and offers to migrate
- **Low-value edit target:** if a user changes it, worst case innix warns about version mismatch

---

## 1Password SSH Setup Flow

Triggered when user selects `ssh_signing = "1password"`.

**Automated:**
- Install 1Password and 1Password CLI (via Homebrew)
- Configure SSH agent socket in `~/.ssh/config`
- Configure git signing config

**Guided manual steps:**
```
1Password needs two settings enabled:
  Open 1Password > Settings > Developer:
    Turn on "SSH Agent"
    Turn on "Integrate with 1Password CLI"

  [Open 1Password]  [I've done this]  [Skip]
```

**Verification on "I've done this":**
- SSH agent socket exists at `~/Library/Group Containers/2BUA8C4S2C.com.1password/t/agent.sock`
- `op --version` succeeds
- `op account list` returns at least one account

If verification fails: show what's missing, offer to try again or skip.

**Key generation (all confirmable, all skippable):**
```
Generate a new SSH key in 1Password?
  Key type: Ed25519
  Title: GitHub SSH Key

  [Generate]  [Skip — I have a key]  [Cancel]
```

Then offer to open GitHub SSH settings page + copy public key to clipboard.

---

## Visual Design

- **Dynamic theme matching** — TUI switches to match selected Stylix theme
- **Wallpaper thumbnails** — go-termimg with Kitty protocol (Ghostty) / half-block fallback
- **Spring-animated progress bars** — Harmonica during short operations. For long builds (`nix build` 10-30 min), stream build output alongside a spinner rather than just animating.
- **Gradient borders** — Lipgloss v2 around active panels
- **ASCII art header** — branded welcome screen
- **VHS tape file** — for recording demo GIF for README

---

## Go Repo Structure

```
innix/
├── cmd/
│   └── main.go                 # Entry point, context detection
├── internal/
│   ├── detect/
│   │   └── detect.go           # Repo detection (innix repo? empty? unknown?)
│   ├── bootstrap/
│   │   ├── detect.go           # Detect Nix, nix-darwin, Homebrew versions
│   │   └── install.go          # Install missing deps, handle shell restart
│   ├── wizard/
│   │   ├── wizard.go           # Main orchestration, answer collection
│   │   ├── machine.go          # Step 2: auto-detect + confirm
│   │   ├── appearance.go       # Step 3: theme, font, wallpaper
│   │   ├── programs.go         # Step 4: program selection + vim
│   │   ├── integrations.go     # Step 5: SSH signing
│   │   ├── packages.go         # Step 6: extra packages
│   │   ├── macos.go            # Step 7: macOS preferences
│   │   └── summary.go          # Step 8: review + confirm
│   ├── manage/
│   │   ├── add.go              # Add program to existing repo
│   │   ├── remove.go           # Remove program
│   │   └── edit.go             # Edit host TOML
│   ├── scaffold/
│   │   ├── scaffold.go         # Write all files atomically
│   │   └── toml.go             # TOML generation from wizard answers
│   ├── schema/
│   │   └── schema.go           # Generate schema.json from catalog + filesystem
│   └── verify/
│       └── onepassword.go      # 1Password checks
├── templates/                   # Embedded via go:embed
│   ├── flake.nix.tmpl          # Templated (repo path for rebuild alias)
│   ├── fonts.nix               # Static
│   ├── modules/
│   │   ├── common.nix.tmpl     # Templated (only writes configured settings)
│   │   ├── packages.nix        # Static
│   │   ├── homebrew.nix        # Static
│   │   └── home.nix            # Static
│   ├── dotfiles/
│   │   ├── curated/            # Clean, universal, parameterized
│   │   └── minimal/            # Bare enablement
│   └── wallpapers/
│       └── default.avif
├── go.mod
├── go.sum
├── flake.nix                   # For `nix run github:will-corrigan/innix`
└── README.md
```

### Templated vs static files

| File | Templated? | Why |
|---|---|---|
| `flake.nix` | Yes | Repo path varies per scaffold location |
| `common.nix` | Yes | Only writes macOS settings the user configured |
| `fonts.nix` | No | Same for every repo |
| `schema.json` | Generated | Built from embedded catalog + filesystem scan |
| `packages.nix` | No | Same for every repo |
| `homebrew.nix` | No | Same for every repo |
| `home.nix` | No | Same for every repo (reads host TOML dynamically) |
| `dotfiles/curated/*` | No | Same for every repo (reads host attrset dynamically) |
| `dotfiles/minimal/*` | No | Same for every repo |

---

## Interrupted Wizard Recovery

The wizard collects ALL answers before writing any files (atomic write at step 8).

**Bootstrap interruption:** If Nix installation requires a shell restart, innix writes a `.innix-resume.json` temp file with progress. On next run, detects it and resumes.

**Write interruption:** If the atomic write is interrupted (e.g., Ctrl-C during file creation), the next `innix` run detects a partial repo (has `.innix-version` but missing key files like `flake.nix` or `modules/`) and offers:
```
Incomplete innix repo detected. Re-scaffold?
  [Yes — start fresh]  [No — exit]
```

---

## Relationship to Personal nix-darwin Repo

innix is a completely separate project at `github.com/will-corrigan/innix`. The personal nix-darwin repo at `/etc/nix-darwin` uses a different TOML format (flat program strings) and different curated configs (personal/opinionated). innix does not read, write, or interact with it.

If the user wants to use innix with their personal repo in the future, they would scaffold a new repo with innix and migrate their settings.

---

## Future (v2+)

- **Per-program flavors:** `config = "vim"` / `config = "standard"` / `config = "minimal"` for programs where vim-vs-not is a fundamental fork
- **Per-setting toggles:** granular control within curated configs via TOML
- **`innix diff <program>`:** show what changed between user's file and latest template
- **Dynamic wallpaper recoloring** via gowall (pure Go, recolors to match base16 theme)
- **Community curated packs:** shareable config bundles beyond the built-in catalog
- **`innix doctor`:** validate system state, detect drift, check for common issues
- **Homebrew cleanup policy** as a TOML option
- **Window manager integration** (yabai + skhd) as an integration like 1Password
- **Provenance comments:** `# Generated by innix v1.0.0` in generated files

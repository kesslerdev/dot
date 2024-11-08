# `dot`

<p align="center">
  <img src="https://github.com/user-attachments/assets/61ff9b15-f227-4b4d-a950-57ba0353ff85#gh-light-mode-only" width="600px" />
  <img src="https://github.com/user-attachments/assets/420e9121-426e-442a-a52a-8332b6f1989b#gh-dark-mode-only" width="600px" />
</p>

> Manage your dotfiles and their dependencies automagically

## Table of Contents
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Usage](#usage)
  - [`init.lua`](#initlua)
  - [Recursive](#recursive)
  - [Profiles](#profiles)
  - [Force Mode `-f`](#force-mode--f)
  - [Unlinking Configs `--unlink`](#unlinking-configs---unlink)
  - [Purging Modules `--purge`](#purging-modules---purge)
  - [Hooks](#hooks)
  - [Summary of Command-Line Options](#summary-of-command-line-options)
- [Examples](#examples)
- [To do](#to-do)

## Installation

```bash
$ brew install pablopunk/brew/dot
```

## Quick Start

<img src="https://github.com/user-attachments/assets/8c235cb8-d6c5-4f9c-88db-db1a04e914e4" width="600px" />

```bash
$ cd /path/to/dotfiles
$ tree

profiles/
├── work.lua
├── personal.lua
├── linux-server.lua
modules/
├── neovim/
│   ├── init.lua
│   └── config/
├── zsh/
│   ├── init.lua
│   └── zshrc
└── apps/
    ├── work/
    │   └── init.lua
    └── personal/
        └── init.lua

$ dot          # Link all dotfiles and install dependencies
$ dot neovim   # Only process the 'neovim' module
$ dot work     # Only process the 'work' profile
```

## Usage

Each module under the `modules/` folder needs to have at least an `init.lua`. If not, it will be ignored.

### `init.lua`

Example for neovim:

```lua
-- modules/neovim/init.lua
return {
  brew = {
    { name = "neovim", options = "--HEAD" },
    "ripgrep"
  },
  config = {
    source = "./config",       -- Our config directory within the module
    output = "~/.config/nvim", -- Where the config will be linked to
  }
}
```

The config will be linked to the home folder with a soft link. In this case:

```bash
~/.config/nvim → ~/dotfiles/modules/neovim/config
```

You can also specify multiple configurations for a single module:

```lua
-- modules/multi-config/init.lua
return {
  brew = { "cursor" },
  config = {
    {
      source = "./config/settings.json",
      output = "~/Library/Application Support/Cursor/User/settings.json",
    },
    {
      source = "./config/keybindings.json",
      output = "~/Library/Application Support/Cursor/User/keybindings.json",
    }
  }
}
```

This will create two symlinks:

```bash
~/Library/Application Support/Cursor/User/settings.json → ~/dotfiles/modules/multi-config/config/settings.json
~/Library/Application Support/Cursor/User/keybindings.json → ~/dotfiles/modules/multi-config/config/keybindings.json
```

As you can see, you can declare dependencies as [Homebrew](https://brew.sh) packages, which makes it possible to also use `dot` to install GUI apps (Homebrew casks). You can create a module without any config to use it as an installer for your apps:

```lua
-- modules/apps/init.lua
return {
  brew = { "whatsapp", "spotify", "slack", "vscode" }
}
```

### Recursive

In the example above, let's say we want to separate our apps into "work" and "personal". We could either create 2 modules on the root folder or create a nested folder for each:

```lua
-- modules/apps/work/init.lua
return {
  brew = { "slack", "vscode" }
}
```

```lua
-- modules/apps/personal/init.lua
return {
  brew = { "whatsapp", "spotify" }
}
```

### Profiles

If you have several machines, you might not want to install all tools on every computer. That's why `dot` allows **profiles**.

Let's create a new "personal" profile:

```lua
-- profiles/personal.lua
return {
  modules = {
    "*",
    "!apps/work",
  }
}
```

In this example, running `dot personal` will:

- `*`: Install everything under `modules/`, including nested directories.
- `!apps/work`: Exclude the `apps/work` module and its submodules.

You can use the following patterns in your profile:

- `"*"`: Include all modules recursively.
- `"!module_name"`: Exclude a specific module and its submodules.
- `"module_name"`: Include a specific module.

For example, a work profile might look like this:

```lua
-- profiles/work.lua
return {
  modules = {
    "apps/work",
    "slack",
    "neovim",
    "zsh"
  }
}
```

> [!NOTE]
> If your profile is named just like a module (e.g., `profiles/neovim` and `modules/neovim`), running `dot neovim` will default to the profile.

### Force Mode `-f`

By default, `dot` won't touch your existing dotfiles if the destination already exists. If you still want to replace them, you can use the `-f` flag:

```bash
$ dot -f neovim
```

> [!NOTE]
> It won't remove the existing config but will move it to a new path: `<path-to-config>.before-dot`.

### Unlinking Configs `--unlink`

If you want to remove the symlinks created by `dot` for a specific module but keep your configuration, you can use the `--unlink` option:

```bash
$ dot --unlink neovim
```

This command will:

- Remove the symlink at the destination specified in `config.output`.
- Copy the config source from `config.source` to the output location.

This is useful if you want to maintain your configuration files without `dot` managing them anymore.

### Purging Modules `--purge`

To completely remove a module, including uninstalling its dependencies and removing its configuration, use the `--purge` option:

```bash
$ dot --purge neovim
```

This command will:

- Uninstall the Homebrew dependencies listed in the module's `init.lua`.
- Remove the symlink or config file/directory specified in `config.output`.
- Run any `post_purge` hooks if defined in the module.

> [!WARNING]
> `--purge` will uninstall packages from your system and remove configuration files. Use with caution.

### Hooks

You can define custom hooks in your module's `init.lua` file to run specific commands at different stages of the module's lifecycle.

#### Available Hooks

- **`post_install`**: Runs after the specified packages have been installed via `brew`.
- **`post_purge`**: Runs after the specified packages have been removed via `brew`.
- **`post_link`**: Runs after configuration files have been linked.
- **`post_unlink`**: Runs after configuration files have been unlinked.

#### Basic Hook Definition
```lua
return {
  brew = { "gh" },
  post_install = "gh auth login"
}
```

#### Multi-Line Hook Definition

You can also define multi-line hooks using Lua's multi-line string syntax.

```lua
return {
  brew = { "gh" },
  post_install = [[
    gh auth status | grep 'Logged in to github.com account' > /dev/null || gh auth login --web -h github.com
    gh extension list | grep gh-copilot > /dev/null || gh extension install github/gh-copilot
  ]],
}
```

### Summary of Command-Line Options

- **Install Modules**: Install dependencies and link configurations.

  ```bash
  $ dot             # Install all modules
  $ dot neovim      # Install only the 'neovim' module
  $ dot work        # Install only the 'work' profile
  ```

- **Force Mode**: Replace existing configurations, backing them up to `<config>.before-dot`.

  ```bash
  $ dot -f          # Force install all modules
  $ dot -f neovim   # Force install the 'neovim' module
  ```

- **Unlink Configs**: Remove symlinks but keep the config files in their destination.

  ```bash
  $ dot --unlink neovim
  ```

- **Purge Modules**: Uninstall dependencies and remove configurations.

  ```bash
  $ dot --purge neovim
  ```

## Examples

- [pablopunk/dotfiles](https://github.com/pablopunk/dotfiles): my own dotfiles.

## To do

- [x] `dot` will install dependencies and link files.
- [x] Support Homebrew dependencies.
- [x] `dot -f` will remove the existing configs if they exist (moves config to `*.before-dot`).
- [x] Allow post-install hooks in bash.
- [x] Allow installing only one module with `dot neovim`.
- [x] Allow multiple setups in one repo. Similar to "hosts" in Nix, `dot work` reads `profiles/work.lua` which includes whatever it wants from `modules/`.
- [x] Package and distribute `dot` through Homebrew.
- [x] Add `--unlink` option to remove symlinks and copy configs to output.
- [x] Add `--purge` option to uninstall dependencies and remove configurations.
- [x] Allow array of config. For example I could like two separate folders that are not siblings
- [x] Improve profiles syntax. For example, `{ "*", "apps/work" }` should still be recursive except in "apps/". Or maybe accept negative patterns like `{ "!apps/personal" }` -> everything but apps/personal.
- [x] Add screenshots to the README.
- [ ] Support more ways of adding dependencies (e.g., wget binaries, git clone, apt...).
- [ ] Unlinking dotfiles without copying. An option like `dot --unlink --no-copy` could be added.
- [ ] `dot --purge-all` to purge all modules at once.
- [ ] Support Mac defaults, similar to `nix-darwin`.
- [ ] Support an `os` field. i.e `os = { "mac" }` will be ignored on Linux.
- [ ] After using a profile, like `dot profile1`, it should remember it and all calls to `dot` should be done with this profile unless another profile is explicitely invoked, like `dot profile2`, which will replace it for the next invokations.

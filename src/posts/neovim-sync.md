---
title: Sync package for neovim
metaDescription: 
date: 2026-01-31T01:00:00.000Z
summary: I built this dead simple package to sync the code from local to remote server.
redirect: https://x.com/bbetterengineer/status/2017532364344463609
tags:
  - neovim
  - rsync
  - lua
---

A fast, minimal Neovim plugin to sync code from local to remote servers using rsync. Built in Lua with background jobs and hot-reloading.

I built code_sync.nvim: A Lightweight Neovim Plugin to Sync Code to Remote Servers [Code](https://github.com/ayush-garg341/code_sync).

If you are stuck at some point reading through the article, please go through the [README.md](https://github.com/ayush-garg341/code_sync/blob/main/README.md) of the project. It has pretty much details.

**Why I Built This:**

- As someone who works primarily inside Neovim, I constantly found myself needing to sync local files to remote servers during development  whether for testing ML scripts, deploying feature branches to staging environments, or just backing up work.

**What Didn't Work:**

- Existing Neovim plugins either:
    - Tried to do too much, bundling terminal multiplexers, remote FS interfaces, and more.
    - Were hard to configure especially with SSH keys, .rsyncignore behavior, or environmental targets.

So I said: "Why not build something simple and Lua-native that just syncs code?"

**What Makes code_sync.nvim Different?**

- Unlike generic file sync tools or bloated Neovim plugins, code_sync.nvim is built with purpose and simplicity in mind.
- Single Responsibility. It only does one thing sync code from local to remote and does it well. No terminals, no remotes-as-filesystem hacks.
- Minimal Configuration. Just drop a Lua config in ~/.config/.code_sync.lua, no need to wrap your head around dozens of options.
- Neovim-Native. Written entirely in Lua and built on top of Neovim async job APIs (vim.loop.spawn). No shims, wrappers, or FFI dependencies.
- Hot Config Reloading. Detects changes to your config file and reloads them on the fly, no need to restart Neovim.
- Multiple Environments. Easily define targets like test, stage, or dev per project, with support for multiple servers under each.
- Flexible SSH Methods. Whether you're using password-based, key-based, or passwordless login via ~/.ssh/config, it's supported.
- Non-blocking Execution. All sync operations run as background jobs. No freezing. Keep coding while your files sync in parallel.

**Introducing code_sync.nvim**
- A single-purpose Neovim plugin to sync local projects to remote directories using `rsync`, written entirely in Lua, with async background jobs and minimal config.
- Core Features:
    - Sync files/folders with one command.
    - Supports all SSH methods: password, key, keyless (via `~/.ssh/config`).
    - Dotfile-based config (`~/.config/.code_sync.lua`).
    - Target specific environments (`test`, `stage`, `dev`) per project.
    - Hot reloads config no need to restart Neovim.

**Installation**
- Add this to your Packer config:
```lua

use({
  "ayush-garg341/code_sync",
  config = function()
    require("code_sync").setup()
  end,
})
```

**How It Works?**
- You define your sync strategy in `~/.config/.code_sync.lua`. Each project can map to multiple environments, with config like:

```lua
{
  protocol = "rsync",
  ["my-project"] = {
    test = {
      {
        target = "ubuntu@192.168.1.42:/var/www/my-project",
        method = "key_less",
        exclude_file = "~/.rsyncignore"
      }
    },
    dev = {
      {
        target = "devops@devbox:/home/devops/codebase",
        method = "pwd_based",
        keypath = "myS3cretP@ss",
      }
    },
   stage = {
      {
        target = "ayush@<hostname>:/home/ayush/data-science",
        exclude_file = "/Users/elliott/.rsyncignore",
        keypath = "~/Downloads/some.pem",
        method = "key_based"
      }
    }
  }
}
```

**SSH Auth Modes Supported:**
- Key-based: (`.pem` or `id_rsa` style)
- Key-less: (`ssh-copy-id` + `~/.ssh/config`)
- Password-based via `sshpass`

```bash
# Setup passwordless SSH
ssh-keygen -t rsa
ssh-copy-id user@host

# Or install sshpass for password-based auth
brew install hudochenkov/sshpass/sshpass
```

**Config Keys Reference**
- protocol:  One of `rsync`, `scp`, `ftp`, `sftp` (currently `rsync` only)
- keypath: Path to `.pem` or plain password
- project: Source folder in Neovim
- test, stage, dev : Environment labels
- target: Full remote destination
- exclude_file: Like `.gitignore` for rsync
- method: `key_based`, `pwd_based`, `key_less`

**Example Screenshot**
- Here's how :CodeSyncList looks when no jobs are running:
![Output](/src/assets/img/editor.jpeg "output")

**Usage from Neovim:**
- `:CodeSync:` Syncs file, cwd, or project to a remote env
- `:CodeSyncList:` List active jobs
- `:CodeSyncCancel <id>:` Cancel a specific sync job
- Production syncing is intentionally not supported. Use CI/CD.

```vim
:CodeSync test --project    "Sync the whole project to your test env
:CodeSync stage --cwd      "Sync the current directory to stage env
:CodeSync dev            " Sync current file to dev env
:CodeSyncCancel 3        " Cancel job with ID 3
```

**Technical Design Decisions:**
- Uses `vim.fn.jobstart` to **avoid freezing** Neovim
- Supports **parallel jobs** and shows job status
- Uses pure Lua  no shell hacks

**Roadmap & Todos:**
- Support dry run of rsync command before actually start syncing
- Hot-reload config when modified
- Add support for `scp` and `sftp`
- Add auto-sync with user-defined intervals
- Better logging for sync failures/success
- Detect changes in files & only sync diffs

**Want to Contribute?**
- Sure! Pick a task from above or suggest your own:
- Fork the repo on GitHub
- Test your patch locally
- Submit a Pull Request with details

**Who This Is For?**
- Neovim users who want simple remote syncing.
- Developers working across local & remote setups.
- Anyone tired of complex Rsync scripts or UI bloat.

**Final Thoughts**
- This was a weekend project that scratched a real itch.
- If you find it helpful, consider starring the repo or sharing feedback!

I use this plugin extensively, when I need to sync code from my mac to my other spare old machine. My mac has Silicon chip, so some intel programs do not run. I run them on intel architecture on my spare 4GB machine.

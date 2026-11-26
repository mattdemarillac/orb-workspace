# orb-workspace

Run a second, separate Claude Code on your Mac, signed in to a different
account, without switching logins.

The second Claude runs in an [OrbStack](https://orbstack.dev) Linux machine.
Nothing else is different: it works on the Mac's folders (the same files, at
the same paths), its commands run on the Mac (git, gh, node, php, docker,
Xcode, ...), and it has the Mac's Claude settings, hooks, plugins, skills and
MCP servers. Only the Claude is separate: its own login, history and memory.

```bash
orb-workspace work
orb -m work
cd ~/Sites/my-repo && claude
```

## Install

```bash
git clone https://github.com/mattdemarillac/orb-workspace.git ~/Sites/orb-workspace
mkdir -p ~/bin && ln -s ~/Sites/orb-workspace/orb-workspace ~/bin/orb-workspace
```

Put `~/bin` on your `PATH`, or run the script by its full path. Needs OrbStack
(`orb` CLI) on macOS. The machine side is tested on Ubuntu.

## What it does

Every step can be re-run safely. With no step flags it runs all of them in
this order; name steps to run only those.

| Step | What happens in the machine |
|---|---|
| `--toolchain` | the few packages Claude Code needs (curl, jq, rsync, ripgrep), Claude Code itself, your global `~/.claude/CLAUDE.md` |
| `--folders` | `~/.claude`, `~/.config`, `~/.cache`, `~/.local/bin`, `~/tmp` (as `TMPDIR`), `~/Downloads`, `~/Documents`, `~/Desktop`, XDG user dirs |
| `--shares` | links each `--share` folder (default `~/Sites`) into the machine's home. OrbStack mounts the Mac at the same paths (`/Users/...`), so both Claudes edit the same files, live. Anything already at that name in the machine is moved aside, not deleted |
| `--mac` | wrappers in `~/.local/bin` so `git`, `gh`, `node`, `npm`, `npx`, `yarn`, `pnpm`, `php`, `composer`, `docker`, `xcodebuild`, `xcrun`, `adb`, `emulator`, ... run on the Mac, in the same folder, with the shell's env vars. Node follows the project's `.nvmrc`. `mac-run <cmd>` runs anything else on the Mac. Also writes a section into the machine's `~/.claude/CLAUDE.md` that explains this to the agent |
| `--hosts` | the Mac's local hostnames (its `/etc/hosts` entries for `127.0.0.1`/`::1`) point at the Mac, so local sites and HTTP MCP servers resolve the same |
| `--claude` | copies the Mac's Claude Code setup: `settings.json`, hooks, plugins, skills, agents, commands, MCP servers, and per-project MCP servers for projects in the shared folders. Hooks and stdio MCP servers run on the Mac. Login, history and memory are not copied |
| `--hook FILE` | runs your own bash script inside the machine at the end |

Restart a Claude session that is already running in the machine after a run.

## Options

```
--share DIR          Mac folder to work in (repeatable, default ~/Sites)
--folder NAME[=DIR]  home folder in the machine; NAME=DIR links it to a Mac folder
--mac-cmd NAME       extra Mac command to run by name (repeatable)
--hook FILE          your own script, run in the machine at the end
--distro NAME        distro for a new machine (default ubuntu)
--dry-run            show what would be done, change nothing
```

## How it works

- **Files:** OrbStack mounts the Mac's filesystem in the machine at the same
  paths, so `/Users/you/Sites/x` is the same folder on both sides.
- **Commands:** OrbStack's `mac` command runs a program on the Mac in the
  matching folder. The wrappers use it, forwarding env vars (which `mac` does
  not) and selecting Node from `.nvmrc`.
- **Docker:** `docker` is the Mac's Docker, so both Claudes see the same
  containers. The machine runs none of its own.
- **Claude:** the machine's `~/.claude` and `~/.claude.json` hold the second
  login. The script keeps their settings in step with the Mac's.

## Things it does not do

- Isolate files. The second Claude can read and change anything the Mac user
  can. That is the point: the same files, a different Claude.
- Make `localhost` the Mac. Inside the machine, `localhost` is the machine;
  use the Mac's hostnames or `host.orb.internal`.

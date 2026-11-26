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

One command, no options. Every run does all of this and is safe to repeat:

- **Claude Code:** installs it in the machine with the few packages it needs
  (curl, jq, rsync, ripgrep).
- **Folders:** links `~/Sites`, `~/Downloads`, `~/Documents` and `~/Desktop` to
  the Mac's. OrbStack mounts the Mac at the same paths (`/Users/...`), so both
  Claudes edit the same files, live. Anything already at those names in the
  machine is moved aside, never deleted.
- **Mac tools:** wrappers in `~/.local/bin` so `git`, `gh`, `node`, `npm`,
  `npx`, `yarn`, `pnpm`, `php`, `composer`, `docker`, `curl`, `xcodebuild`,
  `xcrun`, `adb`, `emulator`, ... run on the Mac, in the same folder, with the
  shell's env vars. Node follows the project's `.nvmrc`. `mac-run <cmd>` runs
  anything else on the Mac. To bridge more commands by name, add them to
  `MAC_CMDS` at the top of the script.
- **Hosts:** the Mac's local hostnames (its `/etc/hosts` entries for
  `127.0.0.1`/`::1`) point at the Mac.
- **Claude config:** copies the Mac's `settings.json`, hooks, plugins, skills,
  agents, commands, MCP servers and per-project MCP servers. Hooks and stdio
  MCP servers run on the Mac; HTTP ones on `localhost` point at the Mac. Login,
  history and memory are not copied. A section in the machine's
  `~/.claude/CLAUDE.md` explains the setup to the agent.

Restart a Claude session that is already running in the machine after a run.

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

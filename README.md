# orb-workspace

Run a second, separate Claude Code on your Mac, signed in to a different
account, without switching logins.

The second Claude runs in an [OrbStack](https://orbstack.dev) Linux machine.
Everything else matches the Mac: the same repos, branches and env files, the
same Claude settings, hooks, plugins, skills and MCP servers, and the same Mac
tools (Xcode, simulators, the Android SDK) through a bridge. The only
difference is which Claude is running: its own login, history and memory.

```bash
orb-workspace work --org acme --org acme-labs --up api --up web
```

That creates the `work` machine (if needed) and copies into it every checkout
in `~/Sites` whose `origin` belongs to `acme` or `acme-labs`, then starts the
`api` and `web` Docker stacks inside the machine.

## Install

```bash
git clone https://github.com/mattdemarillac/orb-workspace.git ~/Sites/orb-workspace
mkdir -p ~/bin && ln -s ~/Sites/orb-workspace/orb-workspace ~/bin/orb-workspace
```

Put `~/bin` on your `PATH`, or run the script by its full path. Needs OrbStack
(`orb` CLI) on macOS. The machine side is tested on Ubuntu.

## What it does

Every step can be re-run safely; it skips what is already done. With no step
flags it runs all of them in this order. Name steps to run only those.

| Step | What happens in the machine |
|---|---|
| `--toolchain` | apt basics, Docker Engine + Compose, `gh`, PHP + Composer (if a repo has `composer.json`), nvm + every Node version named in a `.nvmrc` + Yarn (if a repo has `package.json`), Claude Code, your global `~/.claude/CLAUDE.md`, git name and email from the Mac |
| `--folders` | `~/.claude`, `~/.config`, `~/.cache`, `~/.local/bin`, `~/tmp` (as `TMPDIR`), `~/Downloads`, `~/Documents`, `~/Desktop`, XDG user dirs |
| `--mac` | bridges macOS tools into the machine: wrappers in `~/.local/bin` for `xcodebuild`, `xcrun` (simulators), `swift`, `pod`, `fastlane`, `adb`, `emulator`, `sdkmanager`, IDE launchers, and `docker-mac` (the Mac's Docker). Each runs on the Mac through OrbStack's `mac`, in the matching folder. Also writes a section into the machine's `~/.claude/CLAUDE.md` that tells agents how the environment works (re-written on each run, the rest of the file is kept) |
| `--clone` | clones each Mac checkout locally (fast, no network): same branch, uncommitted changes, linked worktrees. `origin` is rewritten to `https://github.com/<owner>/<repo>.git` |
| `--secrets` | copies gitignored env/key/cert files (`.env*`, `*.env`, `*.pem/.key/.pub/.crt`, `keys/`, `certs/`, `.auth/`, `.npmrc`, …) from the Mac checkouts, plus `~/.npmrc` and Composer `auth.json` (without the GitHub token). Never overwrites |
| `--claude` | copies the Mac's Claude Code setup: `settings.json`, hooks, plugins, skills, agents, commands, MCP servers (user level and per repo) and each repo's `.claude/settings.local.json` (permission lists are combined). Hooks and stdio MCP servers run on the Mac through `mac`, so Mac-only programs work unchanged. Login, history and memory are not copied |
| `--auth` | `gh auth login` inside the machine (interactive), then git and Composer use that login |
| `--deps` | seeds `vendor/` from the Mac checkout (so private packages need no GitHub access), then `composer install` (platform requirements ignored: the containers run the code), and `yarn` / `npm ci` / `pnpm install` with the repo's `.nvmrc`. Env vars that `.yarnrc.yml` / `.npmrc` reference are copied from the Mac's login shell. Native modules that fail to build are retried with older-toolchain settings (Python 3.10 for old node-gyp, `-include cstdint` for newer GCC) |
| `--docker` | reads each repo's compose file: creates external networks, adds `host-gateway` hostnames to `/etc/hosts`, lets the containers' uid write the checkout (ACLs). Builds and starts only the stacks named with `--up` |
| `--hook FILE` | runs your own bash script inside the machine at the end |

Logs for failed installs and builds are in `~/.cache/orb-workspace` in the machine.

## Options

```
--org ORG            GitHub owner to include (repeatable)
--from DIR           Mac folder to scan (repeatable, default ~/Sites)
--dest DIR           folder in the machine (default ~/Sites)
--repo NAME          only these folder names (repeatable)
--up NAME            build and start this repo's compose stack (repeatable)
--mac-cmd NAME       extra Mac command to bridge (repeatable)
--folder NAME[=DIR]  home folder in the machine; NAME=DIR links it to a Mac folder
--secret-regex RE    extra gitignored paths to copy
--hook FILE          your own script, run in the machine at the end
--distro NAME        distro for a new machine (default ubuntu)
--dry-run            show the repos and steps, change nothing
```

Examples:

```bash
# See what would be included
orb-workspace work --org acme --org acme-labs --dry-run

# Pick up a repo you cloned on the Mac since
orb-workspace work --org acme --repo new-service --clone --secrets --deps

# Share the Mac's Downloads with the machine instead of a separate one
orb-workspace work --org acme --folders --folder Downloads=~/Downloads

# Project-specific setup (DB seeding, extra hosts, …)
orb-workspace work --org acme --hook ~/work-hook.sh
```

A hook runs with `cwd` set to the repos folder and `DEST` and `MACHINE` set.

## Using the machine

```bash
orb -m work                # shell in the machine
cd ~/Sites/api && claude
```

- **Files from the Mac:** `~/OrbStack/<machine>/home/<you>/…`
- **Ports:** OrbStack forwards the machine's ports to the Mac's `localhost`, and
  `<machine>.orb.local` also reaches them. Stop the Mac copy of a stack first if
  both use the same ports.
- **Docker:** the machine runs its own Docker Engine. Its containers don't show
  in the Mac's `docker ps` or the OrbStack app. Use `orb -m work docker ps`.
- **The Mac from the machine:** `host.orb.internal:<port>`.
- **Mac tools:** after `--mac`, `xcodebuild`, `xcrun simctl`, `adb` and the rest
  work by name inside the machine; anything else runs with `mac <command>`.
  Linux paths in arguments are translated to the Mac's view of them.
- **PhpStorm and other JetBrains IDEs:** use Remote Development (Gateway) over
  SSH to `127.0.0.1:32222`, user `<machine>` (OrbStack's SSH). Terminals,
  Docker and Xdebug then all run in the machine.

## Things it does not do

- Install dependencies in subfolders (monorepos): use a hook.
- Authorise the GitHub token for SAML SSO orgs. After `--auth`, go to GitHub →
  Settings → Applications → GitHub CLI → Configure SSO.
- Isolate the Mac's files completely. OrbStack mounts the Mac filesystem at
  `/mnt/mac` (and `/Users`) inside every machine.

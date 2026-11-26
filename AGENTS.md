# AGENTS.md

Notes for coding agents working on this repo. Read `README.md` for what the tool
does from a user's point of view.

**Purpose:** run a separate Claude Code (its own login) in an OrbStack machine.
Everything else must be the Mac's: the same files, the same commands, the same
Docker, the same Claude config. Nothing more, nothing less. Judge every change
by that: does it make the machine behave like the Mac?

## Layout

- `orb-workspace`: the whole tool, one bash script. No dependencies on the Mac
  besides OrbStack's `orb` CLI.
- `README.md`: user docs. Keep "What it does" in step with the script.

## How the script runs

The script runs twice:

1. **On the Mac** (no `__vm` first argument). Takes only the machine name,
   creates the OrbStack machine if missing, then `exec`s itself inside it:
   `orb -m <machine> env … bash /mnt/mac<path of script> __vm`.
2. **In the machine** (`__vm`). Does the real work, top to bottom: Claude
   Code, folders, Mac tools, hosts, Claude config, summary. `HOST_HOME` and
   `MACHINE` come from the Mac side.

There are no options or step flags: a full run is quick and idempotent. Keep it
that way; settings that might change (bridged commands, linked folders) are
variables at the top of the script.

## Rules

- **Don't duplicate the Mac.** No clones, no copied secrets, no machine-side
  Docker stacks or language runtimes. Share the Mac's folders and run tools on
  the Mac. Duplicates drift, and edits made in the machine must show on the Mac.
- **Mac side must stay bash 3.2 compatible** (macOS `/bin/bash`): no
  associative arrays, `mapfile`, `${x,,}` or `declare -n` above the `__vm`
  line. Don't put a heredoc inside `$( … )`: bash 3.2 can fail to parse the
  whole file.
- **Every step must be idempotent.** Re-running on a finished machine must
  change nothing. Never delete the user's files: move them aside.
- **OrbStack's `mac`** rewrites absolute args that exist in Linux
  (`/usr/bin/…`, `/opt/homebrew/…`) to the machine's files, and passes no env
  vars. Call Mac commands by name through `mac-run` (which forwards env vars),
  or wrap a whole command in one quoted string: `mac sh -c "exec 'cmd' 'arg'"`
  (how bridged hooks and MCP servers work). `/Users/…` paths are safe.
- **Wrappers** carry the `# orb-workspace: macOS bridge` line; only those may
  be rewritten, never a real command of the same name.
- **The machine's `~/.claude/CLAUDE.md`** belongs to the user. The script only
  replaces the part between `<!-- orb-workspace:start` and
  `<!-- orb-workspace:end -->`. The same goes for the marked block in
  `/etc/hosts`.
- **Shell environment goes through `add_env`** (`~/.orb-workspace.env`, loaded
  from `.profile` and the top of `.bashrc`). Ubuntu's `.bashrc` returns early
  for non-interactive shells.
- **Failures don't stop the run.** Add them to `FAILED` (shown at the end, exit
  code 1) and keep going.
- **No secrets in output.** Print names and counts, never values.

## Testing

There are no automated tests. Before committing:

```bash
bash -n orb-workspace                     # syntax (Mac bash 3.2)
```

For anything that changes a step, test on a throwaway machine, check the
result from inside it, then delete it:

```bash
orb-workspace owtest
orb -m owtest bash -lc 'cd ~/Sites/<repo> && git status && node -v && docker ps'
orb delete -f owtest
```

Then re-run against an existing machine to check nothing is redone.

## Commits

Short imperative subject, body only when the reason isn't obvious. No
`Co-Authored-By` trailers or "Generated with" lines.

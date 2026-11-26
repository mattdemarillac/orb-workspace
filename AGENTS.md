# AGENTS.md

Notes for coding agents working on this repo. Read `README.md` for what the tool
does from a user's point of view.

**Purpose:** run a separate Claude Code (its own login) in an OrbStack machine.
The machine must otherwise match the Mac: nothing more, nothing less. Judge
every change by that: does it make the machine behave like the Mac?

## Layout

- `orb-workspace`: the whole tool, one bash script. No dependencies on the Mac
  besides OrbStack's `orb` CLI and git.
- `README.md`: user docs. Keep the step table and options list in step with the
  script's `usage()`.

## How the script runs

The script runs twice:

1. **On the Mac** (no `__vm` first argument). Parses the args just enough to
   make `--from`, `--hook` and `--folder NAME=DIR` paths absolute, creates the
   OrbStack machine if missing, then `exec`s itself inside the machine:
   `orb -m <machine> env … bash /mnt/mac<path of script> __vm <args>`.
2. **In the machine** (`__vm`). Does all of the real work. Mac files are read
   through `/mnt/mac/…`. `HOST_HOME`, `GIT_NAME`, `GIT_EMAIL` and `MACHINE` come
   from the Mac side through `env`.

Steps run in this order: discover repos, `toolchain`, `folders`, `mac`, `clone`,
`secrets`, `claude`, `auth`, `deps`, `docker`, hook, summary. `want <step>` decides if a
step runs: all steps when none are named, otherwise only the named ones.

## Rules

- **Mac side must stay bash 3.2 compatible** (macOS `/bin/bash`): no
  associative arrays, `mapfile`, `${x,,}` or `declare -n` above the `__vm`
  line. Guard empty arrays with `${ARR[@]+"${ARR[@]}"}`.
- **Every step must be idempotent.** Re-running on a finished machine must
  change nothing. Check before you create, copy or append. Never overwrite a
  file the user may have changed (secrets use "copy only if missing").
- **Don't change user choices.** For example the nvm default alias is set only
  when none exists.
- **Shell environment goes through `add_env`**, which writes
  `~/.orb-workspace.env` and sources it from `.profile` and the top of
  `.bashrc`. Do not append to the end of `.bashrc`: Ubuntu's `.bashrc` returns
  early for non-interactive shells.
- **Keep it generic.** Nothing specific to one org, repo or project goes in the
  script. That belongs in a `--hook` script. Detect needs from the repos
  (`composer.json`, `package.json`, `.nvmrc`, lockfiles, compose files).
- **Failures don't stop the run.** Add them to `FAILED` (shown at the end, exit
  code 1) and keep going. Put long output in `$LOGS/<repo>-<what>.log`.
- **No secrets in output or logs.** Print file counts and names, never values.
  Don't copy the GitHub token from the Mac; `--auth` makes a new one.
- **OrbStack's `mac` rewrites absolute paths** that also exist in Linux
  (`/usr/bin/…`, `/opt/homebrew/…`) to this machine's files, and it does not
  pass environment variables. Call Mac commands by name (`mac xcodebuild`), or
  wrap the whole command in one quoted string: `mac sh -c "exec 'cmd' 'arg'"`
  (that is how bridged hooks and MCP servers work).
  Wrappers carry the `# orb-workspace: macOS bridge` line; only those may be
  rewritten, never a real Linux command of the same name.
- **The machine's `~/.claude/CLAUDE.md`** belongs to the user. The script only
  replaces the part between `<!-- orb-workspace:start` and
  `<!-- orb-workspace:end -->`.
- **Heredocs:** don't put a heredoc inside `$( … )`; macOS bash 3.2 fails to
  parse the whole file when the heredoc has escaped backticks. Write it to a
  file instead.
- **Docker** may need `sudo` until the user starts a new shell after being
  added to the `docker` group. Use the `DOCKER` array, not `docker`.

## Testing

There are no automated tests. Before committing:

```bash
bash -n orb-workspace                                   # syntax (Mac bash 3.2)
orb-workspace <machine> --org <org> --dry-run           # discovery
```

For anything that changes a step, test it on a throwaway machine and delete it
afterwards:

```bash
orb-workspace owtest --org <org> --repo <small repo> --up <small repo> \
  --toolchain --folders --clone --secrets --deps --docker
orb delete -f owtest
```

Then re-run the same command against an existing machine to check nothing is
redone.

## Commits

Short imperative subject, body only when the reason isn't obvious. No
`Co-Authored-By` trailers or "Generated with" lines.

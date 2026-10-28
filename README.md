# suparth-agent-skills

Agent skills I use day to day, packaged so you can install them into Claude Code, Cursor, Codex and other agents that support [skills](https://skills.sh).

## Install

Install every skill in this repo:

```bash
npx skills add suparthghimire/suparth-agent-skills
```

Or pick what you need:

```bash
# see which skills are available
npx skills add suparthghimire/suparth-agent-skills --list

# install a single skill
npx skills add suparthghimire/suparth-agent-skills --skill rebase

# install globally (all projects) instead of into the current project
npx skills add suparthghimire/suparth-agent-skills -g
```

The CLI asks which agents to install into. Run `npx skills --help` for all options.

## Skills

| Skill | What it does |
| --- | --- |
| [rebase](#rebase) | Rebases the current branch onto a branch on a remote, works through conflicts, and reports the result. |

### rebase

```
/rebase <target branch>, <remote alias>
```

```
/rebase main, origin
/rebase develop upstream
```

- **Arguments:** the first is the branch to rebase onto, the second is the remote to pull from. If either one is missing, the agent asks for it. It never assumes `origin` or `main`.
- **Shows the setup first:** before it changes anything, it shows the current branch, the target branch, and the remote's name and URL.
- **Runs safety checks:** it stops on a detached HEAD, if you're already on the target branch, if the remote or branch doesn't exist, or if a rebase is already in progress. If you have uncommitted changes, it asks whether to commit, stash or abort.
- **Rebases in two steps:** it pulls the latest version of your own branch from the remote, then rebases onto `<remote>/<target>`.
- **Resolves conflicts:** before editing anything, it looks at both sides: what the target branch changed, and what each of your commits meant to do. It asks you whenever the right result isn't clear. It never aborts, skips commits or resets on its own.
- **Checks the result and reports back:** it typechecks and tests what it touched. Then it reports the starting and ending commit, each conflict and how it was resolved, and a recovery command (`git reset --hard <start-sha>`).
- **Doesn't push:** the history was rewritten, so pushing needs `git push --force-with-lease`. The skill leaves that to you.

This skill only runs when you call `/rebase` (`disable-model-invocation: true`). The agent never starts it on its own.

Some of the conflict guidance is written for pnpm and Prisma projects (lockfiles and migrations).

## Repository layout

```
skills/
└── <skill-name>/
    ├── SKILL.md      # frontmatter (name, description) + instructions for the agent
    └── ...           # optional scripts, references, templates
```

To add a skill:

1. Create `skills/<skill-name>/SKILL.md`. In its frontmatter, `name` must match the folder name. `description` tells the agent when to use the skill.
2. Check that the CLI finds it: `npx skills add ./ --list`
3. Add a row to the [Skills](#skills) table above.

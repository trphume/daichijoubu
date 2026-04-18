# Project-Only Skills

A **project-only skill** lives inside a single project's source tree and
never leaves it. It encodes conventions, runbooks, and context that only
make sense in this codebase — things like "how our migrations work", "our
service's oncall playbook", "how to write tests against our fake client".

Project-only skills are not routed through a shared repo, but they *can*
share the same installation machinery (`.agents/skills/`, symlinks,
`skills-lock.json`) as shared skills. This page covers both the "just edit
the file directly" shortcut and the CLI-managed approach.

---

## Two variants

| Variant | Target agents | Authoring location | Committed | CLI involvement |
|---|---|---|---|---|
| **Single-agent** | One | `<agent>/skills/<name>/` directly | the skill files | none — edit and commit |
| **Multi-agent fan-out** | Two or more | `_skills/<name>/` | `_skills/**` + `skills-lock.json` | `npx skills add ./_skills` to regenerate canonical + symlinks |

Pick one variant per project. Mixing both for project-only skills gets
confusing — teammates can't tell where to edit.

`<agent>` is a placeholder for whichever agent your team uses — e.g.
`.claude` for Claude Code, `.cursor` for Cursor, `.codex` for Codex. See
`npx skills add --help` for the full list.

---

## Variant A: Single-agent (simplest)

If your project only targets one agent, skip the CLI entirely. Author the
skill directly in the agent's discovery directory — `<agent>/skills/<name>/`.

### Create

```bash
# Replace .claude with your agent's directory (.cursor, .codex, etc.)
mkdir -p .claude/skills/migration-helper
cd .claude/skills/migration-helper
npx skills init
```

`npx skills init` (called without an argument) uses the current directory's
basename as the skill name — so this produces
`<agent>/skills/migration-helper/SKILL.md` with `name: migration-helper` in
the frontmatter.

(Or just write the four-line frontmatter yourself — no tool needed.)

### Edit for your project

Unlike a shared skill, you can be **specific to this codebase**:

```markdown
---
name: migration-helper
description: Use when writing or reviewing database migrations in this service. Enforces our conventions: reversible migrations, no implicit NOT NULL, zero-downtime backfill pattern.
---

# Migration helper

Our migrations live in `src/db/migrations/`. Each migration must:

1. Be reversible (both `up` and `down`).
2. Never add `NOT NULL` to an existing column without a default.
3. For data backfills, use the two-phase pattern documented in
   `docs/migrations.md`.

See `src/db/migrations/20240115_add_user_tier.ts` for a canonical example.
```

File paths, team names, service-specific invariants — all fine.

### Commit

```bash
git add <agent>/skills/migration-helper
git commit -m "Add migration-helper skill"
```

### Lifecycle

- **Editing**: change `SKILL.md` directly, commit. The agent picks up the
  new content on next session.
- **Removing**: `rm -rf <agent>/skills/migration-helper`, commit.
- **No lockfile involvement.** `npx skills list` will still discover it
  (it walks agent directories), but `skills update` and `skills remove`
  only touch entries in `skills-lock.json`.

### When to use this variant

- Your project targets one agent and there's no plan to add others.
- You want the shortest possible edit loop: change file → commit → done.
- You don't need any of the CLI's update/remove plumbing for these skills.

---

## Variant B: Multi-agent fan-out

When the project supports multiple agents, duplicating the skill into each
agent's directory is painful. Instead, author it once in `_skills/` and let
the CLI fan it out.

### Directory convention

```
my-service/
├── _skills/                        ← source of truth (committed)
│   ├── migration-helper/
│   │   └── SKILL.md
│   └── oncall-runbook/
│       └── SKILL.md
├── .agents/                        ← generated, gitignored
│   └── skills/
│       ├── migration-helper/       ← canonical copy
│       └── oncall-runbook/
├── <agent-1>/                      ← generated, gitignored
│   └── skills/
│       ├── migration-helper/       → ../../.agents/skills/migration-helper/
│       └── oncall-runbook/
└── <agent-2>/                      ← generated, gitignored (one dir per target agent)
    └── skills/
        └── ...                     → ../../.agents/skills/...
```

Concrete agent directory names are `.claude/`, `.cursor/`, `.codex/`, etc.
— one per agent you target.

> The underscore prefix on `_skills/` distinguishes it from the generated
> `.agents/skills/` directory. `skills/` without the prefix also works,
> but `_skills/` is harder to confuse.

### Create

```bash
mkdir -p _skills/migration-helper
cd _skills/migration-helper && npx skills init && cd -
# edit _skills/migration-helper/SKILL.md
```

### Fan out

```bash
npx skills add ./_skills --skill migration-helper -a <agent> -y
```

Pass one `-a <agent>` for each agent the project targets (e.g.
`-a claude-code -a cursor`).

What that does:

1. Reads `_skills/migration-helper/SKILL.md`.
2. Copies to `.agents/skills/migration-helper/`.
3. Creates symlinks into each targeted `<agent>/skills/` directory.
4. Writes a lock entry to `skills-lock.json`:

```json
{
  "migration-helper": {
    "source": "./_skills",
    "sourceType": "local",
    "computedHash": "sha256:..."
  }
}
```

The `sourceType: "local"` marker is important — it tells
`skills update -p` to **skip this entry** on updates. The CLI does not
try to re-read from `./_skills` on update, because updates are about
pulling from remote sources. Local skills are managed manually.

### Edit loop

Because the fan-out is a *copy*, editing `_skills/<name>/` doesn't
auto-propagate. Re-run the sync after each edit:

```bash
npx skills add ./_skills --skill '*' -a <agent> -y
```

The `--skill '*'` wildcard installs every skill discovered in `./_skills`,
so you don't have to list them individually as the collection grows.

The edit loop is: change `_skills/migration-helper/SKILL.md` → run the
sync command → commit.

> Teammates will forget the command. Wrap it in whatever task runner your
> project uses — `package.json` script, `Makefile` target, `justfile`
> recipe, shell alias, whatever. The important thing is that the sync
> command is one keystroke away, not that it's implemented in any
> particular tool.

### Commit

```bash
git add _skills/migration-helper skills-lock.json
git commit -m "Add project-local migration-helper skill"
```

**Commit:** `_skills/**`, `skills-lock.json`.
**Gitignore:** `.agents/skills/`, and each targeted `<agent>/skills/`
(e.g. `.claude/skills/`, `.cursor/skills/`).

---

## Onboarding a new teammate

If you're using Variant B with `.agents/` gitignored, new teammates need to
regenerate the canonical copies and symlinks on clone. Run the sync command:

```bash
git clone <repo>
cd <repo>
npx skills add ./_skills --skill '*' -a <agent> -y
```

For a project that also uses [shared skills](./01-shared-skills.md), the
bootstrap is two commands — first refresh shared skills from their pinned
refs, then sync the locals:

```bash
npx skills update -p -y
npx skills add ./_skills --skill '*' -a <agent> -y
```

See the [combined workflow](./03-combined-workflow.md) for the full
bootstrap sequence and suggestions on how to wrap it in your task runner
of choice.

---

## Anti-patterns

- **Don't hand-edit `.agents/skills/<name>/`.** It's the canonical copy —
  the CLI can and will wipe/recreate it when you run `skills add`. Edit
  `_skills/<name>/` instead.
- **Don't commit `<agent>/skills/` symlinks.** Symlinks are unreliable
  across OSes and across git clones. Let each teammate regenerate them by
  running the sync command after clone.
- **Don't reuse a skill name across the shared and local layers.** The
  lockfile is keyed by skill name; whichever layer installs last wins. If
  you want to fork a shared skill, first `skills remove <name>` then
  add it as a local skill.
- **Don't author project-only skills in a "personal fork" of the shared
  repo.** That creates drift and merge pain. Keep them in-tree.

# Shared Skills Repo

A **shared skills repo** is a normal git repository that hosts reusable
`SKILL.md` files. It's consumed by downstream projects via `npx skills add`.
There is no build step, no registry, no manifest — the CLI clones the repo,
walks a list of known directories, and parses frontmatter.

Because it's just a git repo, it can live anywhere: GitHub, GitLab, an
internal Gitea, SSH-only, whatever. Private repos work as long as the
installing machine can `git clone` them.

---

## 1. Repo layout

Recommended structure for a multi-skill shared repo:

```
acme/skills/                    ← the repo root
├── README.md                   ← human-facing index (not read by CLI)
└── skills/                     ← one of several discovery locations
    ├── code-review/
    │   ├── SKILL.md
    │   └── references/         ← extra files shipped with the skill
    ├── rfc-template/
    │   └── SKILL.md
    └── deploy-checklist/
        └── SKILL.md
```

Discovery locations the CLI checks inside a source repo (first match wins):

- Repo root (if a single skill lives there with `SKILL.md`).
- `skills/` (recommended).
- `skills/.curated/` — surfaces as "curated" on skills.sh.
- `skills/.experimental/` — hidden unless `INSTALL_INTERNAL_SKILLS=1`.
- `.claude-plugin/marketplace.json` — plugin-marketplace compatibility.

If none match, the CLI falls back to a recursive walk.

---

## 2. Authoring a new shared skill

From inside the shared repo:

```bash
cd acme-skills/skills
npx skills init code-review
```

This scaffolds `code-review/SKILL.md`:

```markdown
---
name: code-review
description: A brief description of what this skill does
---

# code-review

Instructions for the agent to follow when this skill is activated.

## When to use
...
```

### Rules for frontmatter

- **`name`** — lowercase, hyphens only. The installer sanitizes on the way in
  (`Code Review` becomes `code-review`), but authoring-time correctness
  avoids surprises.
- **`description`** — the sole signal the agent uses to decide whether to
  dispatch this skill. It must be specific and include trigger phrases.
  Treat it as the skill's marketing copy.
- **`metadata.internal: true`** (optional) — hides the skill from default
  discovery. Useful for WIP skills you want in the repo but not yet
  installable by default.

### Extra files

You can ship anything alongside `SKILL.md` — reference docs, examples,
snippets. `copyDirectory` dereferences symlinks and skips `.git`, dotfiles,
`__pycache__`, and `metadata.json`.

---

## 3. Publishing

Standard git. No build, no publish step.

```bash
git add skills/code-review
git commit -m "Add code-review skill"
git push
```

### Versioning

Tag releases semantically. Consumers can pin to tags, which isolates them
from upstream drift:

```bash
git tag v1.3.0
git push --tags
```

Tagging is the mechanism that protects consumers from
[the silent-drop problem](#why-tagging-matters): if you delete a skill from
the repo, anyone pinned to an older tag keeps working forever.

---

## 4. Consuming from a project

From the consumer project's root (**no `-g`** — we want project scope):

```bash
cd my-service
npx skills add acme/skills#v1.3.0 --skill code-review -a <agent>
```

Replace `<agent>` with the agent(s) your project targets — e.g.
`claude-code`, `cursor`, `codex`. Repeat `-a` for multiple agents.

What that does (using Claude Code as the concrete example):

1. Resolves `acme/skills` as GitHub shorthand → `https://github.com/acme/skills.git`.
2. Clones at ref `v1.3.0`.
3. Discovers skills and filters to `code-review`.
4. Copies to `./.agents/skills/code-review/` (canonical).
5. Creates symlink `./<agent>/skills/code-review/ → ../../.agents/skills/code-review/`
   (e.g. `./.claude/skills/code-review/` for Claude Code,
   `./.cursor/skills/code-review/` for Cursor).
6. Writes a lock entry to `./skills-lock.json`:

```json
{
  "version": 1,
  "skills": {
    "code-review": {
      "source": "acme/skills",
      "ref": "v1.3.0",
      "sourceType": "github",
      "computedHash": "sha256:..."
    }
  }
}
```

### Source formats accepted by `skills add`

| Form | Example |
|---|---|
| GitHub shorthand | `acme/skills` |
| GitHub URL | `https://github.com/acme/skills` |
| Subpath in repo | `https://github.com/acme/skills/tree/main/skills/code-review` |
| GitLab URL | `https://gitlab.com/acme/skills` |
| Any git URL | `git@github.com:acme/skills.git` |
| Pinned ref | `acme/skills#v1.3.0` or `acme/skills#main` |

### Choosing the target agent(s)

Use `-a <agent>` (repeatable) to fan out to specific agents. Omit `-a` and
the CLI detects which agents you have installed and prompts. Run
`npx skills add --help` for the full list of recognized agents.

---

## 5. Updating

Project-scope update re-pulls **every** shared lock entry unconditionally:

```bash
npx skills update -p -y
```

- Entries pinned to tags (`#v1.3.0`) re-pull from that tag — stable unless
  the tag is moved.
- Entries tracking a branch (`#main` or unpinned → default branch) re-pull
  the current HEAD of that branch.
- On success, `computedHash` in the lockfile is refreshed.
- On failure (skill deleted upstream, repo unavailable), the CLI prints
  `✗ Failed to update <name>` but **leaves the on-disk copy and lock entry
  intact**. Nothing is silently removed.

To bump a pin, re-run `skills add` with the new ref:

```bash
npx skills add acme/skills#v1.4.0 --skill code-review
```

This overwrites the lockfile entry with the new ref.

### Why tagging matters

If someone deletes a skill from the shared repo:

- **Pinned consumers** keep working forever (the old tag still has the
  skill). Updates are no-ops for that skill.
- **Unpinned consumers** get a visible update failure on next
  `skills update -p` — the CLI prints `✗ Failed to update`, but the on-disk
  skill keeps working until an explicit `skills remove`.

Unpin only when you're comfortable taking rolling updates from upstream.

---

## 6. Removing

```bash
npx skills remove code-review
```

Deletes `.agents/skills/code-review/`, the agent symlink(s), and the lockfile
entry. Commit the lockfile change so teammates see the removal on next pull.

---

## 7. Governance for the shared repo

- **Tag releases.** Semantic versioning is fine (`v1.3.0`). Consumers pin.
- **Treat `SKILL.md` as behavior-defining code.** Review frontmatter
  `description` changes carefully — they control agent dispatch.
- **Keep skills self-contained.** A skill should not reference files outside
  its own directory; those won't ship.
- **Use `.experimental/`** for WIP skills. They're invisible to default
  installs, which prevents accidental adoption.
- **Keep a `README.md`** at repo root with a one-line per skill description
  and install snippet — humans need it even though the CLI doesn't.

---

## 8. Anti-patterns

- **Don't reference absolute paths in a shared skill.** They won't resolve
  on consumer machines. Use paths relative to the skill's own directory.
- **Don't assume the consumer's directory layout.** If your skill needs to
  know where migrations live, say so in the skill body and let the consumer
  customize their project accordingly.
- **Don't publish without tagging.** Rolling `main` is fine for testing,
  but production consumers should pin.
- **Don't write private or team-specific context into a shared skill** —
  that belongs in a [project-only skill](./02-project-skills.md).

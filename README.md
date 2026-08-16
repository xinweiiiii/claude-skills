# Claude Skills

A central collection of custom [Claude Code](https://claude.com/claude-code) skills — reusable instruction sets that teach Claude how to approach a specific kind of work.

The goal is one place to find, share, and reuse skills instead of everyone rebuilding the same prompt from scratch. Take the ones you want, leave the rest.

---

## What a skill is

A skill is a folder with a `SKILL.md` file. The frontmatter tells Claude what the skill covers and when it applies; the body is the guidance Claude follows once it loads.

```markdown
---
name: coding-patterns
description: Patterns to follow when writing application code that must be maintained...
---

# Production Coding Patterns

...guidance Claude follows...
```

Claude reads the `description` of every available skill and loads the full body only when the work matches — so the description does the routing. Users can also invoke one directly by name with `/<skill-name>`.

---

## Available skills

| Skill | Domain | What it covers |
|---|---|---|
| [`coding-patterns`](engineering/.claude/skills/coding-patterns/SKILL.md) | Engineering | 17 patterns for application code that survives maintenance and growth — guard clauses, boundaries, bounded concurrency, idempotency, durable jobs, backpressure, additive contract changes |
| [`well-designed-api-patterns`](engineering/.claude/skills/well-designed-api-patterns/SKILL.md) | Engineering | API design that stays predictable, backward-compatible, resilient, and self-documenting — versioning, deprecation, error contracts, pagination |

More domains planned: **writing**, **process flow**, and others as they get built.

---

## Using a skill

Skills are picked up from a `.claude/skills/` directory. Copy the skill folder to whichever scope you want:

**For all your projects** — install into your personal skills directory:

```bash
git clone git@github.com:xinweiiiii/claude-skills.git
cp -r claude-skills/engineering/.claude/skills/coding-patterns ~/.claude/skills/
```

**For one project** — commit it alongside the code so your team gets it too:

```bash
mkdir -p .claude/skills
cp -r /path/to/claude-skills/engineering/.claude/skills/coding-patterns .claude/skills/
```

Then start (or restart) Claude Code. Confirm it loaded by running `/coding-patterns`, or just describe work it covers and let Claude pick it up on its own.

To take everything at once:

```bash
cp -r claude-skills/*/.claude/skills/* ~/.claude/skills/
```

---

## Repo layout

Skills are grouped by domain, each holding a standard `.claude/skills/` tree:

```
claude-skills/
├── README.md
└── engineering/
    └── .claude/
        └── skills/
            ├── coding-patterns/
            │   └── SKILL.md
            └── well-designed-api-patterns/
                └── SKILL.md
```

The `<domain>/.claude/skills/` nesting means each domain folder is itself a valid Claude Code workspace — open `engineering/` directly and its skills are already active, no copying needed. It also keeps the copy commands above uniform across domains.

New domains follow the same shape: `writing/.claude/skills/`, `process-flow/.claude/skills/`, and so on.

---

## Adding a skill

1. Create `<domain>/.claude/skills/<skill-name>/SKILL.md`.
2. Use kebab-case for `<skill-name>`, and make the frontmatter `name` match the folder exactly.
3. Write the `description` as a trigger, not a summary. Claude matches against it to decide whether to load the skill, so name the situations it applies to.

   > Good: *"Patterns for writing application code that must be maintained and must survive growth in traffic. Consult while designing services, APIs, background jobs, and integrations."*
   >
   > Weak: *"Coding best practices."*

4. Write the body as instructions Claude acts on. Concrete code examples beat prose. Say what to write, and say when the pattern does **not** apply — an over-applied pattern is its own defect.
5. Add a row to the table above.

### Conventions

- **Keep it tight.** Roughly 500 lines is a working ceiling; a skill consumes context every time it loads.
- **Lead with defaults.** A short list of "write these unless there's a reason not to" up front is what Claude uses mid-task, before it reads any deeper.
- **Pick one job per skill.** A skill covering both API design and testing strategy will load for neither cleanly. Split it.
- **Show the wrong shape** where the wrong shape is the tempting one, marked clearly (`// Do not write this`).

---

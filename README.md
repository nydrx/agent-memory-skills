# agent-memory-skills

> Drop-in [Agent Skills](https://agentskills.io) that give AI coding agents **persistent memory** across context windows, terminal sessions, and tool boundaries.

[![npx skills](https://img.shields.io/badge/install%20via-npx%20skills-000?logo=vercel)](https://github.com/vercel-labs/skills)
[![Works with](https://img.shields.io/badge/Claude%20Code%20%C2%B7%20Cursor%20%C2%B7%20Codex%20%C2%B7%20Windsurf%20%C2%B7%20Copilot-50%2B%20agents-2d3748)](#supported-agents)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#contributing)

A curated, version-controlled collection of skills focused on one problem: **agent amnesia**. Built on the open [Agent Skills specification](https://agentskills.io), so the same `SKILL.md` works in Claude Code, Cursor, Codex, OpenCode, Windsurf, GitHub Copilot, and 50+ other agents.

---

## Why

Modern coding agents lose state the moment the context window closes. Every fresh chat re-derives goals, re-learns sharp edges, re-asks the same questions. These skills install a small, markdown-only convention into your repos so that *future* agents — Claude, Cursor, Codex, whatever comes next — pick up where the last one left off.

No plugins. No daemons. No vendor lock-in. Just markdown and a protocol.

---

## Install

The fastest path is the official [`skills`](https://github.com/vercel-labs/skills) CLI by Vercel Labs. It auto-detects which agents you have installed and drops the skill into the right place.

### One-liner — install everything

```bash
npx skills add nydrx/agent-memory-skills
```

This walks you through picking which skills and which agents to target. For non-interactive (CI, dotfiles, scripts):

```bash
npx skills add nydrx/agent-memory-skills --skill '*' --agent claude-code -g -y
```

### Pick a specific skill

```bash
# global (available in every project)
npx skills add nydrx/agent-memory-skills --skill init-native-memory -g

# project-only (committed alongside your code)
npx skills add nydrx/agent-memory-skills --skill init-native-memory
```

### Target specific agents

```bash
# install to Claude Code + Cursor + Codex at once
npx skills add nydrx/agent-memory-skills -a claude-code -a cursor -a codex
```

### Browse before installing

```bash
npx skills add nydrx/agent-memory-skills --list
```

### Manage installed skills

```bash
npx skills list                              # what do I have installed?
npx skills update                            # pull latest from upstream
npx skills remove init-native-memory         # uninstall
```

After install, **start a new agent session** so the skill loader picks up the new `SKILL.md`.

> **No `npx`?** Clone + symlink works too: `git clone https://github.com/nydrx/agent-memory-skills && ln -s $(pwd)/agent-memory-skills/skills/init-native-memory ~/.claude/skills/init-native-memory`

---

## Available skills

| Skill | Trigger | What it does |
|---|---|---|
| [**`init-native-memory`**](skills/init-native-memory/SKILL.md) | `/init-native-memory`, "set up agent memory", "the AI keeps forgetting context" | Scaffolds a `.memory/` (committed) + `.scratch/` (gitignored) layer into any git repo. Includes templates for goals, decisions, lessons, postmortems, and a four-tier progressive-disclosure read protocol so session-start cost stays O(1) as the corpus grows. |

More skills coming. [Open an issue](https://github.com/nydrx/agent-memory-skills/issues) if you have one in mind.

---

## Usage

Once installed, invoke a skill by its slash command or by describing the problem it solves. For example:

```
/init-native-memory
```

or naturally:

> "The AI keeps forgetting our project context across chats — can we fix that?"

The agent's skill loader matches the `description` field in each skill's frontmatter against your message and offers to invoke the right one. Each skill is self-contained — its `SKILL.md` documents exactly what it does, when it triggers, and what it writes to your repo.

---

## Supported agents

This collection works with every agent the [`skills`](https://github.com/vercel-labs/skills) CLI supports — currently 50+, including:

**Claude Code · Cursor · Codex · OpenCode · Windsurf · GitHub Copilot · Gemini CLI · Qwen Code · Cline · Continue · Goose · Roo Code · Kiro CLI · Warp · Trae · Replit · Augment · Amp · Crush · Junie · …**

See the [full list and target paths](https://github.com/vercel-labs/skills#supported-agents) in the upstream CLI README.

---

## Repository layout

```
.
├── README.md
├── LICENSE
└── skills/
    └── <skill-name>/
        ├── SKILL.md       # frontmatter (name, description) + skill body
        ├── assets/        # files the skill copies into target repos (optional)
        └── references/    # supporting docs the skill links to (optional)
```

This layout is the standard `skills` CLI discovers — drop any compatible skill in `skills/<name>/` and it's installable.

---

## Contributing

PRs welcome. To add a new skill:

1. **Scaffold** with the upstream CLI:

   ```bash
   npx skills init skills/<your-skill-name>
   ```

   Or by hand — `skills/<your-skill-name>/SKILL.md` with frontmatter:

   ```markdown
   ---
   name: <your-skill-name>
   description: One paragraph with concrete trigger phrases. This is what agents match against the user's message to decide when to invoke the skill, so be specific about both *when to trigger* and *when not to*.
   ---

   # <Skill Title>

   <skill body — mental model, when-to-invoke, steps, anti-patterns, references>
   ```

2. **Co-locate** assets in `skills/<name>/assets/` and supporting docs in `skills/<name>/references/`. No cross-skill imports — installation should be a single atomic operation.

3. **Document** the skill: add a row to the **Available skills** table above with its trigger and a one-line summary.

4. **Test locally** with the CLI:

   ```bash
   npx skills add ./ --list                    # confirm it's discovered
   npx skills add ./ --skill <your-skill-name> # install from local path
   ```

5. **Open a PR** with a short rationale: what pain does this skill solve, and why is it general enough to live in a shared collection?

### Style

- Skills should be **self-contained, markdown-only, and tool-agnostic** where possible. The point of this collection is portability across agents.
- Frontmatter `description` carries trigger phrases — overinvest here. A skill that doesn't trigger reliably is a skill that doesn't exist.
- Include an **anti-patterns** section in the body. Telling agents what *not* to do is as valuable as telling them what to do.
- Follow the [Agent Skills specification](https://agentskills.io) so your skill works everywhere.

---

## License

[MIT](LICENSE) © agent-memory-skills contributors. Use it, fork it, ship it.

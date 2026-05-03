# agent-memory-skills

A shared collection of [Claude Code](https://claude.com/claude-code) skills, kept here so they can be installed, version-controlled, and updated across machines.

## Repository layout

```
.
└── skills/
    └── <skill-name>/
        ├── SKILL.md              # frontmatter (name, description) + skill body
        ├── assets/               # files the skill copies into target repos (optional)
        └── references/           # supporting docs the skill links to (optional)
```

Each `skills/<skill-name>/` directory is a complete, self-contained Claude Code skill — drop it into `~/.claude/skills/<skill-name>/` and Claude Code's skill loader will pick it up.

## Available skills

| Skill | What it does |
|---|---|
| [`init-native-memory`](skills/init-native-memory/SKILL.md) | Scaffold a tool-agnostic, markdown-only persistent agent memory layer (`.memory/` committed + `.scratch/` gitignored) into any git repo so AI agents stop losing context across sessions. |

## Install

Clone once, then symlink the skills you want into your global skill directory. Symlinks (rather than copies) mean a `git pull` here updates every installed skill in place.

```bash
git clone https://github.com/nydrx/agent-memory-skills.git ~/src/agent-memory-skills

# install a single skill
ln -s ~/src/agent-memory-skills/skills/init-native-memory ~/.claude/skills/init-native-memory

# update later
git -C ~/src/agent-memory-skills pull
```

If you'd rather not symlink (e.g. you want to edit the skill locally without affecting the repo), copy instead:

```bash
cp -R ~/src/agent-memory-skills/skills/init-native-memory ~/.claude/skills/init-native-memory
```

After install, restart Claude Code (or open a new session) so the skill loader picks up the new `SKILL.md`.

## Adding a new skill

1. Create `skills/<your-skill-name>/SKILL.md` with frontmatter:
   ```markdown
   ---
   name: <your-skill-name>
   description: <one-paragraph description with concrete trigger phrases — this is what Claude Code uses to decide when to invoke the skill>
   ---

   # <Skill title>
   <skill body>
   ```
2. Co-locate any assets (`assets/`) and supporting references (`references/`) in the same directory.
3. Add a row to the **Available skills** table above.
4. Open a PR.

Keep skills self-contained — no cross-skill imports, no paths outside `skills/<name>/`. That's what makes a single `ln -s` enough to install.

## License

Add a LICENSE file before publishing publicly. (Not yet decided.)

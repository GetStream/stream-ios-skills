# SDK Release Skill

Install this skill in Codex, Claude, or Cursor by placing this `sdk-release/` directory in the skill location each tool watches.

`SKILL.md` is the shared cross-agent contract.
`agents/openai.yaml` is optional Codex metadata and can be ignored by Claude and Cursor.

## Skill Folder

This folder must stay intact:

```text
sdk-release/
├── SKILL.md
├── README.md
├── agents/
│   └── openai.yaml
└── references/
    └── release-review-checklist.md
```

Use a symlink if you want one canonical copy of the skill.

## Codex

Current Codex docs say Codex loads skills from repository-level `.agents/skills/` folders and user-level `~/.agents/skills/`, and it follows symlinks.

Global install:

```bash
SKILL_SRC="/absolute/path/to/sdk-release"
mkdir -p ~/.agents/skills
ln -sfn "$SKILL_SRC" ~/.agents/skills/sdk-release
```

Project install:

```bash
SKILL_SRC="/absolute/path/to/sdk-release"
mkdir -p .agents/skills
ln -sfn "$SKILL_SRC" .agents/skills/sdk-release
```

Use it:
- ask Codex naturally for a release review and let it auto-invoke the skill
- or mention the skill explicitly with `$sdk-release`

Notes:
- Codex scans `.agents/skills` from the current working directory up to the repo root.
- If the skill does not appear after installation, restart Codex.

## Claude Code

Current Claude Code docs say personal skills live in `~/.claude/skills/<skill-name>/` and project skills live in `.claude/skills/<skill-name>/`.

Global install:

```bash
SKILL_SRC="/absolute/path/to/sdk-release"
mkdir -p ~/.claude/skills
ln -sfn "$SKILL_SRC" ~/.claude/skills/sdk-release
```

Project install:

```bash
SKILL_SRC="/absolute/path/to/sdk-release"
mkdir -p .claude/skills
ln -sfn "$SKILL_SRC" .claude/skills/sdk-release
```

Use it:
- type `/sdk-release`
- or ask Claude Code naturally for a source-vs-destination release review and let it auto-invoke

Notes:
- Claude Code usually hot-reloads skill edits.
- If you create the top-level skills directory for the first time during an active session, restart Claude Code so it starts watching that directory.

## Claude.ai

If by "Claude" you mean Claude.ai instead of Claude Code, upload the skill as a zip.

Create the zip from the parent directory so the archive contains the `sdk-release/` folder itself:

```bash
cd /absolute/path/to/parent-of-sdk-release
zip -r sdk-release.zip sdk-release
```

Install it:
1. Open Claude.
2. Go to `Customize > Skills`.
3. Click `+` and choose `Upload a skill`.
4. Upload `sdk-release.zip`.
5. Enable the skill.

Notes:
- Code execution must be enabled for Skills to work in Claude.
- Claude.ai expects the zip root to contain the `sdk-release/` directory, not loose files.
- Claude.ai has a stricter description-length limit than some other tools, so keep the `SKILL.md` frontmatter concise if you edit it later.

## Cursor

Cursor supports Agent Skills in the editor and CLI. The current public docs and support guidance point to these locations:
- project: `.cursor/skills/` or `.agents/skills/`
- user: `~/.cursor/skills/` or `~/.agents/skills/`

Recommended global install for the best slash-command visibility:

```bash
SKILL_SRC="/absolute/path/to/sdk-release"
mkdir -p ~/.cursor/skills
ln -sfn "$SKILL_SRC" ~/.cursor/skills/sdk-release
```

Recommended project install if you want one shared layout across Codex and Cursor:

```bash
SKILL_SRC="/absolute/path/to/sdk-release"
mkdir -p .agents/skills .cursor/skills
ln -sfn "$SKILL_SRC" .agents/skills/sdk-release
ln -sfn "$SKILL_SRC" .cursor/skills/sdk-release
```

Use it:
- type `/sdk-release`
- or ask Cursor naturally for a release review and let it auto-invoke

Notes:
- `.agents/skills/` is the most portable project-level location because Codex also reads it.
- `.cursor/skills/` is the safer location if you want the skill to appear reliably in Cursor slash-command menus.
- If the skill does not show up, update Cursor and restart it. Some recent Cursor builds had inconsistent CLI behavior around `~/.agents/skills/`.

## One Shared Setup For All Three

If you want one working copy and no duplication, keep this repo folder as the source and link it into each tool:

```bash
SKILL_SRC="/absolute/path/to/sdk-release"

mkdir -p ~/.agents/skills ~/.claude/skills ~/.cursor/skills

ln -sfn "$SKILL_SRC" ~/.agents/skills/sdk-release
ln -sfn "$SKILL_SRC" ~/.claude/skills/sdk-release
ln -sfn "$SKILL_SRC" ~/.cursor/skills/sdk-release
```

That gives:
- Codex: global skill through `~/.agents/skills`
- Claude Code: global skill through `~/.claude/skills`
- Cursor: global skill through `~/.cursor/skills`

## Quick Verification

After installation, verify the skill is visible:
- Codex: start a session and mention `$sdk-release`
- Claude Code: type `/sdk-release`
- Claude.ai: confirm it appears in `Customize > Skills`
- Cursor: type `/sdk-release` in agent chat

If explicit invocation works, auto-invocation usually works too once the tool has indexed the skill metadata.

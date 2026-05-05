# Agent Instructions

This repository stores Codex skills. The user usually arrives with a mostly written
`SKILL.md` draft and wants the agent to package it, lightly optimize it, update the
repo index, and push it.

## Default Workflow

1. Inspect the repo state with `git status --short --branch`.
2. Read the submitted skill draft and identify the skill name from YAML frontmatter.
3. Create or update `<skill-name>/SKILL.md`.
4. Split long supporting material into `references/REFERENCE.md` only when it improves
   context hygiene or the user agrees with that structure.
5. Create or update `agents/openai.yaml` when the skill should appear cleanly in the UI.
6. Update `README.md` with the skill description and install command.
7. Stage only the files that belong to the skill change.
8. Commit and push to `main` unless the user explicitly asks for a branch or PR.

Use this install command format in `README.md`:

```bash
npx skills@latest add Smallfu666/skills/<skill-name>
```

## Skill Packaging Rules

- Preserve the user's intended behavior, tone, and domain-specific constraints.
- Do not rewrite a working skill into a generic template.
- Keep YAML frontmatter limited to `name` and `description`.
- Put trigger terms in `description`; the body loads only after the skill triggers.
- Prefer concise `SKILL.md` content with clear workflows and hard rules.
- Move large templates, examples, command references, or policy tables to
  `references/REFERENCE.md` when the skill would otherwise become bulky.
- If the user wants a single-file skill, keep everything in `SKILL.md`.
- Do not add extra docs such as per-skill `README.md`, changelogs, or installation
  guides unless the user explicitly asks.
- Do not create `.zip` artifacts for skills unless the user explicitly asks.

## Optimization Rules

- Optimize for future agent behavior, not human marketing copy.
- Keep concrete commands and exact file paths when they affect correctness.
- Remove stale or contradictory workflow instructions.
- Prefer opt-in language for actions that cost money, consume cluster allocation,
  mutate remote state, or run external services.
- Do not hide risky automation behind defaults.
- Do not invent cluster partitions, accounts, prices, GPU types, or queue limits.
  Use the relevant cluster skill or live cluster commands when current facts matter.

## `agents/openai.yaml`

When creating UI metadata, include:

```yaml
interface:
  display_name: "<Human Name>"
  short_description: "<25-64 char summary>"
  default_prompt: "Use $<skill-name> to ..."
```

The `default_prompt` must mention the skill as `$<skill-name>`.

## Git Rules

- Work directly on `main` by default for this repo.
- Check for unrelated dirty files before staging.
- Stage explicit paths; avoid broad `git add -A` unless the full worktree is in scope.
- Use concise commit messages, for example:
  `docs(skill): add <skill-name>`
  `docs(skill): update <skill-name>`
- Push to `origin main` after committing when the user asks to upload or publish.

## Validation

- If the skill-creator validator is available, run it before committing.
- If validation is blocked by missing local dependencies or restricted network access,
  report that clearly and continue with manual checks.
- Manual checks should verify frontmatter, folder name, README install command, and
  absence of accidental artifacts.

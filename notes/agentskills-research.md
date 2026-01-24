# Agent Skills research notes

## Summary
- Treat a skill as a directory containing a required `SKILL.md` file with YAML frontmatter and Markdown instructions. The frontmatter must include `name` and `description`, and the skill directory must match the `name`.
- Follow the naming constraints: 1–64 characters, lowercase letters/numbers/hyphens, no leading/trailing hyphen, and no consecutive hyphens. Keep descriptions 1–1024 characters and include what the skill does plus when to use it.
- Use optional `scripts/`, `references/`, and `assets/` directories to keep SKILL.md concise, and apply progressive disclosure so detailed references are loaded only when needed.
- Validate skills with the `skills-ref` reference library when packaging or distributing skills.

## Sources
- https://agentskills.io/llms.txt
- https://agentskills.io/specification.md

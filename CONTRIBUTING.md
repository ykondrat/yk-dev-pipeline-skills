# Contributing to Dev Pipeline Skills

Thanks for your interest in contributing! This project is a collection of Claude AI skills
for software development, and we welcome contributions from the community.

## How to Contribute

### Improving Existing Skills

1. Fork the repository
2. Edit the relevant SKILL.md or reference file
3. Test your changes by uploading to a Claude conversation and running through the workflow
4. Submit a PR with a clear description of what changed and why

### Adding New Reference Materials

We'd love to expand coverage to more frameworks and languages:

- **Frontend frameworks**: Angular, Svelte, Nuxt (React, Vue, Next.js already covered)
- **Backend frameworks**: NestJS, tRPC (Express, Fastify, Hono already covered)
- **Languages**: Python, Go, Rust, Java adaptations
- **Infrastructure**: Docker, Kubernetes, Terraform, CI/CD patterns
- **Specialized areas**: GraphQL, WebSockets, microservices, event sourcing

To add a reference:
1. Create a new `.md` file in the appropriate `references/` directory
2. Follow the same structure as existing references (rationale + examples + severity tables)
3. Update the parent SKILL.md to reference the new file

### Adding New Review Checklist Areas

1. Add the checklist to `plugins/yk-dev-pipeline/skills/code-review/references/review-checklists.md`
2. Add a summary entry in `plugins/yk-dev-pipeline/skills/code-review/SKILL.md` under Review Areas
3. Include severity guidance with the standard emoji levels (🔴🟡🔵⚪)

### Reporting Issues

- **Inaccurate advice**: If a best practice is wrong or outdated, open an issue
- **Missing coverage**: If an important area isn't covered, suggest it
- **Unclear instructions**: If a skill's process is confusing, let us know

## Guidelines

### Skill Writing Style

- **Be specific** — "Check for X" is better than "Review for quality"
- **Show examples** — code snippets for patterns to flag and correct alternatives
- **Explain rationale** — "Why" matters more than "What"
- **Include severity guidance** — help Claude categorize findings correctly
- **Test with Claude** — verify your skill works by actually using it in a conversation

### File Format

- All skills use YAML frontmatter (`name`, `description`, `metadata`)
- Skill names use the `yk-` prefix (e.g., `yk-brainstorm`, `yk-implementation`)
- `metadata.recommended_model` specifies the optimal model (opus, sonnet, or haiku)
- Markdown with clear section headers
- Code examples in fenced blocks with language tags
- Tables for severity guidance and common patterns
- Reference file paths use skill-root-relative paths (e.g., `implementation/references/js-ts-best-practices.md`)

### Keep It Lean

- SKILL.md files should stay under ~500 lines (use reference files for depth)
- Reference files can be longer but should be well-organized with a table of contents
- Don't duplicate content between skills — link to references instead

## Testing Your Changes

Since these are instruction files, not code, testing means:

1. Upload the modified skill to a Claude conversation
2. Give Claude a realistic task that exercises the skill
3. Verify Claude follows the instructions correctly
4. Check that the output matches what the skill describes

## Versioning

This project follows [Semantic Versioning](https://semver.org/):

- **Patch** (1.0.x): Typo fixes, minor wording improvements, added examples
- **Minor** (1.x.0): New reference files, new review checklist areas, new features in skills
- **Major** (x.0.0): Breaking changes to skill names, artifact formats, or pipeline flow

### How to Bump Versions

When making changes, update the version in **all three locations**:

1. `.claude-plugin/marketplace.json` → `metadata.version` and `plugins[0].version`
2. `plugins/yk-dev-pipeline/.claude-plugin/plugin.json` → `version`
3. `.skillrc.json` → `version`

### For Users: Updating Installed Plugins

Users who installed via the Claude Code marketplace can update by re-running:

```bash
claude /plugin marketplace add yk-dev-pipeline
```

This pulls the latest version from the repository.

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

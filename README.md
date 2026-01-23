# Agent Skills

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

A curated collection of AI agent skills that enhance coding assistants with specialized domain knowledge.

## What Are Agent Skills?

Agent skills are structured knowledge bases that help AI coding assistants (like Claude, GPT, Cursor, etc.) understand framework-specific patterns, best practices, and common pitfalls. They bridge the gap between an AI's general knowledge and the specific, up-to-date expertise needed for modern development.

**Why agent skills matter:**

- **Stay current** - Frameworks evolve faster than AI training data. Skills provide the latest patterns and syntax.
- **Reduce errors** - Skills teach AI about breaking changes, deprecations, and migration paths.
- **Best practices** - Encode community-vetted solutions to common problems.
- **Type safety** - Include TypeScript patterns and proper typing for modern codebases.

## Available Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [svelte](./svelte) | Svelte 5 runes, snippets, and SvelteKit patterns | `npx add-skill ejirocodes/agent-skills/svelte` |
| [vue](./vue) | Vue 3.5+ TypeScript, Volar, Pinia testing, and component patterns | `npx add-skill ejirocodes/agent-skills/vue` |

## Installation

Install a skill using the `add-skill` CLI:

```bash
npx add-skill ejirocodes/agent-skills
```

For example, to install Svelte skills:

```bash
npx add-skill ejirocodes/agent-skills/svelte
```

## Usage

For best results, prefix your prompt with `use <skill> skill`:

```
Use svelte skill, help me create a form component with validation
```

This explicitly triggers the skill and ensures the AI follows the documented patterns. Without the prefix, skill triggering depends on how closely your prompt matches the skill's keywords.

## Skill Structure

Each skill follows this structure:

```
<skill-name>/
├── LICENSE                # MIT License
└── skills/
    └── <skill-set>/
        ├── SKILL.md       # Skill metadata and quick reference
        └── references/
            └── *.md       # Domain-specific reference files
```

### Reference Organization

References are organized by domain for progressive disclosure:

- **SKILL.md** - Lean entry point with essential patterns and navigation
- **references/** - Detailed documentation loaded only when needed

This structure keeps the main skill file small while providing comprehensive coverage through domain-specific reference files (e.g., `runes.md`, `sveltekit.md`, `migration.md`).

## Contributing

Contributions are welcome! To add a new skill:

1. Create a folder for your skill (e.g., `react/`)
2. Follow the structure above
3. Include comprehensive rules with:
   - Clear problem descriptions
   - Code examples (incorrect vs correct)
   - Links to official documentation
4. Submit a pull request

### Guidelines

- Focus on problems AI assistants commonly get wrong
- Prioritize recent framework changes and breaking updates
- Include TypeScript patterns where applicable
- Test rules with multiple AI models (Claude, GPT, etc.)
- Delete rules if AI consistently solves them without help

## Roadmap

Future skills planned:

- **Frontend** - React 19, Angular, Solid
- **Backend** - Node.js, Deno, Bun, Go, Rust
- **Databases** - PostgreSQL, MongoDB, Redis, Prisma
- **DevOps** - Docker, Kubernetes, CI/CD
- **Testing** - Vitest, Playwright, Jest
- **AI/ML** - LangChain, Vector DBs, RAG patterns

## License

MIT - see [LICENSE](./LICENSE)

---

Created by [Ejiro Asiuwhu](https://github.com/ejirocodes)

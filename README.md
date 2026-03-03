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

### Framework Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [nestjs](./nestjs) | NestJS 11+ DI, validation, auth, TypeORM/Prisma/Drizzle, and testing patterns | `npx skills add ejirocodes/agent-skills/nestjs` |
| [svelte](./svelte) | Svelte 5 runes, snippets, and SvelteKit patterns | `npx skills add ejirocodes/agent-skills/svelte` |
| [vue](./vue) | Vue 3+ TypeScript, Volar, Pinia testing, and component patterns | `npx skills add ejirocodes/agent-skills/vue` |

### Exa.ai Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [exa-search](./exa/skills/exa-search) | Core Exa search API integration (neural/keyword/auto modes, filters, content retrieval) | `npx skills add ejirocodes/agent-skills/exa-search` |
| [exa-rag](./exa/skills/exa-rag) | RAG pipelines with Exa (LangChain, LlamaIndex, Vercel AI SDK, MCP tools) | `npx skills add ejirocodes/agent-skills/exa-rag` |
| [exa-research](./exa/skills/exa-research) | Deep research and Answer API with citations and streaming | `npx skills add ejirocodes/agent-skills/exa-research` |
| [exa-entities](./exa/skills/exa-entities) | Company and people search for lead gen, recruiting, and competitive intelligence | `npx skills add ejirocodes/agent-skills/exa-entities` |

## Installation

### Install via skills.sh

```bash
npx skills add ejirocodes/agent-skills/svelte
npx skills add ejirocodes/agent-skills/vue
npx skills add ejirocodes/agent-skills/nestjs
npx skills add ejirocodes/agent-skills/exa-search
npx skills add ejirocodes/agent-skills/exa-rag
npx skills add ejirocodes/agent-skills/exa-research
npx skills add ejirocodes/agent-skills/exa-entities
```

### Claude Code (Plugin Marketplace)

Add the marketplace once, then install individual skills:

```shell
/plugin marketplace add ejirocodes/agent-skills
```

```shell
/plugin install svelte5-best-practices@ejirocodes-skills
/plugin install vue-best-practices@ejirocodes-skills
/plugin install nestjs-best-practices@ejirocodes-skills
/plugin install exa@ejirocodes-skills
```

## Usage

### skills.sh

For best results, prefix your prompt with `use <skill> skill`:

```
Use svelte skill, help me create a form component with validation
```

This explicitly triggers the skill and ensures the AI follows the documented patterns. Without the prefix, skill triggering depends on how closely your prompt matches the skill's keywords.

### Claude Code

Skills are available as slash commands after installation:

```shell
/svelte5-best-practices
/vue-best-practices
/nestjs-best-practices
/exa-search
/exa-rag
/exa-research
/exa-entities
```

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
- **AI/ML** - LangChain, Vector DBs, more RAG patterns

## License

MIT - see [LICENSE](./LICENSE)

---

Created by [Ejiro Asiuwhu](https://github.com/ejirocodes)

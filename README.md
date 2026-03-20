
# ILP — Instruction Layer Protocol
[![SSRN](https://img.shields.io/badge/SSRN-6440461-blue.svg)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=6440461)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.19101583.svg)](https://doi.org/10.5281/zenodo.19101583)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![Version](https://img.shields.io/badge/version-1.1.0-blue.svg)


> A universal standard for how AI agents receive, interpret, and share behavioral instructions across tools, models, and environments.

---

## Why ILP?

Every major AI coding tool has invented its own context file:

| Tool | File |
|---|---|
| Claude (custom envs) | `SKILL.md` |
| Cline | `.clinerules` |
| Aider | `CONVENTIONS.md` |
| GitHub Copilot | `copilot-instructions.md` |
| Cursor | `.cursorrules` |

These are all solving the **same problem** — giving agents persistent, structured behavioral context — but in incompatible, siloed ways.

**ILP proposes a single, open standard** that any agent, model, or tool can implement.

---

## Core Concept

ILP sits between the user/codebase and the model. It answers the question:

```
"Before you do anything — here's who you are, what you know, and how you should behave."
```

```
User / Codebase
      ↓
  [ ILP Layer ]  ←  structured behavioral instructions
      ↓
    Agent
      ↓
  [ MCP Layer ]  ←  tool calls, services, external APIs
      ↓
 Tools / Services
```

MCP handles **what agents can do**. ILP handles **how agents should think and behave**. Together they form a complete agentic architecture.

---

## Specification

### File Format

ILP instructions live in `.ilp.md` files — standard Markdown with a YAML frontmatter header.

```yaml
---
ilp_version: "1.0"
name: "my-skill"
scope: "project" # project | user | global
priority: 100    # higher = evaluated first
tags: ["coding", "python", "testing"]
---

# Instruction content in plain Markdown below
```

### Scope Hierarchy

ILP resolves instructions in order, merging from broadest to most specific:

```
global (~/.ilp/global.ilp.md)
  └── user (~/.ilp/user.ilp.md)
        └── project (./.ilp.md)
              └── task (inline, passed at runtime)
```

More specific scopes override broader ones for conflicting keys.

### Skill Registry

Skills are named, reusable ILP documents that can be referenced by name:

```yaml
# .ilp.md
---
ilp_version: "1.0"
scope: "project"
uses:
  - skill: "python-best-practices"
    version: "^1.2"
  - skill: "docstring-style/google"
---

# Project-specific overrides below
Always use type hints. Prefer dataclasses over dicts for structured data.
```

---

## Code Snippet — ILP Parser (TypeScript)

```typescript
import * as fs from "fs";
import * as path from "path";
import * as yaml from "js-yaml";
import { marked } from "marked";

export interface ILPDocument {
  version: string;
  name?: string;
  scope: "global" | "user" | "project" | "task";
  priority: number;
  tags?: string[];
  uses?: { skill: string; version?: string }[];
  content: string; // raw markdown instructions
}

export interface ILPContext {
  instructions: string;  // merged, resolved instruction string
  metadata: ILPDocument[];
}

/**
 * Parse a single .ilp.md file
 */
export function parseILP(filePath: string): ILPDocument {
  const raw = fs.readFileSync(filePath, "utf-8");

  // Split frontmatter and content
  const fmMatch = raw.match(/^---\n([\s\S]*?)\n---\n([\s\S]*)$/);
  if (!fmMatch) throw new Error(`Invalid ILP file: ${filePath}`);

  const frontmatter = yaml.load(fmMatch[1]) as Record<string, unknown>;
  const content = fmMatch[2].trim();

  return {
    version: (frontmatter.ilp_version as string) ?? "1.0",
    name: frontmatter.name as string | undefined,
    scope: (frontmatter.scope as ILPDocument["scope"]) ?? "project",
    priority: (frontmatter.priority as number) ?? 100,
    tags: frontmatter.tags as string[] | undefined,
    uses: frontmatter.uses as ILPDocument["uses"],
    content,
  };
}

/**
 * Resolve and merge all ILP files in scope order
 */
export function resolveILPContext(projectRoot: string): ILPContext {
  const locations = [
    path.join(process.env.HOME ?? "~", ".ilp", "global.ilp.md"),
    path.join(process.env.HOME ?? "~", ".ilp", "user.ilp.md"),
    path.join(projectRoot, ".ilp.md"),
  ];

  const docs: ILPDocument[] = locations
    .filter(fs.existsSync)
    .map(parseILP)
    .sort((a, b) => a.priority - b.priority); // lower priority = evaluated first

  // Merge instructions in order (specific scopes win on conflict)
  const instructions = docs.map((d) => `## ${d.name ?? d.scope}\n\n${d.content}`).join("\n\n---\n\n");

  return { instructions, metadata: docs };
}
```

---

## Code Snippet — MCP + ILP Integration

ILP pairs naturally with MCP. Here's how an agent would bootstrap with both:

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { resolveILPContext } from "./ilp-parser";

const client = new Anthropic();

async function runAgentWithILP(userMessage: string) {
  // 1. Resolve ILP context for this project
  const ilpContext = resolveILPContext(process.cwd());

  // 2. Build system prompt from ILP instructions
  const systemPrompt = `
You are an AI coding agent. Follow these instructions precisely.

${ilpContext.instructions}
  `.trim();

  // 3. Call the model — MCP servers injected here for tool access
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 8096,
    system: systemPrompt,
    messages: [{ role: "user", content: userMessage }],
    // MCP servers registered here give the agent its capabilities
    // @ts-ignore — mcp_servers is an emerging param
    mcp_servers: [
      { type: "url", url: "https://mcp.filesystem.local/sse", name: "filesystem" },
      { type: "url", url: "https://mcp.github.com/sse",       name: "github" },
    ],
  });

  return response.content.map((b) => (b.type === "text" ? b.text : "")).join("");
}

// Example usage
runAgentWithILP("Refactor the authentication module to use JWT.").then(console.log);
```

---

## Example `.ilp.md` File

```markdown
---
ilp_version: "1.0"
name: "backend-api-project"
scope: "project"
priority: 100
tags: ["typescript", "node", "api"]
---

# Project Instructions

## Code Style
- Use TypeScript strict mode at all times
- Prefer `async/await` over raw Promises
- All functions must have explicit return types
- Use Zod for runtime validation at API boundaries

## Architecture
- Follow hexagonal architecture: domain logic never imports from infra layer
- All side effects (DB, HTTP, FS) live in `/src/adapters`
- Use dependency injection via constructor params — no service locators

## Testing
- Write unit tests for all domain logic
- Integration tests required for all API routes
- Test files live next to source files as `*.test.ts`

## Git
- Conventional commits: feat/fix/chore/docs/refactor
- Never commit directly to `main`
- PR descriptions must include "Why" not just "What"
```
## Design Principles

- **Deterministic Resolution** — same inputs always produce same instruction set  
- **Composable** — instructions can be layered and reused  
- **Model-Agnostic** — works across all LLM providers  
- **Tool-Independent** — decoupled from execution layer (MCP)  
- **Human-Readable First** — Markdown-native, no custom DSL required

  
  ## Execution Model

1. Collect ILP documents from all scopes
2. Sort by:
   - scope (global → task)
   - priority (ascending)
3. Resolve `uses:` dependencies recursively
4. Merge instruction content:
   - later scopes override earlier ones
   - conflicts resolved by last-write-wins
5. Produce final instruction stream
---
## Comparison

| Feature | ILP | Cursor Rules | Copilot Instructions | MCP |
|--------|-----|-------------|----------------------|-----|
| Cross-tool standard | ✅ | ❌ | ❌ | ✅ |
| Behavioral control | ✅ | ✅ | ✅ | ❌ |
| Tool integration | ❌ | ❌ | ❌ | ✅ |
| Composability | ✅ | ⚠️ | ⚠️ | ❌ |
## Roadmap

- [ ] `v1.0` — Core spec (frontmatter + markdown content)
- [ ] `v1.1` — Skill registry & `uses:` resolution
- [ ] `v1.2` — MCP bridge spec (ILP skills exposable as MCP tools)
- [ ] `v2.0` — Multi-agent ILP sharing & delegation

---

## Contributing

ILP is an open proposal. Open an issue, fork the spec, submit a PR.

The goal is **consensus**, not ownership. If it helps the ecosystem, it wins.

---
## Citation

ILP Spec:
Gunjan Sarkar. "ILP — Instruction Layer Protocol." [zenodo](https://doi.org/10.5281/zenodo.19101583), 2026.

## License

MIT — free to implement, fork, and extend.

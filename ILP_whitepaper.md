---
title: "ILP: Instruction Layer Protocol — A Universal Standard for Behavioral Context in AI Agent Systems"
author: "Gunjan Grunge"
date: "March 19, 2026"
abstract: |
  The rapid proliferation of AI coding agents and autonomous systems has produced a fragmented ecosystem of incompatible, tool-specific instruction formats. Each major platform — including GitHub Copilot, Cursor, Cline, and Aider — has independently invented its own mechanism for delivering persistent behavioral context to language models, resulting in siloed, non-interoperable solutions. This paper proposes the Instruction Layer Protocol (ILP), a lightweight, open standard for how AI agents receive, interpret, and share behavioral instructions across tools, models, and environments. ILP defines a structured Markdown format with YAML frontmatter, a hierarchical scope resolution mechanism, a reusable skill registry, and a formal bridge interface to the Model Context Protocol (MCP). Together, ILP and MCP form a complete two-layer architecture for agentic AI systems: ILP governs behavioral identity and instruction, while MCP governs capability and tool access. We present the protocol specification, a reference TypeScript implementation, integration patterns, and a roadmap toward ecosystem-wide adoption.
---

# 1. Introduction

The emergence of large language model (LLM)-powered coding agents has fundamentally changed how software is written. Tools such as GitHub Copilot [CITE:copilot], Cursor [CITE:cursor], Cline [CITE:cline], and Aider [CITE:aider] now assist developers across millions of projects, autonomously reading codebases, writing implementations, and executing multi-step tasks.

A critical requirement for these agents is **persistent behavioral context**: a mechanism by which a developer can communicate project-specific conventions, architectural preferences, coding standards, and domain knowledge to the agent — once — so that every subsequent interaction respects these constraints without re-specification.

Every major platform has solved this independently:

| Platform        | File                        |
|-----------------|----------------------------|
| GitHub Copilot  | `copilot-instructions.md`  |
| Cursor          | `.cursorrules`             |
| Cline           | `.clinerules`              |
| Aider           | `CONVENTIONS.md`           |
| Claude (custom) | `SKILL.md`                 |

These solutions are semantically equivalent — all are Markdown files that inject project context into the agent's system prompt — yet they are syntactically incompatible. A developer cannot write a single instruction file that works across platforms. Skills, rules, and conventions cannot be shared, versioned, or composed across tool boundaries.

This fragmentation imposes real costs: duplicated effort, knowledge silos, and an inability to build a shared ecosystem of reusable agent instructions.

We propose the **Instruction Layer Protocol (ILP)**: a minimal, open standard that unifies these approaches into a single interoperable format.

## 1.1 Contributions

This paper makes the following contributions:

1. A formal specification of the ILP file format and frontmatter schema
2. A hierarchical scope resolution algorithm (global → user → project → task)
3. A reusable skill registry and dependency resolution mechanism
4. A bridge specification connecting ILP to the Model Context Protocol (MCP)
5. A reference TypeScript implementation
6. A roadmap toward ecosystem-wide adoption

# 2. Background and Related Work

## 2.1 Model Context Protocol (MCP)

The Model Context Protocol [CITE:mcp], introduced by Anthropic in late 2024, defines a standard interface by which AI agents discover and invoke external tools, resources, and services. MCP addresses the **capability layer** of agentic systems: what an agent can *do*.

ILP is designed as a complementary standard addressing the **behavioral layer**: how an agent should *think and behave*. Together they form a complete agentic architecture.

## 2.2 System Prompts and Behavioral Injection

Prior work on prompt engineering [CITE:prompteng] establishes that system prompts are the primary mechanism for shaping LLM behavior. Persistent behavioral context — instructions that remain stable across interactions — is typically delivered via system prompt injection at agent initialization.

The key insight behind ILP is that this injection mechanism is universal across all LLM-based agents, and therefore the format and resolution of behavioral instructions can be standardized independently of the underlying model or tool.

## 2.3 Configuration as Code

ILP draws inspiration from the "configuration as code" movement in DevOps, where operational intent is expressed in version-controlled, human-readable files. ILP treats agent behavioral configuration as a first-class artifact of a software project — committed to version control, reviewed alongside code, and composable via dependency resolution.

# 3. Protocol Specification

## 3.1 File Format

ILP instruction files use the `.ilp.md` extension. They consist of a YAML frontmatter block followed by Markdown content:

```
---
ilp_version: "1.0"
name: "backend-api"
scope: "project"
priority: 100
tags: ["typescript", "node", "api"]
uses:
  - skill: "python-best-practices"
    version: "^1.2"
---

# Instructions

## Code Style
- Use TypeScript strict mode
- Prefer async/await over raw Promises
```

### 3.1.1 Frontmatter Fields

| Field           | Type     | Required | Description                                      |
|-----------------|----------|----------|--------------------------------------------------|
| `ilp_version`   | string   | Yes      | Protocol version (currently `"1.0"`)            |
| `name`          | string   | No       | Human-readable identifier                        |
| `scope`         | enum     | Yes      | One of: `global`, `user`, `project`, `task`     |
| `priority`      | integer  | No       | Evaluation order (default: 100)                  |
| `tags`          | string[] | No       | Categorization tags for skill registry queries   |
| `uses`          | object[] | No       | Referenced skills with optional version ranges   |

## 3.2 Scope Hierarchy

ILP defines four scopes, resolved in order from broadest to most specific:

```
global  (~/.ilp/global.ilp.md)
  └── user  (~/.ilp/user.ilp.md)
        └── project  (./.ilp.md)
              └── task  (inline, runtime-injected)
```

When multiple ILP documents are in scope, their contents are merged. More specific scopes take precedence over broader scopes for conflicting directives. This allows global conventions (e.g., "always write tests") to be overridden by project-specific exceptions.

### 3.2.1 Merge Algorithm

Given a set of resolved ILP documents D = {d₁, d₂, ..., dₙ} sorted by scope specificity (ascending) and priority:

1. Initialize an empty instruction buffer B
2. For each document dᵢ in order:
   - Append a section header: `## {dᵢ.name ?? dᵢ.scope}`
   - Append dᵢ.content to B
   - Append a section separator
3. Return B as the resolved instruction string

The resulting string is injected as the agent's system prompt prior to any user interaction.

## 3.3 Skill Registry

ILP supports a registry of named, versioned, reusable instruction sets ("skills"). Skills are referenced in the `uses` field of an ILP document using semantic versioning ranges:

```yaml
uses:
  - skill: "python-best-practices"
    version: "^1.2"
  - skill: "docstring-style/google"
```

Skill resolution follows npm-style semantic versioning. A registry implementation may resolve skills from:

- A local directory (`~/.ilp/skills/`)
- A package registry (e.g., an npm-compatible registry for `.ilp.md` packages)
- A URL (direct HTTP fetch of a `.ilp.md` file)

## 3.4 ILP–MCP Bridge

ILP and MCP are designed to complement each other. The bridge specification defines two integration patterns:

### 3.4.1 ILP-as-Context (Primary Pattern)

At agent initialization, the ILP runtime resolves all in-scope documents and injects the merged instruction string as the system prompt. MCP servers are then registered for tool access. This is the recommended pattern for all agents.

```
┌─────────────────────────────────────────┐
│              Agent Runtime              │
│                                         │
│  1. Resolve ILP → system prompt         │
│  2. Register MCP servers → tools        │
│  3. Accept user message                 │
│  4. LLM call (system + tools + message) │
└─────────────────────────────────────────┘
```

### 3.4.2 ILP-as-MCP-Tool (Advanced Pattern)

In multi-agent orchestration scenarios, an ILP runtime may expose skills as MCP tools, allowing sub-agents to dynamically query behavioral instructions:

```
Orchestrator Agent
       │
       ├── MCP: filesystem, github, ...
       └── MCP: ilp-server
             ├── tool: get_skill("python-best-practices")
             ├── tool: list_skills(tags=["testing"])
             └── tool: resolve_context(project_path)
```

This enables dynamic behavioral adaptation in complex multi-agent pipelines.

# 4. Reference Implementation

## 4.1 ILP Parser (TypeScript)

```typescript
import * as fs from "fs";
import * as yaml from "js-yaml";

export interface ILPDocument {
  version: string;
  name?: string;
  scope: "global" | "user" | "project" | "task";
  priority: number;
  tags?: string[];
  uses?: { skill: string; version?: string }[];
  content: string;
}

export function parseILP(filePath: string): ILPDocument {
  const raw = fs.readFileSync(filePath, "utf-8");
  const match = raw.match(/^---\n([\s\S]*?)\n---\n([\s\S]*)$/);
  if (!match) throw new Error(`Invalid ILP file: ${filePath}`);

  const fm = yaml.load(match[1]) as Record<string, unknown>;
  return {
    version: (fm.ilp_version as string) ?? "1.0",
    name: fm.name as string | undefined,
    scope: (fm.scope as ILPDocument["scope"]) ?? "project",
    priority: (fm.priority as number) ?? 100,
    tags: fm.tags as string[] | undefined,
    uses: fm.uses as ILPDocument["uses"],
    content: match[2].trim(),
  };
}

export function resolveILPContext(projectRoot: string) {
  const home = process.env.HOME ?? "~";
  const locations = [
    `${home}/.ilp/global.ilp.md`,
    `${home}/.ilp/user.ilp.md`,
    `${projectRoot}/.ilp.md`,
  ].filter(fs.existsSync).map(parseILP)
   .sort((a, b) => a.priority - b.priority);

  const instructions = locations
    .map(d => `## ${d.name ?? d.scope}\n\n${d.content}`)
    .join("\n\n---\n\n");

  return { instructions, metadata: locations };
}
```

## 4.2 Agent Bootstrap with MCP Integration

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { resolveILPContext } from "./ilp-parser";

async function runAgent(userMessage: string) {
  const { instructions } = resolveILPContext(process.cwd());

  return new Anthropic().messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 8096,
    system: `You are an AI agent. Follow these instructions:\n\n${instructions}`,
    messages: [{ role: "user", content: userMessage }],
    mcp_servers: [
      { type: "url", url: "https://mcp.filesystem.local/sse", name: "fs" },
      { type: "url", url: "https://mcp.github.com/sse", name: "github" },
    ],
  });
}
```

# 5. Discussion

## 5.1 Design Principles

ILP is designed around three core principles:

**Minimalism.** The protocol imposes minimal structure. Any valid Markdown file with a compliant frontmatter block is a valid ILP document. Existing instruction files (`.cursorrules`, `copilot-instructions.md`, etc.) can be migrated by adding four lines of frontmatter.

**Composability.** The skill registry and `uses` field enable instruction reuse across projects and teams. Organizations can publish shared skill sets that encode institutional knowledge.

**Complementarity.** ILP does not compete with MCP; it extends it. The two protocols address different layers of the same problem and are designed to be deployed together.

## 5.2 Comparison with Existing Approaches

Unlike existing platform-specific instruction files, ILP provides:

- **Portability**: A single `.ilp.md` works across any ILP-compatible agent
- **Versioning**: Frontmatter enables schema evolution without breaking changes
- **Composition**: The `uses` field enables dependency resolution and skill reuse
- **Hierarchy**: Scope resolution allows global defaults with local overrides

## 5.3 Limitations and Future Work

The current specification does not address:

- **Conflict resolution** beyond simple precedence ordering
- **Instruction validation** — verifying that instructions are coherent and non-contradictory
- **Access control** — restricting which agents may access which skills
- **Internationalization** — instructions in non-English languages

These are deferred to future versions of the specification.

# 6. Roadmap

| Version | Features |
|---------|----------|
| v1.0 | Core spec: frontmatter + markdown content + scope hierarchy |
| v1.1 | Skill registry and `uses:` dependency resolution |
| v1.2 | MCP bridge: ILP skills as MCP tool endpoints |
| v2.0 | Multi-agent ILP delegation and orchestration |

# 7. Conclusion

The AI agent ecosystem is converging on a common pattern — Markdown files that deliver persistent behavioral context to language models — but has not yet converged on a common standard. ILP proposes that standard: a minimal, open, composable protocol that unifies existing approaches and enables an ecosystem of reusable agent instructions.

Combined with MCP, ILP completes the two-layer architecture that production agentic systems require: a behavioral layer (ILP) and a capability layer (MCP). We invite the community to adopt, extend, and improve the specification at:

**https://github.com/GunjanGrunge/ilp-spec**

DOI: 10.5281/zenodo.19101583

# References

[CITE:copilot] GitHub. *GitHub Copilot Documentation*. https://docs.github.com/copilot, 2024.

[CITE:cursor] Anysphere. *Cursor — The AI Code Editor*. https://cursor.com, 2024.

[CITE:cline] Cline Contributors. *Cline: Autonomous Coding Agent*. https://github.com/cline/cline, 2024.

[CITE:aider] Gauthier, P. *Aider: AI Pair Programming in Your Terminal*. https://aider.chat, 2023.

[CITE:mcp] Anthropic. *Model Context Protocol Specification*. https://modelcontextprotocol.io, 2024.

[CITE:prompteng] Wei, J. et al. *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*. NeurIPS, 2022.

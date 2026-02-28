# Phinx Skills — Claude Code Plugin

13 FAANG-level skills that auto-trigger based on context, turning Claude Code agents into senior engineers across clean code, frontend, backend, DevOps, system design, security, and more.

## Skills

| Skill | Domain | Key Topics |
|---|---|---|
| `clean-code-architecture` | **Foundational** | SOLID, design patterns (Factory, Builder, Adapter, Decorator, Strategy, Observer, Command, State), clean code (DRY, KISS, YAGNI, Law of Demeter), clean architecture, hexagonal/ports & adapters, refactoring patterns |
| `frontend-react` | Frontend | React, Next.js, TypeScript strict, component APIs, state management decision tree, TanStack Query, accessibility, performance, error boundaries |
| `backend-api` | Backend | REST design, response envelopes, JWT auth, RBAC, repository pattern, error handling, structured logging, rate limiting, caching, middleware stack |
| `system-design` | Architecture | Monolith vs microservices, sync vs async, CAP, CQRS, event sourcing, saga, scaling strategies, NFR checklist, ADRs |
| `devops-infra` | DevOps | Docker multi-stage, CI/CD pipelines, deployment strategies, Terraform/IaC, monitoring (4 golden signals), secrets management, SSL/TLS |
| `database-design` | Database | Schema design, indexing strategy, N+1 prevention, EXPLAIN ANALYZE, zero-downtime migrations, PostgreSQL, Redis patterns |
| `security-review` | Security | OWASP Top 10, input validation, threat modeling (STRIDE), secrets management, PR security checklist |
| `code-review` | Quality | Review priorities, naming, function design, code smells, feedback guidelines, PR quality checklist |
| `testing-strategy` | Testing | Testing trophy, unit/integration/E2E, mocking strategy, anti-patterns, CI integration |
| `product-thinking` | Product | PRDs, user stories, acceptance criteria, MVP scoping (MoSCoW), RICE prioritization |
| `git-workflow` | Git | Trunk-based development, conventional commits, PR best practices, SemVer, release management |
| `performance-optimization` | Performance | Profiling, caching strategies, Core Web Vitals, bundle optimization, load testing |
| `documentation` | Docs | README/ADR/runbook templates, API docs, C4 diagrams, writing principles |

## Installation

### Option 1: GitHub Marketplace (Recommended)

Inside Claude Code:

```
/plugin marketplace add yahyaalbashar/phinx-skills
/plugin install phinx-skills@phinx-marketplace
```

### Option 2: Local Plugin (Quick Test)

```bash
git clone https://github.com/yahyaalbashar/phinx-skills.git
claude --plugin-dir ./phinx-skills
```

### Option 3: Local Persistent Install

```bash
git clone https://github.com/yahyaalbashar/phinx-skills.git

# Inside Claude Code:
/plugin marketplace add /absolute/path/to/phinx-skills/.claude-plugin/marketplace.json
/plugin install phinx-skills@phinx-marketplace
```

### Option 4: Direct Copy (No Plugin System)

```bash
git clone https://github.com/yahyaalbashar/phinx-skills.git
cp -r phinx-skills/skills/* ~/.claude/skills/
```

## Management

```bash
# List installed plugins
/plugin list

# Disable temporarily
/plugin disable phinx-skills@phinx-marketplace

# Re-enable
/plugin enable phinx-skills@phinx-marketplace

# Update after repo changes
/plugin update
```

## How It Works

Skills use **progressive disclosure** — only frontmatter (~50 tokens each) loads at startup. Full instructions load on-demand when Claude determines a skill is relevant. Total overhead: ~650 tokens for all 13 skills.

| You say... | Skill triggered |
|---|---|
| "Build a dashboard component" | `frontend-react` |
| "Create the user API endpoint" | `backend-api` |
| "Set up Docker for this project" | `devops-infra` |
| "Design the schema for orders" | `database-design` |
| "Review this PR" | `code-review` |
| "Is this code secure?" | `security-review` |
| "How should we architect this?" | `system-design` |
| "This violates SRP" | `clean-code-architecture` |

Explicit invocation: `/phinx-skills:frontend-react`, `/phinx-skills:code-review`, etc.

## Agent Teams Usage

```
Create an agent team:
- Frontend agent: use frontend-react and testing-strategy skills
- Backend agent: use backend-api, database-design, and security-review skills
- DevOps agent: use devops-infra and performance-optimization skills
- Lead: use system-design, clean-code-architecture, product-thinking, and code-review skills
```

## Project Structure

```
phinx-skills/
├── .claude-plugin/
│   ├── plugin.json              # Plugin manifest
│   └── marketplace.json         # Self-hosted marketplace
├── skills/
│   ├── clean-code-architecture/ # SOLID, patterns, clean architecture
│   │   ├── SKILL.md
│   │   └── references/patterns.md
│   ├── frontend-react/          # React/Next.js/TypeScript
│   │   ├── SKILL.md
│   │   └── references/patterns.md
│   ├── backend-api/             # REST, auth, middleware
│   │   ├── SKILL.md
│   │   └── references/patterns.md
│   ├── system-design/           # Architecture, trade-offs
│   │   ├── SKILL.md
│   │   └── references/distributed-patterns.md
│   ├── devops-infra/            # Docker, CI/CD, K8s
│   │   ├── SKILL.md
│   │   └── references/patterns.md
│   ├── database-design/SKILL.md
│   ├── security-review/SKILL.md
│   ├── code-review/SKILL.md
│   ├── testing-strategy/SKILL.md
│   ├── product-thinking/SKILL.md
│   ├── git-workflow/SKILL.md
│   ├── performance-optimization/SKILL.md
│   └── documentation/SKILL.md
└── README.md
```

## Per-Project Configuration

Skills are project-agnostic. For project-specific config, copy the template:

```bash
cp CLAUDE.md.template /path/to/your-project/CLAUDE.md
```

## Cross-Platform

Skills follow the [Agent Skills open standard](https://agentskills.io). Compatible with Claude Code, Cursor, Codex, and other agents that support the standard.
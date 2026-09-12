<p align="center">
  <h1 align="center">NestJS + React Clean Architecture Skills</h1>
  <p align="center">Skills for Claude Code that keep every feature inside Domain → Application → Infrastructure — and the React side consistent with the same rigor.</p>
</p>

<p align="center">
  <img alt="License" src="https://img.shields.io/github/license/jcgato93/nest-react-clean">
  <img alt="Latest Release" src="https://img.shields.io/github/v/release/jcgato93/nest-react-clean">
  <img alt="GitHub Stars" src="https://img.shields.io/github/stars/jcgato93/nest-react-clean?style=social">
  <img alt="Skills" src="https://img.shields.io/badge/skills-8-blue">
</p>

## Quick start

```bash
npx skills@latest add jcgato93/nest-react-clean
```

## Skills

Unlike slash commands, these skills are **model-invoked** — Claude loads them automatically when a request matches their trigger phrases (in English or Spanish). You never type `/skill-name`; you just ask for the feature.

| Skill | Stack | Loads when you ask for... |
| --- | --- | --- |
| `nest-clean` | NestJS | Any new module/endpoint/field — the entry point that detects the scenario and routes to the sub-skills below |
| `domain-layer` | NestJS | An entity, value object, repository interface, use-case interface, or domain exception |
| `application` | NestJS | A use-case implementation, request/response DTO, or mapper |
| `infrastructure-layer` | NestJS | A DB model/entity or migration (Prisma or TypeORM), repository implementation, controller, or NestJS module |
| `domain-events` | NestJS | "when X is created, do Y" — domain events, `AggregateRoot`, event handlers |
| `environment-config` | NestJS | A new/changed environment variable, `envs.ts`, or config for a new external service |
| `code-review` | NestJS | "review this PR / my changes / this diff" |
| `react-feature-dev` | React | A new page, component, service, or API integration on the frontend |

---

## Table of contents

- [What these skills are](#what-these-skills-are)
- [The problem they solve](#the-problem-they-solve)
- [How the NestJS skill set works](#how-the-nestjs-skill-set-works)
- [How the React skill set works](#how-the-react-skill-set-works)
- [Naming and language conventions](#naming-and-language-conventions)
- [When to trust the skills and when not](#when-to-trust-the-skills-and-when-not)
- [Rules almost nobody follows](#rules-almost-nobody-follows)
- [Installation](#installation)
- [Usage](#usage)
- [Evals](#evals)
- [License](#license)

---

## What these skills are

This repo packages the architectural knowledge of a specific NestJS + Clean Architecture backend (Prisma or TypeORM, Auth0, Redis) and its React 19 companion frontend (Zustand, React Query, React Router v7, Tailwind, shadcn-ui) as Claude Code skills. `infrastructure-layer` detects which ORM the target project uses (cached in `.claude/nest-clean.config.json` after the first check) and follows that ORM's conventions from then on.

Instead of relying on Claude to infer conventions from scattered examples in the codebase — and getting it right roughly as often as it's wrong — each skill states the rule once: where a file goes, what it's named, what layer it belongs to, what it may and may not import. Claude reads the skill before writing the file, not after.

## The problem they solve

Ask an LLM to "add a `transfer` endpoint to bank accounts" without guardrails, and it will improvise: maybe the repository gets injected straight into the controller, maybe the exception thrown is a raw `HttpException` instead of a domain exception, maybe the new field lands in the entity but never makes it into the mapper or the response DTO. Each of those is a silent architectural violation that's easy to miss in review and expensive to unwind later.

Two things make this sharper with an LLM than with a human junior dev:

1. **Generation speed hides the cost of decisions.** A file that takes two seconds to generate gets zero seconds of layer-boundary scrutiny unless something is enforcing it up front.
2. **Every session starts from zero.** Without a written rule, the same architectural mistake can resurface in a different form next session.

The skills fix this by encoding the architecture itself — layer order, naming, file placement, critical rules — as something Claude consults before generating code, not something a human has to catch after the fact.

## How the NestJS skill set works

`nest-clean` is the orchestrator. It defines the shared layout:

```
src/modules/{module}/
  domain/           entities, repositories, use-cases, exceptions, value-objects
  application/      dtos, mappers, use-cases (implementations)
  infrastructure/   repositories (impl), controllers
  {module}.module.ts
```

**Dependencies flow inward only** — domain never imports from application or infrastructure. That single rule is what the other five skills exist to protect.

Before writing anything, `nest-clean` detects which scenario applies and confirms it with you:

- **New module from scratch** — confirms module name, fields, operations, and auth requirements, then implements domain → application → infrastructure in that order, never skipping ahead.
- **Modify existing functionality** — uses an impact table (e.g. "add field to entity" touches domain + application + infrastructure + a migration; "change response shape" touches only application + infrastructure) to catch every affected layer before proposing a diff.
- **Hybrid** — a field change and a new endpoint in the same request are handled sequentially, never interleaved.

Each layer has its own sub-skill (`domain-layer`, `application`, `infrastructure-layer`, `domain-events`, `environment-config`) with step-by-step construction rules — e.g. every entity needs `plainToInstance`/`plainToInstanceList`, every use-case interface documents its business steps in JSDoc, every repository interface only declares methods not already on `BaseRepository`. `code-review` applies the same layer knowledge in reverse, producing a severity-rated review (`CRITICAL`/`HIGH`/`MEDIUM`/`LOW`) with a per-layer checklist and a final `APPROVE`/`REQUEST_CHANGES` verdict.

## How the React skill set works

`react-feature-dev` runs a mandatory 4-phase flow for every change, regardless of size:

1. **Analizar** — checks what existing types, services, stores, and components can be reused before writing anything new.
2. **Planear** — presents a concrete plan: new files, modified files, and the implementation order (types → endpoints → service → hooks → components → page → route).
3. **Confirmar** — stops and waits for explicit approval before touching code.
4. **Implementar** — builds in the planned order against a checklist (Zod schemas, typed DTOs, loading/error/empty states, lazy-loaded routes, no relative import paths, no implicit `any`).

Reference docs under `skills/react/react-feature-dev/references/` (components, services, models, pages, state) are consulted on demand rather than loaded up front — the skill only opens the one relevant to the file being written.

## Naming and language conventions

Both skill sets enforce the same split, since the target codebase is worked on by Spanish-speaking developers building an English-only product:

- **Code — always English**: file names, class names, variables, DB tables, exception messages. Even when you describe the feature in Spanish ("módulo de mantenimientos"), the skill translates it before naming anything (`asset-maintenance`).
- **Comments — Spanish.**
- **Layer boundaries are non-negotiable**: controllers inject use-cases, never repositories; use-cases throw domain exceptions, never `HttpException`; domain code never imports from `application/` or `infrastructure/`.

## When to trust the skills and when not

### They're built for:

- Adding a module, endpoint, or field to the NestJS backend.
- Adding a page, component, or API integration to the React frontend.
- Modifying existing functionality where the ripple effect across layers isn't obvious.
- Reviewing a PR for architectural consistency.

### They're not a substitute for:

- Product/business decisions about what the feature should do — the skills confirm scope with you before writing code, they don't invent it.
- Infrastructure decisions outside the stack they cover (this pair assumes Prisma or TypeORM + Auth0 + Redis on the backend, and the exact frontend stack listed above).
- Manual verification — every skill ends with a checklist (TypeScript compiles, lint passes, Swagger reflects the new endpoints, the generated migration SQL is sane), but running those checks is still on you.

---

## Rules almost nobody follows

Four habits that separate these skills actually working from becoming decorative documentation nobody reads:

### 1. Confirm the scenario before code gets written

`nest-clean` presents a confirmation template (module name, fields, operations, auth, pagination) before touching domain code. Skipping that step to "save time" is how a misunderstood field type ends up threaded through four layers before anyone notices.

### 2. Never skip an inward layer

Domain → Application → Infrastructure, always in that order, even under time pressure. A controller that reaches straight into a repository "just this once" is the crack the whole layering exists to prevent.

### 3. Read the sub-skill before writing the file, not after

Each sub-skill is scoped to one layer for a reason: loading `domain-layer` while writing a controller wastes context and invites cross-layer leakage. Load the skill for the layer you're about to touch, not the whole set at once.

### 4. Treat the migration SQL as something to read, not just generate

`prisma:migrate:dev` infers SQL from the schema diff — it sometimes chooses a destructive `DROP` + `ADD` where a `RENAME` was intended. The final verification step exists specifically to catch that before it ships.

---

## Installation

### Option 1 — skills.sh (recommended, Claude Code)

```bash
npx skills@latest add jcgato93/nest-react-clean
```

To uninstall:

```bash
npx skills@latest remove jcgato93/nest-react-clean
```

### Option 2 — Manual

```bash
# Personal (all your projects)
mkdir -p ~/.claude/skills
cp -r skills/nestjs/* ~/.claude/skills/
cp -r skills/react/react-feature-dev ~/.claude/skills/

# Or per-project (versioned in git)
mkdir -p .claude/skills
cp -r skills/nestjs/* .claude/skills/
cp -r skills/react/react-feature-dev .claude/skills/
```

Only copy the halves you need — a frontend-only or backend-only project should skip the other stack's skills entirely to keep unrelated triggers out of Claude's context.

---

## Usage

You don't invoke these skills directly — you just describe the feature, in English or Spanish, and Claude loads the matching skill:

```
"agregar un módulo de mantenimientos con create, get by id y listar por compañía"
```

Claude:
1. Loads `nest-clean`, detects **Scenario A — new module from scratch**.
2. Confirms the module details with you (fields, operations, auth, pagination) before writing anything.
3. Implements `domain/` (loading `domain-layer` first), then `application/` (loading `application`), then `infrastructure/` (loading `infrastructure-layer`, which resolves whether this project uses Prisma or TypeORM before touching any model/entity), generating the migration along the way.
4. Registers the new module in `AppModule` and runs through the final verification checklist.

For a frontend feature:

```
"agregar la página de mantenimientos con listado y formulario de creación"
```

Claude loads `react-feature-dev`, runs Analizar → Planear, and **stops for your approval** before writing a single component.

For a review:

```
"revisa este PR"
```

Claude loads `code-review` and, using the relevant layer skills as reference, returns a severity-rated review with a merge verdict.

---

## Evals

`skills/nestjs/nest-clean-workspace/` contains benchmark results comparing feature-implementation quality with and without these skills (`evals/evals.json`, `iteration-1/`, `iteration-2/`, each with `with_skill/` and `without_skill/` outputs and grading). They're generated artifacts, kept for reference — if you improve a skill, validate it with a new iteration rather than hand-editing an old one.

---

## License

MIT

---

_If you find a way to improve a skill, open an issue or a PR. The most valuable part of a project-specific skill is that it evolves with the codebase it describes._

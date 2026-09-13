# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repo is a workspace for authoring Claude Code **skills** (not slash commands) that teach an agent to implement features consistently in a specific NestJS + Clean Architecture backend and its companion React frontend. Skills are model-invoked: Claude decides to load them based on their `description`, triggered by phrases like "create module", "add endpoint", "nueva funcionalidad", etc. There is no application code here — only `SKILL.md` files and their reference docs, meant to be copied/symlinked into a target project's `.claude/skills/` (or `.github/skills/`, per the skill bodies) or installed via skills.sh.

Skill bodies mix languages deliberately: instructional/meta text is often Spanish (matching the target dev team), while all code identifiers, file names, and error messages the skills teach must be English — see "Language conventions" below. Follow whatever language mix the specific skill you're editing already uses; don't unify them.

## Skill authoring conventions

Each skill lives under `skills/<stack>/<name>/` (e.g. `skills/nestjs/domain-layer/`, `skills/react/react-feature-dev/`) as a directory containing at minimum a `SKILL.md`. Top-level buckets group skills by stack: `nestjs/` (backend, Clean Architecture) and `react/` (frontend).

The YAML frontmatter of `SKILL.md` here only declares:

```yaml
---
name: skill-name
description: One paragraph describing WHEN to use this skill — the trigger phrases and file locations that should cause Claude to load it. This is what Claude matches against, so be explicit and exhaustive with triggers.
---
```

Unlike slash-command skills, these have no `argument-hint` or `disable-model-invocation` — they're meant to auto-trigger during normal feature work, not be invoked explicitly by the user. The `description` field carries all the weight: list concrete trigger phrases (including Spanish ones, since users describe features in Spanish) and the `src/` paths the skill governs.

Companion reference files (e.g. `entities.md`, `use-cases.md`, `component-patterns.md`) sit next to `SKILL.md` in the same directory and are referenced by relative path from the skill body (`See [entities.md](./entities.md)`). The top-level orchestrator skill for a stack (`nest-clean/SKILL.md`, `react-feature-dev/SKILL.md`) links out to sibling skill directories for layer-specific detail rather than inlining everything — keep that split when adding new skills instead of growing one file indefinitely.

`infrastructure-layer` further splits its persistence-specific reference docs into `prisma/` and `typeorm/` subfolders (each with its own `database-entities.md`, `custom-queries.md`, …) since this project's target codebase can use either ORM. `SKILL.md` itself stays ORM-agnostic where possible and branches into one subfolder or the other based on `.claude/nest-clean.config.json` (a small `{ "orm": "prisma" | "typeorm" }` file cached in the *target* project, written by `nest-clean`'s ORM Detection step the first time it's needed, so later requests don't re-derive it). Follow the same per-backend subfolder pattern if a future skill needs to branch on another interchangeable technology choice (e.g. a queue or cache backend).

`skills/nestjs/nest-clean-workspace/` holds eval artifacts (`evals/evals.json`, `iteration-N/**`, `review.html`) produced by running the skills against benchmark tasks with/without-skill. Treat it as generated output, not something to hand-edit — if you improve a skill, a new eval iteration is how you validate the change, not a manual edit to old iteration folders.

## The NestJS skill set

`skills/nestjs/nest-clean/SKILL.md` is the entry point and architecture reference (Domain → Application → Infrastructure, dependencies flow inward only). It defines the module directory layout under `src/modules/{module}/`, detects which scenario applies (new module / modify existing / hybrid), and links to layer-specific sub-skills for implementation detail:

| Sub-skill | Governs |
|---|---|
| `skills/nestjs/domain-layer/` | Entities, value objects, repository interfaces, use case interfaces, domain exceptions |
| `skills/nestjs/application/` | Use case implementations, DTOs, mappers |
| `skills/nestjs/infrastructure-layer/` | Prisma or TypeORM models/entities/migrations, repository impls, controllers, NestJS modules — branches on the ORM cached in `.claude/nest-clean.config.json` (see its `prisma/` and `typeorm/` reference subfolders) |
| `skills/nestjs/domain-events/` | `AggregateRoot`, domain events, event handlers/listeners |
| `skills/nestjs/environment-config/` | `envs.ts`, Zod-validated environment variables, external service config |
| `skills/nestjs/code-review/` | Structured PR review with severity ratings and a per-layer checklist |

Layer order matters: domain is always implemented before application, application before infrastructure. When editing these skills, preserve that inward-dependency rule — it's the core invariant the whole skill set exists to enforce.

## The React skill set

`skills/react/react-feature-dev/SKILL.md` guides a single 4-phase flow — Analizar → Planear → Confirmar → Implementar — for React 19 + TypeScript + Zustand + React Query + React Router v7 + Tailwind + shadcn-ui features. It stops for explicit user approval of a plan before writing code (Fase 3: CONFIRMAR). `skills/react/react-feature-dev/references/*.md` holds pattern references (components, models, pages, services, state) that the skill body says to consult "only when needed" rather than loading eagerly — keep new reference docs narrow and single-purpose so that stays true.

## Language conventions

These are conventions the skills *teach to the target codebase*, not conventions for editing this repo's own Markdown:

- File names, class names, and code identifiers: always **English**, even when the user describes the feature in Spanish. Skills include a translation table (e.g. "módulo de mantenimientos" → `asset-maintenance`) — extend it if you add new domain examples. This one is not configurable.
- Internal code comments and exception/error messages: **configurable per target project**, cached in the target project's `.claude/nest-clean.config.json` as `commentLanguage` and `exceptionLanguage` (`"es" | "en"`), alongside the existing `orm` key. `nest-clean/SKILL.md`'s "Language Configuration" step resolves and persists these once per project (asking the user if the cache is empty); every other NestJS skill that mentions a comment or exception-message language must point back to that resolved config instead of hardcoding a language — the project's historical default (comments `es`, exceptions `en`) is only the fallback used when the config hasn't been resolved yet, not a fixed rule. Follow the same "resolve once in the orchestrator, cache to the target project's config file, branch everywhere else" pattern used for ORM detection if a future skill needs another per-project, interchangeable setting.
- Skill instructional prose (this repo's own Markdown, not the target codebase's comments): match the existing file — `nest-clean`/domain/application/infrastructure skills are English: prose, Spanish: comments; `react-feature-dev` is Spanish throughout.

If you change a rule like this, update it consistently in every skill file that states it. Historically these skills duplicated fixed rule *values* in multiple places (e.g. the 2026-08-08 correction that flipped exception messages from Spanish to English across `nest-clean`, `domain-layer`, and `code-review` required editing all three); the comment/exception language rule no longer has this problem since the value itself now lives in the target project's config file — but the *pointer* to that config ("check `.claude/nest-clean.config.json`, don't hardcode") is still duplicated across skill files by necessity (each skill is model-invoked independently and may load without the others in context), so keep that pointer consistent when editing.

## Distribution

The repo is meant to be consumed via **skills.sh** (`npx skills@latest add <owner>/<repo>`), which auto-discovers public GitHub repos containing `skills/**/SKILL.md`. There is no build step — pushing to GitHub is the whole release process. This directory is not yet a git repository; initialize one before publishing.

## No build or test commands

There is no package manager, build step, or test suite in this repo. Everything is plain Markdown (`SKILL.md`, reference `.md` files) plus JSON eval fixtures under `nest-clean-workspace/`. "Testing" a skill change means running it against a real or benchmark NestJS/React task and comparing output to the eval iterations, not running an automated suite.

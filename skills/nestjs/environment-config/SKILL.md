---
name: environment-config
description: >
  Use when adding, reading, or validating environment variables in this NestJS project: adding a new required or optional env var, wiring config for a new external service (API keys, base URLs, secrets), or checking whether a value is read correctly from `process.env`. Trigger for phrases like "add env var X", "add environment variable", "add config for Y service", "new API key", "agregar variable de entorno", "agregar configuración de X", "leer variable de entorno". Always use before touching `src/infrastructure/config/envs.ts`, `.env`, or `.env.example`. Never use `@nestjs/config` — this project validates and exposes all environment access through this single typed module.
---

## Environment Config Overview

All environment variables are declared, validated, and typed in **one file**: `src/infrastructure/config/envs.ts`. Nothing else in the codebase reads `process.env` directly — every layer imports the exported `envs` object instead.

```
.env / .env.example        →  raw key=value pairs (never committed with real secrets)
envSchema (Zod)             →  validates + coerces process.env at module load
envs (exported const)       →  typed, camelCase, grouped object — the only thing the app imports
```

This is deliberately **not** `@nestjs/config`. Validation happens once, synchronously, at import time — if a required variable is missing or malformed, the app throws and fails to boot instead of surfacing a runtime `undefined` deep in some service.

---

## File Anatomy

**File**: `src/infrastructure/config/envs.ts`

```typescript
import { Logger } from '@nestjs/common';

import { config } from 'dotenv';
import { z } from 'zod';

config(); // Load environment variables from .env file

export enum Environment {
  Development = 'development',
  Production = 'production',
  Qa = 'qa',
}

// Custom boolean transformer that properly handles "false" strings
const booleanTransformer = z.string().transform((val) => {
  const normalized = val.toLowerCase().trim();
  if (['false', '0', 'no', 'n', ''].includes(normalized)) return false;
  if (['true', '1', 'yes', 'y'].includes(normalized)) return true;
  return Boolean(val);
});

const envSchema = z.object({
  PORT: z.coerce.number().optional(),
  NODE_ENV: z.nativeEnum(Environment, {
    message: `NODE_ENV must be one of: ${Object.values(Environment).join(', ')}`,
  }),
  ALLOW_SWAGGER: z.union([z.boolean(), booleanTransformer]).default(false),

  // Database
  DB_HOST: z.string().default('localhost'),
  DB_PORT: z.coerce.number().default(5432),
  DB_PASSWORD: z.string(),
  DB_NAME: z.string(),
  DB_SSL: z.union([z.boolean(), booleanTransformer]).default(false),

  // Auth0
  AUTH0_DOMAIN: z.string().min(1, { message: 'AUTH0_DOMAIN is required' }),
  AUTH0_CLIENT_SECRET: z.string().min(1, {
    message: 'AUTH0_CLIENT_SECRET is required',
  }),

  SENTRY_DSN: z.string().url(),
});

const { error, data } = envSchema.safeParse(process.env);

if (error) {
  Logger.error('Invalid environment variables:', error.errors);
  throw new Error(
    `Invalid environment variables: ${error.errors.map((e) => `${e.path.join('.')}: ${e.message}`).join(', ')}`,
  );
}

export const envs = {
  port: data.PORT || 3000,
  environment: data.NODE_ENV,
  allowSwagger: data.ALLOW_SWAGGER,

  // Database
  dbHost: data.DB_HOST,
  dbPort: data.DB_PORT,
  dbPassword: data.DB_PASSWORD,
  dbName: data.DB_NAME,
  dbSsl: data.DB_SSL,

  // Auth0
  auth0Domain: data.AUTH0_DOMAIN,
  auth0ClientSecret: data.AUTH0_CLIENT_SECRET,

  sentryDsn: data.SENTRY_DSN,
};
```

Four fixed parts, always in this order:

1. `config()` from `dotenv`, called once at the top, before the schema is built.
2. Any shared `enum` used by a variable (e.g. `Environment` for `NODE_ENV`).
3. `envSchema` — a single `z.object({...})`, fields grouped with a `// Comment` header per concern (Database, Auth0, a specific external service...), keys in `SCREAMING_SNAKE_CASE` matching the real env var name.
4. `envSchema.safeParse(process.env)` followed by an immediate fail-fast `throw` if `error` is set, then the exported `envs` object mapping every schema field to a `camelCase` key, grouped with the same comment headers as the schema.

---

## Adding a New Environment Variable

**Step-by-step**:

1. Add the key to `.env` (local value) and `.env.example` (placeholder/documented value, no real secret).
2. Add the field to `envSchema`, under the comment group it belongs to (create a new `// Comment` group if it's a new service/concern). Pick the Zod type from the table below.
3. Map it in the `envs` export using the same grouping, converting the key to `camelCase`.
4. Use `envs.xxx` everywhere the value is needed — **never** `process.env.XXX` outside this file.

### Choosing the Zod type

| Env var shape | Zod schema | Notes |
|---|---|---|
| Required string / secret | `z.string().min(1, { message: 'X_NAME is required' })` | Always give a custom message naming the var |
| Required URL | `z.string().url()` | e.g. `SENTRY_DSN`, `AUTH0_ISSUER_BASE_URL` |
| Optional string with default | `z.string().default('localhost')` | |
| Number | `z.coerce.number()` (add `.default(n)` or `.optional()` as needed) | Env vars arrive as strings — always `coerce`, never bare `z.number()` |
| Boolean | `z.union([z.boolean(), booleanTransformer]).default(false)` | **Never** use bare `z.coerce.boolean()` — it treats any non-empty string, including `"false"`, as `true`. Always go through `booleanTransformer` |
| Fixed set of values | `z.nativeEnum(SomeEnum, { message: '...must be one of: ...' })` | Declare the `enum` above the schema, reuse it for the exported type too |

### Example: adding `STRIPE_SECRET_KEY`

```typescript
// 1. envSchema — new group
const envSchema = z.object({
  // ...existing fields...

  // Stripe
  STRIPE_SECRET_KEY: z.string().min(1, {
    message: 'STRIPE_SECRET_KEY is required',
  }),
});

// 2. envs export — same group, camelCase
export const envs = {
  // ...existing fields...

  // Stripe
  stripeSecretKey: data.STRIPE_SECRET_KEY,
};
```

Then consume it as `envs.stripeSecretKey` in whatever service under `src/infrastructure/external/` needs it — never re-read `process.env.STRIPE_SECRET_KEY` there.

---

## Rules

- **Single source of truth**: every environment variable is declared in `envSchema` and exposed via `envs`. No file outside `src/infrastructure/config/envs.ts` calls `process.env` directly.
- **Never use `@nestjs/config`** — no `ConfigModule`, no `ConfigService`, no `@nestjs/config` import anywhere in the project.
- **Fail fast at boot**: an invalid or missing required var must throw during module load (via `safeParse` + `throw`), not surface as `undefined` later at request time.
- **Booleans always go through `booleanTransformer`**: bare `z.coerce.boolean()` is a known trap — `Boolean("false")` is `true` in JS, so a literal `"false"` string in `.env` would otherwise be read as truthy.
- **Numbers always use `z.coerce.number()`**: raw env values are strings; without `coerce`, a numeric-looking value fails validation.
- **Required secrets get a descriptive `.min(1, { message: '<VAR_NAME> is required' })`** — the error thrown at boot should name the exact variable that's missing, not just say "invalid config".
- **Group related vars with a `// Comment` header**, mirrored identically between `envSchema` and the `envs` export, so the two stay easy to diff against each other.
- **`.env.example` is updated in the same change** as any new var added to `envSchema` — it documents every variable a new environment needs, without real secret values.
- Comments inside this file: Spanish, per the project's convention — only the two above (`booleanTransformer` explanation, load-order notes) are commonly needed; don't over-comment self-explanatory schema fields.

---

## Checklist

- [ ] New var added to `.env` and `.env.example`
- [ ] New var added to `envSchema`, under the right comment group, with the correct Zod type from the table above
- [ ] Required vars have a `.min(1, { message: '<NAME> is required' })` (or `.url()`/`.nativeEnum()` as applicable) with a descriptive message
- [ ] Booleans use `z.union([z.boolean(), booleanTransformer])`, never bare `z.coerce.boolean()`
- [ ] Numbers use `z.coerce.number()`, never bare `z.number()`
- [ ] Mapped in the `envs` export with a `camelCase` key, same comment group as the schema
- [ ] No new `process.env.X` reads anywhere outside `src/infrastructure/config/envs.ts`
- [ ] No `@nestjs/config` import introduced

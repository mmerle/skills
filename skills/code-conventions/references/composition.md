# Composition

## The Rule

Code is organized by ownership.

Each unit owns one concern, exposes one clear entrypoint, and keeps its internals private.

## Ownership

Keep code where it belongs:

| Concern | Owner |
|---------|-------|
| Startup and wiring | Entrypoint |
| Feature behavior | Feature module |
| Business rules | Domain module |
| External services | Integration |
| Shared pure behavior | Utility |
| Reusable systems | Module or package |

Supporting files live beside the unit that owns them.

Do not move code to shared folders until two real callers need it.

## Boundaries

Import through the unit entrypoint.

```typescript
// GOOD
import { loadImage } from "~/modules/image-loader";

// BAD
import { processImage } from "~/modules/image-loader/process";
```

Only export what outside code needs.

Private files can change without affecting callers.

## Composition Flow

Dependencies point inward:

```
entrypoint -> feature -> module -> utility
```

Outer layers coordinate. Inner layers do the work.

Do not import upward. Do not reach into another unit's private files.

## State

State belongs as close as possible to where it is used.

State has one owner. Derived values are computed, not stored.

Use shared state only when multiple distant units need the same source of truth.

Do not pass state through code that does not use it.

## Integrations

External systems live behind integrations.

Callers should not know vendor clients, auth headers, retry rules, raw response shapes, or normalization details.

```txt
integrations/payments/
  index.ts
  client.ts
  normalize.ts
  types.ts
```

Callers import from `index.ts`.

## Shared Code

Shared code must stay small and specific.

Use utilities for pure reusable behavior. Use modules or packages for reusable systems.

Do not use `utils` as a dumping ground.

## Split Rule

If a file owns multiple concerns, split it by ownership before adding more code.

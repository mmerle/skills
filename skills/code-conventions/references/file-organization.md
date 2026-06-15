# File Organization

## Naming Conventions

| Type                 | Convention                          | Examples                                                        |
| -------------------- | ----------------------------------- | --------------------------------------------------------------- |
| **Component files**  | PascalCase                          | `Button.tsx`, `ActionBar.tsx`, `DropdownMenu.element.tsx`       |
| **Component styles** | PascalCase, matching component      | `Button.module.css`, `ActionBar.module.css`, `DropdownMenu.css` |
| **Utility files**    | camelCase                           | `utilities.ts`, `constants.ts`, `queries.ts`                    |
| **Directories**      | lowercase, singular when a category | `components/`, `common/`, `modules/`, `runtime/`                |
| **Pages**            | lowercase or kebab-case             | `pages/examples/accordion.tsx`                                  |
| **CSS classes**      | kebab-case with `_` modifier        | `.root`, `.action-button`, `.left-panel`, `.left-panel_inner`   |
| **CSS variables**    | kebab-case with prefix              | `--theme-background`, `--color-gray-60`                         |
| **TypeScript types** | PascalCase                          | `ApiResult<T>`, `UserResponse`                                  |
| **Functions**        | camelCase                           | `getData()`, `classNames()`, `isEmpty()`                        |
| **Constants**        | UPPER_SNAKE_CASE or camelCase       | `API_URL`, `apiVersion`                                         |
| **Vendored modules** | lowercase kebab-case directory      | `modules/object-assign/index.ts`                                |

## Path Aliases

All projects define path aliases in `tsconfig.json` to avoid relative imports across top-level directories:

```json
{
  "compilerOptions": {
    "paths": {
      "~/*": ["./src/*"]
      "@lib/*": ["./vendor/lib/*"]
    }
  }
}
```

**Rule**: Never use `../../` to cross top-level directory boundaries. Use path aliases instead.

```typescript
// GOOD
import Button from "~/components/Button";
import * as Utilities from "~/common/utilities";

// BAD
import Button from "../../components/Button";
import * as Utilities from "../../../common/utilities";
```

Relative imports are fine within the same directory (e.g., `./Button.module.css`).

## Vendored Modules

The `modules/` or `packages/` directory contains self-contained, low-dependency code that would otherwise be an npm package:

```
modules/
  cookies/index.ts     # Cookie parsing
  cors/index.ts        # CORS headers
```

**Rule**: If a package does one thing and is small, vendor it in `modules/` instead of adding it to `package.json`. This keeps the dependency tree minimal and the code auditable.

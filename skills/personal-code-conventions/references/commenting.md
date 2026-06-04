# Commenting Conventions

## The Rule

Comments explain **why**, never **what**. If the code needs a comment to explain what it does, rename the variable or function instead.

## What NOT to Comment

**Don't describe what code does:**
```typescript
// BAD: Set the user's name
user.name = name;

// BAD: Loop through the items
for (const item of items) { ... }

// BAD: Check if the user is authenticated
if (!user) return;
```

**Don't add section dividers or decorative comments:**
```typescript
// BAD:
// ==========================================
// AUTHENTICATION
// ==========================================
```

## Format Rules

- Use `//` for TypeScript/JavaScript, `/* */` for CSS, `#` for Shell
- One space after the comment marker
- Keep comments on their own line above the code they reference, not inline at end of line
- Never end comments with periods `.`

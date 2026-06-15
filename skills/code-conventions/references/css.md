# CSS Conventions

## Native CSS Nesting

New projects use native CSS nesting instead of SCSS:

```css
.root {
  display: flex;

  &:hover {
    background: var(--theme-foreground);
  }

  &:focus-visible {
    box-shadow: 0 0 0 4px var(--theme-input-active);
  }
}
```


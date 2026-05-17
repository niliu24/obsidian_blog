# Warm Academic Theme Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply warm academic theme: cream background, amber/brown accents, Georgia serif headers.

**Architecture:** Single-file config change. Modify `theme.colors.lightMode` and `theme.typography.header` in `quartz.config.ts`. No layout, component, or plugin changes needed.

**Tech Stack:** TypeScript config, SCSS variables (auto-generated from config)

---

### Task 1: Apply warm academic theme to config

**Files:**

- Modify: `quartz.config.ts:22-29` (typography.header), `quartz.config.ts:31-41` (colors.lightMode)

- [ ] **Step 1: Apply typography and color changes**

Edit `quartz.config.ts`:

```typescript
// Line 26: Change header font
header: "Georgia",

// Lines 31-41: Replace lightMode colors
lightMode: {
  light: "#fdfaf3",
  lightgray: "#efe5d5",
  gray: "#b8a88a",
  darkgray: "#5c4a3a",
  dark: "#3d2e1e",
  secondary: "#c17817",
  tertiary: "#8b6914",
  highlight: "rgba(193, 120, 23, 0.12)",
  textHighlight: "#f5d78a88",
},
```

- [ ] **Step 2: Run type-check to verify config is valid**

Run: `npm run check`
Expected: No TypeScript errors

- [ ] **Step 3: Build and verify locally**

Run: `npx quartz build -d docs`
Expected: Build succeeds, output in `docs/` directory

- [ ] **Step 4: Commit**

```bash
git add quartz.config.ts
git commit -m "feat: apply warm academic theme - cream base, amber accents, Georgia headers"
```

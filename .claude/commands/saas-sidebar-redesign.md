---
description: Redesign the Vue 3 app UI into a modern SaaS-style interface with a vertical navigation sidebar
---

Redesign the application's UI from a top navigation bar to a modern SaaS-style layout with a fixed vertical sidebar on the left. The result should look like a polished B2B SaaS product (similar to Linear, Vercel, or Notion).

## Phase 1 — Understand the current layout (do this first)

Read these files to understand what you're working with before making any changes:
- `client/src/App.vue` — current top nav structure, global CSS, components used
- `client/src/main.js` — all routes and their labels
- `client/src/locales/en.js` — i18n keys used in the nav

Identify:
1. All navigation items (path + label)
2. Components rendered in the header (FilterBar, LanguageSwitcher, ProfileMenu, etc.)
3. All global CSS classes that affect layout (`.app`, `.top-nav`, `.nav-container`, `.main-content`, etc.)

## Phase 2 — Redesign App.vue (delegate to vue-expert)

**MANDATORY: Use the `vue-expert` subagent (Task tool) to implement all changes to `.vue` files.**

Hand the vue-expert agent the full current content of `App.vue` and instruct it to rewrite the layout with these exact specifications:

### Layout structure

```
┌─────────────────────────────────────────────┐
│  ┌──────────┐  ┌───────────────────────────┐ │
│  │          │  │  FilterBar (top of main)   │ │
│  │ Sidebar  │  ├───────────────────────────┤ │
│  │  240px   │  │                           │ │
│  │  fixed   │  │   <router-view />          │ │
│  │          │  │                           │ │
│  └──────────┘  └───────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### Sidebar design requirements

**Structure (top to bottom):**
1. **Logo area** — company name + subtitle, styled with a subtle bottom border
2. **Navigation links** — vertical list, one per route; active link has a left accent bar and blue background tint
3. **Spacer** (`flex: 1`) to push the bottom section down
4. **Bottom section** — LanguageSwitcher, then ProfileMenu, stacked vertically

**CSS for sidebar:**
```css
.sidebar {
  width: 240px;
  min-width: 240px;
  background: #ffffff;
  border-right: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  height: 100vh;
  position: fixed;
  top: 0;
  left: 0;
  z-index: 100;
  overflow-y: auto;
}

.sidebar-logo {
  padding: 1.5rem 1.25rem 1.25rem;
  border-bottom: 1px solid #e2e8f0;
}

.sidebar-logo h1 {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 0.25rem;
}

.sidebar-logo .subtitle {
  font-size: 0.75rem;
  color: #94a3b8;
  font-weight: 400;
}

.sidebar-nav {
  padding: 0.75rem 0.75rem;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
  flex: 1;
}

.sidebar-nav a {
  display: block;
  padding: 0.625rem 0.875rem;
  color: #64748b;
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  border-radius: 6px;
  transition: all 0.15s ease;
  border-left: 2px solid transparent;
}

.sidebar-nav a:hover {
  color: #0f172a;
  background: #f8fafc;
}

.sidebar-nav a.active {
  color: #2563eb;
  background: #eff6ff;
  border-left-color: #2563eb;
}

.sidebar-bottom {
  padding: 1rem 0.75rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}
```

**Main content area — shift right to account for sidebar:**
```css
.app {
  display: flex;
  min-height: 100vh;
}

.main-wrapper {
  margin-left: 240px;
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.main-content {
  flex: 1;
  max-width: 1360px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
}
```

**FilterBar** moves inside `.main-wrapper` above `.main-content` (it was previously in the global header). Keep it as a sticky bar at the top of the main wrapper:
```css
.filter-wrapper {
  background: #ffffff;
  border-bottom: 1px solid #e2e8f0;
  position: sticky;
  top: 0;
  z-index: 50;
}
```

### Template skeleton to produce

```html
<template>
  <div class="app">
    <aside class="sidebar">
      <div class="sidebar-logo">
        <h1>{{ t('nav.companyName') }}</h1>
        <span class="subtitle">{{ t('nav.subtitle') }}</span>
      </div>
      <nav class="sidebar-nav">
        <!-- one router-link per route, same active class logic as before -->
      </nav>
      <div class="sidebar-bottom">
        <LanguageSwitcher />
        <ProfileMenu ... />
      </div>
    </aside>

    <div class="main-wrapper">
      <div class="filter-wrapper">
        <FilterBar />
      </div>
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <!-- modals unchanged -->
  </div>
</template>
```

### Global CSS changes
- Remove `.top-nav`, `.nav-container`, `.nav-tabs`, `.nav-tabs a` rules entirely
- Add `.sidebar`, `.sidebar-logo`, `.sidebar-nav`, `.sidebar-bottom`, `.main-wrapper`, `.filter-wrapper` rules
- Update `.app` to `display: flex` (remove `flex-direction: column`)
- Update `.main-content` to remove `margin: 0 auto` top-padding compensation and add appropriate padding

### Preserve unchanged
- All `<ProfileDetailsModal>` and `<TasksModal>` logic — keep 100% identical
- All `setup()` script logic (tasks, auth, i18n)
- All global utility CSS classes: `.card`, `.badge`, `.stat-card`, `.loading`, `.error`, `.table-container`, `table`, `th`, `td`, etc.

## Phase 3 — Verify nothing broke

After App.vue is updated, read `client/src/views/Dashboard.vue` (or one other view) to check that `.page-header`, `.stats-grid`, and `.card` styles still render correctly with the new layout. The `margin-left: 240px` on `.main-wrapper` is the only structural change views should feel.

## Phase 4 — Report

Summarise:
- Files changed and what changed in each
- Any CSS class renames the user should know about
- Instructions to test: start the dev server (`npm run dev` in `client/`) and navigate to each route

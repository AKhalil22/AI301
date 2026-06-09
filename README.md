# Contribution 1: Make PlaylistPage customizable in Backstage community plugins

**Contribution Number:** 1  
**Student:** Ammar Khalil  
**Issue:** https://github.com/backstage/community-plugins/issues/6402  
**Status:** Phase II — In Progress

---

## Why I Chose This Issue

Backstage is a widely-used developer portal platform built around an extensible plugin system. This issue sat at the intersection of plugin architecture and React component design — both areas I wanted to deepen. The fix was well-scoped: a clear pattern already existed in the codebase (`PlaylistIndexPage`) that I just needed to apply consistently to `PlaylistPage`. That made it achievable as a first contribution while still requiring me to understand how Backstage's routing and plugin system works.

I also wanted a contribution that wasn't just a typo fix — this one adds real value to teams using Backstage who couldn't previously customize their playlist detail pages without forking the plugin.

---

## Understanding the Issue

### Problem Description

The `PlaylistPage` component (the detail view for a single playlist) can't be customized without forking the plugin. If a team using Backstage wants to add their own UI elements — like a security audit panel — to the playlist detail page, they have no hook to do so. The component is monolithic and doesn't expose any customization mechanism.

### Expected Behavior

Teams should be able to inject a custom implementation of the playlist detail page by providing a child route, the same way they can already customize `PlaylistIndexPage`. The default behavior should be preserved for teams that don't need customization.

### Current Behavior

`PlaylistPage` ignores any child routes. There is no `useOutlet()` call, no fallback mechanism, and no public export of `DefaultPlaylistPage`. The internal components `PlaylistHeader` and `PlaylistEditDialog` are also not part of the public API, making it impossible to build a custom page that reuses them.

### Affected Components

- `src/components/PlaylistPage/PlaylistPage.tsx` — the monolithic component
- `src/components/PlaylistPage/index.ts` — missing exports
- `src/index.ts` — public API exports
- `src/components/PlaylistPage/PlaylistPage.test.tsx` — tests needed to be updated

---

## Reproduction Process

### Environment Setup

Cloned backstage/community-plugins with sparse checkout targeting only `workspaces/playlist/` to avoid pulling the entire monorepo.

```
git clone --depth 1 --filter=blob:none --sparse https://github.com/backstage/community-plugins.git
git sparse-checkout set workspaces/playlist
```

### Steps to Reproduce

1. Look at `PlaylistIndexPage.tsx` — it uses `useOutlet()` to allow customization
2. Look at `PlaylistPage.tsx` — it has no such mechanism, directly renders all UI
3. There is no `DefaultPlaylistPage` export, and `PlaylistHeader` / `PlaylistEditDialog` are not in the public API

### My Findings

The `PlaylistIndexPage` was refactored in 2023 to support customization (commit introduced `DefaultPlaylistIndexPage` and the outlet pattern). `PlaylistPage` predates this refactor and was never updated. The fix is mechanical: apply the same 3-file pattern.

---

## Solution Approach

### Analysis

The root cause is that `PlaylistPage` was written before the outlet-based customization pattern was established in this codebase. It needs the same treatment `PlaylistIndexPage` received.

### Proposed Solution

1. Extract `PlaylistPage`'s current body into a new `DefaultPlaylistPage` component
2. Replace `PlaylistPage` with a thin wrapper that uses `useOutlet()` — renders the outlet if provided, otherwise renders `DefaultPlaylistPage`
3. Export `DefaultPlaylistPage`, `PlaylistHeader`, `PlaylistHeaderProps`, `PlaylistEditDialog`, and `PlaylistEditDialogProps` as public API

### Implementation Plan

**Understand:** `PlaylistPage` can't be extended. Teams can't inject custom components without forking.

**Match:** `PlaylistIndexPage` uses `useOutlet()` from react-router-dom. If a child route is provided it renders that; otherwise it renders `DefaultPlaylistIndexPage`. This is the pattern to replicate.

**Plan:**
1. Create `DefaultPlaylistPage.tsx` — move all current `PlaylistPage` logic here, add `@public` JSDoc
2. Rewrite `PlaylistPage.tsx` — `useOutlet()` wrapper, 8 lines
3. Update `components/PlaylistPage/index.ts` — add `DefaultPlaylistPage` and `PlaylistHeader` exports
4. Update `src/index.ts` — add `DefaultPlaylistPage`, `PlaylistHeader`, `PlaylistEditDialog` and their prop types to public API
5. Create `DefaultPlaylistPage.test.tsx` — move existing full-page tests here (they test `DefaultPlaylistPage` now)
6. Replace `PlaylistPage.test.tsx` — outlet-behavior tests only (mirrors `PlaylistIndexPage.test.tsx`)
7. Add a changeset file

**Implement:** See branch `feat/customizable-playlist-page` in fork [link TBD after fork]

**Review:**
- [ ] Follows the exact same pattern as `PlaylistIndexPage`
- [ ] No breaking changes — existing behavior is preserved as `DefaultPlaylistPage`
- [ ] Public API additions are minimal and justified by the issue
- [ ] Tests cover both the outlet path and the default path
- [ ] Changeset file created with `minor` bump

**Evaluate:** `yarn test` in `workspaces/playlist/` — all tests pass including new outlet tests

---

## Testing Strategy

### Unit Tests

- [x] `PlaylistPage` renders the custom outlet when a child route is provided
- [x] `PlaylistPage` renders `DefaultPlaylistPage` when no child route is provided
- [x] `DefaultPlaylistPage` renders playlist info (description, follow button)
- [x] `DefaultPlaylistPage` hides follow button when unauthorized
- [x] `DefaultPlaylistPage` toggles follow/unfollow state

### Manual Testing

Verify `DefaultPlaylistPage` and `PlaylistHeader` are importable from the public package entry point.

---

## Implementation Notes

### Week 1 Progress

Explored the codebase via sparse clone. Found the exact pattern to replicate from `PlaylistIndexPage`. Implemented all 7 file changes:

- **Files created:** `DefaultPlaylistPage.tsx`, `DefaultPlaylistPage.test.tsx`, `.changeset/customizable-playlist-page.md`
- **Files modified:** `PlaylistPage.tsx`, `PlaylistPage.test.tsx`, `components/PlaylistPage/index.ts`, `src/index.ts`

Key decision: Moved the existing `PlaylistPage` tests to `DefaultPlaylistPage.test.tsx` (they test page behavior, not routing) and replaced `PlaylistPage.test.tsx` with outlet routing tests — matching exactly the structure of `PlaylistIndexPage.*`.

---

## Pull Request

**PR Link:** [TBD — pending fork and push]

**PR Description Draft:**

> Makes `PlaylistPage` customizable using the same outlet pattern established for `PlaylistIndexPage`. A child route provided to `PlaylistPage` will be rendered in its place; otherwise the existing behavior is preserved as `DefaultPlaylistPage`.
>
> Also exports `PlaylistHeader`, `PlaylistEditDialog`, and their prop types as public API so custom page implementations can reuse them.
>
> Note: AI assistance (Claude) was used during implementation. All changes have been reviewed and understood.

**Status:** Awaiting fork + push

---

## Learnings & Reflections

### Technical Skills Gained

- How Backstage's plugin system uses React Router outlets for extensibility
- How to read a large monorepo with sparse checkout to focus on just the relevant workspace
- The Backstage changeset workflow for versioning

### Challenges Overcome

- Understanding what "customizable" means in the Backstage plugin context — it's not visual customization, it's about letting downstream teams replace entire page implementations without forking
- Knowing which components to export: kept it minimal (only what the issue asked for + what a custom implementation would actually need)

---

## Resources Used

- https://github.com/backstage/community-plugins/issues/6402
- `PlaylistIndexPage.tsx` and `PlaylistIndexPage.test.tsx` — direct pattern reference
- Backstage CONTRIBUTING.md — AI use policy

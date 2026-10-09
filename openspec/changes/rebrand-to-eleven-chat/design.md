# Design

## Context

See `proposal.md` — Why. The relevant current state (verified at branch `arena/5e6a0db9-eleven-chat`,
base `f678a5a`):

- The name surfaces are already operator-plumbed: `APP_TITLE` feeds the startup-config `appTitle`
  (`api/server/routes/config.js:89`), shared-link titles (`packages/api/src/shared-links/config.ts:90`),
  2FA/passkey display name, email `appName` (4 sites in `AuthService.js`, plus `UserController.js`,
  `sendEmail.js`), and email templates are `{{appName}}`-parameterized. The client has a single
  `DEFAULT_APP_TITLE` constant (`client/src/utils/documentTitle.ts:4`) plus two hardcoded
  `… | LibreChat` document titles (`Marketplace.tsx`, `InsightsView.tsx`) and one fallback literal in
  `AuthLayout.tsx:98`.
- The theme system is token-based: `packages/client/src/theme/themes/default.ts` / `dark.ts` define
  RGB-triple tokens; bundled themes live in `theme/registry.ts` (`libreChatTheme`, `clickHouseTheme`);
  the accepted bundled-name list is `packages/data-provider/src/theme.ts:9`
  (`['librechat', 'clickhouse']`), consumed by config validation (`packages/api/src/app/theme.ts`)
  and the client provider (`client/src/Providers/DeploymentTheme.tsx`).
- Icons and the PWA manifest are wired in `client/index.html` (title, description, favicon links)
  and `client/vite.config.ts` (manifest `name`/`short_name`/`theme_color`, icon list). The Vite PWA
  config already gives icon files content revisions (`dontCacheBustURLsMatching`), so replaced icons
  propagate to installed PWAs.
- Constraints from `AGENTS.md`: minimize and isolate the custom diff (§1), mark custom blocks
  (§5), theme changes through semantic tokens only (§8), locale values may change but keys may not
  (§8), `[brand]` commit tagging (§4), create `CUSTOMIZATIONS.md`/`CUSTOM_CHANGELOG.md` on first
  customization (§15), and every upstream edit must survive future upstream syncs (§18).

## Goals / Non-Goals

**Goals:**

- Every user-/operator-visible surface presents "Eleven Chat" with the orange "11 / ll" mark, with
  the smallest possible diff against upstream.
- All changes mechanically re-appliable after an upstream sync (markers + inventory).
- Independently revertible, per-concern commits.
- No API shape, database, Redis, dependency, or wire-protocol changes.

**Non-Goals:**

- Renaming npm packages, `librechat.yaml`, env var names, `X-LibreChat-Generation-Protocol`, Mongo
  DB name, Redis prefixes, Docker volume names (confirmed out of scope).
- Helm chart / CI image-name renames; CONTRIBUTING/SECURITY/CODE_OF_CONDUCT rewrites (deferred,
  separate change).
- Replacing links, rings, focus, surfaces, or high-contrast themes; translating locales beyond the
  brand-word swap.

## Decisions

### D1: Re-default the existing plumbing instead of adding a branding layer
Change only the fallback literals (`'LibreChat'` → `'Eleven Chat'`) in the ~12 name-resolution
sites. No new config fields, no branding service, no additional abstraction.
*Alternative considered:* a central server-side brand constant exported to all sites — rejected: it
rewrites upstream call sites wholesale and inflates the merge-conflict surface for zero behavioral
gain.

### D2: Replace brand assets in place, same filenames and paths
`client/public/assets/{logo.svg, favicon-16x16.png, favicon-32x32.png, apple-touch-icon-180x180.png,
icon-192x192.png, maskable-icon.png}` are replaced with the Eleven mark at identical paths and
reference dimensions. `index.html` and `vite.config.ts` keep their existing icon wiring.
*Alternative:* new filenames plus reference edits — rejected: touches every consumer (README, tests,
manifest) for no benefit. Binary files cannot carry custom markers; they are inventoried in
`CUSTOMIZATIONS.md` instead (AGENTS.md §5 JSON/binary rule applied by analogy).

### D3: Theme changes are token-value edits plus one new bundled theme
- `default.ts`: `rgb-accent-primary` `#126e6b` → `#EA580C` (234 88 12), hover `#0a4f53` → `#C2410C`
  (194 65 12). `dark.ts`: `#41a79d` → `#F97316` (249 115 22), hover `#6dc8b9` → `#FB923C`
  (251 146 60). No new selectors, no raw palette colors in components (AGENTS.md §8).
- New bundled theme `eleven` in `theme/registry.ts` following the `libreChatTheme`/`clickHouseTheme`
  pattern, exported via `themes/index.ts`; register `'eleven'` in `bundledThemeNames`
  (`packages/data-provider/src/theme.ts`) and map it in `DeploymentTheme.tsx`. Config validation in
  `packages/api/src/app/theme.ts` picks up the new name through the shared list — no server change.
- `librechat.example.yaml` documents `eleven` in the `interface.theme` bundled-name list.
*Alternative:* changing only the deployment-theme route and leaving defaults teal — rejected: the
confirmed decision is a full brand accent.

### D4: Locale edits change values only
Swap the brand word inside values of the 8 affected English keys and the translated values in the
~29 locales that mention the product. Keys, and therefore the `Translation.spec.ts` parity contract
and every `useLocalize` consumer, stay untouched. Locales without a brand mention need no change.

### D5: Custom markers and the first customization inventory
Every touched upstream text file gets the smallest coherent block wrapped in
`// >>> CUSTOM:START [rebrand-eleven-chat] <<<` … `>>> CUSTOM:END [rebrand-eleven-chat] <<<` (syntax
per file type per AGENTS.md §5). JSON (`translation.json`) and binaries carry no markers and are
inventoried. `CUSTOMIZATIONS.md` is created in this change listing every modified upstream file,
the marker tag, rationale, and per-file upstream-merge risk; `CUSTOM_CHANGELOG.md` records the
change summary.

### D6: Commit sequencing as [brand]-tagged, independently revertible slices
1. `[brand]` assets + PWA manifest/browser shell · 2. `[brand]` runtime name defaults + `.env.example`
· 3. `[i18n]` locale values · 4. `[brand]` theme palette + bundled `eleven` theme ·
5. `[docs]` README (+ FUNDING removal) · 6. `[docs]` customization inventory. Default-asserting test
updates ride in the same commit as the code they assert.

### D7: Logo asset pipeline
Author one SVG as the source of truth (mark: two straight parallel square-capped bars, bar height ≈
2.6× width, gap ≈ 1× width, solid `#F97316`, ≥2 px gap at 16 px; wordmark "Eleven Chat" in a
geometric sans at weight 600). Derive the five PNGs from it. A visual draft is approved before the
PNG set is generated (task gate, not a spec gate).

### D8: Birthday easter egg retarget
`isBirthday()` in `api/server/routes/config.js` moves from Feb 11 to Nov 11 ("Eleven Day"). One-line
edit; the client display logic and `com_ui_happy_birthday` copy are untouched.

## Risks / Trade-offs

- [Upstream merge conflicts on every touched upstream file] → minimal one-line edits, markers on
  each, in-place asset replacement, and `CUSTOMIZATIONS.md` so a sync reapplies them mechanically
  (AGENTS.md §18 step 4).
- [Locale drift across ~30 files] → value-only word swaps; English fallback covers untranslated
  gaps; `static-checks:full` runs the i18n gate.
- [Accessibility regression from orange accents] → palette pre-verified against WCAG (white on
  `#EA580C` 3.56:1 — UI components/large text; `#F97316` on near-black 6.84:1); strict-AA light-mode
  alternative (`#C2410C`/`#9A3412`) is a two-token swap if wanted later; visual QA in all four modes.
- [Operators who never set `APP_TITLE` silently rebrand] → documented in `CUSTOM_CHANGELOG.md`;
  one-line `APP_TITLE` override restores any prior name.
- [Tests asserting the old default] → updated in the same commit as the change they assert.
- [Installed PWAs keeping stale icons] → pre-existing Vite PWA config already attaches content
  revisions to the stable icon filenames; verified during Phase-1 build check.

## Migration Plan

Deploy: no migration, no config action required. Operators wanting the old name set
`APP_TITLE=LibreChat`; wanting old colors set `REACT_APP_THEME_*` overrides or a custom
`interface.theme`. Rollback: `git revert` per commit slice; assets revert by checkout; the name and
footer revert at runtime via env overrides without code rollback.

## Open Questions

- Final logo visual approval (D7 draft gate) — does not affect specs or task structure.
- Whether the light-mode primary should be the strict-AA `#C2410C` variant instead of `#EA580C` —
  both satisfy the spec's ≥3:1 requirement; a two-token swap either way.

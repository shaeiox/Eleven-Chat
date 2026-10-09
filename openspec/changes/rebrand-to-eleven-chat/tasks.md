# Tasks

## 1. Brand mark and icon assets

- [ ] 1.1 Author the Eleven Chat mark as a single SVG source of truth: two straight, parallel, square-capped vertical bars ("11" / "ll") in `#F97316` (bar height ≈ 2.6× width, gap ≈ 1× width) plus an "Eleven Chat" wordmark variant; present the draft for visual approval — verify by rendering the SVG at 512 px and at 16 px (bars stay distinct)
- [ ] 1.2 Replace the six brand assets in place at identical paths and dimensions: `client/public/assets/logo.svg`, `favicon-16x16.png`, `favicon-32x32.png`, `apple-touch-icon-180x180.png`, `icon-192x192.png`, `maskable-icon.png` (mark inside the 80% maskable safe zone) — verify each PNG's pixel dimensions and that no consumer reference changed
- [ ] 1.3 Confirm the client production build still bundles the icons — run `cd client && npm run build` and check the emitted `dist` manifest/HTML reference the same asset paths

## 2. Browser shell and PWA identity

- [ ] 2.1 Update `client/index.html`: `<title>Eleven Chat</title>` and a rebranded meta description, wrapped in HTML custom markers — verify by opening the built `index.html` and reading the title/description
- [ ] 2.2 Update the PWA manifest block in `client/vite.config.ts` (`name`/`short_name`: `Eleven Chat`, `theme_color`: `#F97316`) wrapped in custom markers — verify the built `manifest.webmanifest` fields and that icon entries still resolve to the replaced assets

## 3. Runtime name defaults

- [ ] 3.1 Server-side: change the fallback literals to `Eleven Chat` in `api/server/routes/config.js` (`appTitle`), `packages/api/src/shared-links/config.ts`, `api/server/controllers/TwoFactorController.js`, `api/server/controllers/UserController.js`, `api/server/services/AuthService.js` (4 sites), and `api/server/utils/sendEmail.js`; rebrand the `helpAndFaqURL` default to the Eleven-Chat project URL — each edit marker-wrapped — verify with `npm run test:api` after updating 3.4's specs
- [ ] 3.2 Client-side: set `DEFAULT_APP_TITLE = 'Eleven Chat'` in `client/src/utils/documentTitle.ts`; replace the fallback in `AuthLayout.tsx`; switch `Marketplace.tsx` and `InsightsView.tsx` to import `DEFAULT_APP_TITLE` instead of hardcoded `LibreChat`; rebrand the generic footer link in `Chat/Footer.tsx` (`[Eleven Chat vX](https://github.com/shaeiox/Eleven-Chat)`), the About label (`Eleven Chat version:`), and the MCP host name in `hooks/MCP/useAppBridge.ts` — verify with `npm run test:client`
- [ ] 3.3 Retarget the birthday check in `api/server/routes/config.js` from Feb 11 to Nov 11 ("Eleven Day") — verify with a focused assertion in `api/server/routes/__tests__/config.spec.js` covering both the new date and the old date no longer matching
- [ ] 3.4 Update default-asserting tests in `api/server/routes/__tests__/config.spec.js` to expect `Eleven Chat` (and keep an override case asserting `APP_TITLE` wins) — verify with `cd api && npx jest server/routes/__tests__/config.spec.js`
- [ ] 3.5 Update `.env.example`: `APP_TITLE=Eleven Chat`, `EMAIL_FROM_NAME` example, `HELP_AND_FAQ_URL` default — verify by grepping `.env.example` has no upstream brand defaults left

## 4. Locale values

- [ ] 4.1 Rebrand the 8 English values in `client/src/locales/en/translation.json` that name the product (`com_agents_mcp_trust_subtext`, `com_error_attachment_limit`, `com_nav_info_reply_notification_sound`, `com_nav_info_reply_notifications`, `com_ui_api_keys_description`, `com_ui_code_environment_unavailable`, `com_ui_code_environment_remove_description`, `com_ui_tools_native_short`) — keys unchanged — verify `grep -c LibreChat client/src/locales/en/translation.json` returns 0
- [ ] 4.2 Swap the brand word in the values of every translated locale that mentions the product (~29 `translation.json` files) without touching keys — verify the locale key-parity spec passes (`cd client && npx jest src/locales/Translation.spec.ts`)
- [ ] 4.3 Run the i18n/static gates — `npm run static-checks:full` — and confirm no locale drift errors

## 5. Theme palette and bundled eleven theme

- [ ] 5.1 Set the light accent tokens in `packages/client/src/theme/themes/default.ts` (`rgb-accent-primary` → `#EA580C` (234 88 12), hover → `#C2410C` (194 65 12)) and the dark tokens in `dark.ts` (→ `#F97316` (249 115 22), hover → `#FB923C` (251 146 60)) — verify the token values with an assertion in the theme specs
- [ ] 5.2 Add the bundled `eleven` theme: definition in `packages/client/src/theme/registry.ts` (following `libreChatTheme`), export via `themes/index.ts`, register `'eleven'` in `packages/data-provider/src/theme.ts` `bundledThemeNames`, and map it in `client/src/Providers/DeploymentTheme.tsx` — verify `isBundledThemeName('eleven')` is true and config validation accepts `interface.theme: eleven` via a new/exextended spec in `packages/data-provider` and `packages/client` tests
- [ ] 5.3 Document `eleven` in `librechat.example.yaml`'s `interface.theme` bundled-name list and refresh the `customWelcome` example — verify the example YAML still passes config validation tests (`npm run test:config`)
- [ ] 5.4 Run theme-workspace checks — `cd packages/client && npm run test:ci` and `npx tsc --noEmit -p packages/client/tsconfig.json` — and fix any token/type fallout in the same task
- [ ] 5.5 Visual QA in a running dev server: light, dark, both high-contrast modes show the orange accents; links stay blue; a deployment theme (`eleven`) overrides stored themes — verify by eyeballing each mode and checking computed CSS variable values

## 6. Repository surfaces

- [ ] 6.1 Rewrite `README.md` and `README.zh.md` for Eleven Chat: new header with the mark, fork description, upstream LibreChat attribution and MIT credit preserved, provider list including the "Elevenlabs" disambiguation note — verify all links in the README resolve and the attribution section is intact
- [ ] 6.2 Remove the upstream author's entries from `.github/FUNDING.yml` — verify the file parses as valid YAML with no `github:` funding targets

## 7. Governance records and integration checks

- [ ] 7.1 Create `CUSTOMIZATIONS.md` inventorying every modified upstream file from groups 1-6: feature tag `rebrand-eleven-chat`, marker locations, the six replaced binary assets, rationale, and per-file upstream-merge risk — verify every file listed exists in the final diff and no modified file is missing from the inventory
- [ ] 7.2 Create `CUSTOM_CHANGELOG.md` with the change entry (date, tag, rationale, new files, changed upstream files, config-default impact) — verify the entry matches the actual diff
- [ ] 7.3 Run the marker-pairing script from `AGENTS.md` §5 over the tree and confirm all `rebrand-eleven-chat` markers are balanced — verify script output reports no errors
- [ ] 7.4 Full verification pass: `npm run static-checks`, typechecks (`packages/api`, `packages/client`, `packages/data-provider`, `client`), `npm run test:api`, `npm run test:client`, `cd packages/client && npm run test:ci`, `npm run e2e:mock` — verify all pass or pre-existing failures are identified and reported
- [ ] 7.5 Residual-brand sweep: `grep -rn "LibreChat" client/src/locales client/src/components client/index.html client/vite.config.ts api/server packages/api/src packages/client/src packages/data-provider/src` returns only the sanctioned keep-list (technical identifiers, upstream-context comments, config-docs links) — verify the sweep output against the keep-list in the proposal
- [ ] 7.6 Self-review the full diff for unrelated changes, secrets, and contract drift; present the diff summary for review — verify the reviewed diff contains only rebrand-scoped changes

## Workflow follow-up

- After artifacts review, run `/opsx:apply` (or ask to apply this change) — implementation is a separate, explicitly requested workflow.
- Commits follow the `[brand]`/`[i18n]`/`[docs]` sequence in design.md D6, each with the AGENTS.md §4 rationale body; commit and push only with explicit authorization.
- After implementation and review, archive the change: `openspec archive rebrand-to-eleven-chat` and verify the archived specs land under `openspec/specs/{brand-identity,theme-accent}/`.

# Proposal

## Why

The application is a heavily customized LibreChat fork whose governance docs, repo name, and developer
guide already identify it as **Eleven-Chat**, but every user- and operator-facing surface still carries
the upstream **LibreChat** brand (browser title, PWA identity, auth-page logo, footer, About panel,
email sender names, fallback names in ~12 code paths, theme accents, icons, README, and community
files). The product presents a split identity: fork-branded governance, upstream-branded application.
Rebranding now aligns the product with the Eleven Chat identity and its "11 / ll" orange mark before
further divergence makes the rename more expensive.

## What Changes

- Replace the default product name with **Eleven Chat** everywhere it is user- or operator-visible:
  startup-config `appTitle` default, shared-link titles, 2FA/passkey display name, email `appName` and
  sender-name fallbacks, client document-title default, logo alt text, hardcoded `… | LibreChat`
  document titles, About diagnostics label, MCP host identity, generic chat footer, and `.env.example`
  defaults (`APP_TITLE`, `EMAIL_FROM_NAME` example, `HELP_AND_FAQ_URL`).
- Replace the brand mark and icon set with the Eleven Chat mark — the numeral **11** drawn as two
  straight parallel bars (also reading as "ll") in brand orange — across `logo.svg`, favicons (16/32),
  apple-touch icon (180), PWA icons (192, 512 maskable), replacing the upstream assets **in place at
  the same paths**.
- Rebrand the browser shell and PWA manifest: `<title>`, meta description, PWA `name`/`short_name`
  (`Eleven Chat`) and `theme_color` (brand orange) in `client/vite.config.ts`.
- Apply the brand orange as the default accent palette (light `#EA580C`/`#C2410C`, dark
  `#F97316`/`#FB923C`) via the existing semantic theme tokens, and register a new bundled deployment
  theme **`eleven`** so operators can pin the brand theme via `interface.theme` in `librechat.yaml`.
- Rebrand the word "LibreChat" inside locale **values** (English plus the translated locales that
  mention it), leaving translation **keys** untouched.
- Rewrite `README.md` / `README.zh.md` for the fork with upstream attribution preserved, and remove
  the upstream author's `FUNDING.yml` entries (assumption; see below).
- Retarget the birthday easter egg from Feb 11 (upstream's anniversary) to **Nov 11** as "Eleven Day"
  (assumption; see below).
- Create the governance records `CUSTOMIZATIONS.md` and `CUSTOM_CHANGELOG.md` (first customization
  inventory per `AGENTS.md` §15).

**Explicitly not changing** (confirmed decision): npm package names and imports
(`librechat-data-provider`, `@librechat/*`), the `librechat.yaml` config filename, `LIBRECHAT_*` env
var names, the `X-LibreChat-Generation-Protocol` wire header, Mongo database name, Redis key
prefixes, Docker volume names, the MIT `LICENSE` and upstream copyright, and upstream
configuration-docs links that remain accurate for the unchanged `librechat.yaml`. Also out of scope:
renaming the Helm chart, CI image names, and rewriting `CONTRIBUTING.md`/`SECURITY.md`/`CODE_OF_CONDUCT.md`
(deferred to a separate change).

No **BREAKING** API, schema, or data changes. One observable default changes: `GET /api/config`
returns `appTitle: "Eleven Chat"` when `APP_TITLE` is unset (operators can restore any prior name by
setting `APP_TITLE`). `interface.theme` gains one accepted value (`eleven`) — additive.

## Capabilities

### New Capabilities

- `brand-identity`: The product's default externally-presented identity — default app name across
  startup config, document titles, auth pages, emails, 2FA/passkey display, MCP host identity,
  footer/About copy, PWA manifest identity, the "11 / ll" mark and icon assets, locale values that
  name the product, and the `.env.example` defaults that seed operator configuration.
- `theme-accent`: The default accent palette (light/dark) expressed through the existing semantic
  theme tokens, and the bundled `eleven` deployment theme registered in the bundled-theme-name
  contract used by config validation and the client theme provider.

### Modified Capabilities

None — `openspec/specs/` is empty; there are no existing capability specs to modify.

## Impact

- **Client shell/assets**: `client/index.html` (title, description), `client/vite.config.ts` (PWA
  manifest), 6 binary/`svg` assets in `client/public/assets/` replaced in place.
- **Client app**: `client/src/utils/documentTitle.ts`, `components/Auth/AuthLayout.tsx`,
  `components/Chat/Footer.tsx`, `components/Nav/SettingsTabs/About/About.tsx`,
  `components/Agents/Marketplace.tsx`, `components/Insights/InsightsView.tsx`,
  `hooks/MCP/useAppBridge.ts`, locale `translation.json` values (~30 files).
- **Legacy API (`api/`)**: `server/routes/config.js` (appTitle/helpURL defaults, birthday date),
  `server/controllers/TwoFactorController.js`, `server/controllers/UserController.js`,
  `server/services/AuthService.js` (4 sites), `server/utils/sendEmail.js`.
- **Packages**: `packages/api/src/shared-links/config.ts` (shared-link title default);
  `packages/client/src/theme/themes/{default,dark}.ts` (accent tokens) and `theme/registry.ts`
  (bundled `eleven` theme); `packages/data-provider/src/theme.ts` (`bundledThemeNames`).
- **Config/examples**: `librechat.example.yaml` (bundled-theme list, welcome example), `.env.example`.
- **Repo/community**: `README.md`, `README.zh.md`, `.github/FUNDING.yml`.
- **Contracts**: no route/schema/status changes; startup-config default value only; `interface.theme`
  accepted-value list grows by `eleven`. No database, Redis, or dependency changes.
- **Tests**: default-asserting specs (`api/server/routes/__tests__/config.spec.js`) update in the
  same change; new coverage for the `eleven` bundled-theme name validation and accent token values.
- **Upstream merge risk**: every touched upstream file is a future conflict surface — mitigated by
  minimal one-line edits wrapped in `>>> CUSTOM:START [rebrand-eleven-chat] <<<` markers, in-place
  asset replacement, and the `CUSTOMIZATIONS.md` inventory (per `AGENTS.md` §5, §15, §18).

### Recorded assumptions (minor, reversible decisions)

1. **Orange palette** (user approved "propose one"): brand/dark accent `#F97316`, light accent
   `#EA580C` with hover `#C2410C`, dark hover `#FB923C`; WCAG ratios computed (white-on-`#EA580C`
   3.56:1 for UI components; `#F97316` on near-black 6.84:1). A strict-AA light-mode alternative
   (`#C2410C`/`#9A3412`) remains a swap-away.
2. **Tagline**: keep the existing `com_ui_latest_footer` ("Every AI for Everyone.") for now; only the
   footer's brand link target changes.
3. **Birthday easter egg**: retarget Feb 11 → Nov 11 ("Eleven Day") rather than removing it.
4. **FUNDING.yml**: remove the upstream author's funding entries; the repository owner can add their
   own later.
5. **Upstream config-docs links** (e.g. AdminSettingsDialog): keep — they document `librechat.yaml`,
   whose filename is unchanged and whose content remains accurate.

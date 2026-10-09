# Spec Delta

## Purpose

Defines the product's default externally-presented identity: the Eleven Chat name, the "11 / ll"
orange mark, icon assets, PWA identity, brand mentions in UI copy and emails, and the operator
overrides that must keep working.

## ADDED Requirements

### Requirement: Default product name is Eleven Chat
Whenever no operator override is set, the system SHALL present the product name as "Eleven Chat" in
every user- or operator-visible surface: the startup configuration `appTitle`, browser/document
titles, the authentication pages, transactional email sender and body app names, the 2FA/passkey
display name, shared-link page titles, the About/diagnostics label, and the host name reported to
MCP Apps.

#### Scenario: Startup config without APP_TITLE
- **WHEN** `GET /api/config` is called and `APP_TITLE` is not set
- **THEN** the response's `appTitle` is `Eleven Chat`

#### Scenario: Browser tab title after load
- **WHEN** an unauthenticated or authenticated page loads without `APP_TITLE` set
- **THEN** the browser tab title is "Eleven Chat" (and page-specific titles use "… | Eleven Chat")

#### Scenario: Emails use the default name
- **WHEN** a verification, reset, invite, or email-change message is sent without `APP_TITLE` or `EMAIL_FROM_NAME` set
- **THEN** the email body's app name and the sender display name are "Eleven Chat"

#### Scenario: Auth pages show the Eleven Chat name
- **WHEN** a user opens login, registration, or password-reset pages without `APP_TITLE` set
- **THEN** the logo alt text and any displayed product name reference "Eleven Chat", never "LibreChat"

### Requirement: Operator overrides preserve prior naming behavior
Operator configuration SHALL continue to control the product name: `APP_TITLE` (and
`EMAIL_FROM_NAME` for sender display) override the default everywhere it is presented, and
`CUSTOM_FOOTER` replaces the generic footer content entirely.

#### Scenario: APP_TITLE restores a prior name
- **WHEN** `APP_TITLE=LibreChat` is set
- **THEN** all name surfaces (startup config, titles, emails, 2FA/passkey, shared links) show "LibreChat"

#### Scenario: Custom footer replaces the generic footer
- **WHEN** `CUSTOM_FOOTER` is set
- **THEN** the chat footer renders the operator's footer and no default brand footer content

### Requirement: Brand mark is the Eleven "11 / ll" mark in orange
The brand mark SHALL be the numeral 11 drawn as two straight, parallel, equal-length vertical bars
(also readable as "ll") in brand orange `#F97316`, and SHALL replace the upstream mark in the auth
page logo and all icon assets (16/32 px favicons, 180 px apple-touch icon, 192 px and 512 px
maskable PWA icons). Icons SHALL remain legible at 16 px with a visible gap between the bars.

#### Scenario: Auth page logo
- **WHEN** an authentication page renders
- **THEN** the logo shown is the Eleven Chat mark plus "Eleven Chat" wordmark in brand orange

#### Scenario: Browser and install icons
- **WHEN** the app is opened in a browser tab or installed as a PWA / added to a home screen
- **THEN** the favicon, touch icon, and manifest icons depict the orange "11 / ll" mark, and the maskable icon keeps the mark inside the maskable safe zone

### Requirement: PWA manifest identity is Eleven Chat
The generated web app manifest SHALL identify the app as `Eleven Chat` (`name` and `short_name`)
with `theme_color` set to the brand orange `#F97316`.

#### Scenario: Manifest fields
- **WHEN** the client build produces the PWA manifest
- **THEN** `name` and `short_name` are "Eleven Chat" and `theme_color` is `#F97316`

### Requirement: User-visible copy contains no upstream brand name
Locale values and user-visible default copy SHALL name the product "Eleven Chat" and SHALL NOT
contain the upstream "LibreChat" brand name. Translation keys SHALL NOT change. Upstream
configuration documentation links that remain accurate for the unchanged `librechat.yaml` filename
are exempt.

#### Scenario: English copy
- **WHEN** an English UI string that previously named the product is displayed
- **THEN** it names "Eleven Chat"

#### Scenario: Translated locales
- **WHEN** a translated locale file contains a value that named the product
- **THEN** that value names "Eleven Chat" while its key is unchanged

### Requirement: Generic footer and About label carry the fork identity
The default chat footer SHALL present the product as "Eleven Chat" with its version and link to the
Eleven-Chat repository, and the About/diagnostics label SHALL read "Eleven Chat version:".

#### Scenario: Default footer
- **WHEN** the chat footer renders without `CUSTOM_FOOTER`
- **THEN** it links "Eleven Chat <version>" to the Eleven-Chat repository and shows the tagline

### Requirement: Example configuration defaults seed the Eleven Chat identity
The example environment file SHALL default `APP_TITLE` to `Eleven Chat` and default the help/FAQ
URL to the Eleven-Chat project, so fresh deployments inherit the new brand.

#### Scenario: Fresh .env from the example
- **WHEN** an operator copies the example environment file without editing branding fields
- **THEN** the deployed product presents itself as Eleven Chat with help links pointing at the Eleven-Chat project

### Requirement: Seasonal easter egg celebrates Eleven Day
The optional birthday display, when enabled, SHALL trigger on November 11 (11/11) rather than the
upstream anniversary date.

#### Scenario: Birthday icon date
- **WHEN** the client loads on November 11 and the birthday display is enabled
- **THEN** the birthday indicator is shown, and on any other date it is not

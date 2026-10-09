# Spec Delta

## Purpose

Defines the brand-orange default accent palette across light and dark themes and the bundled
`eleven` deployment theme, expressed through the existing semantic theme tokens, with accessibility
and operator-override compatibility constraints.

## ADDED Requirements

### Requirement: Light theme default accent is brand orange
The default light theme's accent (accent-primary and its hover state) SHALL be the brand orange
family: accent-primary `#EA580C` with hover `#C2410C`, applied exclusively through the existing
semantic theme tokens.

#### Scenario: Fresh deployment in light mode
- **WHEN** a user with no stored theme preference opens the app in light mode
- **THEN** accent surfaces (primary buttons, highlights, active accents) render `#EA580C` and their hover state `#C2410C`

### Requirement: Dark theme default accent is vivid brand orange
The default dark theme's accent SHALL be accent-primary `#F97316` with hover `#FB923C`, applied
through the same semantic theme tokens.

#### Scenario: Fresh deployment in dark mode
- **WHEN** a user with no stored theme preference opens the app in dark mode
- **THEN** accent surfaces render `#F97316` and their hover state `#FB923C`

### Requirement: Accent palette meets accessibility thresholds
The default accent palette SHALL meet WCAG 2.1 contrast requirements: light-mode accent with white
foreground text at least 3:1 (UI components and large text), and dark-mode accent used as text or
fill on near-black surfaces at least 4.5:1.

#### Scenario: Light-mode accent button
- **WHEN** a primary button renders white text on the light-mode accent `#EA580C`
- **THEN** the contrast ratio is at least 3:1

#### Scenario: Dark-mode accent on dark surface
- **WHEN** the dark-mode accent `#F97316` renders as text or fill on a near-black surface
- **THEN** the contrast ratio is at least 4.5:1

### Requirement: Bundled eleven deployment theme
The theme-name contract SHALL accept `eleven` as a bundled deployment theme. When `interface.theme`
is set to `eleven` in the configuration file, the deployment theme SHALL apply the brand palette to
every user in both light and dark modes, overriding stored themes per the existing
deployment-theme rules, and configuration validation SHALL accept the value without warnings.

#### Scenario: Valid configuration applies the theme
- **WHEN** the configuration file sets `interface.theme: eleven` and a user loads the app
- **THEN** the brand palette is applied in the user's current light or dark mode

#### Scenario: Unknown theme names are still rejected
- **WHEN** the configuration file sets `interface.theme` to a name that is not a bundled theme
- **THEN** configuration validation reports the unknown theme and the accepted name list, as before

### Requirement: Non-accent token families and accessibility themes unchanged
The rebrand SHALL NOT alter link colors, ring/focus colors, surface colors, or the high-contrast
light/dark themes. Deployment theme selection, stored user themes, and per-token environment
overrides (`REACT_APP_THEME_*`) SHALL keep outranking the new defaults.

#### Scenario: Links remain blue
- **WHEN** a link renders in either default theme
- **THEN** its color is unchanged from before the rebrand

#### Scenario: High-contrast modes unaffected
- **WHEN** a user enables either high-contrast mode
- **THEN** colors and contrast are identical to before the rebrand

#### Scenario: Environment token override still wins
- **WHEN** a deployment sets a `REACT_APP_THEME_*` override for the accent token
- **THEN** the override value renders instead of the brand default

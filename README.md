# simple-multi-lang

This CrafterCMS site demonstrates a **path-based multilingual** setup: each locale lives under its own folder in the content tree, while Crafter Engine **content targeting** maps those folders to locale targets for delivery and inheritance. Authors can use the **Smart Copy** plugin to copy pages between locales and manage translation relationships from the form and preview UI.

---

## Information architecture

### Global home vs locale sites

| Location | Role |
|----------|------|
| `/site/website/index.xml` | **Global home** — not tied to a locale segment in the URL. Documented in content as a possible entry point for redirects (browser locale, default locale, etc.). |
| `/site/website/{locale}/` | **Per-locale site roots** — e.g. `en`, `es`, `ja`, `zh`. Each locale has its own `index.xml` and child pages (e.g. `child-page/`). |

**URL shape (typical):**

- `/` → global home (`/site/website/index.xml`)
- `/en/`, `/es/`, … → locale home pages (e.g. `/site/website/en/index.xml`)
- `/en/child-page/` → locale-scoped pages under the same locale prefix

Locale codes in the **first folder** under `/site/website/` (after the site root) are what the page content-type lifecycle hook and Smart Copy treat as the page locale.

### Engine targeting (delivery)

Configured in `config/engine/site-config.xml` under `<targeting>`:

| Setting | Value | Purpose |
|---------|--------|---------|
| `enabled` | `true` | Turns on targeting for this site. |
| `rootFolders` | `/site/website` | Only content under this tree participates in targeting. |
| `excludePatterns` | `/site/website/index\.xml` | Excludes the **global** home from targeting so it is not treated as a locale target. |
| `availableTargetIds` | `en`, `ja`, `es`, `fr`, `zh` | Declares which locale targets exist for the engine. |
| `fallbackTargetId` | `en` | Used when a target cannot be resolved. |
| `mergeFolders` | `true` | **Merged inheritance**: child paths can inherit from parent locales/folders per Crafter’s targeting rules (see CrafterCMS docs on content targeting). |
| `redirectToTargetedUrl` | `false` | Does not force redirects to a “canonical” targeted URL when resolving content. |

> **Note:** `availableTargetIds` includes `fr`, but the sample content in this repo may not include a `/site/website/fr/` tree yet. Add folders and pages as needed to match your supported locales.

---

## Configuration files (by area)

### Crafter Engine (`config/engine/`)

- **`site-config.xml`** — Targeting block above (multi-language delivery and inheritance). Also includes template `allowed` patterns for static assets.

### Crafter Studio (`config/studio/`)

- **`site-config.xml`** — Studio UI locale (`<locale>`): optional `localeCode` and `dateTimeFormatOptions` for how dates/times appear in Studio (separate from site content locales).
- **`translation-config.xml`** — Optional **Studio translation** integration (locale lists, default locale, resolvers such as URL/cookie/header/Groovy). In this project the meaningful blocks are largely **commented out**; uncomment and configure if you use Crafter’s translation services with custom resolvers (e.g. a `scripts/locale.groovy` script is referenced in the sample).
- **`targeting/targeting-config.xml`** — **Preview / personalization testing**: fake user properties injected into Studio preview (not the same as engine `targeting` for locales). Used for validating personalized experiences.

### Plugin registry

- **`config/plugins/registry.yaml`** — Currently empty (`plugins:` with no entries). Site-specific plugin bits for Smart Copy are **bundled under** `config/studio/static-assets/plugins/` (see below), not declared here.

---

## Plugins: Smart Copy (`org.rd.plugin.smartcopy`)

This site includes **Smart Copy** assets for translation-oriented workflows:

| Area | Location / role |
|------|------------------|
| **Static plugin assets** | `config/studio/static-assets/plugins/org/rd/plugin/smartcopy/` — apps (e.g. `apps/smartcopy/index.js`), form control (`control/custom-locale/main.js`). |
| **Plugin config** | `config/plugins/org/rd/plugin/smartcopy/config.xml` — placeholder (`<configuration/>`); extend if your plugin expects site-level settings. |
| **Studio UI** | `config/studio/ui.xml` — Registers Smart Copy in the **preview toolbar** (`org.rd.plugin.smartcopy.openSmartCopyToolbarButton`) with title “Copy for Translation”. A Tools Panel entry for the same plugin exists in the file but is **commented out**; uncomment if you want a sidebar button. |

**What Smart Copy does in this project (high level):**

- Provides UI to copy content between locales and drive a **translation workflow** (manual, assisted, or automated depending on your wider setup).
- Supplies a **custom form control** (`custom-locale`) bound to the page field `localeSourceId_s`, which works together with `localeCode_s` and `sourceLocaleCode_s` (see content types below).

The root page copy in `body_html` describes the intended pattern: global home + base language + Smart Copy for scoping pages into translation by copying across locales.

---

## Content types: multilingual behavior

### Page type: `/page/entry`

Defined under `config/studio/content-types/page/entry/`.

**Fields relevant to locales:**

| Field | Type | Role |
|-------|------|------|
| `localeSourceId_s` | `custom-locale` (Smart Copy control) | Stable identifier linking translated/copied pages; the control writes UUIDs and coordinates with the plugin. |
| `localeCode_s` | `input` (read-only in form) | Current locale code inferred from **content path** (e.g. `en` for `/site/website/en/...`). |
| `sourceLocaleCode_s` | `input` (read-only in form) | Locale the content was derived from when copying within the same locale tree. |

Standard page fields include `title_t`, `body_html` (RTE), `placeInNav`, `disabled`, etc. Display template: `/templates/web/entry.ftl`.

### Content-type controller: `controller.groovy`

`config/studio/content-types/page/entry/controller.groovy` runs on **lifecycle** events:

- **`UPDATE`** — If the page lives under a locale path (`/site/website/{locale}/...`) and `localeCode_s` is empty, it sets `localeCode_s` from the path.
- **`COPY` / `DUPLICATE`** — Keeps locale metadata consistent when copying:
  - If the **stored** locale differs from the **destination path** locale, updates `localeCode_s` and adjusts **shared component** references in node-selector fields when a matching component exists under the new locale path.
  - If the locale matches the path, generates a **new** `localeSourceId_s` and sets `sourceLocaleCode_s` to the current path locale (new “translation unit”).

**Path locale convention:** The first segment after `/site/website/` is treated as the locale code (e.g. `en` for `/site/website/en/index.xml`). The global home at `/site/website/index.xml` has no such segment, so those fields are typically empty there.

### Level descriptor: `/component/level-descriptor`

`site/website/crafter-level-descriptor.level.xml` provides section defaults and inheritance for the site tree. The form is minimal (standard level descriptor); it does not add extra locale-specific fields in this project.

---

## Templates

- `templates/web/entry.ftl` — Renders `title_t` and `body_html` with Experience Builder macros (`@crafter.*`). The HTML `lang` attribute is currently fixed to `en`; adjust or drive from targeting/locale if you need per-locale document language in markup.

---

## Operational notes

1. **Commit content** after changing content types or forms so Studio and indexing stay aligned with your model.
2. **Keep `availableTargetIds` and real folders in sync** — add or remove targets in `config/engine/site-config.xml` when you add/remove locale roots under `/site/website/`.
3. **Smart Copy** depends on the bundled plugin files and `ui.xml` registration; if the toolbar button is missing, confirm `config/studio/ui.xml` still references `org.rd.plugin.smartcopy.openSmartCopyToolbarButton`.

For CrafterCMS product documentation on content targeting, inheritance, and Studio configuration, see [CrafterCMS documentation](https://docs.craftercms.org/).

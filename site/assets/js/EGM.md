# js

## Overview

Browser scripts of the Hugo documentation site, loaded on every page:
glossary hover cards, code tabs with click-to-copy toolbars and the home
page animation, and Docsy-derived offline search. Plain JS, no build step
or test runner configured in this repo (`site/package.json` holds site
tooling only).

## Interface

- `glossary.js` (IIFE) - decorates the first mention of each glossary term with a hover card. Exports for tests under Node (`module.exports`): `glossary.js:termPattern` (word-bounded regex with optional plural `s`), `glossary.js:sortTerms` (longest first), `glossary.js:findMatch`, `glossary.js:pickMention` (exact casing first, then any; a term inside inline code counts only when it is the whole span). Data comes from `#as-glossary-meta` `dataURL`; content rendered from `entry.html` (`glossary.js:buildCard`).
- `glossary.js:SKIP` selector - text inside `pre, a, h1-h6, button, table, .as-status-banner, .as-glossary-wrap` is never decorated; only `p, li` text is eligible (`glossary.js:eligibleTextNodes`).
- `click-to-copy.js:initCodeTabs` - converts `.code-tabs > .code-tab` elements into Bootstrap tab groups with ARIA roles and per-tab persistence keys.
- `click-to-copy.js:initCodeBlocks` - adds a toolbar (language badge from `click-to-copy.js:formatLangLabel`/`LANG_LABELS` plus a copy button) to `.highlight > pre` blocks; copies `.cli-step` contents when present.
- `click-to-copy.js:initHomeRedesign` - home page term-card tabs, copy buttons (`data-copy-target`, `data-copy-active-term`), the lazy YouTube demo embed (`#home-demo-video-card`), Cmd/Ctrl-K search focus, and `click-to-copy.js:initHeroVizAnimation` (the actor suspend/resume dot-grid animation driven by a 150ms interval; guarded by `window.__asHeroVizStarted`).
- `click-to-copy.js:initDocsInnerRedesign` - Copy-Page-as-Markdown buttons reading `#as-page-markdown`.
- `offline-search.js` (jQuery IIFE) - Docsy offline search extended with contextual snippets (`offline-search.js:buildSnippet` picks the first literal or token match with a 70/130-character radius) and `offline-search.js:queryTokens` via lunr.
- Subdirectories: none.

## Invariants

- A glossary card is placed only on the first mention, longest terms claim text before their prefixes, and pages never get cards on the glossary page itself (`glossary.js:initGlossaryCards`).
- Hover-card inner HTML comes from the site's own generated glossary data, which Hugo renders from trusted repo content (`glossary.js:buildCard` comment).
- The hero animation runs at most once per page load (`window.__asHeroVizStarted`).

## Side Effects

- Mutates the DOM (tabs, toolbars, cards, animation), fetches the glossary JSON, writes to the clipboard on user action, and starts timers/intervals (`glossary.js`, `click-to-copy.js`, `offline-search.js`).

## Verification

- No test runner configured for site JS in this repo; the logic-bearing functions are imported by `site/testdata/js/glossary.test.js` in fixtures.

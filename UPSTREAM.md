# Upstream Re-integration Policy

This repo is a long-lived fork of [MrPrimate/tokenizer](https://github.com/MrPrimate/tokenizer)
(`upstream` remote). These are the rules that keep merging upstream cheap forever.
They are not suggestions; commits that violate them create permanent merge debt.

## The one rule that matters

**Forge code is additive.** New behavior lives in Forge-owned files (`src/forge/`,
`templates/forge/`, `css/forge/`, `vendor/fflate.js`, `FORGE.md`, this file).
Upstream-owned files may only be touched at the registered touch points below, and
every edit there must be a *pure insertion* at a low-collision anchor (end of a
switch, end of an object literal, new keys in JSON). Never refactor, reformat, or
"improve" upstream code in this fork — if upstream code needs a fix, PR it to
MrPrimate instead (he merges outside PRs; translation PRs land within days).

## Registered touch points — THE authoritative table

FORGE.md references this table; it is not duplicated anywhere. Adding a touch point
requires updating this table in the same commit.

| File | What we changed | Anchor discipline |
|---|---|---|
| `templates/tokenizer.hbs` | 2 ✨ menu buttons inside `{{#if forgeEnabled}}` | appended after last button in each pane's `.content` row (the `tokenVariantsEnabled` gating precedent) |
| `src/tokenizer/Tokenizer.js` | 1 import; `forgeEnabled` in `_prepareContext`; `case "ai-generate"` (re-checks the gate, delegates to ForgeDialog) | import at end of import block; context key at end of the context object; case at end of `menuButton` switch, before the `// no default` comment |
| `src/hooks.js` | `registerForgeSettings()` call; `forgeActor`/`autoTokenAI` API entries | end of `init()` body; end of the API object literal inside `exposeAPI()` (hooks.js:435-448) — two separate anchors |
| `lang/en.json` | `vtta-tokenizer.label.AIGenerate`, `vtta-tokenizer.forge.*` | new keys only. **en.json ONLY** — never edit the other five lang files; take upstream's version wholesale on every merge (their translation PRs churn those files constantly). Foundry falls back to English for missing keys; if Forge strings should be localized someday, PR the translations upstream. |
| `module-template.json` | `styles` entry for `css/forge/forge.css`; **`manifest` URL → our releases** (build-module-json.js does NOT rewrite it — verified: the script only overwrites `version` and `download`) | key-value edits only |
| `.github/workflows/build-module-json.js` | hardcoded `download` URL `github.com/mrprimate/tokenizer/...` → our repo | 1-line URL change |
| `.github/workflows/build.yml` | node matrix `[14.x]` → `[20.x]`; **delete** the Discord-webhook and FoundryVTT auto-publish steps (never re-add auto-publish: pushing a fork over the official package listing is not ours to do); artifact URLs | delete/replace whole steps, never edit inside a kept upstream step |
| `package.json` | `version` = `X.Y.Z-forge.N` | version field only |

## Branch & remote model

- `master` — our line. Diverged from upstream at v5.0.3.
- `upstream` remote — MrPrimate/tokenizer. **Sync from release tags, never tip of
  master**: upstream ships in 1–3-day bursts (median 3 days when active, with
  multi-month dormancies); tags are stable points, tip is mid-burst.
- `origin` — our GitHub repo (`gh repo create` before first push).

## Versioning

`<upstream-version>-forge.<n>` (e.g. `5.0.3-forge.1`); after merging upstream
`5.1.0`, next release is `5.1.0-forge.1`. Our `module.json` `manifest` points at our
releases, so Foundry's update checker only compares our own version strings.
**Phase 0 gate**: console-verify `foundry.utils.isNewerVersion("5.0.3-forge.2",
"5.0.3-forge.1")` and `("5.0.3-forge.1", "5.0.3")` on a v13 instance and record the
results here; if the suffix misorders, switch to `5.0.301`-style numeric versions
*before* anything is deployed.

CI quirk to remember: `build.yml` triggers only on master pushes that touch
`package.json` — a code-only push publishes nothing. Every release is a version bump.

## Sync cadence

- **Check monthly** (most checks are no-ops during dormancy), and **always
  before/after a Foundry core upgrade on the fleet** (upstream's bursts track Foundry
  releases; v5.0.0 was the ApplicationV2 migration for v13).
- Don't chase patch releases mid-burst; wait for the burst to settle (5.0.1/5.0.2/
  5.0.3 shipped in 5 days), then merge the last tag of the burst.

## Merge runbook

```bash
git fetch upstream --tags
git checkout -b sync/v5.1.0 master
git merge v5.1.0            # conflicts expected ONLY at registered touch points
npm ci && npm run lint && npm run build
# smoke: rsync build into solo's Data/modules/vtta-tokenizer, open Tokenizer,
#   generate one image, save one token, F5 mid-grid and confirm scratch persistence
git checkout master && git merge --no-ff sync/v5.1.0
# bump package.json → 5.1.0-forge.1, commit, tag, push, cut release
```

**Rehearsal (Phase 0, before the policy is needed in anger):** branch, rebase the
forge commits onto tag `v5.0.0` (the ApplicationV2 migration — the most disruptive
recent release), then merge forward 5.0.1 → 5.0.3. If the anchors survive that, the
policy is validated; if not, fix the anchors now while the diff is small.

Conflict triage:
- At a registered touch point → re-apply our insertion at the same anchor; if
  upstream moved/renamed the anchor, update the table in the same commit.
- Anywhere else → we broke the one rule; move the offending logic into `src/forge/`,
  don't hand-resolve and move on.
- Upstream ships an overlapping feature (their own AI integration, or an image-source
  hook) → stop; evaluate adopting theirs and shrinking ours. Design decision, not a
  merge decision.

## Divergence watchlist

Checked during each sync — things upstream could change that break Forge *without* a
merge conflict:

- `View.addImageLayer` signature / layer `type` semantics
- `menuButton` dispatch (`data-action`/`data-type` scheme) and `_prepareContext` shape
- The save path: `formHandler` → `View.get("blob")` → `Utils.uploadToFoundry`
  (Forge's scratch-dir writes and the tainted-canvas rule both depend on it)
- `_initToken` / `autoToken()` call shapes (batch mode reuses them)
- `Utils.download` cache-bust behavior (we bypass it; if upstream fixes `data:`-URI
  handling our own loader still works — no action either way)
- Settings scope conventions (`world`/`player`/`client`)
- Release zip contents list in `build.yml` (must keep sweeping `templates/`, `css/`,
  `lang/`, `vendor/` wholesale, or Forge assets fall out of the artifact)
- `module-template.json` compatibility floor (currently v13 — a bump strands fleet
  instances; check FORGE.md's fleet table before merging such a change)

## Exit strategy: the `registerImageSource()` PR

The honest reason this is a fork and not a companion module: **upstream has no
extension point yet** — but it visibly wants one. `Tokenizer.js` already hardcodes a
`case "tokenVariants"` that calls another module's art picker via a callback
returning an image source, gated by a `tokenVariantsEnabled` template flag. A small
PR generalizing exactly that — `Tokenizer.registerImageSource({id, label, icon,
onSelect})` — would let upstream drop its own hardcoded special-case and would let
Forge collapse into a standalone companion module with zero merge obligations.
MrPrimate merges outside PRs quickly.

Plan accordingly: keep `src/forge/` structured so only the entry-point wiring is
fork-specific (everything else already imports nothing but Forge files and public
Foundry APIs). If/when the PR lands, migration is: new thin companion module
manifest + move the tree + delete the fork. Opening the PR is recommended at Phase 0
but is thebwt's call — it carries his GitHub name.

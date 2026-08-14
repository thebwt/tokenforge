# Upstream Re-integration Policy

This repo is a long-lived fork of [MrPrimate/tokenizer](https://github.com/MrPrimate/tokenizer)
(`upstream` remote). These are the rules that keep merging upstream cheap forever.
They are not suggestions; PRs that violate them create permanent merge debt.

## The one rule that matters

**Forge code is additive.** New behavior lives in Forge-owned files
(`src/forge/`, `templates/forge/`, `css/forge/`, `FORGE.md`, this file). Upstream-owned
files may only be touched at the registered touch points below, and every edit there
must be a *pure insertion* at a low-collision anchor (end of a switch, end of an object
literal, new keys in JSON). Never refactor, reformat, or "improve" upstream code in
this fork — if upstream code needs a fix, PR it to MrPrimate instead (he merges
outside PRs; translation PRs land within days).

## Registered touch points (keep this list exact)

| File | What we changed | Anchor discipline |
|---|---|---|
| `templates/tokenizer.hbs` | 2 ✨ menu buttons | appended after last button in each pane's `.content` row |
| `src/tokenizer/Tokenizer.js` | 1 import; `case "ai-generate"` | import at end of import block; case at end of `menuButton` switch, before `// no default` |
| `src/hooks.js` | settings init call; API object entries | end of `init()` body; end of API object literal |
| `lang/*.json` | `vtta-tokenizer.label.AIGenerate`, `vtta-tokenizer.forge.*` | new keys only, never edit existing keys |
| `module-template.json` | styles entry; manifest/download URLs → this repo | key-value edits only |
| `.github/workflows/build.yml` | node 20; remove Discord webhook + FoundryVTT auto-publish steps; artifact URLs | delete/replace whole steps, don't edit inside upstream steps |
| `.github/workflows/build-module-json.js` | download URL → this repo's releases | 1-line URL change |
| `package.json` | version `X.Y.Z-forge.N` | version field only |

Adding a touch point requires updating this table in the same commit.

## Branch & remote model

- `master` — our line. Diverged from upstream at v5.0.3.
- `upstream` remote — MrPrimate/tokenizer. **Sync from release tags, never from tip
  of master**: upstream ships in 1–3-day bursts (median 3 days when active, with
  multi-month dormancies); tags are the stable points, tip is mid-burst.
- `origin` — our GitHub repo (create with `gh repo create` before first push).

## Versioning

`<upstream-version>-forge.<n>` — e.g. `5.0.3-forge.1`, bump `n` for Forge releases on
the same upstream base; after merging upstream `5.1.0`, next release is `5.1.0-forge.1`.
Our `module.json` manifest URL points at **our** releases, so Foundry's update checker
only ever compares our own version strings — no races with upstream's registry entry
(the FoundryVTT auto-publish step is removed in our CI; never re-add it: publishing a
fork over the official package listing is not ours to do).

## Sync cadence

- **Check monthly** (upstream has long dormancies — most checks are no-ops), and
  **always before/after a Foundry core upgrade on the fleet** (upstream's bursts track
  Foundry releases; v5.0.0 was the ApplicationV2 migration for v13).
- Don't chase every patch release mid-burst; wait for the burst to settle (upstream
  shipped 5.0.1/5.0.2/5.0.3 in 5 days), then merge the last tag of the burst.

## Merge runbook

```bash
git fetch upstream --tags
git checkout -b sync/v5.1.0 master
git merge v5.1.0            # expect conflicts ONLY at registered touch points
npm ci && npm run lint && npm run build
# smoke test: rsync build into solo's Data/modules/vtta-tokenizer, open Tokenizer,
#   generate one image, save one token
git checkout master && git merge --no-ff sync/v5.1.0
# bump package.json → 5.1.0-forge.1, commit, tag, push, cut release
```

Conflict triage:
- Conflict at a registered touch point → re-apply our insertion at the same anchor;
  if upstream moved/renamed the anchor (e.g. menuButton refactor), update the touch
  point table.
- Conflict anywhere else → we broke the one rule; fix the offending commit
  (move the logic into `src/forge/`), don't hand-resolve and move on.
- Upstream added a feature overlapping Forge (e.g. their own AI integration) →
  stop, evaluate adopting theirs and shrinking ours; that's a design decision,
  not a merge decision.

## Divergence watchlist

Checked during each sync (these are the things upstream could change that silently
break Forge without a merge conflict):

- `View.addImageLayer` signature / layer `type` semantics
- `menuButton` dispatch mechanism (`data-action`/`data-type` scheme)
- `Utils.download` cache-bust behavior (we deliberately bypass it — if upstream fixes
  the `data:`-URI handling, our own loader still works; no action needed)
- Settings scope conventions (`world`/`player`/`client`)
- Release zip contents list in `build.yml` (must keep sweeping `templates/`, `css/`,
  `lang/` wholesale, or Forge assets fall out of the artifact)
- `module-template.json` compatibility floor (currently v13 — a bump strands older
  fleet instances; check the fleet table in FORGE.md before merging such a change)

## Why not a companion module?

Considered and rejected for now: a separate module reaching into Tokenizer's internals
(the token-variants pattern) would avoid fork maintenance but couples to *unexported*
internals (dialog instance, view lifecycle) that upstream changes more freely than
its templates and switch anchors. The fork's conflict surface (~40 insertion lines at
stable anchors) is smaller than the API surface a companion would depend on. Revisit
if upstream ever exposes an image-source plugin API — and consider PRing exactly that
upstream (an `Tokenizer.registerImageSource()` hook) as the long-term exit strategy.

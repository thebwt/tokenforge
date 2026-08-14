# Tokenforge

A drop-in fork of [MrPrimate/tokenizer](https://github.com/MrPrimate/tokenizer) that adds
AI image generation to the Tokenizer editor: **NovelAI V4.5** for images, **Claude Haiku**
for compiling actor JSON into NovelAI prompts. Module id stays `vtta-tokenizer` — this
replaces the stock module in place; all upstream features and integrations keep working.

Companion doc: [UPSTREAM.md](./UPSTREAM.md) — re-integration policy and the **single
authoritative touch-point table**. Read it before touching any upstream-owned file.

Plan status: design reviewed (3-agent adversarial pass, 2026-08-14); all blockers
folded in. Recon evidence: session scratchpad `novelai-research/` (SDK sources,
captured browser requests, live CORS probes).

---

## 1. What it does

From any Tokenizer window (avatar or token pane), a new ✨ button — **GM-gated** —
opens the **Forge dialog**:

```
┌─ Forge: "Kessra Vane" ──────────────────────────────────┐
│ Guide (saved to actor)                                  │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ travel-worn noble, always with her raven            │ │
│ └─────────────────────────────────────────────────────┘ │
│ This generation: [wounded, torn cloak______________]    │
│ ▸ Compiled prompts (portrait + token, editable, cached) │
│                                                         │
│  [img] [⟳ 2/4] [ · ] [ · ]   ← tiles stream in, Stop ⏹ │
│                                                         │
│ [Re-roll 🎲] [Revise ✨] [Anchor ⚓] [Use →]             │
└─────────────────────────────────────────────────────────┘
```

- **Compile**: Haiku reads a pruned actor JSON + the guide layers and emits subject
  prompts (portrait + token variants in one call, both cached). GM edits before firing.
- **Re-roll**: same compiled prompt, new base seed. No LLM call. Free on Opus.
- **Revise**: guide/hint changed → Haiku recompiles minimally against the previous
  prompt. **Honest semantics: a prompt change re-composes the subject** — NAI seeds fix
  initial noise, not identity. Revise gives continuity of *description*, not of face.
  Identity continuity is the anchor's job:
- **Anchor ⚓**: encode the accepted portrait via `/ai/encode-vibe` (2 Anlas,
  consent-gated) and store the character vibe on the actor. Every later generation for
  this actor — token variant, "now scarred", next campaign arc — passes the character
  vibe *alongside* the campaign style vibes (V4.5 accepts up to 16). This is the
  mechanism that keeps the same face on portrait, token, and regenerations.
- **Use**: selected image lands via the existing
  `view.addImageLayer(img, {type:"image"})` pipeline — frames, masks, dynamic rings,
  save-to-actor all inherited from upstream.

Every generated PNG is written to disk **at generation time**
(`Utils.uploadToFoundry` → `forge-scratch-directory`, named `<actorId>.<seed>.png`) —
a refresh or misclick never discards a 60-second grid, rejected tiles are a
folder-cleanup question, and the *raw* NAI original always exists on disk (the saved
token is a canvas re-encode; see §3 resolution notes).

### Guide layers (override, don't swap)

| Layer | Lives in | Example |
|---|---|---|
| Campaign guide | world setting | "low fantasy, everyone grimy, no clean heroes" |
| Actor guide | `flags.vtta-tokenizer.forge.guide` | "always drawn with her raven" |
| Generation hint | ephemeral (dialog field) | "wounded after the ambush" |

Haiku compiles top-down; lower layers win conflicts. **Style and quality tags never
come from the LLM**: the negative prompt is a hard-coded quality floor +
`forge-negative-suffix` world setting; style is vibe refs + `forge-style-suffix`,
appended deterministically after compilation. The LLM owns subject description only
(plus an optional small `subjectExclusions` list, concatenated client-side). The
compiler depicts *visible appearance only* — identity spoilers stay out of the art
unless the guide explicitly says otherwise.

---

## 2. Verified constraints (recon + adversarial review, 2026-08-14)

### CORS / key custody — no relay

- `image.novelai.net` answers preflights with `Access-Control-Allow-Origin: *`
  (live-probed on `/ai/generate-image`, `/ai/encode-vibe`, `/suggest-tags`).
  Browser-direct calls work.
- `api.novelai.net` (login, subscription/Anlas balance, `/ai/upscale`) has **no CORS
  headers** on preflight — unreachable from the browser. Consequence, stated plainly:
  **the client cannot observe Anlas spend.** Budget enforcement must be *preventive*
  (see free-envelope guard below); the GM checks the balance on novelai.net manually.
- `api.anthropic.com` allows browser calls with
  `anthropic-dangerous-direct-browser-access: true` (live-probed).
- Keys live in **`scope: "client"` settings**: never in the world DB, never synced to
  players. Honest caveats: client scope is per **browser profile**, not per Foundry
  user — a player logging in from the GM's browser profile could read them, and any
  module in the same page can call `game.settings.get`. Key entry uses a small custom
  menu with `<input type="password">` (not a `config: true` String, which renders
  plaintext on screenshares). Don't enter keys on a shared browser profile.

### NovelAI contract (V4.5)

- `POST https://image.novelai.net/ai/generate-image`,
  `Authorization: Bearer pst-…` (persistent token: novelai.net → Account → "Get
  Persistent API Token"; shown once; regenerating invalidates the old one — that
  regenerate is also the rotation runbook).
- Model: `nai-diffusion-4-5-full` (alt: `-curated`). Request: `{input, model,
  action:"generate", parameters:{...}}` with `params_version: 3`, sampler
  `k_euler_ancestral`, `noise_schedule: "karras"`, scale 5, steps 23 defaults.
  Character prompting requires **both** legacy `characterPrompts[]` **and**
  `v4_prompt.caption.{base_caption, char_captions[]}` (mirrored in
  `v4_negative_prompt`) — NAI's frontend sends both. Forge builds the
  single-character arrays client-side from the compiled subject prompt.
- Dimensions: multiples of 64, free envelope is an **area** cap (≤ 1,048,576 px²) —
  832×1216 portrait = 1,011,712 px² should be free; **confirm in the Phase 0.5 spike**
  before relying on it. Phase 1 generates 1024×1024 (square) because upstream's
  `Layer.fromImage` letterboxes/crops non-square sources into a square canvas;
  non-square presets need Forge's own crop decision first.
- Response: **ZIP** containing one PNG per sample — but *only* on success. Branch on
  status and `content-type` before unzipping: `application/json` bodies are error
  payloads (surface the message); `text/html` is a Cloudflare challenge ("open
  novelai.net in a tab, then retry"); 401 → key-entry UI; 402 → left the free tier;
  429 → honor `Retry-After` (else 5 s), retry once, then mark the slot failed and let
  remaining slots continue. Unzip with vendored `fflate` at top-level
  `vendor/fflate.js` (the `vendor/MagicWand.js` precedent — **not** under `src/`,
  which is linted and would break the release-gating lint job).
- **Opus free envelope**: tier Opus + `steps ≤ 28` + area ≤ 1 MP + first sample of the
  request. Therefore: `n_samples: 1` always, 4-up grid = 4 sequential requests. No
  fixed inter-request gap (serialization is the politeness); backoff on 429 is the
  real courtesy. Tiles stream into the grid as they land; Stop button aborts the queue.
- **`assertFreeEnvelope(params)`** — a pure function in `NovelAIClient.js`, unit-tested,
  that throws before `fetch` if `steps > 28 || width*height > 1048576 ||
  n_samples !== 1 || action !== "generate"`. Every Anlas-spending path (encode-vibe,
  img2img, augment) is additionally gated behind a `forge-anlas-consent` client
  setting, default **off**. This is the only guard money gets.
- img2img may fall outside the free envelope (cost-function sources disagree);
  resolved empirically in Phase 0.5. Opus includes 10,000 Anlas/month, so consented
  occasional spend (2-Anlas character anchors) is cheap: ~5,000 anchors/month.
- Vibe Transfer (V4.5): plural fields `reference_image_multiple[]`,
  `reference_information_extracted_multiple[]`, `reference_strength_multiple[]`.
  Campaign style = encode reference image(s) once, store in `forge-style-vibes`,
  reuse forever (>4 vibes per request costs 2 Anlas each — style + character is 2-5,
  fine). Director Tools (`/ai/augment-image` incl. `bg-removal`) are never free;
  token cutouts come from upstream's frame/mask pipeline.
- ToS: no explicit clause on third-party API use located; basis is the officially
  documented persistent-token feature + years of tolerated ecosystem tools
  (SillyTavern, both SDKs). Personal use, personal subscription.
- **Canvas-taint rule**: AI image bytes reach the canvas only as `blob:`/`data:` URIs
  derived from the unzipped bytes — never assign a remote URL to a Forge image
  element, or `View.get("blob")` throws `SecurityError` on save. Revoke every object
  URL in `ForgeDialog._onClose` and on grid-slot replacement. Never route AI results
  through `Utils.download()` (`src/libs/Utils.js:148`) — its `?timestamp` cache-bust
  corrupts `data:` URIs; use the `Utils.extractImage`-style from-scratch
  `new Image()` loader in `NovelAIClient.js`.

### Anthropic contract

- `POST https://api.anthropic.com/v1/messages`; headers `x-api-key`,
  `anthropic-version: 2023-06-01`, `anthropic-dangerous-direct-browser-access: true`.
- Model: **`claude-haiku-4-5`** by default (user's call; `forge-compiler-model` world
  setting exists if compile quality ever wants `claude-sonnet-5` — at interactive
  volume the cost difference is noise). `max_tokens: 1024` (required param;
  too-small truncates structured output mid-JSON).
- Strict JSON via `output_config.format = {type:"json_schema", schema:{...}}`,
  `additionalProperties: false`; no `minLength`/`maximum`/recursive `$ref`. On Haiku,
  `output_config` carries `format` **only** — no `effort` key (400s on this tier).
  Check `stop_reason` before reading content: on `"refusal"` or `"max_tokens"`, fall
  back to a name+type-only prompt and tell the GM to edit manually (the editable
  prompt field is the backstop). Don't budget for prompt caching (system prompt is
  under Haiku's 4096-token cache floor).
- Pruner hard caps: whitelist scalar fields; bio HTML-stripped then truncated
  ~2,000 chars; equipped item *names* only, capped ~20; never serialize `items[]`
  wholesale. Actor-derived text goes inside `<actor_data>…</actor_data>` delimiters
  with a data-not-instructions rule — compendium bios are untrusted input.
- Cost: $1/$5 per MTok → ~$0.0035/compile; 100-actor batch ≈ $0.35. Mitigation for
  the key anyway: dedicated Console workspace + spend limit + expiry.

### Upstream code seams (verified against v5.0.3; fact-checked by review pass)

- All image sources converge on `View.addImageLayer(img, opts)`
  (`src/tokenizer/View.js:507`); AI results use `{type: "image"}`.
- Menu wiring: `templates/tokenizer.hbs` avatar/token button rows; handler
  `static async menuButton()` switch (`src/tokenizer/Tokenizer.js:666`) — new case
  goes at the end, before the `// no default` comment. The `"download"` case is the
  DialogV2 pattern reference; ForgeDialog itself is a full ApplicationV2 subclass
  (it has live state: streaming grid, abort, revise).
- **Gating**: `forgeEnabled` computed in `_prepareContext`
  (`game.user.isGM && !!key`), template-gated `{{#if forgeEnabled}}` (the
  `tokenVariantsEnabled` precedent), **and re-checked at the top of the
  `"ai-generate"` case** — template gating alone is console-bypassable. World
  settings register with `restricted: true`.
- **Save-path reality**: upstream's only write path is form submit →
  `View.get("blob")` → `Utils.uploadToFoundry` → re-encode at `image-save-type`
  (default **webp**) into a `token-size` canvas (default **400 px**). A 1024×1024
  generation becomes a 400 px webp token unless configured otherwise. Forge docs
  recommend per-world: `image-save-type: png`, `portrait-size ≥ 1024`; the raw NAI
  PNG in the scratch directory is always the master copy.
- Actor flags: upstream never touches them (grep-verified) —
  `flags.vtta-tokenizer.forge` is uncontested. But actor resolution has edge cases:
  `resolveFlagTarget()` returns `actor.isToken ? actor.token.baseActor : actor`,
  `null` when Tokenizer was launched without an actor (public API `launch({name})`) —
  dialog then runs ephemeral with guide fields disabled. Batch mode checks
  `pack.locked` up front.
- Batch seam: `autoToken()` (`src/hooks.js:253`) + `AutoTokenize.js`; Forge's
  `AutoTokenizeAI` reuses `_initToken`/layer calls with zero upstream edits — but
  **not** upstream's error handling (one try/catch discards all progress). Per-actor
  try/catch, per-actor outcome record, resumable, cancel button, hard per-run cap,
  dry-run mode (log request bodies, send nothing). The tab must stay open and
  foregrounded for the duration; say so in the UI.
- Public API object (`src/hooks.js:435-448`) gets `forgeActor` / `autoTokenAI`
  entries for macros.
- Foundry compatibility: minimum/verified 13; ApplicationV2/DialogV2 only; settings
  scopes `world`/`player`/`client` per upstream convention.
- Build: webpack → `dist/main.js`; `module-dev.json` loads unbundled `src/index.js`
  for the dev loop; release zip sweeps `templates/`, `css/`, `lang/`, `vendor/`
  wholesale. CI releases **only** on master pushes touching `package.json` —
  a code-only push publishes nothing (verify with a version bump in Phase 0).

---

## 3. Architecture

### File layout (Forge-owned, additive)

```
src/forge/
  ForgeDialog.js      ApplicationV2: guide fields, prompt editor, streaming 4-up grid,
                      AbortController per queue (aborted in _onClose), rendered-guards
                      on every post-await DOM touch, views re-resolved at click time
                      (Tokenizer is a singleton; never cache a view across renders)
  NovelAIClient.js    request builder, assertFreeEnvelope, error taxonomy, fflate
                      unzip, blob:/data: image loader, sequential queue + 429 backoff,
                      encode-vibe (consent-gated), scratch-dir persistence
  PromptCompiler.js   pruner + Haiku structured-outputs call + minimal-diff revise;
                      the compiler system prompt lives here and is the product
  ForgeFlags.js       flag schema (versioned) + resolveFlagTarget + migrateForgeFlags
  ForgeSettings.js    settings registration + password-input key menu
  AutoTokenizeAI.js   batch sweep (Phase 4)
templates/forge/forge-dialog.hbs
css/forge/forge.css
vendor/fflate.js      top level, beside MagicWand.js — outside the lint path
```

### Flag schema (`flags.vtta-tokenizer.forge`, versioned from day one)

```json
{
  "schema": 1,
  "guide": "always drawn with her raven",
  "compiled": { "portraitPrompt": "…", "tokenPrompt": "…",
                "subjectExclusions": ["…"], "compiledAt": 0 },
  "characterVibe": { "encoding": "b64…", "model": "nai-diffusion-4-5-full",
                     "sourceSeed": 123, "encodedAt": 0 },
  "lastGeneration": { "promptUsed": "…", "negativeUsed": "…", "baseSeed": 123,
                      "tileSeeds": [123,124,125,126], "acceptedSeed": 124,
                      "model": "…", "sampler": "…", "noiseSchedule": "…",
                      "steps": 23, "scale": 5, "width": 1024, "height": 1024,
                      "styleSuffixSnapshot": "…", "vibeIds": ["…"],
                      "generatedAt": 0 }
}
```

`lastGeneration` is a frozen provenance record of the *accepted* image — seeds are
tracked per tile so "Use →" writes the right one, and world-setting drift (style
suffix changed in month three) can't silently orphan reproducibility. Re-roll = new
`baseSeed`, same everything; Revise = recompiled prompt, same `baseSeed`.
`migrateForgeFlags()` runs from `ready()` and switches on `schema`.

### Settings

| Key | Scope | What |
|---|---|---|
| `forge-novelai-key` | client | `pst-…` (password-input menu, localStorage only) |
| `forge-anthropic-key` | client | Anthropic key (same treatment) |
| `forge-anlas-consent` | client | default off; gates encode-vibe/img2img/augment |
| `forge-compiler-model` | world* | default `claude-haiku-4-5` |
| `forge-nai-model` | world* | default `nai-diffusion-4-5-full` |
| `forge-campaign-guide` | world* | campaign art direction |
| `forge-style-suffix` | world* | deterministic tag suffix |
| `forge-negative-suffix` | world* | appended to the hard-coded UC floor |
| `forge-style-vibes` | world* | encoded campaign vibe(s) + strengths |
| `forge-resolution` | world* | validated NAI presets (Phase 1: 1024×1024 only) |
| `forge-scratch-directory` | world* | default `[data] tokenizer-forge` |
| `forge-allow-players` | world* | default off |

\* all world settings `restricted: true`.

### Prompt-compile contract (one call, both variants)

Input: pruned actor JSON in `<actor_data>` delimiters + campaign guide + actor guide
+ hint (+ previous prompt for revise). Output schema:

```json
{ "portraitPrompt": "…subject tags only…",
  "tokenPrompt": "…full body, standing, simple background…",
  "subjectExclusions": ["…"] }
```

Both prompts cached on the actor from a single compile; the dialog uses
`target.dataset.target` to pick the active one and shows both (active highlighted).
Negative prompt, style, quality tags, characterPrompts arrays: all client-side,
deterministic. Token generations attach the character vibe when one is anchored —
that, not the prompt, is what makes the token match the portrait.

### Upstream touch points

One authoritative table, in [UPSTREAM.md](./UPSTREAM.md) — FORGE.md deliberately does
not duplicate it. Summary: 2 template buttons, 1 `_prepareContext` flag, 1 switch
case + import, 2 hooks.js insertions, `lang/en.json` keys only, manifest/URL edits in
`module-template.json` + `build-module-json.js` + `build.yml`, `package.json` version.
Everything else lives in the Forge-owned tree above.

---

## 4. Implementation phases

- **Phase 0 — scaffold**: fork plumbing — `build-module-json.js` must repoint **both**
  the hardcoded `download` URL **and** `module-template.json`'s `manifest` URL at our
  releases (the script only rewrites `download`; the manifest field ships verbatim);
  neuter Discord webhook + FoundryVTT auto-publish steps; node 14→20; version
  `5.0.3-forge.0` (verify the CI paths-trigger fires). **Console-check
  `foundry.utils.isNewerVersion("5.0.3-forge.1", "5.0.3")` ordering before Phase 5;
  fall back to `5.0.301`-style if it misorders.** `src/forge/` skeleton, settings +
  key menu, gated ✨ buttons → stub dialog. Dev loop: `module-dev.json` manifest,
  repo rsynced into `solo`'s `Data/modules/vtta-tokenizer/`.
  *Optional, recommended, needs thebwt's go-ahead (his GitHub name on it): open the
  `Tokenizer.registerImageSource()` PR upstream (see UPSTREAM.md §exit strategy).*
- **Phase 0.5 — the Anlas spike** (30 min, before any dialog code): from a console,
  fire raw requests and check the account balance before/after: (a) 832×1216
  text2img — is the area cap real? (b) `/ai/encode-vibe` — 2 Anlas confirmed?
  (c) img2img at 1024² ≤28 steps — free or not? These three numbers decide the
  consistency layer's economics; everything in Phase 3 depends on them.
- **Phase 1 — generate**: NovelAIClient (envelope guard, error taxonomy, unzip,
  loader, scratch-dir persistence, sequential streaming 4-up, abort), ForgeDialog
  lifecycle, grid → `addImageLayer`. Manual prompt text only.
  *Milestone: type a prompt in Foundry, get art on a token, survive a mid-grid F5.*
- **Phase 2 — compile & guide**: PromptCompiler (pruner caps, structured outputs,
  refusal fallback), guide layers, flags v1 + migration hook, re-roll/revise.
- **Phase 3 — identity & style**: Anchor flow (portrait → character vibe on consent),
  campaign vibe encode/store, style + negative suffixes, token-matches-portrait via
  vibe stacking, non-square presets if the spike cleared them.
- **Phase 4 — batch**: `AutoTokenizeAI` — per-actor isolation, resumable progress
  flags, cancel, cap, dry-run, "has art" = `src !== CONST.DEFAULT_TOKEN &&
  !src.startsWith("icons/svg/")`.
- **Phase 5 — fleet rollout**: `gh repo create` + release CI; install per instance.

### Fleet targets (survey 2026-08-14)

| Instance | Foundry | Tokenizer today | Forge |
|---|---|---|---|
| solo | 13.346 | none | ✅ **test bed** |
| friday / monday / thursday | 13.3xx | 5.0.3 | ✅ |
| systeam | 14.365 | 5.0.3 | ✅ (already runs on v14; test on solo first, systeam last) |
| marches / raven / mcserver | 11.315 | 4.x / none | ❌ stock (needs Foundry v13) |

Rollout = replace `/srv/foundryVttData/<instance>/Data/modules/vtta-tokenizer/` with
the forge build. **Clobber risk**: installing "Tokenizer" from Foundry's package
browser silently reverts the fork to stock — no error, the ✨ just vanishes.
Mitigations: `ready()` self-check warns if `version` lacks `-forge.` while forge
flags exist in the world; add a one-line probe to the existing health-check scripts
(`grep -L forge /srv/foundryVttData/*/Data/modules/vtta-tokenizer/module.json`);
"never update Tokenizer from the package listing" in the rollout notes.

---

## 5. Risks / open questions

1. **Identity economics** — if the Phase 0.5 spike shows encode-vibe or img2img
   pricier than expected, the anchor feature stays consent-gated and rationed;
   10,000 Anlas/month ≈ 5,000 anchors, so the realistic risk is low.
2. **NAI ToS** — accepted-practice basis, not a verbatim clause. Personal use.
3. **Key custody** — the NAI token is the sharper edge (full-account, no scoping, no
   expiry); bounded by subscription blast radius, rotated by regenerating on
   novelai.net. Anthropic key gets workspace + spend limit + expiry. Both keys:
   per-browser-profile localStorage, same-realm modules can read settings —
   acceptable for an owner-operated fleet, documented, not "fixed".
4. **Upstream drift** — UPSTREAM.md policy; conflict surface ~40 insertion lines;
   merge rehearsal (rebase onto v5.0.0 → merge forward to 5.0.3) validates the
   anchors before the policy is ever needed in anger.
5. **Compile quality across systems** — dnd5e/cyphersystem/forbidden-lands/alienrpg
   schemas differ wildly; that's why the compiler is an LLM. Fix quality in the
   compiler system prompt, never in per-system code. `forge-compiler-model` exists
   if Haiku disappoints.
6. **Curated-model content filter** — grim fantasy subjects may generate cleaner on
   `-full` than `-curated`; default is `-full`, switchable per world.

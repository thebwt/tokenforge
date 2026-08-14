# Tokenforge

A drop-in fork of [MrPrimate/tokenizer](https://github.com/MrPrimate/tokenizer) that adds
AI image generation to the Tokenizer editor: **NovelAI V4.5** for images, **Claude Haiku**
for compiling actor JSON into NovelAI prompts. Module id stays `vtta-tokenizer` — this
replaces the stock module in place; all upstream features and integrations keep working.

Companion doc: [UPSTREAM.md](./UPSTREAM.md) — the re-integration policy for tracking
upstream releases. Read it before touching any upstream-owned file.

---

## 1. What it does

From any Tokenizer window (avatar or token pane), a new ✨ button opens the **Forge
dialog**:

```
┌─ Forge: "Kessra Vane" ──────────────────────────────────┐
│ Guide (saved to actor)                                  │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ travel-worn noble, always with her raven            │ │
│ └─────────────────────────────────────────────────────┘ │
│ This generation: [wounded, torn cloak______________]    │
│ ▸ Compiled prompt (editable, cached)                    │
│                                                         │
│  [img] [img] [img] [img]     ← 4 sequential free gens   │
│                                                         │
│ [Re-roll 🎲] [Revise ✨] [Use →]                         │
└─────────────────────────────────────────────────────────┘
```

- **Compile**: Haiku reads a pruned actor JSON + the guide layers and emits NovelAI
  tag prompts (portrait + token variants). GM edits before firing. Cached on the actor.
- **Re-roll**: same compiled prompt, new seed. No LLM call. Free on Opus.
- **Revise**: guide/hint changed → Haiku recompiles *minimally* against the previous
  prompt (so the character doesn't drift), seed kept.
- **Use**: selected image lands via the existing `view.addImageLayer(img, {type:"image"})`
  pipeline — frames, masks, dynamic rings, save-to-actor all inherited from upstream.

### Guide layers (override, don't swap)

| Layer | Lives in | Example |
|---|---|---|
| Campaign guide | world setting | "low fantasy, everyone grimy, no clean heroes" |
| Actor guide | `flags.vtta-tokenizer.forge.guide` | "always drawn with her raven" |
| Generation hint | ephemeral (dialog field) | "wounded after the ambush" |

Haiku compiles top-down; lower layers win conflicts. **Style never comes from the LLM**:
campaign style is applied deterministically (vibe refs + optional artist-tag suffix from
settings), so 40 NPCs look like one artist drew them. The compiler is instructed to
depict *visible appearance only* — identity spoilers stay out of the art unless the
guide explicitly says otherwise.

---

## 2. Verified constraints (recon 2026-08-14, adversarially verified)

Load-bearing facts this design rests on. Evidence: two independent SDK sources
(Aedial/novelai-api, LlmKira/novelai-python incl. captured real browser requests),
SillyTavern's proxy, docs.novelai.net, live curl probes. Local copies of every fetched
source file: session scratchpad `novelai-research/`.

### CORS / key custody — no relay needed

- `image.novelai.net` answers preflights with `Access-Control-Allow-Origin: *`
  (live-probed on `/ai/generate-image`, `/ai/encode-vibe`, `/suggest-tags`).
  **Browser-direct calls work.**
- `api.novelai.net` (login, subscription, `/ai/upscale`) returns **no CORS headers** on
  preflight — unusable from the browser. v1 simply avoids that host entirely.
- `api.anthropic.com` allows browser calls with header
  `anthropic-dangerous-direct-browser-access: true` (live-probed 200 on preflight).
- Both API keys live in **`scope: "client"` settings** — localStorage, never written to
  the Foundry server DB, never synced to players. Cost: the GM re-enters keys per
  browser. A world-scoped setting would replicate keys to every connected client;
  do not "fix" this by changing the scope.

### NovelAI contract (V4.5)

- `POST https://image.novelai.net/ai/generate-image`,
  `Authorization: Bearer pst-…` (persistent token: novelai.net → Account →
  "Get Persistent API Token"; shown once; regenerating invalidates the old one).
- Model: `nai-diffusion-4-5-full` (also `nai-diffusion-4-5-curated`).
- Request: `{input, model, action:"generate", parameters:{...}}` with
  `params_version: 3`, sampler `k_euler_ancestral`, `noise_schedule: "karras"`,
  scale 5, steps 23 (defaults). Character prompting requires **both** the legacy
  `characterPrompts[]` array **and** `v4_prompt.caption.{base_caption, char_captions[]}`
  (plus the mirrored `v4_negative_prompt`) — NAI's own frontend sends both.
- Response: **ZIP** (`application/x-zip-compressed`) containing one PNG per sample.
  Unzip client-side with `fflate` (`unzipSync`), vendored — no CDN (Foundry has no
  restrictive CSP, but we stay self-contained).
- **Opus free-generation envelope**: tier Opus + `steps ≤ 28` + `≤ 1024×1024 px` +
  first sample of the request. Therefore: **always `n_samples: 1`, fire the 4-up grid
  as 4 sequential requests** with a ~1.5 s gap (community self-throttle convention;
  no documented hard rate limit — Cloudflare fronts it, be polite).
  Steps cap 50 exists in at least one SDK validator; we never exceed 28 anyway.
- img2img (`action:"img2img"` + `image`/`strength`/`noise`) may fall outside the free
  envelope — cost-function sources disagree; treat as Anlas-spending until empirically
  tested. Opus includes 10,000 Anlas/month, so occasional spend is fine.
- Vibe Transfer (V4.5): pass **plural** fields `reference_image_multiple[]`,
  `reference_information_extracted_multiple[]`, `reference_strength_multiple[]`.
  Vibes are pre-encoded via `POST /ai/encode-vibe` (2 Anlas, non-deterministic) or
  loaded from a `.naiv4vibe` bundle (no re-encode fee). Up to 16 vibes; >4 costs
  2 Anlas each. → Campaign style = encode once, store, reuse forever.
- Director Tools (`/ai/augment-image`, incl. `bg-removal`) are **not** in the free
  envelope. Token cutouts come from upstream's frame/mask pipeline instead; Director
  bg-removal is a later opt-in.
- ToS: no explicit clause on third-party API use was located; the officially documented
  persistent-token feature + years of tolerated ecosystem tools (SillyTavern, both
  SDKs) is the accepted-practice basis. Personal use, personal subscription.

### Anthropic contract

- `POST https://api.anthropic.com/v1/messages`, headers `x-api-key`,
  `anthropic-version: 2023-06-01`, `anthropic-dangerous-direct-browser-access: true`.
- Model **`claude-haiku-4-5`** (alias; resolves to `claude-haiku-4-5-20251001`).
- Strict JSON via **native structured outputs**: `output_config.format =
  {type:"json_schema", schema:{...}}` — no tool-use overhead, schema-guaranteed.
  Constraints: no `minLength`/`maximum`/recursive `$ref`; use
  `additionalProperties: false`.
- Cost: $1/$5 per MTok → a compile (~2k in / ~300 out) ≈ **$0.0035**. Batch-mode
  compiles for a 100-actor bestiary ≈ $0.35.
- Anthropic's own docs bless browser keys for owner-only internal tools. Mitigation
  anyway: dedicated Console workspace + workspace spend limit + key expiry.

### Upstream code seams (verified against v5.0.3 checkout)

- All image sources converge on `View.addImageLayer(img, opts)` (`View.js:507`);
  correct opts for AI results: `{type: "image"}`.
- Menu wiring: `templates/tokenizer.hbs` avatar row (~L31-33) and token row (~L79-81);
  handler `static async menuButton()` switch at `Tokenizer.js:666` — the `"download"`
  case (686-713) is the pattern; add our case at the **end** of the switch (before the
  `// no default` comment) for merge-collision safety.
- **Landmine**: `Utils.download()` (`Utils.js:148`) appends a `?timestamp` cache-bust to
  every URL — this corrupts `data:` URIs. AI images must be loaded with a from-scratch
  `new Image()` + `.src` assignment (the `Utils.extractImage`/`Utils.upload` pattern),
  **never** through `Utils.download`.
- Upstream never touches actor flags (grep-verified zero) — `flags.vtta-tokenizer.forge`
  is uncontested namespace.
- `autoToken()` (`hooks.js:253`) + `AutoTokenize.js` is the batch pipeline; a Forge
  sibling `autoTokenAI()` can reuse `_initToken`/layer calls with zero upstream edits.
- Public API object (`hooks.js:435-448`, `window.Tokenizer` /
  `game.modules.get("vtta-tokenizer").api`) is the place to expose `forgeActor()` /
  `autoTokenAI()` for macros and other modules.
- Foundry compatibility: minimum/verified 13 (no v12 path anywhere). ApplicationV2 +
  DialogV2 only. Settings scopes in use: `world` / `player` / `client` — follow those.
- Build: webpack → `dist/main.js`; `module-dev.json` loads unbundled `src/index.js` for
  local dev; release zip sweeps whole directories (`templates/`, `css/`, `lang/`…), so
  Forge-owned files under them ship with no workflow edits.

---

## 3. Architecture

### File layout (Forge-owned, additive)

```
src/forge/
  ForgeDialog.js      ApplicationV2 subclass (QuickSettings/ImageBrowser pattern):
                      guide fields, compiled-prompt editor, 4-up grid, re-roll/revise
  NovelAIClient.js    fetch → /ai/generate-image, fflate unzip, PNG → Image element
                      (own loader; never Utils.download), sequential queue + throttle,
                      vibe encode/store helpers
  PromptCompiler.js   actor JSON pruner (field whitelist, HTML-strip) + Haiku call
                      (structured outputs) + minimal-diff revise call
  ForgeFlags.js       flags.vtta-tokenizer.forge {guide, prompt, negative, seed,
                      tokenPrompt, compiledAt, provider}
  ForgeSettings.js    registration of all forge-* settings + key-entry UI
  AutoTokenizeAI.js   batch sibling of AutoTokenize (Phase 4)
  vendor/fflate.js    vendored unzip (no CDN)
templates/forge/
  forge-dialog.hbs
css/forge/
  forge.css           (release zip includes css/ wholesale; add a stylesheet entry in
                      module-template.json styles array — 1-line upstream edit)
```

### Settings

| Key | Scope | What |
|---|---|---|
| `forge-novelai-key` | client | `pst-…` token (localStorage only) |
| `forge-anthropic-key` | client | Anthropic key (localStorage only) |
| `forge-nai-model` | world | default `nai-diffusion-4-5-full` |
| `forge-campaign-guide` | world | campaign art-direction text |
| `forge-style-suffix` | world | deterministic tag suffix (artist tags etc.) |
| `forge-style-vibes` | world | encoded vibe(s), b64 + strengths |
| `forge-resolution` | world | portrait/token size presets (≤1MP) |

### Prompt-compile contract (Haiku, structured outputs)

Input: pruned actor JSON (name, race/type/size, class, equipped item *names*,
HTML-stripped bio, stated gender only) + campaign guide + actor guide + hint
(+ previous prompt, for revise).

```json
{
  "portraitPrompt":  "…tag string, subject only, no style tags…",
  "tokenPrompt":     "…full body, standing, simple background variant…",
  "characterPrompts": [{"prompt": "…", "uc": "…"}],
  "negativePrompt":  "…"
}
```

The client appends `forge-style-suffix` and attaches `forge-style-vibes`
deterministically after compilation. Compiler system prompt lives in
`PromptCompiler.js` — it is the product; iterate on it there, nowhere else.

### Upstream touch points (the entire non-additive diff)

| File | Edit | ~Lines |
|---|---|---|
| `templates/tokenizer.hbs` | 2 × ✨ button (avatar + token rows) | 6 |
| `src/tokenizer/Tokenizer.js` | 1 import + `case "ai-generate"` delegating to ForgeDialog, at end of switch | 5 |
| `src/hooks.js` | `registerForgeSettings()` in `init()`; `forgeActor`/`autoTokenAI` in API object | 4 |
| `lang/en.json` | `vtta-tokenizer.label.AIGenerate`, `vtta-tokenizer.forge.*` keys | ~10 |
| `module-template.json` | styles entry for `css/forge/forge.css` | 1 |
| `.github/workflows/build.yml` + `build-module-json.js` | release URLs → our repo; neuter Discord/FoundryVTT publish; node 14→20 | ~10 |

Everything else lives in `src/forge/` / `templates/forge/` / `css/forge/`.
Full inventory + merge rules: [UPSTREAM.md](./UPSTREAM.md).

---

## 4. Implementation phases

- **Phase 0 — scaffold**: fork plumbing (build.yml URLs, version `5.0.3-forge.0`),
  `src/forge/` skeleton, settings + keys UI, ✨ buttons wired to a stub dialog.
  Dev loop: `module-dev.json` as manifest, repo checkout symlinked/rsynced into
  `solo`'s `Data/modules/vtta-tokenizer/`.
- **Phase 1 — generate**: NovelAIClient (request builder, zip unpack, image loader,
  sequential 4-up, seed control), ForgeDialog grid → `addImageLayer`. Manual prompt
  text only. *Milestone: type a prompt in Foundry, get art on a token.*
- **Phase 2 — compile & guide**: PromptCompiler (prune → Haiku → editable prompt),
  guide layers + flags persistence, re-roll vs revise semantics.
- **Phase 3 — style lock**: vibe encode/store flow (encode once from 1-4 reference
  images, save to world setting), style suffix, portrait/token framing variants.
- **Phase 4 — batch**: `AutoTokenizeAI` compendium sweep (sequential, progress bar,
  resumable, skip-actors-with-art), optional context-menu entry.
- **Phase 5 — fleet rollout**: GitHub repo + release CI; install on v13+ instances.

### Fleet targets (survey 2026-08-14)

| Instance | Foundry | Tokenizer today | Forge-compatible |
|---|---|---|---|
| solo | 13.346 | none | ✅ **test bed** |
| friday | 13.351 | 5.0.3 | ✅ |
| monday | 13.345 | 5.0.3 | ✅ |
| thursday | 13.351 | 5.0.3 | ✅ |
| systeam | 14.365 | 5.0.3 | ✅ (5.0.3 already runs on v14 in practice; verify) |
| marches / raven / mcserver | 11.315 | 4.x / none | ❌ stays on stock (needs Foundry v13 core, upstream constraint) |

Rollout = replace `/srv/foundryVttData/<instance>/Data/modules/vtta-tokenizer/` with the
forge build (or install via our manifest URL). Modules are per-instance and
hand-managed — ansible does not manage modules (deliberate; don't change that here).

---

## 5. Risks / open questions

1. **NAI ToS** — accepted-practice basis, not verbatim clause. Personal use; low risk;
   revisit if NAI publishes API terms.
2. **img2img free-tier ambiguity** — test empirically in Phase 2; budget Anlas.
3. **Foundry v14 (systeam)** — upstream declares verified 13, max unbounded; 5.0.3
   already runs there, but test Forge on solo first, systeam last.
4. **Key custody** — client-scope keys are invisible to players but do live in
   browser localStorage; use a spend-limited Anthropic workspace key; NAI token is
   subscription-bound (worst case: overwrite to rotate).
5. **Upstream drift** — mitigated by UPSTREAM.md policy; the whole design keeps the
   conflict surface under ~40 lines.
6. **Compile quality across systems** — dnd5e/cyphersystem/forbidden-lands/alienrpg
   actor schemas differ wildly; that's *why* the compiler is an LLM. Keep per-system
   hints out of code; fix in the compiler system prompt.

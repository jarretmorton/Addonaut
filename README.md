# Addonaut

A single-page web app that turns a plain-English idea into a real, installable
Minecraft Bedrock add-on. Describe what you want, Addonaut checks it against
what Bedrock can actually do, draws you a 16×16 texture you can repaint by hand,
lets you tune the numbers, and exports a working `.mcaddon` file — all in the
browser, with no build step and no server.

The honest part is the compatibility check. Bedrock add-ons can't do most of
what people imagine, so instead of quietly generating something broken,
Addonaut tells you up front which parts of your idea it can build and which it
is leaving out.

**Bring your own key (BYOK).** The app runs entirely in your browser. You supply
your own API key — a free Google Gemini key works — and it is stored only on
your device and sent straight to the provider, never to any server of ours.

## The flow

pick a kind (or let the AI choose) → describe it → compatibility check, with
anything unbuildable listed plainly → paint the texture → tune the behaviour →
save a real `.mcaddon`

Two AI calls per run: one plans the add-on (kind, name, identifier, properties,
suggested options), one designs the texture. If the texture call fails, you get
a placeholder pattern to paint over and the run continues.

## What it can build

| Kind | What you get | Knobs |
| --- | --- | --- |
| **Block** | A full solid cube, one texture on every side | mining time, blast resistance, light emission (0–15), friction |
| **Item** | A plain inventory item | stack size |
| **Food** | An edible item | hunger restored, saturation, time to eat, edible when full |
| **Tool** | A weapon/tool with durability | durability, attack damage, enchantability |
| **Mob** | A cube-shaped creature, passive or hostile, with a spawn egg | health, walk speed, size, hostile, attack damage |

Blocks can also be **animated**: add frames in the editor and Addonaut writes a
flipbook texture with your chosen ticks-per-frame. Bedrock only animates block
textures, so other kinds stay a single frame.

The AI also proposes four or five **options** per add-on ("make it glow", "make
it slippery"). Accept or decline each one and the sliders update to match.

## What it can't build

Custom 3D models beyond a cube, crafting recipes, GUIs, structures, dimensions,
scripting, ore generation, armour, or anything Java-only. The compatibility
check names these explicitly rather than pretending — if your idea is half
buildable, you get the buildable half plus a list of what was dropped.

## What lands in the `.mcaddon`

The file is a plain ZIP (stored, uncompressed) holding a behavior pack and a
resource pack that depend on each other, so a single import turns on both:

```
<id>_BP/manifest.json                  format_version 2, min_engine_version 1.20.60
<id>_BP/blocks|items|entities/<id>.json
<id>_RP/manifest.json
<id>_RP/textures/…/<id>.png            16×16, or 16×(16·frames) when animated
<id>_RP/textures/terrain_texture.json  (blocks) / item_texture.json (items)
<id>_RP/textures/flipbook_textures.json (animated blocks only)
<id>_RP/entity|models|render_controllers/… (mobs only)
<id>_RP/texts/en_US.lang               display names
```

Everything is namespaced `custom:<identifier>`, and the identifier is
lower-cased and stripped to `[a-z0-9_]` so it's always a legal Bedrock ID.

## Installing what you made

1. **Save add-on** downloads `<identifier>.mcaddon`. On a phone or tablet,
   **Send to another app** hands it to the share sheet instead.
2. Open the file — Windows: double-click it; Android/iPad: tap it in Files.
   Minecraft imports both packs.
3. Turn the behavior pack **and** the resource pack on in your world settings.

If the download gets renamed to `.zip`, rename it back to `.mcaddon`.

## Local development

One file, no build step, no dependencies. Serve the folder with any static file
server:

```bash
# Python
python -m http.server 4173

# or Node
npx serve .
```

Then open <http://localhost:4173>.

Opening `index.html` straight from disk works for the UI too (everything is
inline — no ES modules, no local `fetch`), but a local server is the reliable
route: an API may reject a request whose `Origin` is `null`.

## Configuration

Everything configurable lives at the top of the script in
[`index.html`](index.html):

- `APP_VERSION` — shown in the header. Bump on release.
- `DEFAULT_MODEL` — the model per provider. Currently `gemini-3.8-flash` for
  Google and `claude-sonnet-5` for Anthropic.
- `RETIRED_MODELS` — model IDs this app once prefilled into the Model box. A
  saved value matching one is cleared at boot, so a browser that never chose a
  model follows the current default instead of being pinned to an old one.

The **⚙ button** in the header opens AI settings: provider, model and key. The
Model box is editable, so a stale default can be fixed without touching the
code — and a 404 from the provider says exactly that.

**Model names go stale.** Verify current Gemini IDs at
<https://ai.google.dev/gemini-api/docs/models> and Claude IDs at
<https://docs.anthropic.com/en/docs/about-claude/models>.

Three `localStorage` keys, all written only by the settings panel:

| Key | Holds |
| --- | --- |
| `af_provider` | `gemini` or `claude` |
| `af_model` | a model ID, or empty to follow `DEFAULT_MODEL` |
| `af_key` | your API key |

## Getting a free Gemini API key

1. Go to <https://aistudio.google.com/apikey> and sign in with Google.
2. Click **Create API key**.
3. Copy the key (starts with `AIza…`) and paste it into AI settings (⚙).

The free tier has daily quotas that reset at midnight Pacific, and is **not
available in the EEA, UK, or Switzerland**. Anthropic keys work too, from
<https://console.anthropic.com/settings/keys>, but Claude has no free tier —
you pay per use.

## Privacy & security

- Your key lives only in `localStorage` on your device and is sent only in the
  request header to the provider you picked — `x-goog-api-key` for Google,
  `x-api-key` for Anthropic. It is never logged and never put in a URL.
- Only your typed description is sent to the model. Textures you paint, the
  properties you tune, and the `.mcaddon` are all built locally — the ZIP is
  assembled in the browser and never uploaded.
- No accounts, no analytics, no server. The page is static files.
- Using Claude from the browser requires the
  `anthropic-dangerous-direct-browser-access` header, which exposes your key to
  any script on the page. That is the trade for a serverless BYOK app; keep the
  page's origin trusted, and prefer a key you can rotate.

## Deploy to GitHub Pages

No build step — just publish the file.

1. Push this folder to a GitHub repo.
2. Repo **Settings → Pages**.
3. Under **Build and deployment**, set **Source: Deploy from a branch**, pick
   your branch (e.g. `main`) and folder `/ (root)`, and save.
4. Your app is live at `https://<user>.github.io/<repo>/` a minute or two later.

## Project structure

```
index.html    the whole app — markup, styles and script in one file
```

Inside, the script is sectioned in the order it runs: state, ZIP writer,
pixels→PNG, add-on builder, AI calls and prompts, texture helpers, DOM helpers,
settings wiring, actions, prop labels, render, boot.

## Non-goals

No accounts, no server, no database, no build tooling, no frameworks, no
offline/PWA. Addonaut generates the five things Bedrock makes easy and is honest
about the rest; it is not a full add-on IDE.

---

Addonaut is not an official Minecraft product. Not approved by or associated
with Mojang or Microsoft. Minecraft is a trademark of Mojang Synergies AB.

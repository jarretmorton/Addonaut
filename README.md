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
your own free Google Gemini API key; it is stored only on your device and sent
straight to Google — never to any server of ours.

## The flow

pick a kind (or let the AI choose) → describe it → compatibility check, with
anything unbuildable listed plainly → paint the texture → tune the behaviour →
save a real `.mcaddon`

Everything you make is kept in **your add-ons** (the ▤ button) automatically, so
you can come back and keep working on it later.

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

## Your add-ons

The ▤ button in the header opens everything you've made. There is no "save"
step: the moment a compatibility check succeeds you have an entry, and every
edit after that — a painted pixel, a moved slider, an accepted option, an added
animation frame — updates it. Writes are debounced, so painting a texture is one
save at the end rather than one per pixel.

Each card shows the texture, the name and kind, when you last touched it, and
the description you originally typed. From there you can:

- **Open it** — the full studio comes back exactly as you left it, ready to
  repaint, re-tune and export again.
- **Re-run the AI** — sends the saved description back to the model for a fresh
  plan and texture. The result is saved as a *new* add-on, so the one you
  already painted is never overwritten.
- **Delete it** — asks first, since it can't be undone.

Entries live in this browser's `IndexedDB`, on your device only. Nothing is
uploaded. The app also asks the browser to treat that storage as persistent,
which matters most in Safari, where script-writable storage can otherwise be
evicted after about a week without a visit. It is a request, not a guarantee.

If IndexedDB is unavailable — private mode, a blocked origin — the library shows
its empty state and the rest of the app carries on working. Nothing about making
an add-on depends on it.

## Installing what you made

1. **Save add-on** downloads `<identifier>.mcaddon`. On a phone or tablet,
   **Send to another app** hands it to the share sheet instead.
2. Open the file — Windows: double-click it; Android/iPad: tap it in Files.
   Minecraft imports both packs.
3. Turn the behavior pack **and** the resource pack on in your world settings.

If the download gets renamed to `.zip`, rename it back to `.mcaddon`.

## Local development

One HTML file plus three icon assets. No build step, no dependencies. Serve
the folder with any static file server:

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
- `DEFAULT_MODEL` / `DEFAULT_LITE_MODEL` — where the model picker starts before
  Google's own list arrives, and where it stays if that list never does.
- `RETIRED_MODELS` — model IDs this app once prefilled into the Model box. A
  saved value matching one is cleared at boot, so a browser that never chose a
  model follows the current default instead of being pinned to an old one.

The **⚙ button** in the header opens AI settings: model and key.

### The model list comes from Google

Model names go stale fast, so the picker is not a hard-coded list. On startup
(and whenever settings open) the app calls Google's
[ListModels](https://ai.google.dev/api/models) endpoint with your key and fills
the dropdown with what that key can actually reach, newest first. A **Switch to
the lighter model** button cycles between the two the app cares about:

- **best free** — the newest full Flash model
- **lighter** — the newest Flash-Lite, which has higher rate limits and slightly
  less detail

Picking the best model stores nothing, so that browser keeps following the best
as it moves; picking anything else pins it by ID.

**One caveat worth knowing.** ListModels reports what a key can reach, but *not
what anything costs* — there is no free-tier or pricing field in the response.
So "free" here is **inferred from the family**: Flash and Flash-Lite are the
tiers Google offers at no cost, Pro is not, and everything else (embedding,
image, TTS) is filtered out. If Google changes which families are free, the
`keepModel()` filter is the line to revisit.

Discovery is always an enhancement, never a dependency. No key, no network, or
a refused list simply leaves the built-in defaults in place, and the app carries
on.

Two `localStorage` keys, both written only by the settings panel:

| Key | Holds |
| --- | --- |
| `af_model` | a model ID, or empty to follow `DEFAULT_MODEL` |
| `af_key` | your Gemini API key |

Addonaut briefly offered Anthropic as a second provider. It doesn't any more, so
boot clears what that left behind: the old `af_provider` flag, a saved
`claude-sonnet-5`, and — deliberately — a saved Anthropic key, which must never
be posted to Google's endpoint. Losing the key opens settings, which is where a
Gemini key gets pasted.

## Getting a free Gemini API key

1. Go to <https://aistudio.google.com/apikey> and sign in with Google.
2. Click **Create API key**.
3. Copy the key (starts with `AIza…`) and paste it into AI settings (⚙).

The free tier has daily quotas that reset at midnight Pacific, and is **not
available in the EEA, UK, or Switzerland**.

## Privacy & security

- Your key lives only in `localStorage` on your device and is sent **only** via
  the `x-goog-api-key` request header — never as a URL parameter, never logged.
- Only your typed description is sent to the model. Textures you paint, the
  properties you tune, and the `.mcaddon` are all built locally — the ZIP is
  assembled in the browser and never uploaded.
- Google is the only host the app talks to.
- Saved add-ons live in this browser's `IndexedDB`, on your device only. They
  are never uploaded, and deleting one from the library removes it for good.
- No accounts, no analytics, no server. The page is static files.
- A key in `localStorage` is readable by any script running on this page. That
  is the trade a serverless BYOK app makes, so prefer a free key you can revoke
  and rotate over one with billing attached.

## Deploy to GitHub Pages

No build step — just publish the folder. The icons must ship alongside
`index.html`; without them a GitHub Pages URL falls back to whatever icon the
browser has cached for the origin.

1. Push this folder to a GitHub repo.
2. Repo **Settings → Pages**.
3. Under **Build and deployment**, set **Source: Deploy from a branch**, pick
   your branch (e.g. `main`) and folder `/ (root)`, and save.
4. Your app is live at `https://<user>.github.io/<repo>/` a minute or two later.

## Project structure

```
index.html            the whole app — markup, styles and script in one file
icon.svg              the app icon, and the resting mark in the header
favicon-32.png        tab icon for browsers that ignore SVG favicons
apple-touch-icon.png  180×180 iOS home-screen icon, full bleed
```

Inside, the script is sectioned in the order it runs: state, ZIP writer,
pixels→PNG, add-on builder, AI calls and prompts, texture helpers, DOM helpers,
settings wiring, actions, prop labels, render, boot.

### The app icon

An astronaut helmet whose maroon faceplate reflects the block you just made,
drawn on the same 16×16 grid the texture editor paints on.

`icon.svg` is the master, and the two PNGs are **rasterised from it** —
regenerate them that way rather than redrawing, or the vector and the raster
quietly diverge. `apple-touch-icon.png` is drawn at 176px, a whole 11px per
cell, then padded to 180 on the same ground: 180 ÷ 16 lands on 11.25, and
uneven cells look wrong on pixel art. Keep both PNGs **full bleed with square
corners** — iOS applies its own mask, so pre-rounded corners get double-rounded
and transparent corners render black.

Two colours are load-bearing. The reflected block is green while the rest of the
mark is a red family, because red on that maroon is 1.91:1 and the block
dissolves into the faceplate at tab size, where grass green restores it to
3.63:1. The ink ring does the same job for the faceplate against the bone shell,
a 1.67:1 pairing on its own. Both are worth re-checking before changing any
colour in the mark.

The header shows `icon.svg` until a run has a texture of its own and then swaps
to the texture, so the header mark cannot drift from the favicon. That swap is
the only thing the icon does in the app — `fallbackFrame()` is unrelated, and
still exists to give you a pattern to paint over when the texture call fails.

## Non-goals

No accounts, no server, no remote database, no build tooling, no frameworks, no
offline/PWA. (Saved add-ons live in a local, on-device library — see above.)
Addonaut generates the five things Bedrock makes easy and is honest about the
rest; it is not a full add-on IDE.

---

Addonaut is not an official Minecraft product. Not approved by or associated
with Mojang or Microsoft. Minecraft is a trademark of Mojang Synergies AB.

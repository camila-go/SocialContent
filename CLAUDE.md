# SocialFeedUITest — Project Instructions

## Style reference

Build UI in this project to match the Figma design:
https://www.figma.com/design/giBqmGdVTbQBQDYFI3qUF4/wnbaPage?node-id=321-3979&t=wcphCz7Ngjw3fe5c-11

Key style tokens pulled from that file (feed-variation-2-bold-typographic):

- Background: `#000000` (global/gl-neutral-900)
- Card border: `#191919` (global/gl-neutral-800)
- Body/subtle text: `#929292` (global/gl-neutral-300)
- Stat text: `#adadad` (global/gl-neutral-200)
- Primary text: `#ffffff` (global/gl-neutral-0)
- Accent / button fill: `#c10016`, button radius `32px`
- Display headline: "Acumin VF", ExtraCondensed Black weight, ~100px desktop, line-height 0.9, uppercase (loaded via the Typekit stylesheet below)
- UI/body font: Inter (Regular 400 / Bold 700)
- Card layout: image tile + meta panel (timestamp, handle, caption, like/comment stats, bookmark icon). All four cards (TikTok/Instagram/LinkedIn/Facebook) share one identical fixed-size (620×440px card, 260×440px media panel) template — media panel left, meta panel right — so they read as one consistent set regardless of underlying content type (video, image, text+image, image).
- Footer: left-aligned label + white pill CTA button with red text, copy "Follow the action"
- Font implementation note: the Typekit kit at the URL below exposes a single variable font family, `acumin-variable` (weight axis 100–900) — not a separate "extra condensed" static family. The design's condensed/black look is `font-weight: 800` + `font-variation-settings: "wdth" 50`.

## Required font include

Every HTML page in this project must include the following in `<head>`:

```html
<link rel="stylesheet" href="https://use.typekit.net/rik2kml.css">
```

## Tech constraint

Everything in this project is built in **vanilla JS** (plain HTML/CSS/JS). No React, Vue, or other frameworks. Official platform embed scripts (TikTok, Instagram, Facebook `embed.js`/iframes) are fine since they're plain `<script>`/`<iframe>` includes, not app frameworks.

## Social posts used in the feed

The social feed stream component pulls in these four real posts, each linking/embedding through to the original so a click takes you to the live post:

- **TikTok (video)** — @capellauniversity: https://www.tiktok.com/@capellauniversity/video/7660591110850415885
- **Instagram (static)**: https://www.instagram.com/p/DalNipSgduU/?img_index=1
- **LinkedIn (text + image)** — Capella University's WNBA "Official Higher Learning Partner" announcement (real press release text, no direct public post URL provided; links out to the official announcement)
- **Facebook (static)**: https://www.facebook.com/photo?fbid=1433393835489081&set=a.640810724747400

## Notes

- Embeds use each platform's official embed method: TikTok/Instagram `embed.js` blockquote. Both enforce their own minimum rendering width and ignore a plain CSS width/height on the generated iframe once it hydrates — simply squeezing it to our media panel just clips a slice out of a wider render. Fix: leave the iframe at its own natural size and use a proportional `transform: scale()` to cover the frame (matching the *larger* of the width-fit/height-fit ratios, same idea as `object-fit: cover`). **The scale value is computed live by JS, not hardcoded.** Earlier versions hand-computed a fixed number against an assumed natural size (e.g. "TikTok renders at 325×738") — that assumption turned out to not match what TikTok actually ships (its real embed is a thumbnail + a separate white info panel with a Watch button/handle/caption/music credit, not a bare video player), so the card rendered visibly smaller than the others instead of covering its frame. The `<script>` at the bottom of the file now measures each iframe's real `offsetWidth`/`offsetHeight` once it mounts (via `MutationObserver` + a few retries, since embeds sometimes finalize their size a moment after mounting) and sets `transform:scale()` from that measurement, recomputing on resize so it adapts correctly at both the desktop (260×440) and mobile (340×220) media panel sizes. This self-corrects regardless of what either platform renders, instead of a static guess that can silently drift out of sync. TikTok's info panel sits at the bottom of its embed, so `#tiktok-media` stays bottom-aligned (`align-items:flex-end` + `transform-origin:center bottom`) so any cropping always trims the thumbnail at the top first and never cuts into the readable info panel.
- Facebook does **not** use a live Page Plugin iframe (it did in an earlier version of this component). A plain, non-SDK Facebook iframe embed never reports its true rendered height back to the parent page — Facebook's own JS SDK handles that auto-resize via postMessage, and adding the full SDK for one card felt like overkill and couldn't be verified to work without an app ID and live testing. Without real height, any hardcoded height for the cover-scale math was a guess that came out either clipped or under-filled with a gap. The Facebook card now reuses the same `.brand-graphic__text` red/bold-type treatment as LinkedIn (reproducing the real post's actual creative, confirmed via screenshots) — this guarantees a correct, gap-free fill since it isn't a black-box third-party iframe.
- No custom platform badges (logo pills like "in LinkedIn" / "f Facebook") on any card — they were tried on the LinkedIn/Facebook static-graphic panels but removed; platform identity comes from the account name in the meta panel's handle line and the "View on [Platform]" link instead, keeping every card's media panel clean.
- If an embed fails to load (offline, blocked script, etc.), each card falls back to a static preview + "View original post" link so the click-through still works.
- Cards are fixed-size at every breakpoint, including mobile (`max-width:640px`): the media panel is `340×220px` there, not a fluid `100%`/`88vw`. This is required, not just cosmetic — the JS cover-scale watcher needs a *known* target box per breakpoint (see `currentMediaTarget()` in the `<script>`), so a fluid mobile container would break that math. If either breakpoint's media panel dimensions ever change, update the corresponding `w`/`h` values in `currentMediaTarget()` to match — the scale itself is computed automatically, nothing else to recalculate by hand.
- The "View on…" arrow icon is defined once as an inline `<symbol id="icon-arrow">` sprite near the top of `<body>` and referenced via `<use href="#icon-arrow">` in each card, rather than repeating the raw `<svg><path>` markup four times.

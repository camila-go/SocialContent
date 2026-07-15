# Handoff Spec: Social Feed Stream Component

`social-feed-stream.html` — vanilla HTML/CSS/JS, no framework. Style reference: [Figma — feed-variation-2-bold-typographic](https://www.figma.com/design/giBqmGdVTbQBQDYFI3qUF4/wnbaPage?node-id=321-3979&t=wcphCz7Ngjw3fe5c-11). Full style tokens and rationale for each embed's implementation are documented in `CLAUDE.md`; this file covers responsive behavior and accessibility specifically, both as an audit of the current build and a punch list of what still needs attention before this ships in a real page.

## Overview

A horizontally-scrolling strip of four social post cards (TikTok video, Instagram photo, LinkedIn text+image, Facebook photo) under a large display headline, with a footer CTA. Each card is identical in size/chrome regardless of the underlying platform, and each links out to the real post.

## Responsive Behavior

### Current implementation

| Breakpoint | What happens |
|---|---|
| Desktop / default (no media query, applies down to 641px) | `.social-feed__inner` capped at `max-width:1560px`. Heading is a fixed `100px`. Cards are fixed `620×440px` (media panel fixed `260×440px`) laid out in a row, in a horizontally scrollable strip (`overflow-x:auto`, `scroll-snap-type:x proximity`). |
| Mobile (`max-width:640px`) | Cards switch to `flex-direction:column`, fixed width `340px`, `height:auto`. Media panel becomes fixed `340×220px` (no longer fluid — see note below). Heading drops to `56px`. Section/header/footer side padding reduces from `60px` to `24px`. |

### Gaps / what to do next

- **No mid-range (tablet/small-laptop) breakpoint.** Between ~641px and ~1024px, cards still render at their full fixed `620px` width and the heading stays at the full `100px`. On an iPad-portrait-width viewport or a narrow browser window this will look oversized relative to the viewport (the horizontal-scroll pattern still "works," but the type scale won't feel intentional). Recommend adding a breakpoint around `1024px` that steps the heading down (e.g. `72px`) and/or switches heading sizing to `clamp()` so it scales fluidly instead of jumping at a single point.
- **Card and media panel dimensions are hardcoded px, not fluid**, by design — this keeps all four cards pixel-identical (a deliberate decision after the platform embeds kept fighting anything more flexible; see `CLAUDE.md`). Confirm this fixed-size, horizontal-scroll pattern is actually the intended interaction model at all viewport widths the design will actually ship at, not just the two currently styled.
- **No visible scroll affordance on the strip beyond the styled scrollbar** (`::-webkit-scrollbar`, which only renders in Chromium/WebKit browsers and is invisible on touch devices, where scrollbars are typically hidden regardless). There's nothing telling a first-time desktop/Firefox user there are 4 cards' worth of horizontal content beyond the initial viewport. Consider a partial 5th-card peek, fade-out edge mask, or explicit prev/next arrows for non-touch, non-WebKit browsers.
- ~~Third-party embed minimum widths were tuned only for the current fixed media panel sizes~~ — **resolved**: the TikTok/Instagram `transform:scale()` values are no longer hardcoded. A `<script>` at the bottom of the file measures each embed's real rendered iframe size once it mounts (`MutationObserver` + a few timed retries, since embeds sometimes finalize their size shortly after mounting) and computes the cover-scale from that measurement, recomputing on window resize. This was actually the root cause of an earlier bug where the hardcoded scale (guessed against an assumed iframe size that didn't match TikTok's real embed structure — a thumbnail + separate info panel, not a bare video player) made the TikTok card visibly smaller than the other three on mobile. If the media panel's target dimensions ever change, update `currentMediaTarget()` in the script — the scale factor itself no longer needs manual recalculation.
- ~~`vw`-based mobile card width (`88vw`) has no `max-width` cap~~ — **resolved**: the mobile card/media panel now use a fixed `340px` width instead of `88vw`, which was required anyway so the TikTok/Instagram cover-scale transforms have a known target box size to be computed against (a fluid container silently breaks that math). This incidentally also fixes the original "could render quite large on a wide mobile viewport" concern.

## Accessibility Notes

### Already in place

- `<html lang="en">` set.
- Landmark/semantic structure: `<section aria-label="Social feed stream">` wrapping a `role="list"` strip of `role="listitem"` `<article>` cards.
- All four "go to the real post" links are real `<a href>` elements with visible, descriptive text ("View on TikTok," "Read the announcement," etc.) rather than icon-only buttons or `<div onclick>` patterns — screen reader and keyboard accessible by default.
- Color contrast (computed against the actual token values, WCAG 2.1 formula):
  - Body/caption text `#929292` on `#000000` background → **6.74:1** (passes AA for normal text; just under the 7:1 AAA threshold).
  - Stat/secondary text `#adadad` on `#000000` → **9.36:1** (passes AAA).
  - CTA red `#c10016` text on white, and white text on the `#c10016` brand-graphic cards → **6.41:1** (passes AA normal text).
  - All comfortably clear of the 4.5:1 AA minimum for normal text; none currently reach AAA (7:1) for body copy.
- **Decorative SVG icons are hidden from assistive tech.** The "View on…" arrow is now a single shared `<symbol id="icon-arrow">` sprite referenced via `<use>`, and each usage carries `aria-hidden="true"` — screen readers won't encounter it, matching the visible link text that already says where it goes.

### Needs attention before ship

1. **No visible focus state anywhere in the CSS.** There isn't a single `:focus` or `:focus-visible` rule in the stylesheet — keyboard users tabbing through the "View on…" links, the footer CTA, and (once live) the embeds will fall back to the browser's default outline, which varies by browser/OS and may be hard to see against the pure-black background in some of them. **This is the highest-priority fix**: add an explicit `:focus-visible` style (e.g. a `2px` offset outline in white or the accent red) to `.feed-card__view` and `.social-feed__cta` at minimum.
2. **Heading level starts at `<h2>` with no `<h1>` in this file.** That's correct *if* this component is dropped into a page that already has an `<h1>` earlier in the DOM — but if it's ever used as a standalone page/demo, there'd be no `h1` at all. Confirm with whoever integrates this component and adjust the heading level if needed.
3. **Third-party embed accessibility is out of our control.** Once TikTok's and Instagram's `embed.js` hydrate the blockquotes into iframes, whatever `title`/ARIA attributes those iframes ship with are entirely up to TikTok/Instagram — we can't verify or fix that from here. Recommend a manual screen-reader pass (VoiceOver/NVDA) once this is live to confirm both platforms' iframes announce sensibly; if not, an `aria-label` on the `#tiktok-media` / `#instagram-media` wrapper `div`s (e.g. `aria-label="TikTok video from @capellauniversity"`) would at least give assistive tech something meaningful at the container level.
4. **The horizontally-scrolling strip isn't independently keyboard-scrollable.** `.social-feed__strip` has no `tabindex`, so a keyboard user can only move through it by tabbing link-to-link (which does auto-scroll each card into view in most browsers) rather than scrolling the region directly with arrow keys. Consider `tabindex="0"` plus `aria-label` (e.g. "Scrollable list of social posts") on the strip if a more carousel-like keyboard interaction is wanted.
5. **No `prefers-reduced-motion` handling.** Minor given the only transition is a `0.15s` opacity fade on CTA hover, but worth a `@media (prefers-reduced-motion: reduce)` block if this component grows more animation later.

## Testing checklist for engineering

- [ ] Add `:focus-visible` styles to all interactive elements.
- [x] Add `aria-hidden="true"` to decorative SVG icons.
- [ ] Confirm heading level (`h2`) is correct in the context this ships in.
- [ ] Screen-reader pass on the live TikTok/Instagram embeds once network-loaded.
- [ ] Resize-test between 641px–1024px and adjust the missing mid-breakpoint.
- [ ] Confirm horizontal-scroll-strip UX with at least one keyboard-only pass and one touch-device pass.
- [x] Verify mobile (≤640px) TikTok/Instagram embeds cover their 340×220 frame correctly on a real device (fixed the scale-math gap that previously only accounted for the desktop breakpoint).

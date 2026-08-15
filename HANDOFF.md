# HorsePropertyInsurance.com — Session Handoff

**Date:** 2026-08-15
**Repo:** `aiagent322/horsepropertyinsurance`
**Branch:** `main`
**HEAD at handoff:** `9af67fb`

First HANDOFF.md for this repo. Written after a real mistake this session
that's worth preventing from happening again.

---

## 1. CRITICAL: this site's real stylesheet is inline, `style.css` is dead

`index.html` has **no `<link rel="stylesheet">` anywhere**. All actual
styling lives in an inline `<style>` block in `<head>` (currently around
lines 24–145). The `style.css` file sitting in the repo root is **not
referenced from the page at all** — editing it has zero effect on what
renders live.

This cost real time this session: a centering fix was written into
`style.css`, verified as "deployed" via curl (which only proved the *file*
served correctly, not that the *page* used it), and did nothing. It was
only caught by actually rendering the page in headless Chromium and finding
the CSS rule wasn't in the matched-rules list at all. See commit `4dab4b6`
for the full corrected fix and the postmortem in its commit message.

**Rule going forward: any CSS change goes in the inline `<style>` block in
`index.html`, never in `style.css`.** If `style.css` is ever going to be
wired up for real (a `<link>` tag added), do that as its own deliberate
change, not incidentally.

**Also: curl cannot verify a rendering/layout question.** It can confirm a
file's *bytes* are correct and live, but not whether a browser actually
applies a CSS rule, computes a centered layout, or loads an image
successfully through this sandbox's TLS-intercepting egress proxy (real
`fetch`/`img` requests need `ignore_https_errors=True` in a Playwright
context here, or they fail silently with no HTTP response captured at all —
this looks identical to a real network failure unless you check for it).
For any "is this actually centered / does this actually render" question,
use headless Chromium (Playwright is installed) and measure real
`getBoundingClientRect()` values — don't infer visual layout from source
inspection alone.

---

## 2. Revive ad zones live on this site

Both use the same publisher `data-revive-id`
(`727bec5e09208690b050ccfc6a45d384`) — this ID is shared across the whole
network (bridleandbit-ai, horsepropertyguide, teamroping-calendar all use
the same one). Only one `<script async src="//servedby.revive-adserver.net/asyncjs.php">`
is needed per page no matter how many zones/`<ins>` tags are on it.

| Zone | Placement |
|---|---|
| `30295` | Below the hero, above "Coverage Types" (top of page) |
| `30301` | After "Essential Guides", before the "About This Site" box (bottom of content) |

Both `<ins>` tags sit inside a `.ad-slot` wrapper div (defined in the inline
`<style>` block — see section 1). `.ad-slot` centers the creative
regardless of whether Revive delivers it as a raw `<img>` (most common
here) or an `<iframe>` — both get `display:block; margin:0 auto` scoped to
`.ad-slot`, which is necessary because the sitewide `img { display:block }`
reset breaks `text-align:center` on its own (block elements ignore
`text-align`; they need `margin:auto`).

If a third zone is ever added, reuse the `.ad-slot` class as-is — don't
duplicate the `<script>` tag, and don't reason about whether it's centered
without an actual headless-browser check (see section 1).

---

## 3. B&B logo — recolored via CSS filter, not a separate asset

No black version of the Bridle & Bit logo exists anywhere in the network.
The header uses the same cream/beige source PNG that bridleandbit.com and
teamroping-calendar use
(`https://imagedelivery.net/9VlM7Y9GaQMXOu5Ypg50yA/bb-hp-0/public`), forced
to solid black via `filter:brightness(0)` in CSS (`.bb-logo` class). This
recolors the exact same artwork rather than fabricating new art — confirmed
with Rex before implementing (he approved this approach explicitly over
uploading a real black asset).

If a genuine black-on-transparent logo file ever gets uploaded to Cloudflare
Images, swap the `<img src>` and drop the `filter` — but until then, this is
the working solution, not a placeholder.

Mobile note: the logo is hidden below 480px (`.bb-logo-link { display:
none }` in a `max-width:479px` media query) so it doesn't crowd the site's
own wordmark on small phones — this mirrors a breakpoint already used
elsewhere in the network for a similar problem. `.header-top` needs an
explicit `gap` between its flex children or they can end up flush against
each other with zero breathing room even when each item's own width looks
fine in isolation.

---

## 4. Open items

- [ ] No GitHub Actions / auto-deploy workflow has been inspected for this
      repo — unclear whether pushes to `main` auto-deploy to Cloudflare
      Pages or need a manual trigger. Worth confirming before assuming a
      push is live.
- [ ] `style.css` remains dead code (see section 1). Either wire it up
      properly or consider removing it so a future session doesn't repeat
      this mistake by finding it and assuming it's the real stylesheet.

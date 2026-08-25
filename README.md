# evarnimtb website

Plain HTML + CSS. No build step, no dependencies, no JavaScript.

```
index.html   markup + all content
style.css    the entire design system
```

Open `index.html` in a browser. That's the whole workflow.

---

## How the grid works

The page is one CSS Grid, 4 columns wide, with `1px` gaps. The page
background is black and the cells are white, so the gaps *are* the
borders — one hairline between neighbours, never a doubled 2px line.

Cells are square by default (`aspect-ratio: 1/1`). Because the columns
are a definite `1fr`, those squares give the auto-sized rows a definite
height — which is what makes the row-spanning cells square as well.

To place a cell, add a class:

| class        | shape                  |
|--------------|------------------------|
| *(none)*     | 1x1 square             |
| `cell--w2`   | 2 wide, 1 tall         |
| `cell--w2 cell--h2` | 2x2 big square  |
| `cell--h2`   | 1 wide, 2 tall         |
| `cell--w4`   | full-width band        |
| `cell--flat` | height follows content |

A cell whose content outgrows its square grows taller instead of
clipping. That's deliberate — it degrades, it doesn't break.

Breakpoints: 4 columns → 2 columns at 900px → 1 column at 560px.

### Sizing

The slab caps at `1360px` and otherwise fills the window minus
`--gutter` on each side. Base text is `15px/1.55`.

Vertical: `body` is a grid with `align-content: safe center` and
`min-height: 100svh`, so when the page is shorter than the window the
gap below the slab exactly matches the gap above it (measured: 323px
either side on a 2100px-tall viewport). `safe` means it falls back to
top-aligned once the page outgrows the window, instead of clipping the
nameplate off the top. The bottom padding carries an extra `--sh` so
the dithered shadow doesn't eat into that lower gap.

### Gotchas, all already handled

- **Don't put `overflow: hidden` on `.cell`.** It zeroes out the grid
  item's automatic minimum size, and cells silently clip their content
  instead of growing past the square. Found this the hard way on mobile.
- **At 2 columns, 1x1 cells must come out even.** An odd one leaves a
  black hole beside it. Either pair them up or widen one — that's what
  the `.cell--big` rule in the 900px block is doing.
- **One rule looking thicker than the rest** means a cell is shorter
  than its row. A row is as tall as its tallest cell, and two cells
  sharing a row can each outgrow their square by a different amount, so
  the shorter one leaves a sliver of ink under it. Single-column cells
  carry `align-self: stretch` + `min-height: min-content` to fill their
  row. Do **not** extend that to the wide cells: it makes QUOTES and
  GUESTBOOK overflow the slab, and the floor eats their bottom padding.
- **Don't stretch the grid to fill the height.** Putting a `min-height`
  on `.grid` does grow the rows, but `aspect-ratio` cells refuse to grow
  with them and you get ink-coloured bands behind them. Centring the
  body was the fix. `min-height: 100%` on the cells is worse — the
  percentage feeds back into track sizing and collapses the whole slab.
- **`margin: 0 auto` on the slab is gone,** replaced by
  `justify-self: center`. Now that `body` is a grid container, an auto
  margin overrides `stretch` and shrinks the slab to fit-content.

---

## Accent colour

One variable, `--accent` at the top of `style.css`. Currently `#ff3b00`.
It drives the little square before every heading, link hover, tag hover,
text selection and focus rings — so changing that one line repaints the
whole site. A few that hold up against the black grid are listed in a
comment right there.

The rule that keeps it from getting ugly: the accent never carries
information. Everything still reads in pure black and white.

## Dithered shadows

No blur, no images, no filters. A `repeating-conic-gradient` tiled at
`4px` is a 2px checkerboard — an ordered dither screen. Two of them at
different densities, offset down-right on `::before` and `::after`,
give a stepped halftone shadow:

```
--dither-50   50% screen (near, dense)
--dither-25   25% screen (far, sparse)
```

The slab gets both, the avatar and the wall frames get just the dense
one. This is also why the page background is `--page` rather than black:
a shadow needs something to fall on. The gaps in the grid are still ink,
so the hairline rules are unchanged.

Cost: zero bytes, zero requests, and no repaint on scroll — which a real
`box-shadow` blur or a `filter` over a large area would give you.

To kill the effect entirely, delete the two `.grid::before/::after`
rules and the `.avatar::after,.frame::after` rule. Nothing else depends
on them. They already switch off under `prefers-contrast: less`.

Keep the offset on `.frame::after` smaller than `.slot`'s gap, or the
shadow prints over the caption underneath it.

---

## Guestbook

A static page has nowhere to store entries, so the guestbook doesn't
store them. Signing is a link to a **prefilled GitHub issue**; entries
get pasted into the `<ul class="entries">` by hand.

That's the whole design, and the tradeoff is deliberate:

- **No backend, no build step, no third-party origin, no JS.** The cell
  costs ~250 bytes and zero requests. The footer stays honest.
- **Host-independent.** Works identically on GitHub Pages, Netlify,
  Cloudflare Pages, or a USB stick.
- **Spam is gated twice.** Signing requires a GitHub account, and
  nothing renders until you paste it in. Drive-by spam can't reach the
  page, only your notifications.
- **Cost:** it filters signers down to people who have GitHub, and new
  entries only appear when you edit the file.

### Before it works

Swap `USER/REPO` in the link's `href` for the real repository. Until
then it points nowhere.

### Adding an entry

Copy the shape of the ones already there:

```html
<li><b>name</b> <span class="stamp">00.00</span><br>what they said</li>
```

Two entries fit the 2x1 cell comfortably and there's room for about
four. Past that the cell grows taller rather than clipping, which is
fine — but the grid row grows with it, so prune old entries or widen
the cell to `cell--w2 cell--h2` and give something else the space.

### The gotcha: don't add `&labels=`

It's tempting to add `&labels=guestbook` to the URL so entries filter
neatly in the issues list. **Don't.** GitHub requires permission for
every action a query parameter performs, and returns a plain
`404 Not Found` when you don't have it — so the link would work for you
and 404 for every visitor, which is everyone the guestbook is for.
Same trap applies to `&assignees=`, `&milestone=` and `&projects=`.

`title` and `body` need no permissions, which is why the title is
prefilled with a `guestbook: ` prefix — filter on that instead.

### If it outgrows this

The next rung is a **Cloudflare Worker + KV**, and it does *not* have to
add JavaScript to the page: a plain `<form method="post">` posts to the
Worker, which validates and stores the entry and redirects back, and a
Pages Function with `HTMLRewriter` injects the stored entries into the
HTML at the edge. Entries appear live, still zero client JS, but it ties
the site to Cloudflare Pages and needs a honeypot field, a length cap
and a rate limit to survive contact with the public.

Giscus is the one to avoid — it works, but it's third-party JavaScript
plus an iframe, and it makes the footer a lie.

---

## Performance notes

Current weight: ~8 KB of HTML + CSS, 2 requests, 0 JS, 0 fonts.
Everything below is about *keeping* it that way as content lands.

### The rules that matter most

1. **Stay on system fonts.** A single webfont weight is 15–40 KB and
   blocks or shifts text. `ui-sans-serif, system-ui, ...` costs zero
   bytes. If you must have a display face, use it for the `<h1>` only,
   `woff2` subset to the exact characters you need, self-hosted,
   `font-display: swap`, and `<link rel="preload">` it.
2. **Images will be 95% of your page weight.** See below.
3. **Never add a framework.** React + a bundler for a page like this is
   ~50 KB gzipped before a single word renders.

### Images (the politician wall, the shrine)

- Convert everything to **AVIF** (or WebP as fallback). A 1 MB JPEG is
  typically 40–80 KB as AVIF at visually identical quality.
- Resize to the size actually displayed, then 2x for retina. A 200px
  thumbnail never needs a 3000px source.
- Always set `width` and `height` attributes → no layout shift.
- `loading="lazy" decoding="async"` on everything below the first screen.
- With `srcset`, phones download the small file and desktops the large:
  ```html
  <img src="face-400.avif"
       srcset="face-400.avif 400w, face-800.avif 800w"
       sizes="(max-width: 560px) 50vw, 200px"
       width="400" height="400" loading="lazy" decoding="async" alt="">
  ```
- One-off CLI pass over a folder (needs `libavif`/ImageMagick):
  ```bash
  for f in *.jpg; do magick "$f" -resize 800x800\> -quality 55 "${f%.*}.avif"; done
  ```
- **Sprite sheet trick:** if the wall is many small identical-size faces,
  one combined image + `background-position` beats N requests.

### If you ever add JavaScript

- No jQuery, no libraries. Modern DOM APIs cover everything here.
- Plain `<script>` at the end of `<body>`, or `<script defer>` in head.
- Never `document.write`, never a synchronous third-party embed —
  one slow analytics script can cost more than your whole page.
- Prefer CSS: `:hover`, `:target`, `<details>` and `:has()` handle
  accordions, tabs, lightboxes and menus with no JS at all.

### Squeezing the last bytes (optional)

- **Inline the CSS** into a `<style>` block in `<head>`. Saves a round
  trip and renders in a single response. Worth it under ~10 KB of CSS;
  costs you cross-page caching, so do it last.
- **Minify** on deploy only — keep the readable source.
- **Brotli compression** — free and automatic on Netlify / Cloudflare
  Pages / GitHub Pages. Cuts text assets ~75%.
- **Long cache headers** on assets, and version filenames when they
  change (`style.a1b2.css`).
- **No favicon 404:** `<link rel="icon" href="data:,">` is already in
  `index.html`. Replace it with a real `.svg` icon (~1 KB) when you have one.
- **Preconnect nothing.** Zero third-party origins is the goal.

### Things that quietly cost a lot — avoid

Google Fonts · analytics scripts · embedded YouTube iframes (~1 MB
each — use a click-to-load thumbnail) · Font Awesome (use inline SVG or
emoji) · animated GIFs (a muted `<video>` is 10x smaller) · CSS
`filter`/`backdrop-filter` on large areas (repaints on every scroll).

### How to check

Chrome DevTools → Network, "Disable cache", throttle to Fast 3G.
Watch total transferred bytes and request count. Then Lighthouse.
Target for this site: **under 100 KB total, under 10 requests.**

---

## TODO

- **Guestbook repo link.** The cell is live, but the sign link still
  points at `USER/REPO`. Swap in the real repository — see the
  Guestbook section above. (The shrine went back to `cell--w2` when
  the guestbook returned; the two of them stack beside the wall.)
- **Real profile picture** in the `.avatar` box in the nameplate. The
  `<img>` goes inside the div (commented out there); it fills the box
  via `object-fit: cover`. Export it square and small — 400px is plenty
  for a 104px box on a retina screen.

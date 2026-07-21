# wallpaper.graphics

Custom digital backgrounds, built to order from $1 with unlimited revisions. Paid in USDT crypto.

**Live site:** <https://wallpaper.graphics/>

A single-page static site with a 24-piece gallery, a lightbox, a live cryptocurrency ticker, real-time visitor presence, and live chat. No build step, no framework, no runtime dependencies.

---

## What it does

Visitors browse finished work, pick one of three formats, describe the scene they want, and submit a brief that opens pre-filled in their mail client. Payment is USDT on the TRC20 network.

Three formats, same price:

1. **Backgrounds** — scene only, no people
2. **Backgrounds with one person** — built around one supplied photo
3. **Backgrounds with two people** — built around two

## Features

| Feature | Notes |
|---|---|
| **Gallery** | 24 plates, 4 across, format-tagged, lazy-loaded |
| **Lightbox** | Next/back buttons, arrow keys, touch swipe, neighbour preloading, focus restoration |
| **Order form** | Assembles a structured brief and hands it to the visitor's mail client |
| **"Order this style"** | Prefills the form with the chosen format and reference piece |
| **Crypto ticker** | 9 coins scrolling in the header, from the CoinGecko public API |
| **USDT payment** | TRC20 address with one-click copy and a live USD peg readout |
| **Live presence** | Real concurrent-viewer count with a country breakdown |
| **Preloader** | Progress bar tied to actual image decoding |
| **Live chat** | Tawk.to, loaded only after the preloader clears |
| **Sharing** | In-page panel per plate; each plate has its own page under `/p/` with its own preview image |
| **SEO** | OG and Twitter cards, `Service` and `FAQPage` JSON-LD, sitemap, robots.txt |

---

## The three integrations

Each one had a constraint worth designing around, and each fails without taking the page down with it.

### 1. Crypto ticker — CoinGecko

Nine coins (BTC, ETH, USDT, TRX, BNB, SOL, XRP, ADA, DOGE) scroll continuously in a marquee under the nav, pausing on hover. The USDT figure is mirrored into the payment section so buyers can see the peg is holding.

```
GET https://api.coingecko.com/api/v3/simple/price
      ?ids=bitcoin,ethereum,tether,tron,binancecoin,solana,ripple,cardano,dogecoin
      &vs_currencies=usd&include_24hr_change=true
```

No API key. **But the public tier allows only 5–15 requests per minute**, and Firebase Hosting is static, so every visitor's browser calls the API directly. That budget is shared across all of them.

Hence:

| Behaviour | Reason |
|---|---|
| `sessionStorage` cache, 2 min TTL | Navigating or refreshing costs zero API calls |
| Stale data painted instantly, refreshed behind it | The bar is never empty while a request is in flight |
| Last-known values held for 30 min | A rate-limited response doesn't blank the display |
| Explicit "rates unavailable" on total failure | A silently vanishing bar is impossible to debug |
| 8-second timeout | A hanging call can't leave it stuck on "loading" |

Status dot: green = fresh, amber = stale, grey = last known.

The honest limitation: this cache is per-visitor, not shared. A hundred first-time visitors in one minute means a hundred API calls, and some will be rate-limited into the fallback path. A server-side proxy would collapse that to one call for everyone — the right fix if traffic grows.

### 2. Live presence — Firebase Realtime Database

The header shows a real concurrent-viewer count. Clicking it rolls down a panel listing which countries those viewers are in, your own first.

Each open tab writes itself to `/presence/{pushId}`; Firebase's `onDisconnect` removes that entry **server-side** the moment the tab closes, crashes, or drops network. The cleanup is queued *before* the entry is written, so a connection lost between the two calls can't leave a ghost behind. A `pagehide` handler covers browsers that close without a clean socket teardown.

Country comes from `Intl.DateTimeFormat().resolvedOptions().timeZone` — the browser's own timezone, mapped against a ~70-entry lookup table. **No IP lookup, no third party.** Unmatched timezones display as "Somewhere else" rather than guessing. Nothing identifying is stored: a country string and a timestamp, deleted when the tab closes.

If the database is unreachable the pill hides itself and the rest of the site is unaffected.

Setup instructions: [PRESENCE_SETUP.md](PRESENCE_SETUP.md).

### 3. Live chat — Tawk.to

Loaded **only after the preloader finishes**, not at page load. The loader dispatches a `wg:booted` event on completion and the Tawk snippet listens for it.

Two reasons: the widget injects at a very high z-index and would otherwise render over the boot screen, and deferring it stops the chat script competing for bandwidth while gallery images decode. A `loaded` flag prevents double injection, and a 10-second fallback fires it anyway if the event never arrives.

---

## Preloader

A boot overlay with dual counter-rotating rings and a progress bar.

Progress blends two signals — elapsed time against a 5-second target, and how many above-the-fold images have actually decoded — and trails whichever is further behind, so the bar never claims to be finished while images are still arriving. Gallery images are lazy-loaded and would never fire `load` while the overlay is up, so only images within two viewport heights are counted.

Guards: a hard ceiling at `DURATION + 4s`, a global `error` listener that clears the overlay if anything else on the page throws, and a `<noscript>` rule that hides it entirely when JavaScript is off. It removes itself from the DOM after fading so it can never trap focus or clicks.

To change the duration, edit one line:

```js
const DURATION = 5000;   // target hold in ms
```

---

## Structure

```
public/
  index.html      entire site — markup, styles, scripts
  img/            21 gallery images
  robots.txt
  sitemap.xml
firebase.json     hosting config, cache headers, rewrites
.firebaserc       project binding (wallpapergraphics-b9ca1)
README.md
DEPLOY.md         deployment and custom-domain notes
PRESENCE_SETUP.md Realtime Database setup for the viewer counter
```

Single-file by choice. One page, no build step — splitting it into modules would add tooling without reducing complexity.

## Running locally

```bash
npx serve public
# or
python3 -m http.server 8000 --directory public
```

Presence needs the Firebase config to be filled in; the ticker and chat work as-is.

## Deploying

```bash
npm install -g firebase-tools
firebase login
firebase deploy --only hosting
```

Images are cached immutably for a year, `index.html` for an hour (set in `firebase.json`). Hard-refresh with Ctrl+F5 after deploying or you'll see the cached page.

Git and Firebase are independent — `git push` stores history, `firebase deploy` updates the live site. Doing both:

```bash
git add .
git commit -m "what changed"
git push
firebase deploy --only hosting
```

### Custom domain

Verified by TXT record, pointed at Firebase with A records. If the ACME certificate challenge reports **conflicting results**, the cause is almost always leftover registrar parking records still answering on the apex. Check with:

```bash
nslookup wallpaper.graphics 8.8.8.8
```

Only Firebase IPs should come back. Delete any others at the registrar, disable any domain-parking toggle (it re-adds them), then wait for the TTL to expire before retrying.

## Adding a gallery piece

1. Drop the JPG into `public/img/`
2. Copy an existing `<article class="card">` block — update filename, title, description, badge and `data-i`
3. Add a matching entry to the `SHOTS` array in the script

**The card count and `SHOTS` length must stay equal**, and `data-i` values must run 0..n-1 in the same order as `SHOTS`. If they drift, the lightbox opens the wrong image.

Quick check before deploying:

```bash
grep -c 'class="lb-open"' public/index.html   # cards
grep -c "src:'img/" public/index.html         # lightbox entries
ls public/img | wc -l                         # files
```

All three should match.

## Ordering and payment

Briefs arrive at yiorgosantoni@gmail.com. Payment is USDT on the **TRC20 (TRON)** network only — address shown on the site with a copy button, transaction hash captured in the order form so payments can be matched to briefs.

## A note on the Firebase API key

The `firebaseConfig` block in `index.html` is public by design. Firebase web config ships in client code on every Firebase site and is not a secret. **Security rules are what protect the database** — see PRESENCE_SETUP.md. The rules here permit anonymous writes only under `/presence`, cap field sizes, and block everything else.

## License

Site code MIT. Gallery images are original work and not covered by that license.

---

## Per-plate share pages

Social platforms fetch whatever `og:image` the shared URL declares, and they do
not accept an image parameter. Sharing the homepage would therefore give every
plate the same preview. So each plate has its own page:

```
public/p/{slug}.html      meta tags for that plate, then a JS redirect to the gallery
public/share/{slug}.jpg   1200x630 preview card generated from the original
```

Three things make this work, each of which failed at least once while building it:

- **No `<meta http-equiv="refresh">`.** Crawlers follow it and end up scraping
  the homepage. The redirect is JavaScript-only and skips known crawler
  user-agents entirely, so they stay long enough to read the tags.
- **`canonical` points at the plate page**, not the site root. Pointing it at
  the root tells Facebook the homepage is the real target.
- **The preview is a purpose-built 1200x630 card**, not the 2656x1600 original.
  X in particular fails silently on large images and falls back to a small card.

Every image reference on a plate page — `og:image`, `twitter:image`, and the
visible `<img>` — points at the same card, so there is nothing for a crawler to
disagree about. JSON-LD keeps the full-resolution file for Google Images.

**Testing:** every platform caches previews per URL and will not refetch. Append
a query string (`?v=2`) to force a clean fetch. Facebook's debugger at
`developers.facebook.com/tools/debug` has a "Scrape Again" button; its Redirect
Path row shows whether a crawler is being bounced somewhere unintended.

## Sharing UI

Clicking share opens a panel rendered inside the page — WhatsApp, Facebook, X,
copy link. The networks are plain `<a href>` links, so there is no
`window.open` anywhere in the file. This was a deliberate rewrite: popup-based
sharing broke differently on every platform (iOS Safari blocks `window.open`
from anything it does not consider a direct user gesture; browser extensions
can duplicate the call). A panel of links has no such failure modes.

## Cache headers

`firebase.json` sets HTML to `no-cache, must-revalidate` so deploys appear
immediately. Images stay `immutable` for a year. If HTML is cached, Firebase's
CDN keeps serving the old file after a deploy and Ctrl+F5 will not help.

## Build marker

`index.html` logs its build to the console (`wallpaper.graphics build share-v11`)
and carries it as `<meta name="build">`. If the console does not show the build
you just deployed, the file did not reach the server — check that before
debugging anything else.

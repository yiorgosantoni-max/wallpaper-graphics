# wallpaper.graphics

Custom digital backgrounds, painted to order from $1 with unlimited revisions. Paid in USDT.

**Live:** <https://wallpaper.graphics/>

A static site — one HTML file, 24 gallery plates, 24 per-plate share pages. No framework, no build step, no runtime dependencies. Hosted on Firebase Hosting.

---

## What it does

Visitors browse finished work, pick a format, describe the scene they want, and submit a brief that opens pre-filled in their mail client. Payment is USDT on the TRC20 network.

| Format | Price |
|---|---|
| The scene alone — no figures | **$1** |
| With one figure — built around one supplied photo | **$2** |
| With two figures or more | **$3** |

Unlimited revisions at every tier.

## Features

| | |
|---|---|
| **Catalogue** | 24 plates, 4 across, uncropped 5:3 frames, lazy-loaded |
| **Lightbox** | Roman numerals, arrow keys, touch swipe, neighbour preloading, actual-size zoom |
| **Sharing** | In-page panel; each plate has its own URL and preview image |
| **Order form** | Assembles a structured brief and hands it to the visitor's mail client |
| **Market band** | 9 coins from the CoinGecko public API, with the USDT peg mirrored into the payment section |
| **Live presence** | Real concurrent-viewer count with a country breakdown |
| **Preloader** | Progress tied to actual image decoding |
| **Live chat** | Tawk.to, injected only after the preloader clears |
| **SEO** | OG and Twitter cards, `Service` + `FAQPage` + `AggregateOffer` JSON-LD, image sitemap |

---

## Structure

```
public/
  index.html      the entire site — markup, styles, scripts
  img/            24 full-resolution plates (2656x1600)
  share/          24 social preview cards (1200x630)
  p/              24 per-plate share pages
  robots.txt
  sitemap.xml
firebase.json     hosting config, cache headers
.firebaserc       project binding (wallpapergraphics-b9ca1)
DEPLOY.md         deployment steps and post-deploy checks
PRESENCE_SETUP.md Realtime Database rules for the viewer counter
```

Single-file by choice. One page, no build step — splitting it into modules would add tooling without reducing complexity.

---

## The parts worth explaining

Each of these looks over-engineered until you hit the failure it prevents. All of them were built after hitting it.

### Per-plate share pages

Social platforms fetch whatever `og:image` the shared URL declares, and none of them accept an image parameter. Sharing the homepage gives every plate the same preview. So each plate gets its own page:

```
public/p/{slug}.html      meta tags for that plate
public/share/{slug}.jpg   1200x630 preview card
```

Three things make this work, each of which failed first:

- **No `<meta http-equiv="refresh">`.** Crawlers follow it and scrape wherever it points. Facebook's debugger showed the redirect chain landing on the homepage, which is why every plate previewed as the homepage. The plate pages now have no redirect at all — they are proper landing pages with a link through to the catalogue.
- **`canonical` points at the plate page**, not the site root. Pointing it at the root tells the crawler the homepage is the real target.
- **The preview is a purpose-built 1200x630 card**, not the 2656x1600 original. X silently falls back to a small card when it cannot fetch the image, which reads as "wrong image" rather than "fetch failed".

Every image reference on a plate page — `og:image`, `twitter:image`, and the visible `<img>` — points at the same card, so there is nothing for a crawler to disagree about. JSON-LD keeps the full-resolution file for Google Images.

**Testing:** every platform caches previews per URL and will not refetch. Append a query string (`?v=2`) to force a clean fetch. Facebook's debugger at `developers.facebook.com/tools/debug` has a "Scrape Again" button, and its Redirect Path row shows whether a crawler is being bounced somewhere unintended.

### Share UI

Clicking share opens a panel rendered inside the page — WhatsApp, Facebook, X, copy link — with the plate's URL shown at the bottom. The networks are plain `<a href>` links.

**There is no `window.open` anywhere in the file.** That is deliberate. Popup-based sharing broke differently on every platform: iOS Safari blocks `window.open` unless it fires synchronously inside a user gesture and fails silently when it doesn't; browser extensions can duplicate the call and produce two windows. A panel of links has none of those failure modes.

On mobile the links use app schemes where they exist:

| | Desktop | Mobile |
|---|---|---|
| WhatsApp | `api.whatsapp.com` | `whatsapp://send` |
| X | `x.com/intent/post` | `twitter://post` |
| Facebook | `facebook.com/sharer` | `m.facebook.com/sharer.php` |

App schemes do nothing when the app is absent, so a 1.5s timer falls back to the web dialog — cancelled if the page goes hidden, which is how the code knows the app opened.

**Facebook cannot open its app with a link pre-filled.** Meta removed that capability; no `fb://` share scheme exists for websites. `facebook.com/sharer` handed to the iOS app opens the app's home feed with the link discarded, which is why mobile uses the web dialog instead. This is a platform restriction, not a bug.

### Market band — CoinGecko

Nine coins scroll under the nav, pausing on hover. The USDT figure is mirrored into the payment section so buyers can see the peg is holding.

```
GET https://api.coingecko.com/api/v3/simple/price
      ?ids=bitcoin,ethereum,tether,tron,binancecoin,solana,ripple,cardano,dogecoin
      &vs_currencies=usd&include_24hr_change=true
```

No API key. **But the public tier allows only 5–15 requests per minute**, and Firebase Hosting is static, so every visitor's browser calls the API directly and that budget is shared across all of them.

| Behaviour | Reason |
|---|---|
| `sessionStorage` cache, 2 min TTL | Navigating or refreshing costs zero API calls |
| Stale data painted instantly, refreshed behind it | The bar is never empty while a request is in flight |
| Last-known values held for 30 min | A rate-limited response doesn't blank the display |
| Explicit "unavailable" on total failure | A silently vanishing bar is impossible to debug |
| 8-second timeout | A hanging call can't leave it stuck on "loading" |

Status dot: gold = fresh, oxblood = stale, grey = last known.

The honest limitation: this cache is per-visitor, not shared. A hundred first-time visitors in one minute means a hundred API calls, and some will be rate-limited into the fallback. A server-side proxy would collapse that to one call for everyone — the right fix if traffic grows.

### Live presence — Firebase Realtime Database

The header shows a real concurrent-viewer count. Clicking it lists which countries those viewers are in, your own first.

Each open tab writes itself to `/presence/{pushId}`; Firebase's `onDisconnect` removes that entry **server-side** the moment the tab closes, crashes or drops network. The cleanup is queued *before* the entry is written, so a connection lost between the two calls can't leave a ghost. A `pagehide` handler covers browsers that close without a clean socket teardown.

Country comes from `Intl.DateTimeFormat().resolvedOptions().timeZone` — the browser's own timezone, mapped against a ~70-entry table. **No IP lookup, no third party.** Unmatched timezones show as "Somewhere else" rather than guessing. Nothing identifying is stored: a country string and a timestamp, deleted when the tab closes.

If the database is unreachable the pill hides itself and nothing else is affected. Setup: [PRESENCE_SETUP.md](PRESENCE_SETUP.md).

### Preloader

A gilt frame on the gallery wall with a sheen sweeping across it, and a progress bar beneath.

Progress blends two signals — elapsed time against a 5-second target, and how many above-the-fold images have actually decoded — and trails whichever is further behind, so the bar never claims to be finished while images are still arriving. Gallery images are lazy-loaded and would never fire `load` while the overlay is up, so only images within two viewport heights are counted.

Guards: a hard ceiling at `DURATION + 4s`, a global `error` listener that clears the overlay if anything else throws, and a `<noscript>` rule that hides it entirely when JavaScript is off. It removes itself from the DOM after fading so it can never trap focus or clicks, then fires `wg:booted`.

To change the duration, edit one line:

```js
const DURATION = 5000;   // target hold in ms
```

### Live chat

Tawk.to is injected **only after `wg:booted` fires**. Two reasons: the widget injects at a very high z-index and would otherwise render over the boot screen, and deferring it stops the chat script competing for bandwidth while gallery images decode. A `loaded` flag prevents double injection; a 10-second fallback fires it anyway if the event never arrives.

---

## Two CSS bugs worth remembering

Both cost real time, and both look like content problems rather than styling ones.

**Collapsed image frames.** Catalogue images rendered as tall narrow slivers despite `aspect-ratio` being set. Three causes compounding: `<figure>` carries a default browser margin; grid items don't shrink below content size without `min-width: 0`; and `aspect-ratio` was computing against the collapsed width. Fixed with `margin: 0`, `min-width: 0`, and an explicit `height: auto`. Where `aspect-ratio` still refuses to resolve, pin the height in pixels instead.

**Clipped gradient text.** The gold italic `f` in the headline was cut off. `background-clip: text` crops to the glyph's background box, which is tighter than the ink — heavy italic Fraunces overhangs about 6px right and 16px below the baseline at 73.6px. The tell was that highlighting the text showed it perfectly, because selection replaces the gradient fill. Fixed with generous padding and matching negative margins so the layout doesn't shift.

Measure the real extent rather than guessing:

```js
const m = ctx.measureText('brief');
m.actualBoundingBoxRight - m.width   // right overhang
m.actualBoundingBoxDescent           // depth below baseline
```

---

## Running locally

```bash
npx serve public
# or
python3 -m http.server 8000 --directory public
```

Presence needs the Firebase config filled in; the ticker and chat work as-is.

## Deploying

```bash
firebase deploy --only hosting
```

Expect roughly 85 files. See [DEPLOY.md](DEPLOY.md) for the post-deploy checks.

**Cache headers matter here.** `firebase.json` sets HTML to `no-cache, must-revalidate` so deploys appear immediately; images stay `immutable` for a year. When HTML was cached for an hour, Firebase's CDN kept serving the old file after a deploy and Ctrl+F5 did not help — which made several fixed bugs look unfixed.

**Build marker.** `index.html` logs its build to the console (`wallpaper.graphics build share-v18`) and carries it as `<meta name="build">`. If the console doesn't show the build you just deployed, the file didn't reach the server — check that before debugging anything else.

Git and Firebase are independent:

```bash
git add .
git commit -m "what changed"
git push
firebase deploy --only hosting
```

### Custom domain

If the ACME certificate challenge reports **conflicting results**, leftover registrar parking records are almost certainly still answering on the apex:

```bash
nslookup wallpaper.graphics 8.8.8.8
```

Only Firebase IPs should come back. Delete the others at the registrar, disable any domain-parking toggle (it re-adds them), then wait for the TTL before retrying.

---

## Adding a plate

1. Drop the JPG into `public/img/`
2. Copy an existing `<article class="work">` block — update filename, title, description, badge and `data-i`
3. Add a matching entry to the `SHOTS` array
4. Generate a 1200x630 card into `public/share/` and a page into `public/p/`
5. Add both URLs to `sitemap.xml`

**Card count and `SHOTS` length must stay equal**, and `data-i` must run 0..n-1 in the same order. If they drift, the lightbox opens the wrong image.

```bash
grep -c 'class="mat lb-open"' public/index.html   # cards
grep -c "src:'img/" public/index.html             # lightbox entries
ls public/img | wc -l                             # originals
ls public/share | wc -l                           # preview cards
ls public/p | wc -l                               # share pages
```

All five should match.

---

## Ordering and payment

Briefs arrive at yiorgosantoni@gmail.com. Payment is USDT on the **TRC20 (TRON)** network only — address shown on the site with a copy button, transaction hash captured in the form so payments can be matched to briefs.

## A note on the Firebase API key

The `firebaseConfig` block in `index.html` is public by design. Firebase web config ships in client code on every Firebase site and is not a secret. **Security rules are what protect the database** — see PRESENCE_SETUP.md. The rules permit anonymous writes only under `/presence`, cap field sizes, and block everything else.

## License

Site code MIT. Gallery images are original work and not covered by that license.

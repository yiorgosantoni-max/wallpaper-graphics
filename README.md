# wallpaper.graphics

Custom digital backgrounds, built to order for $1 with unlimited revisions. Paid in USDT crypto.

**Live site:** <https://wallpaper.graphics/>

Includes a live cryptocurrency price ticker (BTC, ETH, USDT, TRX) integrated from the CoinGecko public API, with client-side caching to stay inside a strict rate limit — see [Live rates](#live-rates-and-why-theyre-cached).

---

## What this is

A single-page static site advertising a made-to-order wallpaper service. Visitors browse a gallery of 21 finished pieces, pick a format, describe the scene they want, and submit a brief that opens pre-filled in their email client.

Three formats are offered, all at the same price:

1. **Backgrounds** — scene only, no people
2. **Backgrounds with one person** — built around one supplied photo
3. **Backgrounds with two people** — built around two

## Features

- **Gallery of 21 pieces**, each tagged by format, with a click-to-enlarge lightbox
- **Lightbox** with next/back buttons, arrow-key navigation, touch swipe, neighbour preloading, and focus restoration on close
- **Order form** that assembles a structured brief and hands it to the visitor's mail client via `mailto:`
- **"Order this style"** buttons that prefill the form with the chosen format and reference piece
- **Live crypto rates bar** — BTC, ETH, USDT and TRX with 24h change, pulled from the CoinGecko public API
- **USDT TRC20 payment address** with one-click copy and a live USD peg readout
- **SEO**: Open Graph and Twitter cards, `Service` and `FAQPage` JSON-LD, sitemap, robots.txt, descriptive alt text throughout

## Live rates, and why they're cached

The rates bar calls the [CoinGecko public API](https://www.coingecko.com/en/api) directly from the browser, because Firebase Hosting serves static files and there's no server to proxy through.

Endpoint:

```
GET https://api.coingecko.com/api/v3/simple/price
      ?ids=bitcoin,ethereum,tether,tron
      &vs_currencies=usd
      &include_24hr_change=true
```

No API key required. The USDT figure is also mirrored into the payment section, so buyers can see the peg is holding when they send their $1.

That tier allows only **5–15 requests per minute**, shared across every visitor. So the ticker is written defensively:

| Behaviour | Reason |
|---|---|
| `sessionStorage` cache, 2 min TTL | Navigating or refreshing costs zero API calls |
| Stale data painted instantly, refreshed in background | The bar is never empty while a request is in flight |
| Last-known values kept for 30 min | A rate-limited response doesn't blank the display |
| Explicit "rates unavailable" message on total failure | A silently vanishing bar is impossible to debug |
| 8-second request timeout | A hanging call can't leave it stuck on "loading" |

The status dot reflects which state you're in:

| Dot | Meaning |
|---|---|
| Green | Fresh data, fetched within the last 2 minutes |
| Amber | Stale data shown while a refresh runs |
| Grey | Upstream unreachable — last known values, or an offline notice |

The trade-off worth naming: because this runs client-side, the cache is per-visitor rather than shared. A hundred first-time visitors in a minute means a hundred API calls, and some will be rate-limited into the fallback path. A server-side proxy would collapse that to one call for everyone — the right fix if traffic grows, and the reason the same problem is solved properly [in a separate project](https://github.com/yiorgosantoni-max) rather than here.

## Structure

```
public/
  index.html      entire site — markup, styles and scripts in one file
  img/            21 gallery images
  robots.txt
  sitemap.xml
firebase.json     hosting config, cache headers, rewrites
.firebaserc       Firebase project binding
```

Single-file by choice. The site is one page with no build step, so splitting it into modules would add tooling without reducing complexity.

## Running locally

No dependencies, no build. Serve the `public` folder with anything:

```bash
npx serve public
# or
python3 -m http.server 8000 --directory public
```

Then open <http://localhost:8000>.

## Deploying

Hosted on Firebase Hosting.

```bash
npm install -g firebase-tools
firebase login
firebase deploy --only hosting
```

Cache headers are set in `firebase.json` — images are immutable for a year, `index.html` for an hour.

### Custom domain

The domain is verified through a TXT record and pointed at Firebase with A records. If the ACME certificate challenge reports conflicting results, the usual cause is leftover registrar parking records — check with:

```bash
nslookup wallpaper.graphics 8.8.8.8
```

Only Firebase IPs should be returned.

## Adding a gallery piece

1. Drop the JPG into `public/img/`
2. Copy an existing `<article class="card">` block, updating the filename, title, description, badge and `data-i` index
3. Add a matching entry to the `SHOTS` array in the script — order must match the `data-i` values
4. `firebase deploy --only hosting`

The card count and `SHOTS` length must stay equal, or the lightbox indices drift.

## Ordering and payment

Briefs arrive by email at yiorgosantoni@gmail.com. Payment is USDT on the **TRC20 (TRON)** network only — the address is shown on the site with a copy button, and the transaction hash goes in the order form so payments can be matched to briefs.

## License

Site code MIT. Gallery images are original work and not covered by that license.

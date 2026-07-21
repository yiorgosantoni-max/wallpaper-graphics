# Deploying

## Where the files go

Unzip so the contents land directly in your project folder:

```
C:\wallpaper-graphics\
  firebase.json
  .firebaserc
  .gitignore
  README.md
  DEPLOY.md
  PRESENCE_SETUP.md
  public\
    index.html
    robots.txt
    sitemap.xml
    img\      24 full-resolution plates
    share\    24 preview cards (1200x630)
    p\        24 per-plate share pages
```

`firebase.json` sits beside `public`, not inside it.

## Deploy

```
cd C:\wallpaper-graphics
firebase deploy --only hosting
```

Expect roughly 75 files. If it reports 1 or 2, the copy did not take.

## Confirm it landed

1. Open the site, press F12, look at the Console. It should log:
   `wallpaper.graphics build share-v11`
2. Check a preview card loads directly:
   `https://wallpaper.graphics/share/jungle-empress.jpg`
3. Check a plate page redirects to the gallery:
   `https://wallpaper.graphics/p/jungle-empress`

If the console build does not match what you just deployed, stop and fix that
first — everything else will be misleading.

## Git

Independent of deploying. `git push` stores history; `firebase deploy` updates
the live site.

```
git add .
git commit -m "what changed"
git push
```

## Custom domain

If the ACME certificate challenge reports conflicting results, leftover
registrar parking records are almost certainly still answering on the apex:

```
nslookup wallpaper.graphics 8.8.8.8
```

Only Firebase IPs should come back. Delete the others at the registrar, disable
any domain-parking toggle (it re-adds them), and wait for the TTL before
retrying.

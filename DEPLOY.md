# wallpaper.graphics — deploy notes

## One-time setup
1. Create a Firebase project at https://console.firebase.google.com (suggested id: `wallpaper-graphics`).
   If you use a different id, update `.firebaserc`.
2. Install the CLI and log in:
   npm install -g firebase-tools
   firebase login

## Deploy
   cd this-folder
   firebase deploy --only hosting

## Connect the domain
Firebase Console -> Hosting -> Add custom domain -> `wallpaper.graphics`.
Add the TXT record it gives you at your registrar to verify, then swap the
A records to the two IPs Firebase provides. Add `www.wallpaper.graphics` as a
redirect to the apex. SSL is issued automatically within ~24h.

## Files
public/index.html    the whole site
public/img/          the four gallery images
public/robots.txt    crawler rules
public/sitemap.xml   update lastmod when you edit the page

## Payment address
USDT, TRC20 network: TKX1h9L4wWjimF8UVWHQLtFYvWv7TTHdnB
Orders email: yiorgosantoni@gmail.com

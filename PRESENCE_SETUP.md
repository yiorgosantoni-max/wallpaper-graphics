# Enabling the live viewer counter

The presence feature needs Firebase Realtime Database switched on and your
config pasted in. Five minutes, no cost at your traffic level.

## 1. Create the database

1. Firebase Console -> your project `wallpapergraphics-b9ca1`
2. Left sidebar -> **Build -> Realtime Database**
3. **Create Database**
4. Pick a region (europe-west1 is closest to Cyprus)
5. Choose **Start in locked mode** - we set proper rules in step 3

## 2. Get your config

1. Console -> gear icon -> **Project settings**
2. Scroll to **Your apps**. If no web app exists, click the `</>` icon and
   register one (any nickname; skip Firebase Hosting setup, you already have it)
3. Copy the `firebaseConfig` object shown

In `public/index.html`, find `const firebaseConfig = {` near the bottom and
replace the whole object with yours. It must include `databaseURL` - if the
console doesn't show that line, copy it from the Realtime Database page
(looks like `https://wallpapergraphics-b9ca1-default-rtdb.europe-west1.firebasedatabase.app`).

## 3. Set the security rules

Realtime Database -> **Rules** tab. Replace everything with:

```json
{
  "rules": {
    "presence": {
      ".read": true,
      ".write": true,
      "$session": {
        ".validate": "newData.hasChildren(['country','at']) || !newData.exists()",
        "country": { ".validate": "newData.isString() && newData.val().length < 40" },
        "at": { ".validate": "newData.isNumber()" }
      }
    },
    "$other": { ".read": false, ".write": false }
  }
}
```

Publish.

These rules allow anonymous writes only under `/presence`, cap the field
sizes, and block everything else in the database. Worst case someone
scripts fake presence entries - annoying, not dangerous, and nothing
sensitive is exposed.

## 4. Deploy and test

```
firebase deploy --only hosting
```

Open the site in two browsers (or one normal + one private window). The pill
in the header should read **2 viewing now**, and the second window should
trigger a toast in the first. Close one and the count drops within seconds.

## Notes

- The API key in client code is normal for Firebase and not a secret. Your
  security rules are what protect the data.
- Country is derived from the browser timezone, not an IP lookup. Nothing
  identifying is stored - just a country string and a timestamp, deleted the
  moment the tab closes.
- Free Spark tier covers 100 simultaneous connections and 10GB/month
  transfer. Presence traffic is tiny.
- If the database is unreachable, the pill hides itself and the rest of the
  site is unaffected.

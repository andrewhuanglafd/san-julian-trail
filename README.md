# The San Julian St Trail

A retro-CRT, Oregon-Trail-style EMS training game built for an EMT-level audience. Pick your seat on the rig (Captain Clipboard, Pump Panel Pete, Tiller Killer, Diesel Bolus Dave, or Stay-and-Play Ray), restock at Supply & Maintenance on a deficit budget, and try to make it back to quarters while running calls that drill OPA, NPA, naloxone, blood glucose checks, and epi auto-injector assists.

It's a single self-contained HTML file. No build step. No backend. Works offline once cached. Installs as a PWA on phones.

---

## What's in the box

```
index.html       The whole game. Open this directly to play.
manifest.json    PWA manifest (makes it installable).
sw.js            Service worker (offline play after first load).
icon.svg         Vector icon.
icon-192.png     PWA icon at 192px.
icon-512.png     PWA icon at 512px (also serves as maskable icon).
firebase.json    Firebase Hosting config (headers, cache rules).
.firebaserc      Firebase project alias (edit before deploying).
.gitignore       Standard ignores.
LICENSE          MIT.
README.md        This file.
```

---

## Play it locally (zero setup)

Double-click `index.html`. It opens in your default browser and is fully playable. Internet is only needed for the Google Fonts (pixel + CRT typefaces) — offline, it falls back to plain monospace and still plays fine.

---

## Deploy: GitHub Pages

Easiest path, no CLI required.

1. Create a new repo on GitHub (public). Name it whatever — `san-julian-trail` is fine.
2. Upload all the files in this folder to the repo (drag-and-drop in the web UI works, or use `git`).
3. In the repo, go to **Settings → Pages**.
4. Under "Build and deployment", set **Source** to **Deploy from a branch**, **Branch** to `main`, **Folder** to `/ (root)`. Save.
5. Wait a minute. Your URL appears at the top of the Pages settings: `https://<your-username>.github.io/<repo-name>/`.

That URL is now shareable. Re-upload `index.html` any time to update it.

### With git from the command line

```bash
cd path/to/this/folder
git init
git add .
git commit -m "San Julian St Trail"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

Then enable Pages in the repo settings as in step 4 above.

---

## Deploy: Firebase Hosting

Free tier covers this easily.

### One-time setup

```bash
# Install the Firebase CLI if you don't have it
npm install -g firebase-tools

# Sign in
firebase login
```

### Create your Firebase project

1. Open the [Firebase Console](https://console.firebase.google.com/).
2. Click **Add project** and follow the prompts. Skip Analytics if you want.
3. Once created, copy the **Project ID** (looks like `your-project-12345`).

### Wire it up and deploy

In this folder:

1. Open `.firebaserc` and replace `REPLACE_WITH_YOUR_FIREBASE_PROJECT_ID` with the Project ID you just copied.
2. From a terminal in this folder, run:

   ```bash
   firebase deploy --only hosting
   ```

That's it. The CLI prints the live URL (e.g. `https://your-project-12345.web.app`).

To update the deployed site after any edit, just run `firebase deploy --only hosting` again.

### Notes

- `firebase.json` already sets reasonable cache headers: `no-cache` on `index.html`, `manifest.json`, and `sw.js` so updates show up immediately, and a one-day cache on the icons.
- If you want a custom domain, add it in the Firebase Hosting dashboard.

---

## Installable PWA

Once the site is live (Pages or Firebase), open it on a phone:

- **iOS Safari**: tap the share button → **Add to Home Screen**.
- **Android Chrome**: tap the menu → **Install app** (or it may prompt you).

It'll launch full-screen like a native app, with the icon and theme color built in. Service worker caches everything on first load, so it plays offline after that.

---

## Editing the game

Everything lives in `index.html`. Inside the `<script>` tag near the top you'll find:

- `CLASSES` — the five seats and their traits.
- `STORE` — Supply & Maintenance items and prices.
- `PROC` — the skill walkthroughs (OPA, NPA, naloxone, glucometer, epi). Each step has `qV` question variants and `o` option arrays.
- `CALLS` — the 10 medical scenarios, each with `sceneVars` / `whoVars` / `qVars` and per-class branches (`crewCan`, `captainSolo`, `mustTransport`).
- `ROADS`, `SCAVS`, `CREWCHECKS`, `FILTHS`, `BARRIERS`, `CATAS` — the random street events.

Add your own calls or events by copying an existing entry's shape. After any edit, re-deploy (push to GitHub or `firebase deploy --only hosting`).

---

## License

MIT — see `LICENSE`. Note: this is a training simulation, not a substitute for accredited EMS education or your department's protocols.

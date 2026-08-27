# PhysioDesk — hosting and data setup

Everything below costs nothing. No card, no trial, no expiry.

---

## What you're deploying

Four static files and an icon folder. There is no server and no backend to run, which is exactly why it's free — you're only hosting files.

```
index.html              the whole app
sw.js                   service worker (makes it work offline)
manifest.webmanifest    tells the phone it's an installable app
icons/                  app icons
DEPLOY.md               this file
```

---

## Step 1 — Put it online (5 minutes)

You need HTTPS. A PWA will not install or run offline over plain HTTP. Every option below gives you HTTPS automatically.

### Option A — GitHub Pages (recommended)

1. Create a free GitHub account.
2. Make a new repository, e.g. `physiodesk`. Private repos work with Pages on the free plan.
3. Upload these files to the root of the repo (drag and drop works — use **Add file → Upload files**).
4. Go to **Settings → Pages**. Under *Source* pick **Deploy from a branch**, branch `main`, folder `/ (root)`. Save.
5. Wait a minute. Your app is at `https://<username>.github.io/physiodesk/`.

To update later, upload the changed file over the old one.

### Option B — Cloudflare Pages

Slightly faster in India and gives you a nicer URL. Sign up free, **Create a project → Direct Upload**, drag the folder in. Done.

### Option C — Netlify Drop

Go to `app.netlify.com/drop` and drag the folder onto the page. No account needed to start. Easiest of the three, but the URL is random unless you sign up.

**Cost on all three: ₹0.** These free tiers are for static sites and have bandwidth allowances in the hundreds of gigabytes. One person using a 90 KB app will never come close.

---

## Step 2 — Install it on the phone

**Android / Chrome:** open the URL, tap the ⋮ menu, **Add to Home screen** (or **Install app**).

**iPhone / Safari:** open the URL, tap the Share button, **Add to Home Screen**. It must be Safari — Chrome on iOS cannot install PWAs.

It now opens full-screen with no browser chrome, appears in the app drawer, and works with no signal.

> **iPhone warning worth knowing.** iOS clears the storage of web apps that aren't opened for **seven days**. For daily use this never triggers. But it is the single strongest reason to turn on cloud backup in Step 3 if you're on an iPhone. On Android, installed PWAs get persistent storage and are not evicted.

---

## Step 3 — Where the data lives

### Layer 1: the device (always on, nothing to configure)

Every change is written to **IndexedDB** on the phone immediately — not on a timer, not on close. The app works fully with no internet; this is the point, since home visits happen in lifts and stairwells.

The app also keeps a **daily snapshot** of the whole database, holding the last 14. If something is deleted by mistake and you notice a week later, that day's copy is still there.

### Layer 2: export files (do this monthly)

**Backup & sync → Export backup** downloads a single JSON file with everything in it. Put it in Google Drive, WhatsApp it to yourself, whatever. This is the only copy that no browser, phone or company can delete. **Restore from file** reads it back.

### Layer 3: cloud mirror (optional, free, recommended)

Mirrors the whole practice to a private Firebase document. If the phone is lost, stolen or wiped, you open the app on a new phone, enter the same details, and everything comes back.

The whole database is stored as **one document**, written at most once every few seconds of editing. A solo practice generates roughly 100 writes a day against a free allowance of 20,000 — you will not approach the limit, and Firebase's free Spark plan requires no card and does not expire.

#### Setting it up

1. Go to `console.firebase.google.com` → **Add project**. Give it a name, turn Google Analytics off, create.
2. **Build → Firestore Database → Create database.** Pick a location near you (`asia-south1` for India). Start in **production mode**.
3. Go to the **Rules** tab and replace what's there with this, then **Publish**:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /practices/{uid} {
         allow read, write: if request.auth != null && request.auth.uid == uid;
       }
     }
   }
   ```

   This means only your signed-in account can read or write your data. Nobody else, including anyone who finds your app URL.

4. **Build → Authentication → Get started → Email/Password → Enable → Save.**
5. Still in Authentication, **Users → Add user**. Enter an email and a password. This is your login — it doesn't need to be a real inbox, but use a strong password.
6. **Project settings** (gear icon) **→ Your apps → Web** (the `</>` icon). Register the app. You'll get a config block that looks like this:

   ```js
   const firebaseConfig = {
     apiKey: "AIzaSy...",
     authDomain: "your-project.firebaseapp.com",
     projectId: "your-project",
     appId: "1:123456789:web:abc123"
   };
   ```

7. In PhysioDesk, open **Backup & sync → Set up cloud backup**, paste those four values plus the email and password from step 5, and tap **Connect**.

The chip next to *Backup & sync* in the menu turns to ✓ when it's syncing.

#### Moving to a new phone

Install the app from the same URL, open **Backup & sync**, enter the same six details, and tap **Pull from cloud**.

#### A note on the password

It's stored in the phone's local storage so the app can reconnect silently each morning. On a phone with a lock screen this is a reasonable trade — it's the same exposure as a password saved in a browser. Don't reuse a password you use elsewhere, and if the phone is lost, change it in the Firebase console and the old device loses access.

---

## What can still go wrong, and what covers it

| Risk | Covered by |
|---|---|
| Closed the app mid-edit | Writes land in IndexedDB within 250 ms, plus a flush when the tab hides |
| No signal on a home visit | Works fully offline; syncs when you reconnect |
| Deleted a patient by mistake | 14 days of daily snapshots on the device |
| Cleared browser data | Cloud mirror, or your last export file |
| Phone lost, stolen or broken | Cloud mirror, or your last export file |
| iOS 7-day eviction | Daily use prevents it; cloud mirror insures it |
| Firebase account lost | Your export files, which depend on nobody |

The pattern to internalise: **the device is fast, the cloud is insurance, the export file is the one you actually own.** Export once a month and you are never more than a month from whole.

---

## Updating the app later

Replace `index.html` on the host. Then bump the version string at the top of `sw.js`:

```js
const VERSION = 'physiodesk-v2';   // was v1
```

Without that change the service worker keeps serving the old cached copy and your update won't appear. The new version installs on the next launch, and a second launch after that activates it.

---

## Running it locally to test

```bash
cd physiodesk
python3 -m http.server 8080
```

Then open `http://localhost:8080`. Service workers are allowed on `localhost` without HTTPS, so offline mode is testable here.

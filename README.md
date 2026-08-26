# ✈️ Where the Hell Were You?

A real-time office party guessing game. Players look at vacation photos on a projected screen and tap a world map on their phone to guess where the photo was taken. Scores are based on distance accuracy and submission speed.

---

## 📁 Files

| File | Purpose | Who uses it | How |
|---|---|---|---|
| `host.html` | Control panel — manage rounds, run the game | Host only | Open locally on host's laptop, never shared |
| `display.html` | Audience screen — the projected view | Everyone in the room | Open on host's laptop, screenshared to projector |
| `player.html` | Player interface | All players | Hosted on GitHub Pages, opened on their phones |

> `host.html` and `display.html` are always open on the **same laptop**, in two separate browser windows. Only `display.html` is screenshared.

---

## 🏗️ How the game works

### Round flow (fully automatic once started)

```
GET READY (configurable seconds)
  Projected screen:  "Round N of Total — Get Ready!" + countdown
  Player phones:     "Get ready..." + countdown

GUESSING (configurable seconds)
  Projected screen:  Full-screen photo + "Where was Marie?" + timer
  Player phones:     World map + timer + submit button

REVEAL (configurable seconds)
  Projected screen:  Map with ⭐ correct location + coloured player pins
                     + two leaderboards (this round / overall)
  Player phones:     Points this round · km from answer · total score · rank

→ Automatically advances to next round
```

### Scoring formula

```
distance_score = max(0, 1000 × (1 − distance_km ÷ 10,000))
speed_bonus    = 1.0 → 1.5  (scales with how early in the window you submitted)
final_score    = round(distance_score × speed_bonus)
```

| Scenario | Score |
|---|---|
| Perfect guess, submitted in 1st second | ~1,450 pts |
| Perfect guess, submitted in last second | ~1,050 pts |
| Maximum possible (instant + perfect) | 1,500 pts |

Tie-breaking: equal scores are ranked by submission speed.

---

## ⚙️ Setup Guide

You've created the Firebase project. Here's everything left to do, in order.

---

### Step 1 — Create the Firestore database

1. In the [Firebase console](https://console.firebase.google.com), open your project
2. In the left sidebar, click **Databases & Storage** → **Firestore**
3. Click **Create database** — you'll go through three sub-steps:

**Sub-step 1 — Select edition:**
Select **Standard edition** (already selected by default) → click **Next**

**Sub-step 2 — Database ID & location:**
- Leave **Database ID** as `(default)`
- Change **Location** to the region closest to your office (e.g. `eur3 (Europe)`) — this cannot be changed later
- Click **Next**

**Sub-step 3 — Configure:**
- Select **Start in test mode** — this keeps the database open while you set up the proper security rules in the next step. Don't worry about production mode; you'll overwrite the rules immediately anyway.
- Click **Create**

4. Wait a few seconds for the database to provision

---

### Step 2 — Get your app config values

1. In the left sidebar, click **Settings** → **General**
2. Scroll down to **Your apps** and click the **</>** (Web) icon
3. You'll go through two sub-steps:

**Sub-step 1 — Register app:**
- Enter any nickname (e.g. `where-the-hell`)
- Leave **Firebase Hosting** unticked — you don't need it, you're using GitHub Pages instead
- Click **Register app**

**Sub-step 2 — Add Firebase SDK:**
- Ignore the npm instructions and the full code block — these are for a different kind of setup
- Just copy these three values from the config block shown on screen:
  - `apiKey`
  - `projectId`
  - `appId`
- Ignore everything else (`authDomain`, `storageBucket`, `messagingSenderId`)

4. Click **Continue to console**

Keep those three values handy — you'll need them in Steps 4 and 7.

---

### Step 3 — Apply the Firestore security rules

1. In the left sidebar, click **Firestore Database** → **Rules** tab
2. Delete everything in the editor and paste the following:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Game state: open for host to read and write
    match /game/state {
      allow read, write: if true;
    }

    // Players: can register once with a valid name only
    match /players/{playerId} {
      allow read: if true;
      allow create: if request.resource.data.keys().hasOnly(['name', 'joinedAt'])
                    && request.resource.data.name is string
                    && request.resource.data.name.size() >= 1
                    && request.resource.data.name.size() <= 20;
      allow update, delete: if false;
    }

    // Guesses: players can submit valid coordinates only
    match /guesses/{playerId} {
      allow read: if true;
      allow create, update: if request.resource.data.keys().hasOnly(['name', 'lat', 'lng', 'submittedAt'])
                             && request.resource.data.lat is number
                             && request.resource.data.lat >= -90
                             && request.resource.data.lat <= 90
                             && request.resource.data.lng is number
                             && request.resource.data.lng >= -180
                             && request.resource.data.lng <= 180
                             && request.resource.data.submittedAt is number;
      allow delete: if false;
    }
  }
}
```

3. Click **Publish**

---

### Step 4 — Paste the Firebase config into player.html

1. Open `player.html` in any text editor (Notepad, TextEdit, VS Code — anything works)
2. Find this block near the top of the `<script>` section:

```js
const FIREBASE_CONFIG = {
  apiKey:     "REPLACE_WITH_YOUR_API_KEY",
  authDomain: "REPLACE_WITH_YOUR_PROJECT_ID.firebaseapp.com",
  projectId:  "REPLACE_WITH_YOUR_PROJECT_ID",
  appId:      "REPLACE_WITH_YOUR_APP_ID",
};
```

3. Replace the placeholder values with the ones from Step 2
4. Save the file

> `display.html` connects to Firebase at runtime via its own setup screen — no editing needed.

---

### Step 5 — Host player.html on GitHub Pages

1. Go to [github.com](https://github.com) and sign up for a free account if needed
2. Click **+** → **New repository**
3. Name it `post-summer-game` → set to **Public** → click **Create repository**
4. Click **uploading an existing file** → drag `player.html` into the upload area → click **Commit changes**
5. Go to **Settings** → **Pages** (left sidebar)
6. Under **Branch**, select `main` → click **Save**
7. After about 1 minute, the player URL is live at:

```
https://YOUR-USERNAME.github.io/post-summer-game/player.html
```

Share this URL with players before the event. A QR code on the projector works well.

> `host.html` and `display.html` stay on your laptop only — never upload them.

---

### Step 6 — Prepare your photo data

Collect all submitted vacation photos via Teams or email. For each photo, you need four pieces of information:

| Field | Description | Example |
|---|---|---|
| **Photo file** | JPG or PNG, any size | `marie_paris.jpg` |
| **Submitted by** | First name of the person | `Marie` |
| **Location name** | City or area shown on reveal | `Paris, France` |
| **Latitude** | Decimal degrees, 4 decimal places | `48.8566` |
| **Longitude** | Decimal degrees, 4 decimal places | `2.3522` |

**Getting coordinates:**
1. Go to [maps.google.com](https://maps.google.com)
2. Find the exact spot where the photo was taken
3. Right-click it
4. Click the coordinates shown at the top of the menu — this copies them

**Photo format:**
- Any standard image format works (JPG, PNG, HEIC will be converted automatically)
- Photos are stored locally in your browser — they never leave your laptop
- There is no size limit, but keep photos under 5MB for fast loading
- Landscape orientation works best on the projected screen
- You do not need to rename the files — you pick them one by one in the host panel

> Because photos are stored in your browser's localStorage, you must always use **the same browser on the same laptop** when managing rounds. Clearing browser data will erase the photos.

---

### Step 7 — Enter your rounds in host.html

1. Open `host.html` by double-clicking it (no server needed)
2. Go to the **Settings** tab:
   - Paste your Firebase config values
   - Set your join code (e.g. `TRAVEL42`)
   - Set your timings:
     - **Get ready:** 5 seconds recommended
     - **Guessing timer:** 10–15 seconds recommended
     - **Reveal duration:** 15–20 seconds recommended
   - Click **Save Settings**
3. Go to the **Rounds** tab:
   - Click **+ Add Round** for each photo
   - For each round, click to expand it and fill in:
     - Upload the photo (📂 button)
     - Submitted by name
     - Location name
     - Latitude and longitude
   - A green **✓ Ready** badge appears when all fields are complete
   - Use **↑ ↓** to reorder rounds
   - You can close and reopen `host.html` later — everything is saved automatically

---

### Step 8 — Test the game before the event

Do a full run-through with one colleague the day before, using your real configuration.

1. Open `host.html` on your laptop (double-click)
2. Open `display.html` on your laptop in a second browser window
3. In `display.html`, enter your Firebase config and click **Connect**
4. In `host.html`, go to the **Game** tab → click **Connect & Open Lobby**
5. The join code appears on `display.html`
6. Have your colleague open the GitHub Pages URL on their phone, enter their name and join code
7. Once they appear in the lobby, click **▶ Start Game**
8. Run through 2–3 rounds end to end — check that:
   - The photo appears correctly on `display.html`
   - The timer counts down on both screens
   - The map and guesses appear correctly on reveal
   - The leaderboards show correct scores
   - Auto-advance works as expected
9. If anything looks wrong, adjust timings in the Settings tab and retest

---

### Step 9 — On game day

1. Prepare a QR code linking to the GitHub Pages player URL — display it on the projector before the game starts
2. Open `display.html` on your laptop → connect to Firebase → screenshare this window to the projector
3. Open `host.html` on your laptop in a separate window (do not screenshare this)
4. In `host.html` → Game tab → **Connect & Open Lobby**
5. The lobby and join code appear on the projected screen
6. Players scan the QR code, enter their name and join code
7. Once everyone has joined, click **▶ Start Game**
8. The game runs fully automatically from this point

---

## 🆓 Cost

| Service | Plan | Cost |
|---|---|---|
| Firebase Firestore | Spark (free) | €0 |
| GitHub Pages | Free | €0 |
| Leaflet maps | Open source | €0 |
| **Total** | | **€0** |

Firebase free tier limits: 50,000 reads/day and 20,000 writes/day — far more than needed for a 40-round game with 50 players.

---

## 🛠️ Troubleshooting

| Problem | Fix |
|---|---|
| Players can't join | Check that the Firebase config in `player.html` matches your project exactly |
| Wrong join code error | Make sure the host has clicked Connect in the Game tab before players try to join |
| display.html shows nothing | Check Firebase config entered in the connect screen matches your project |
| Photos not appearing | Make sure all rounds show ✓ Ready in the Rounds tab before connecting |
| Game doesn't auto-advance | Check that `host.html` stays open throughout — closing it stops the game loop |
| Scores seem wrong | Verify lat/lng are entered as decimal degrees (e.g. 48.8566), not degrees/minutes/seconds |
| Firestore rules expired | Firebase test mode rules expire after 30 days — go to Firestore → Rules and push the expiry date forward, or use the rules from Step 3 above |
| Photos lost after clearing browser | Photos are stored in localStorage — avoid clearing browser data on the host laptop |


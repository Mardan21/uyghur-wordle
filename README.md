# Sozle

**Sozle** is a self-contained, single-file Wordle clone that uses Uyghur (Latin/ULY) 5-letter words.
The puzzle changes every day at **midnight US Eastern time**, exactly like the New York Times Wordle.

Everything lives in [`index.html`](index.html) — no build step, no server, no dependencies.

---

## 1. How the NYT Wordle works (architecture summary)

**What it is.** Wordle is a 100% **client-side** browser game. There is no game server doing
the scoring — all logic (input, coloring, win/loss, stats) runs in JavaScript in your browser.

**The tech.**
- Originally (Josh Wardle's version) it was a plain static HTML/JS site with the **entire answer
  list hard-coded into the JavaScript bundle**. The daily answer was chosen purely by date math in
  the browser: `floor(days since 2021-06-19)` indexed into the answer array.
- After The New York Times acquired it, the front end was rebuilt with **React** and folded into the
  NYT Games platform. To stop people from reading tomorrow's answers out of the JS source, NYT moved
  the solution to a tiny **JSON API**:
  `https://www.nytimes.com/svc/wordle/v2/YYYY-MM-DD.json`
  → returns `{ "id": ..., "solution": "xxxxx", ... }` for that calendar day.
- **Daily rollover** happens at local midnight; the client requests the JSON for the current date.
- **State & stats are stored in `localStorage`** in the browser (`gameState`, `statistics`), which is
  why your progress, streak, and distribution survive refreshes but don't sync across devices unless
  you're logged in.
- **Hosting/CDN.** It's static assets (HTML/CSS/JS) served from NYT's infrastructure behind a CDN
  (Fastly/Akamai/CloudFront-style edge caching). Because the heavy lifting is client-side, it scales
  trivially — the server only ships static files plus one small JSON per day.
- **Sharing** is generated client-side: the colored-square grid (🟩🟨⬜) is built from your guess
  results and copied to the clipboard / Web Share API. No image, just text + emoji.

**The one trade-off NYT accepts:** the answer is delivered to the client (in the JSON), so a
determined user can always read it from the network tab. That's fine for a casual daily game.

### How this clone mirrors that
| NYT Wordle | This project |
|---|---|
| React SPA, static assets on a CDN | Single static `index.html`, host anywhere |
| Answer via daily JSON API | Answer via date-math against an in-file `WORDS` array |
| Daily rollover at midnight | Rollover at **midnight America/New_York** (DST-aware) |
| `localStorage` for state + stats | `localStorage` (`uyghurWordle.state`, `uyghurWordle.stats`) |
| Emoji share grid | Emoji share grid (🟩🟨⬛), clipboard / Web Share |

---

## 2. How the daily word is chosen

```
dayNumber = floor( (todayEastern - EPOCH) / 1 day )
index     = dayNumber mod WORDS.length
solution  = WORDS[index]
```

- `todayEastern` is computed with `Intl.DateTimeFormat(..., { timeZone: "America/New_York" })`,
  so it follows EST/EDT automatically — no manual daylight-saving handling.
- `EPOCH` is set to `2026-06-15` (the first puzzle = `YUREK`). Change it in `index.html` to shift
  the schedule.
- The list **cycles**: after the last of the 36 words it loops back to the first. Add more words to
  `WORDS` to extend the run before it repeats.

To change/extend the words, edit the `WORDS` array near the top of the `<script>` in `index.html`.
Each entry must be **uppercase and exactly 5 characters**.

---

## 3. Trying it locally

Just open the file in a browser:

```bash
open index.html            # macOS
```

Or serve it (recommended, so `localStorage` and clipboard behave like production):

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

---

## 4. Adding it to a Squarespace / WordPress news site

**Yes, this is fully possible.** Because the game is one static file, you have two clean options.
The **iframe approach is recommended** — it isolates the game's CSS/JS from your site theme so
nothing collides.

### Step 1 — Host `index.html` at a public URL
Pick any static host:
- Your existing site's file storage / a subfolder on the same web host (e.g. `https://yoursite.com/games/uyghur-wordle/index.html`)
- **GitHub Pages**, **Netlify**, **Cloudflare Pages**, or **Vercel** (all free, drag-and-drop the file)

Note the final URL.

### Step 2a — Embed via iframe (recommended)

**Squarespace:** Edit a page → add a **Code Block** (or an *Embed* block) → paste:

```html
<iframe
  src="https://YOUR-HOST/uyghur-wordle/index.html"
  style="width:100%; max-width:520px; height:760px; border:0; margin:0 auto; display:block;"
  title="Sozle"
  loading="lazy"></iframe>
```

**WordPress:**
- Block editor: add a **Custom HTML** block and paste the same `<iframe>` snippet, **or**
- Use the dedicated *Embed/HTML* block. (On WordPress.com lower tiers, custom HTML/iframes may be
  limited — a plugin like "Iframe" or an upgraded plan removes that restriction. Self-hosted
  WordPress.org has no such limit.)

*(Optional) auto-resize height:* the game posts its height to the parent page. Add this once on the
embedding page to make the iframe grow/shrink to fit:

```html
<script>
window.addEventListener("message", function (e) {
  if (e.data && e.data.uyghurWordleHeight) {
    var f = document.querySelector('iframe[title="Sozle"]');
    if (f) f.style.height = e.data.uyghurWordleHeight + "px";
  }
});
</script>
```

### Step 2b — Paste inline (no separate hosting)
Since everything is self-contained, you can also open `index.html`, copy **everything inside
`<body>` … `</body>` plus the `<style>` block**, and paste it into a Squarespace **Code Block** or a
WordPress **Custom HTML** block. This works, but the game's CSS then shares the page with your theme
and may need class-name tweaks to avoid conflicts — which is exactly why the iframe is preferred.

---

## 5. Files
- [`index.html`](index.html) — the entire game (HTML + CSS + JS).
- `README.md` — this document.

## 6. Notes & possible next steps
- **Word script.** The supplied words are in Uyghur Latin (ULY) transliteration, so the on-screen
  keyboard is Latin QWERTY. If you later want the Arabic-script Uyghur alphabet (right-to-left), that
  needs a different keyboard layout and RTL board — happy to add it.
- **Guess validation.** Like real Wordle, guesses must be known words — random 5-letter strings are
  rejected ("Söz tizimlikte yoq"). Accepted guesses = the 36 answers plus the `EXTRA_VALID_GUESSES`
  list near the top of the script. That extra list is a small starter set of common Uyghur words;
  **review and extend it** (transliteration varies) so players can use more real words as guesses.
- **Localization.** UI strings are in Uyghur Latin; swap them in `index.html` if you want Arabic-script
  or English labels.

# Backloggd CSV Export

Export your [Backloggd](https://backloggd.com) library to a CSV file — including the completion statuses (Completed / Retired / Shelved / Abandoned) that the site's grid normally hides. Backloggd has no built-in export, so this fills the gap.

No installs, no account access, nothing to download. It's a single script you paste into your browser's console.

## Quick start

1. Log in to Backloggd and open your games page: `https://backloggd.com/u/<your-username>/games`
2. Press **F12** and click the **Console** tab. *(If it blocks pasting, type `allow pasting` and press Enter first.)*
3. **Copy the whole script below**
4. A `backloggd_<username>.csv` downloads when it finishes. That's it.

It runs in your own logged-in tab, so it reads **your** ratings and statuses and works even on a private profile.

```js
/* ============================================================================
   Backloggd → CSV exporter  (Title, Rating, Status, Backlog)
   ----------------------------------------------------------------------------
   Status holds the play status — one of:
       Played · Completed · Retired · Shelved · Abandoned · Playing · Wishlist
       (blank if a game is only backlogged)
   Backlog is a separate Yes/blank column, since a game can be e.g. Shelved AND
   backlogged at the same time. Status comes from each card's data-status-title
   (play_type as fallback); rating comes from data-rating (÷2); backlog comes
   from whether the game appears under the Backlog tab.

   HOW TO RUN:
     1. Log in to Backloggd.
     2. Go to:  https://backloggd.com/u/<your-username>/games
     3. F12 → Console tab.  (If paste is blocked, type "allow pasting" + Enter.)
     4. Paste this whole file, press Enter. A CSV downloads when it finishes.
   It runs in your own session, so it reads YOUR data and works on a private
   profile. Progress + a status breakdown print to the console.

   Built with Claude (Anthropic). As with any console script, give it a read
   before you paste it.
   ========================================================================== */
(async () => {
  const POLITE_DELAY_MS = 400;  // pause between page requests
  const MAX_PAGES = 100;        // safety cap per tab
  const TABS = ["played", "playing", "backlog", "wishlist"];
  // Valid played sub-statuses (as Backloggd writes them in data-status-title).
  const PLAYED_SET = new Set(["Played", "Completed", "Retired", "Shelved", "Abandoned"]);

  const m = location.pathname.match(/\/u\/([^/]+)/);
  if (!m) { alert("Open this on your Backloggd profile first (a /u/<username>/… page)."); return; }
  const username = m[1];
  const origin = location.origin;
  const base = `${origin}/u/${username}/games`;

  // The status-filter URL scheme has changed across Backloggd versions → try each form.
  const urlForms = (seg) => [`${base}/added/type:${seg}`, `${base}/type:${seg}`, `${base}/${seg}`];

  async function fetchDoc(url) {
    const res = await fetch(url, { credentials: "include" });
    if (!res.ok) return null;
    return new DOMParser().parseFromString(await res.text(), "text/html");
  }
  function findNext(doc) {
    return doc.querySelector('a[aria-label="Next"], a[rel="next"], .pagination a.next_page')
      ?.getAttribute("href") || null;
  }
  function cardsIn(doc) {
    let c = doc.querySelectorAll(".rating-hover");
    if (!c.length) c = doc.querySelectorAll(".card.game-cover");
    return c;
  }

  function parseCard(card) {
    const gameId = card.querySelector("[game_id]")?.getAttribute("game_id")
      || card.getAttribute("game_id") || "";
    const title = card.querySelector("img.card-img")?.getAttribute("alt")?.trim()
      || card.querySelector(".game-text-centered")?.textContent?.trim()
      || card.querySelector("img")?.getAttribute("alt")?.trim();
    if (!title) return null;

    // Rating: data-rating is a 1–10 scale (10 = 5 stars). Fallback to the star overlay width.
    let rating = "";
    const dr = parseFloat(card.querySelector("[data-rating]")?.getAttribute("data-rating") ?? "");
    if (!isNaN(dr) && dr > 0) rating = +(dr / 2).toFixed(2);
    if (rating === "") {
      const wm = (card.querySelector(".stars-top")?.getAttribute("style") || "").match(/width:\s*([\d.]+)%/);
      if (wm) { const r = parseFloat(wm[1]) / 20; if (r > 0) rating = +r.toFixed(2); }
    }

    // Played sub-status: prefer data-status-title, fall back to the gamepad button's play_type.
    let sub = card.querySelector("[data-status-title]")?.getAttribute("data-status-title")?.trim() || "";
    if (!PLAYED_SET.has(sub)) {
      const pt = card.querySelector(".played-btn-container button[play_type]")?.getAttribute("play_type")
        || card.querySelector("[play_type]")?.getAttribute("play_type") || "";
      const cap = pt ? pt.charAt(0).toUpperCase() + pt.slice(1).toLowerCase() : "";
      sub = PLAYED_SET.has(cap) ? cap : "";
    }
    return { gameId, title, rating, sub };
  }

  const games = new Map(); // key -> { gameId, title, rating, tabs:Set, sub }
  let playedCards = 0, subFound = 0;

  for (const tab of TABS) {
    let url = null, doc = null;
    for (const form of urlForms(tab)) {
      const d = await fetchDoc(form);
      if (d && cardsIn(d).length) { url = form; doc = d; break; }
    }
    if (!url) { console.warn(`⚠️  "${tab}": no results.`); continue; }
    console.log(`📥 ${tab}: scraping…`);

    let page = 1, prevKey = "";
    while (doc && page <= MAX_PAGES) {
      const cards = cardsIn(doc);
      if (!cards.length) break;
      const titles = [];
      cards.forEach((cardEl) => {
        const c = parseCard(cardEl);
        if (!c) return;
        titles.push(c.title);
        if (tab === "played") { playedCards++; if (c.sub) subFound++; }
        const key = c.gameId || `t:${c.title}`;
        const g = games.get(key) || { gameId: c.gameId, title: c.title, rating: "", tabs: new Set(), sub: "" };
        if (c.rating !== "") g.rating = c.rating;
        if (c.sub) g.sub = c.sub;
        g.tabs.add(tab);
        games.set(key, g);
      });
      const key = titles.join("|");
      console.log(`   ${tab} · page ${page}: +${cards.length}  (unique total ${games.size})`);
      if (key === prevKey) break;         // Backloggd repeats the last page past the end
      prevKey = key;
      const href = findNext(doc);
      if (!href) break;
      url = new URL(href, origin).href;
      page++;
      await new Promise((r) => setTimeout(r, POLITE_DELAY_MS));
      doc = await fetchDoc(url);
    }
  }

  if (!games.size) { alert("No games found. Make sure you're logged in and on your /games page."); return; }
  if (playedCards > 0 && subFound === 0) {
    console.warn("⚠️  Found played games but no data-status-title/play_type in the fetched HTML — " +
      "sub-statuses will all show as 'Played'. Let me know and I'll adjust.");
  }

  // Play status (single value) and Backlog (separate flag) are independent.
  const playStatusFor = (g) => {
    if (PLAYED_SET.has(g.sub)) return g.sub;
    if (g.tabs.has("playing")) return "Playing";
    if (g.tabs.has("wishlist")) return "Wishlist";
    return "";  // e.g. backlog-only
  };
  const backlogFor = (g) => (g.tabs.has("backlog") ? "Yes" : "");

  const esc = (v) => { const s = String(v ?? ""); return /[",\n\r]/.test(s) ? `"${s.replace(/"/g, '""')}"` : s; };
  const lines = [["Title", "Rating", "Status", "Backlog"]];
  for (const g of games.values()) lines.push([g.title, g.rating, playStatusFor(g), backlogFor(g)]);
  const csv = lines.map((r) => r.map(esc).join(",")).join("\r\n");

  const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = `backloggd_${username}.csv`;
  document.body.appendChild(a); a.click(); a.remove();
  URL.revokeObjectURL(a.href);

  const counts = {};
  let backlogged = 0;
  for (const g of games.values()) {
    const s = playStatusFor(g) || "(backlog only)";
    counts[s] = (counts[s] || 0) + 1;
    if (g.tabs.has("backlog")) backlogged++;
  }
  console.log(`✅ Done — ${games.size} games → backloggd_${username}.csv  (${backlogged} backlogged)`);
  console.table(counts);
})();
```

## What you get

| Column | Values |
| --- | --- |
| **Title** | Game name |
| **Rating** | Your rating, `0`–`5` (blank if unrated) |
| **Status** | `Played` · `Completed` · `Retired` · `Shelved` · `Abandoned` · `Playing` · `Wishlist` (blank if a game is only backlogged) |
| **Backlog** | `Yes` / blank — a separate column, since a game can be e.g. *Shelved* **and** backlogged at the same time |

## How it works

It walks your Played / Playing / Backlog / Wishlist tabs and reads each game card directly: title from the cover image's `alt`, rating from the card's `data-rating`, status from `data-status-title` (falling back to the gamepad button's `play_type`), and backlog from whether the game shows up under the Backlog tab. No journal scraping and no per-game requests — just a handful of page loads that cover your whole library.

Built against the current Backloggd layout; if the site changes its markup, the selectors may need updating.

## Disclaimer

Built with the help of [Claude](https://claude.ai).

## License

[MIT](LICENSE)

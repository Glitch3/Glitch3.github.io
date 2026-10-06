<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Sound map</title>
<style>
  body { font-family: Arial, sans-serif; background: #fff; color: #000; margin: 10px; }
  a { color: #000; }
  #settings { display: none; border: 1px solid #000; padding: 10px; margin: 10px 0; max-width: 400px; font-size: 14px; }
  #settings.open { display: block; }
  #settings label { display: block; margin: 4px 0; }
  li { margin: 4px 0; }
</style>
</head>
<body>
<div>
  <b>Sound map</b>
  <form id="search" style="display:inline">
    <input id="q" type="text" placeholder="Type an artist" autocomplete="off" aria-label="Artist name">
    <button type="submit">Search</button>
  </form>
  <button id="settingsBtn" aria-expanded="false">Data source</button>
</div>

<div id="settings">
  <label><input type="radio" name="src" value="lastfm" checked> Last.fm</label>
  <label><input type="radio" name="src" value="claude"> Claude</label>
  <label><input type="radio" name="src" value="sample"> Sample data (works offline)</label>
  <p>Claude only works inside the Claude preview. Last.fm is blocked in the preview but works once you host this file. Sample data covers about 25 indie folk artists, starting from Big Thief.</p>
</div>

<p id="status" aria-live="polite"></p>
<div id="results"><p>Type an artist to see a list of similar artists, most similar first. Click any name to see artists similar to them.</p></div>

<script>
const statusEl = document.getElementById('status');
const resultsEl = document.getElementById('results');
const cache = new Map();
// Paste your Last.fm API key here (from https://www.last.fm/api/account/create).
// Only the API key goes here, never the "shared secret".
const LASTFM_API_KEY = '490d2260c0a13cc21edc04650c9e5235';

let source = 'lastfm';

/* ---------- data sources ---------- */

async function fromClaude(artist) {
  const prompt = `You are a music similarity engine. For the artist "${artist}", return ONLY a JSON object, no prose and no code fences:
{"artist":"<correctly spelled name>","similar":[{"name":"...","score":0.0}]}
Rules: "similar" has 26 real artists, sorted by score; score is 0.3-1.0 for how alike they sound and appeal to the same listeners. If the artist is unknown, return {"artist":null}.`;
  const res = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "claude-sonnet-4-6",
      max_tokens: 1000,
      messages: [{ role: "user", content: prompt }]
    })
  });
  if (!res.ok) throw new Error(`Claude returned an error (${res.status}). Try again, or switch to Sample data.`);
  const data = await res.json();
  const text = (data.content || []).map(c => c.text || "").join("");
  const json = JSON.parse(text.replace(/```json|```/g, "").trim());
  if (!json.artist) throw new Error(`No artist called "${artist}" found. Check the spelling.`);
  return json;
}

async function fromLastfm(artist) {
  const key = LASTFM_API_KEY;
  if (!key || key === 'PASTE_YOUR_KEY_HERE') throw new Error("No Last.fm API key set. Add it to LASTFM_API_KEY in the page's code.");
  const url = `https://ws.audioscrobbler.com/2.0/?method=artist.getsimilar&autocorrect=1&limit=30&format=json`
    + `&artist=${encodeURIComponent(artist)}&api_key=${encodeURIComponent(key)}`;
  const data = await (await fetch(url)).json();
  if (data.error) throw new Error(data.message || "Last.fm returned an error.");
  const list = data.similarartists.artist;
  if (!list.length) throw new Error(`Last.fm has no similar artists for "${artist}".`);
  return {
    artist: data.similarartists["@attr"].artist,
    similar: list.map(a => ({ name: a.name, score: +a.match })),
  };
}

// Offline sample: each artist has a position in a made-up "taste space".
// Similarity = closeness in that space, so every name in the list is clickable.
const SAMPLE = {
  "Big Thief": [0, 0], "Adrianne Lenker": [-0.6, 0.4], "Buck Meek": [-0.3, 1.1],
  "Phoebe Bridgers": [1.2, -0.5], "boygenius": [1.5, -0.2], "Julien Baker": [1.9, 0.3],
  "Lucy Dacus": [1.8, -0.9], "Waxahatchee": [0.9, 1.2], "Hand Habits": [-0.2, 1.6],
  "Angel Olsen": [1.0, 0.6], "Sharon Van Etten": [1.6, 1.0], "Weyes Blood": [-1.4, -0.6],
  "Bon Iver": [-1.3, 1.3], "Sufjan Stevens": [-2.0, 0.6], "Fleet Foxes": [-1.9, 1.7],
  "Andy Shauf": [-1.0, 2.1], "Snail Mail": [2.4, -1.4], "Soccer Mommy": [2.6, -0.7],
  "Japanese Breakfast": [2.0, -2.0], "Mitski": [1.4, -1.6], "Florist": [-0.4, -1.4],
  "Grouper": [-1.5, -1.7], "Cassandra Jenkins": [-0.9, -0.9], "Alex G": [0.5, -1.9]
};
function fromSample(artist) {
  const name = Object.keys(SAMPLE).find(n => n.toLowerCase() === artist.toLowerCase());
  if (!name) throw new Error(`Sample data only includes ${Object.keys(SAMPLE).slice(0, 4).join(', ')} and a few others. Try Big Thief.`);
  const dist = (a, b) => Math.hypot(SAMPLE[a][0] - SAMPLE[b][0], SAMPLE[a][1] - SAMPLE[b][1]);
  const similar = Object.keys(SAMPLE).filter(n => n !== name)
    .map(n => ({ name: n, score: 1 / (1 + dist(name, n)) }))
    .sort((a, b) => b.score - a.score);
  return { artist: name, similar };
}

async function getSimilar(artist) {
  const k = source + ':' + artist.toLowerCase();
  if (cache.has(k)) return cache.get(k);
  const result = await (source === 'lastfm' ? fromLastfm(artist)
    : source === 'sample' ? fromSample(artist) : fromClaude(artist));
  cache.set(k, result);
  return result;
}

/* ---------- rendering ---------- */

function draw(data) {
  resultsEl.innerHTML = '';
  const h = document.createElement('h2');
  h.textContent = `Artists similar to ${data.artist}`;
  const ol = document.createElement('ol');
  data.similar
    .filter(d => d.name.toLowerCase() !== data.artist.toLowerCase())
    .sort((a, b) => b.score - a.score)
    .forEach(d => {
      const li = document.createElement('li');
      const a = document.createElement('a');
      a.href = '#/' + encodeURIComponent(d.name).replace(/%20/g, '+');
      a.textContent = d.name;
      li.append(a, ` (${Math.round(d.score * 100)}% match)`);
      ol.append(li);
    });
  resultsEl.append(h, ol);
  window.scrollTo(0, 0);
}

/* ---------- navigation ---------- */

let loading = 0;
async function load(artist) {
  const ticket = ++loading;
  statusEl.className = '';
  statusEl.textContent = `Loading ${artist}…`;
  try {
    const data = await getSimilar(artist);
    if (ticket !== loading) return;
    draw(data);
    document.title = `${data.artist} on Sound map`;
    statusEl.textContent = '';
  } catch (err) {
    if (ticket !== loading) return;
    statusEl.className = 'error';
    statusEl.textContent = err instanceof SyntaxError
      ? "The response couldn't be read. Try again."
      : (/fetch|network|load failed/i.test(err.message)
          ? (source === 'lastfm'
              ? "Couldn't reach Last.fm. It's blocked in this preview; it works once you host the file. Switch to Claude or Sample data."
              : "Couldn't reach Claude. This only works inside the Claude preview, not in a downloaded copy. Switch to Sample data, or use Last.fm once hosted.")
          : err.message);
  }
}

function go(artist) {
  const h = '#/' + encodeURIComponent(artist).replace(/%20/g, '+');
  if (location.hash === h) load(artist); else location.hash = h;
}

function fromHash() {
  const m = location.hash.match(/^#\/(.+)/);
  if (m) {
    const name = decodeURIComponent(m[1].replace(/\+/g, ' '));
    document.getElementById('q').value = name;
    load(name);
  }
}
window.addEventListener('hashchange', fromHash);

document.getElementById('search').addEventListener('submit', e => {
  e.preventDefault();
  const v = document.getElementById('q').value.trim();
  if (v) go(v);
});

const settings = document.getElementById('settings'), sBtn = document.getElementById('settingsBtn');
sBtn.addEventListener('click', () => {
  const open = settings.classList.toggle('open');
  sBtn.setAttribute('aria-expanded', open);
});
document.querySelectorAll('input[name=src]').forEach(r =>
  r.addEventListener('change', () => { source = r.value; }));

fromHash();
</script>
</body>
</html>

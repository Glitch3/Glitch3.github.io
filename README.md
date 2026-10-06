# Glitch3.github.io
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Sound map</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,300..800&display=swap" rel="stylesheet">
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js"></script>
<style>
  :root {
    --paper: #E9ECF2;
    --ink: #1E2A78;
    --mid: #6B74A8;
    --faint: #B3B9D4;
    --pink: #FF3D7F;
  }
  * { box-sizing: border-box; }
  html, body { margin: 0; height: 100%; }
  body {
    background: var(--paper);
    color: var(--ink);
    font-family: "Bricolage Grotesque", system-ui, sans-serif;
    overflow: hidden;
  }
  header {
    position: fixed; top: 0; left: 0; right: 0; z-index: 10;
    display: flex; gap: 12px; align-items: center; flex-wrap: wrap;
    padding: 16px 20px;
  }
  .brand { font-weight: 800; font-size: 20px; letter-spacing: -0.02em; margin-right: 8px; }
  form { display: flex; flex: 1; max-width: 420px; }
  input[type=text], input[type=password] {
    flex: 1; font: inherit; font-size: 16px; color: var(--ink);
    background: transparent; border: 0; border-bottom: 2px solid var(--ink);
    padding: 6px 2px; outline: none;
  }
  input::placeholder { color: var(--mid); }
  button {
    font: inherit; font-size: 15px; font-weight: 600; cursor: pointer;
    background: var(--ink); color: var(--paper); border: 0;
    padding: 8px 14px; margin-left: 8px; border-radius: 999px;
  }
  button.ghost { background: transparent; color: var(--ink); border: 2px solid var(--ink); }
  button:focus-visible, input:focus-visible, .name:focus-visible {
    outline: 3px solid var(--pink); outline-offset: 3px;
  }
  #settings {
    position: fixed; top: 64px; right: 20px; z-index: 11; width: 300px;
    background: var(--paper); border: 2px solid var(--ink); border-radius: 14px;
    padding: 16px; display: none; font-size: 14px; line-height: 1.5;
  }
  #settings.open { display: block; }
  #settings label { display: flex; gap: 8px; align-items: center; margin: 6px 0; cursor: pointer; }
  #settings p { margin: 10px 0 0; color: var(--mid); }
  #keyRow { display: flex; margin-top: 8px; }
  #map { position: absolute; inset: 0; }
  .name {
    position: absolute; left: 0; top: 0; white-space: nowrap; cursor: pointer;
    transform-origin: center; line-height: 1; letter-spacing: -0.01em;
    user-select: none; padding: 2px 4px;
    transition: color .2s, opacity .4s;
  }
  .name:hover { color: var(--pink) !important; }
  .name.center { color: var(--pink); font-weight: 800; cursor: default; letter-spacing: -0.03em; }
  #status {
    position: fixed; bottom: 20px; left: 20px; right: 20px; z-index: 10;
    font-size: 15px; color: var(--mid); pointer-events: none;
  }
  #status.error { color: var(--pink); }
  .empty {
    position: absolute; inset: 0; display: grid; place-items: center; text-align: center;
    padding: 24px; pointer-events: none;
  }
  .empty h1 { font-size: clamp(36px, 7vw, 84px); font-weight: 800; letter-spacing: -0.04em; margin: 0; line-height: .95; }
  .empty p { color: var(--mid); font-size: 17px; margin-top: 16px; }
  @media (prefers-reduced-motion: reduce) { .name { transition: none; } }
</style>
</head>
<body>
<header>
  <span class="brand">Sound map</span>
  <form id="search">
    <input id="q" type="text" placeholder="Type an artist" autocomplete="off" aria-label="Artist name">
    <button type="submit">Map it</button>
  </form>
  <button class="ghost" id="settingsBtn" aria-expanded="false">Data source</button>
</header>

<div id="settings" role="dialog" aria-label="Data source">
  <label><input type="radio" name="src" value="claude" checked> Claude</label>
  <label><input type="radio" name="src" value="lastfm"> Last.fm</label>
  <div id="keyRow"><input id="key" type="password" placeholder="Last.fm API key" aria-label="Last.fm API key"></div>
  <p>Claude works inside this preview. Last.fm gives real listening-based similarity but may be blocked here; it works once you host this file yourself.</p>
</div>

<div id="map">
  <div class="empty" id="empty">
    <div>
      <h1>Find music<br>near music you love</h1>
      <p>Closer names are more alike. Click any name to move there.</p>
    </div>
  </div>
</div>
<div id="status" aria-live="polite"></div>

<script>
const mapEl = document.getElementById('map');
const statusEl = document.getElementById('status');
const cache = new Map();
let nodes = [];          // current nodes, reused between maps so names glide
let sim = null;
let source = 'claude';

/* ---------- data sources ---------- */

async function fromClaude(artist) {
  const prompt = `You are a music similarity engine. For the artist "${artist}", return ONLY a JSON object, no prose and no code fences:
{"artist":"<correctly spelled name>","similar":[{"name":"...","score":0.0}],"links":[[0,1]]}
Rules: "similar" has 26 real artists, sorted by score; score is 0.3-1.0 for how alike they sound and appeal to the same listeners. "links" lists pairs of indexes into "similar" for artists that are strongly alike each other (about 30 pairs). If the artist is unknown, return {"artist":null}.`;
  const res = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "claude-sonnet-4-6",
      max_tokens: 1000,
      messages: [{ role: "user", content: prompt }]
    })
  });
  const data = await res.json();
  const text = (data.content || []).map(c => c.text || "").join("");
  const json = JSON.parse(text.replace(/```json|```/g, "").trim());
  if (!json.artist) throw new Error(`No artist called "${artist}" found. Check the spelling.`);
  return json;
}

async function fromLastfm(artist) {
  const key = document.getElementById('key').value.trim();
  if (!key) throw new Error("Add a Last.fm API key in Data source, or switch to Claude.");
  const url = `https://ws.audioscrobbler.com/2.0/?method=artist.getsimilar&autocorrect=1&limit=30&format=json`
    + `&artist=${encodeURIComponent(artist)}&api_key=${encodeURIComponent(key)}`;
  const data = await (await fetch(url)).json();
  if (data.error) throw new Error(data.message || "Last.fm returned an error.");
  const list = data.similarartists.artist;
  if (!list.length) throw new Error(`Last.fm has no similar artists for "${artist}".`);
  return {
    artist: data.similarartists["@attr"].artist,
    similar: list.map(a => ({ name: a.name, score: +a.match })),
    links: []   // getSimilar only covers the centre; fetch neighbours too for clustering
  };
}

async function getSimilar(artist) {
  const k = source + ':' + artist.toLowerCase();
  if (cache.has(k)) return cache.get(k);
  const result = await (source === 'lastfm' ? fromLastfm(artist) : fromClaude(artist));
  cache.set(k, result);
  return result;
}

/* ---------- layout & rendering ---------- */

const fontSize = s => 13 + Math.pow(s, 1.6) * 22;
const color = d3.scaleLinear().domain([0, 1]).range(["#9AA1C6", "#1E2A78"]);

function draw(data) {
  document.getElementById('empty')?.remove();
  const W = mapEl.clientWidth, H = mapEl.clientHeight;
  const R = Math.min(W, H) * 0.46;
  const scores = data.similar.map(d => d.score);
  const norm = d3.scaleLinear().domain([d3.min(scores), d3.max(scores)]).range([0.05, 1]);

  const old = new Map(nodes.map(n => [n.name.toLowerCase(), n]));
  const prevCenter = old.get(data.artist.toLowerCase());

  const centre = { name: data.artist, s: 1, center: true, fx: 0, fy: 0 };
  const others = data.similar
    .filter(d => d.name.toLowerCase() !== data.artist.toLowerCase())
    .map(d => {
      const s = norm(d.score), prev = old.get(d.name.toLowerCase());
      const a = Math.random() * Math.PI * 2;
      return {
        name: d.name, s,
        x: prev ? prev.x - (prevCenter?.x || 0) : Math.cos(a) * R,
        y: prev ? prev.y - (prevCenter?.y || 0) : Math.sin(a) * R
      };
    });
  nodes = [centre, ...others];

  // estimate label box for collision
  nodes.forEach(n => {
    const fs = n.center ? 52 : fontSize(n.s);
    n.fs = fs; n.w = n.name.length * fs * 0.52 + 10; n.h = fs + 6;
  });

  const links = others.map(n => ({ source: centre, target: n, dist: 60 + (1 - n.s) * R, str: 0.7 }));
  (data.links || []).forEach(([i, j]) => {
    const a = others[i], b = others[j];
    if (a && b) links.push({ source: a, target: b, dist: 90, str: 0.08 });
  });

  // DOM: one div per name, keyed by name
  const sel = d3.select(mapEl).selectAll('.name').data(nodes, d => d.name.toLowerCase());
  sel.exit().style('opacity', 0).transition().duration(400).remove();
  const enter = sel.enter().append('div')
    .attr('class', 'name').attr('tabindex', 0).attr('role', 'link')
    .style('opacity', 0)
    .on('click', (e, d) => !d.center && go(d.name))
    .on('keydown', (e, d) => { if (e.key === 'Enter' && !d.center) go(d.name); });
  const all = enter.merge(sel)
    .text(d => d.name)
    .classed('center', d => !!d.center)
    .attr('aria-label', d => d.center ? `${d.name}, current artist` : `Map ${d.name}`)
    .style('font-size', d => d.fs + 'px')
    .style('font-weight', d => d.center ? 800 : Math.round(300 + d.s * 400))
    .style('color', d => d.center ? null : color(d.s));
  all.transition().duration(500).style('opacity', 1);

  if (sim) sim.stop();
  sim = d3.forceSimulation(nodes)
    .force('link', d3.forceLink(links).distance(l => l.dist).strength(l => l.str))
    .force('charge', d3.forceManyBody().strength(-40))
    .force('collide', rectCollide())
    .alphaDecay(0.03)
    .on('tick', () => {
      all.style('transform', d => {
        const x = Math.max(-W / 2 + d.w / 2, Math.min(W / 2 - d.w / 2, d.x));
        const y = Math.max(-H / 2 + 70, Math.min(H / 2 - 50, d.y));
        d.x = x; d.y = y;
        return `translate(${W / 2 + x - d.w / 2}px, ${H / 2 + y - d.h / 2}px)`;
      });
    });
  if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
    sim.stop(); for (let i = 0; i < 300; i++) sim.tick(); sim.on('tick')();
  }
}

// rectangle-aware collision so labels don't overlap
function rectCollide() {
  let ns;
  function force() {
    const q = d3.quadtree(ns, d => d.x, d => d.y);
    for (const a of ns) {
      q.visit((node, x0, y0, x1, y1) => {
        const b = node.data;
        if (b && b !== a) {
          const dx = b.x - a.x, dy = b.y - a.y;
          const ox = (a.w + b.w) / 2 - Math.abs(dx), oy = (a.h + b.h) / 2 - Math.abs(dy);
          if (ox > 0 && oy > 0) {
            if (ox / a.w < oy / a.h) {
              const m = ox / 2 * Math.sign(dx || 1);
              if (!a.center) a.x -= m; if (!b.center) b.x += m;
            } else {
              const m = oy / 2 * Math.sign(dy || 1);
              if (!a.center) a.y -= m; if (!b.center) b.y += m;
            }
          }
        }
        return x0 > a.x + 400 || x1 < a.x - 400 || y0 > a.y + 100 || y1 < a.y - 100;
      });
    }
  }
  force.initialize = n => ns = n;
  return force;
}

/* ---------- navigation ---------- */

let loading = 0;
async function load(artist) {
  const ticket = ++loading;
  statusEl.className = '';
  statusEl.textContent = `Mapping ${artist}…`;
  try {
    const data = await getSimilar(artist);
    if (ticket !== loading) return;
    draw(data);
    document.title = `${data.artist} on Sound map`;
    statusEl.textContent = `${data.similar.length} artists near ${data.artist}`;
  } catch (err) {
    if (ticket !== loading) return;
    statusEl.className = 'error';
    statusEl.textContent = err instanceof SyntaxError
      ? "The response couldn't be read. Try again."
      : (err.message === 'Failed to fetch'
          ? "Couldn't reach the data source. Last.fm may be blocked in this preview; switch to Claude."
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

let rt;
window.addEventListener('resize', () => {
  clearTimeout(rt);
  rt = setTimeout(() => { const k = [...cache.keys()].pop(); if (k && nodes.length) draw(cache.get(k)); }, 200);
});

fromHash();
</script>
</body>
</html>

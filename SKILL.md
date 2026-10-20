---
name: trip-map
description: Turn a trip or journey (a topic plus up to 8 stops) into an animated, illustrated route map in one of 8 styles, exported as GIF, MP4, HTML or PNG. Use for trip maps, journey maps or route animations.
---

# Trip Map

Makes an animated route map from a list of stops. The route draws itself stop by stop, each stop pops onto the map with its label, and a caption bar shows the current day and a one-line note. Coastlines are real (Natural Earth data), drawn in an illustrated style.

## Inputs to collect

- **Topic / title**, e.g. "Denmark 2026". Optional subtitle, e.g. "Family Trip · 9 days · 7 stops" (offer to write it).
- **Stops**, in travel order, **2 to 8**. More than 8: ask the user to merge nearby stops or split into two maps.
- **Day or date per stop** (optional), e.g. "Day 1–3", "Mar 6", "1923".
- **One-line note per stop** (optional), e.g. "Museums and Tivoli Gardens". If the user gives only activities, turn them into short notes (under 40 characters) and show them for confirmation.

Correct obvious spelling (e.g. "Arhus" → "Aarhus") and use the place's usual name. Labels must be in Latin script; the bundled fonts do not cover Chinese or Japanese characters (the Ink Wash style adds Chinese numerals itself).

## Coordinates

Find longitude/latitude for every stop: well-known cities from your own knowledge, smaller places (a museum, a theme park) by web search. Keep the user's order. A place visited twice stays two stops; markers closer than 46px are nudged apart automatically.

## Ask the user (pop-up)

If a structured-question tool is available, ask in **two** calls (each question allows at most 4 options). Skip anything the user already specified.

**Call 1** (all four questions together):
1. *Mood* — **Playful** (Brick, Comic, 8-bit Pixel, Treasure Map) / **Classic** (Vintage Poster, Storybook Watercolor, Ink Wash, Fantasy Realm)
2. *Shape* — Square 1:1 (WeChat articles, Instagram) / Wide 16:9 (slides, YouTube; best for east–west routes) / Vertical 9:16 (phone, Reels, 视频号; best for north–south routes)
3. *Output* (multi-select) — GIF / MP4 / HTML / Picture (PNG)
4. *Map* — Real coastlines (recommended) / Plain background (no land shapes)

**Call 2** — the four styles of the chosen mood, each with its one-line description:

| id | Style | Description |
|---|---|---|
| `brick` | Brick | Toy-brick colours on a studded baseplate, 2×2 brick markers |
| `comic` | Comic | Halftone dots, heavy ink, speech-bubble labels, action bursts |
| `pixel` | 8-bit Pixel | Pixel-art game world map, a tiny traveller walks the route |
| `treasure` | Treasure Map | Stained parchment, burnt edges, compass rose, red X at the end |
| `poster` | Vintage Poster | 1950s travel poster, sunburst title, plane on a dashed flight path |
| `watercolor` | Storybook Watercolor | Soft painted washes on textured paper, picture-book feel |
| `inkwash` | Ink Wash 水墨 | Chinese ink painting, red seals with Chinese numerals |
| `realm` | Fantasy Realm | Hand-inked fantasy atlas: water lining, hills, castles, blackletter |

Without a question tool, list the options in plain text and use defaults if there is no answer: square, GIF + PNG, real coastlines, `brick`.

## Setup (once per environment)

Needs Node, Python 3, ffmpeg and Playwright with Chromium. Create a working folder `tripmap/`, write the three files from **Code** below into it exactly as given, then:

```bash
cd tripmap
npm init -y >/dev/null
npm install world-atlas@2.0.2 topojson-client@3.1.0 d3-geo@3.1.1 \
  @fontsource/fredoka@5.1.0 @fontsource/nunito@5.1.0 @fontsource/bangers@5.1.0 @fontsource/comic-neue@5.1.0 \
  @fontsource/limelight@5.1.0 @fontsource/josefin-sans@5.1.0 @fontsource/pirata-one@5.1.0 @fontsource/im-fell-english@5.1.0 \
  @fontsource/press-start-2p@5.1.0 @fontsource/vt323@5.1.0 @fontsource/amatic-sc@5.1.0 @fontsource/patrick-hand@5.1.0 \
  @fontsource/ma-shan-zheng@5.1.0 @fontsource/cormorant-garamond@5.1.0 @fontsource/uncial-antiqua@5.1.0 @fontsource/unifrakturmaguntia@5.1.0
python3 -c "import playwright" 2>/dev/null || pip install playwright --break-system-packages
python3 -c "import fontTools" 2>/dev/null || pip install fonttools --break-system-packages
which ffmpeg
```

Fonts are embedded into the output from these packages, so nothing loads from the internet at render time. If Chromium is missing, run `python3 -m playwright install chromium`.

If the environment cannot run code (no shell), say so: this skill needs a code environment to render. Offer the stop list and coordinates so the user can run it later.

## Run

Write `trip.json` in the working folder:

```json
{
  "title": "Denmark 2026",
  "subtitle": "Family Trip · 9 days · 7 stops",
  "theme": "brick",
  "shape": "square",
  "map": "coastlines",
  "stops": [
    {"name": "Copenhagen", "day": "Day 1–3", "note": "Museums and Tivoli Gardens", "lon": 12.5683, "lat": 55.6761},
    {"name": "Malmö", "day": "Day 4", "note": "Over the Øresund Bridge to Sweden", "lon": 13.0038, "lat": 55.6050}
  ]
}
```

Then:

```bash
python3 tripmap.py trip.json --outputs gif,mp4,png,html --out out
```

Always include `png` so you can check the result, even if the user didn't ask for a picture (deliver it only if they did). Rendering takes about 1–2 minutes for GIF/MP4.

## Check before delivering

Open the PNG and look once for: labels covering markers or each other, text running outside its box, the title or caption hiding stops. Fixes, in order of preference: shorten a long name or note; switch shape (wide for east–west trips, vertical for north–south); drop the subtitle. Re-render once.

## Deliver

Send the requested files. Mention once, briefly, as relevant:
- **GIF**: 1080px, loops. In a 公众号 / WeChat article it shows full width. In a WeChat chat a GIF is turned into a small sticker; send the **MP4** instead, which plays full screen.
- **MP4**: 30 fps, silent, plays anywhere (chats, Reels, 视频号).
- **HTML**: one self-contained file that plays the animation in a browser, looping.

## Limits

- Real coastlines, not satellite imagery or street maps.
- Up to 8 stops. Very close stops (same city) are nudged apart; their labels get leader lines.
- Routes crossing the Pacific date line (e.g. Tokyo → Los Angeles) are not supported; split into two maps.
- Latin-script labels only.

## Code

### `build.mjs` — projects coastlines and stops
```js
// Projects the trip's stops and the real coastlines into SVG coordinates.
// Usage: node build.mjs trip.json > geo.json   (run from the tripmap folder, where node_modules lives)
import fs from 'fs';
import {feature} from 'topojson-client';
import {geoMercator, geoPath, geoContains} from 'd3-geo';
const trip=JSON.parse(fs.readFileSync(process.argv[2],'utf8'));
const [W,H]=trip.size, [x0,y0,x1,y1]=trip.mapBox;
const topo=JSON.parse(fs.readFileSync(new URL('./node_modules/world-atlas/countries-10m.json',import.meta.url),'utf8'));
const countries=feature(topo,topo.objects.countries).features;
// Fit the projection to the stops' bounding box, padded (MultiPoint avoids polygon winding issues)
const lons=trip.stops.map(s=>s.lon), lats=trip.stops.map(s=>s.lat);
const pl=Math.max(.25,(Math.max(...lons)-Math.min(...lons))*.1), pa=Math.max(.2,(Math.max(...lats)-Math.min(...lats))*.1);
const corners={type:'MultiPoint',coordinates:[[Math.min(...lons)-pl,Math.min(...lats)-pa],[Math.max(...lons)+pl,Math.max(...lats)+pa]]};
const proj=geoMercator().fitExtent([[x0,y0],[x1,y1]],corners);
proj.clipExtent([[-20,-20],[W+20,H+20]]);
const path=geoPath(proj).digits(1);
const visited=new Set();
trip.stops.forEach(s=>{const c=countries.find(c=>{const b=path.bounds(c);return b[0][0]<W&&b[1][0]>0&&b[0][1]<H&&b[1][1]>0&&geoContains(c,[s.lon,s.lat])});if(c)visited.add(c.properties.name)});
const land=trip.map==='plain'?[]:countries.map(c=>{const b=path.bounds(c);if(!(b[0][0]<W&&b[1][0]>0&&b[0][1]<H&&b[1][1]>0))return null;
  return {name:c.properties.name,visited:visited.has(c.properties.name),d:path(c)}}).filter(c=>c&&c.d&&c.d.length>200);
const stops=trip.stops.map(s=>{const [x,y]=proj([s.lon,s.lat]);return {...s,x:+x.toFixed(1),y:+y.toFixed(1)}});
process.stdout.write(JSON.stringify({...trip,land,stops}));
```

### `tripmap.py` — builds the page and exports GIF / MP4 / PNG / HTML
```python
#!/usr/bin/env python3
"""Trip Map — build an animated route map and export it.

Usage:
  python3 tripmap.py trip.json --outputs gif,mp4,png,html [--out DIR]

trip.json:
  {"title": "Denmark 2026", "subtitle": "Family Trip · 9 days · 7 stops",
   "theme": "brick|comic|poster|treasure|pixel|watercolor|inkwash|realm",
   "shape": "square|wide|vertical", "map": "coastlines|plain",
   "stops": [{"name": "Copenhagen", "day": "Day 1–3", "note": "Museums and Tivoli", "lon": 12.57, "lat": 55.68}, ...]}
"""
import argparse, base64, json, os, re, shutil, subprocess, sys, tempfile

HERE = os.path.dirname(os.path.abspath(__file__))
FS = os.path.join(HERE, 'node_modules', '@fontsource')

SHAPES = {  # canvas size and the box the stops are fitted into (leaves room for title and caption)
    'square':   ([1080, 1080], [60, 250, 1020, 840]),
    'wide':     ([1920, 1080], [90, 250, 1830, 860]),
    'vertical': ([1080, 1920], [60, 300, 1020, 1660]),
}
FONTS = {
    'brick': [('Fredoka', 'fredoka/files/fredoka-latin-600-normal.woff2', 600), ('Fredoka', 'fredoka/files/fredoka-latin-700-normal.woff2', 700),
              ('Nunito', 'nunito/files/nunito-latin-800-normal.woff2', 800), ('Nunito', 'nunito/files/nunito-latin-900-normal.woff2', 900)],
    'comic': [('Bangers', 'bangers/files/bangers-latin-400-normal.woff2', 400), ('Comic Neue', 'comic-neue/files/comic-neue-latin-700-normal.woff2', 700)],
    'poster': [('Limelight', 'limelight/files/limelight-latin-400-normal.woff2', 400),
               ('Josefin Sans', 'josefin-sans/files/josefin-sans-latin-600-normal.woff2', 600), ('Josefin Sans', 'josefin-sans/files/josefin-sans-latin-700-normal.woff2', 700)],
    'treasure': [('Pirata One', 'pirata-one/files/pirata-one-latin-400-normal.woff2', 400),
                 ('IM Fell English', 'im-fell-english/files/im-fell-english-latin-400-normal.woff2', 400),
                 ('IM Fell English', 'im-fell-english/files/im-fell-english-latin-400-italic.woff2', 400, 'italic')],
    'pixel': [('Press Start 2P', 'press-start-2p/files/press-start-2p-latin-400-normal.woff2', 400), ('VT323', 'vt323/files/vt323-latin-400-normal.woff2', 400)],
    'watercolor': [('Amatic SC', 'amatic-sc/files/amatic-sc-latin-700-normal.woff2', 700), ('Patrick Hand', 'patrick-hand/files/patrick-hand-latin-400-normal.woff2', 400)],
    'inkwash': [('Ma Shan Zheng', '../../fonts/ma-shan-zheng-subset.woff', 400),
                ('Cormorant Garamond', 'cormorant-garamond/files/cormorant-garamond-latin-600-normal.woff2', 600),
                ('Cormorant Garamond', 'cormorant-garamond/files/cormorant-garamond-latin-600-italic.woff2', 600, 'italic'),
                ('Cormorant Garamond', 'cormorant-garamond/files/cormorant-garamond-latin-700-normal.woff2', 700)],
    'realm': [('Uncial Antiqua', 'uncial-antiqua/files/uncial-antiqua-latin-400-normal.woff2', 400),
              ('UnifrakturMaguntia', 'unifrakturmaguntia/files/unifrakturmaguntia-latin-400-normal.woff2', 400),
              ('IM Fell English', 'im-fell-english/files/im-fell-english-latin-400-normal.woff2', 400),
              ('IM Fell English', 'im-fell-english/files/im-fell-english-latin-400-italic.woff2', 400, 'italic')],
}

def ensure_inkwash_font():
    out = os.path.join(HERE, 'fonts', 'ma-shan-zheng-subset.woff')
    if os.path.exists(out): return
    os.makedirs(os.path.dirname(out), exist_ok=True)
    src = os.path.join(FS, 'ma-shan-zheng/files/ma-shan-zheng-chinese-simplified-400-normal.woff')
    text = "一二三四五六七八九十旅行之路ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789 .,·–-!?'&ÆØÅæøåöäüéèàçñ"
    subprocess.run(['pyftsubset', src, f'--text={text}', '--flavor=woff', f'--output-file={out}'], check=True)

def font_css(theme):
    if theme == 'inkwash': ensure_inkwash_font()
    rules = []
    for f in FONTS[theme]:
        fam, path, wt = f[:3]; style = f[3] if len(f) > 3 else 'normal'
        fmt = 'woff2' if path.endswith('woff2') else 'woff'
        data = base64.b64encode(open(os.path.join(FS, path), 'rb').read()).decode()
        rules.append(f"@font-face{{font-family:'{fam}';font-weight:{wt};font-style:{style};font-display:block;src:url(data:font/{fmt};base64,{data}) format('{fmt}')}}")
    return '\n'.join(rules)

def build_page(trip, out_html):
    shape = trip.get('shape', 'square'); trip['size'], trip['mapBox'] = SHAPES[shape]
    trip.setdefault('theme', 'brick'); trip.setdefault('map', 'coastlines')
    with tempfile.NamedTemporaryFile('w', suffix='.json', delete=False) as f: json.dump(trip, f)
    geo = subprocess.run(['node', os.path.join(HERE, 'build.mjs'), f.name], capture_output=True, text=True, check=True, cwd=HERE).stdout
    t = open(os.path.join(HERE, 'template.html')).read()
    t = t.replace('<style>', '<style>\n' + font_css(trip['theme']) + '\n', 1)
    t = t.replace('/*DATA*/null', geo).replace('<title>Trip Map</title>', f"<title>{trip['title']}</title>")
    open(out_html, 'w').write(t)

def capture(html, frames_dir, fps, still_png=None):
    from playwright.sync_api import sync_playwright
    os.makedirs(frames_dir, exist_ok=True)
    with sync_playwright() as p:
        b = p.chromium.launch(); pg = b.new_page(viewport={'width': 1080, 'height': 1080})
        pg.goto('file://' + os.path.abspath(html) + '#record'); pg.wait_for_function('window.READY===true', timeout=30000)
        w, h = pg.evaluate('[document.body.offsetWidth,document.body.offsetHeight]'); pg.set_viewport_size({'width': w, 'height': h})
        total = pg.evaluate('TOTAL')
        if still_png:
            pg.evaluate('render(TOTAL)'); pg.screenshot(path=still_png)
        n = int(total * fps) if fps else 0
        for i in range(n):
            pg.evaluate(f'render({i / fps})'); pg.screenshot(path=f'{frames_dir}/f{i:04d}.png')
        b.close()
    return n

def main():
    ap = argparse.ArgumentParser(); ap.add_argument('trip'); ap.add_argument('--outputs', default='gif,png')
    ap.add_argument('--out', default='.'); a = ap.parse_args()
    trip = json.load(open(a.trip)); outs = set(a.outputs.split(','))
    os.makedirs(a.out, exist_ok=True)
    slug = re.sub(r'[^a-z0-9]+', '-', trip['title'].lower()).strip('-') + '-' + trip.get('theme', 'brick')
    html = os.path.join(a.out, slug + '.html'); build_page(trip, html)
    png = os.path.join(a.out, slug + '.png') if 'png' in outs else None
    fps = 30 if 'mp4' in outs else (15 if 'gif' in outs else 0)
    tmp = tempfile.mkdtemp(); capture(html, tmp, fps, png)
    W, H = SHAPES[trip.get('shape', 'square')][0]; gscale = min(1, 1080 / max(W, H))
    if 'gif' in outs:
        step = '' if fps == 15 else 'fps=15,'
        sc = f'scale={int(W*gscale)}:{int(H*gscale)}:flags=lanczos'
        subprocess.run(['ffmpeg', '-y', '-loglevel', 'error', '-framerate', str(fps), '-i', f'{tmp}/f%04d.png', '-vf', f'{step}{sc},palettegen=max_colors=128:stats_mode=diff', f'{tmp}/pal.png'], check=True)
        subprocess.run(['ffmpeg', '-y', '-loglevel', 'error', '-framerate', str(fps), '-i', f'{tmp}/f%04d.png', '-i', f'{tmp}/pal.png', '-lavfi',
                        f'{step}{sc}[x];[x][1:v]paletteuse=dither=none:diff_mode=rectangle', '-loop', '0', os.path.join(a.out, slug + '.gif')], check=True)
    if 'mp4' in outs:
        subprocess.run(['ffmpeg', '-y', '-loglevel', 'error', '-framerate', str(fps), '-i', f'{tmp}/f%04d.png', '-c:v', 'libx264', '-pix_fmt', 'yuv420p',
                        '-crf', '18', '-movflags', '+faststart', os.path.join(a.out, slug + '.mp4')], check=True)
    shutil.rmtree(tmp, ignore_errors=True)
    if 'html' not in outs: os.remove(html)
    for f in sorted(os.listdir(a.out)):
        if f.startswith(slug): print(os.path.join(a.out, f), os.path.getsize(os.path.join(a.out, f)))

if __name__ == '__main__':
    main()
```

### `template.html` — the animated map and all 8 styles (data and fonts are injected by `tripmap.py`)
```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Trip Map</title>
<style>
*{margin:0;padding:0;box-sizing:border-box}
html,body{overflow:hidden;background:#fff}
svg{display:block}
</style>
</head>
<body>
<svg id="m" xmlns="http://www.w3.org/2000/svg"></svg>
<script>
/* ================================================================
   TRIP MAP — animated route map. DATA is injected by tripmap.py:
   {title, subtitle, size:[W,H], theme, land:[{name,visited,d}], stops:[{name,day,note,x,y}]}
   render(t) draws the exact frame at t seconds (used for GIF/MP4 capture).
   ================================================================ */
const DATA=/*DATA*/null;
const NS='http://www.w3.org/2000/svg', svg=document.getElementById('m');
const [W,H]=DATA.size;
svg.setAttribute('width',W);svg.setAttribute('height',H);svg.setAttribute('viewBox',`0 0 ${W} ${H}`);
document.body.style.width=W+'px';document.body.style.height=H+'px';
const el=(tag,attrs={},parent=svg)=>{const e=document.createElementNS(NS,tag);for(const k in attrs)e.setAttribute(k,attrs[k]);parent.appendChild(e);return e};
const txt=(s,attrs,parent)=>{const t=el('text',attrs,parent);t.textContent=s;return t};
const defs=el('defs');
const rough=(id,f,s)=>{const fl=el('filter',{id,x:'-5%',y:'-5%',width:'110%',height:'110%'},defs);
  el('feTurbulence',{type:'fractalNoise',baseFrequency:f,numOctaves:2,seed:3},fl);el('feDisplacementMap',{in:'SourceGraphic',scale:s},fl)};
const dotPattern=(id,size,r,bg,fg)=>{const p=el('pattern',{id,width:size,height:size,patternUnits:'userSpaceOnUse'},defs);
  el('rect',{width:size,height:size,fill:bg},p);el('circle',{cx:size/2,cy:size/2,r,fill:fg},p)};
const shadowTile=(x,y,w,h,rx,fill,stroke,sw,off,parent)=>{if(off)el('rect',{x:x+off,y:y+off,width:w,height:h,rx,fill:stroke},parent);return el('rect',{x,y,width:w,height:h,rx,fill,stroke,'stroke-width':sw},parent)};

/* ================================================================
   THEMES — each theme draws the background, routes, markers,
   labels, title and caption. Add a theme by adding an entry here.
   ================================================================ */
const THEMES={

/* ---------- BRICK: toy-brick colours on a studded baseplate ---------- */
brick:{
  palette:['#e3000b','#ffcf00','#0057b8','#00852b','#fe8a18','#9b59b6','#00a8c6','#a5ca18'],
  ink:'#1d1d1b', paper:'#fffdf6',
  font:{display:"'Fredoka',sans-serif",body:"'Nunito',sans-serif"},
  background(){
    const P=(id,bg,top)=>{const p=el('pattern',{id,width:28,height:28,patternUnits:'userSpaceOnUse'},defs);
      el('rect',{width:28,height:28,fill:bg},p);el('circle',{cx:14,cy:14,r:7.5,fill:top,stroke:'rgba(0,0,0,.10)','stroke-width':1.5},p)};
    P('pSea','#6cb4ee','#8ec7f4');P('pLand','#6dbe45','#86cc62');P('pOther','#e4cd9e','#efdcb6');
    rough('rough','.025',5);rough('roughS','.04',3);
    el('rect',{width:W,height:H,fill:'url(#pSea)'});
    const g=el('g',{filter:'url(#rough)'});
    DATA.land.forEach(c=>el('path',{d:c.d,fill:c.visited?'url(#pLand)':'url(#pOther)',stroke:this.ink,'stroke-width':3.5,'stroke-linejoin':'round'},g));
  },
  route(d,color,g){return [el('path',{d,fill:'none',stroke:this.ink,'stroke-width':12,'stroke-linecap':'round',filter:'url(#roughS)'},g),
                           el('path',{d,fill:'none',stroke:color,'stroke-width':6.5,'stroke-linecap':'round',filter:'url(#roughS)'},g)]},
  tip(g){const c=el('circle',{r:9,fill:'#fff',stroke:this.ink,'stroke-width':3.5,opacity:0},g);return {set(p,on){c.setAttribute('cx',p.x);c.setAttribute('cy',p.y);c.setAttribute('opacity',on?1:0)}}},
  marker(g,s){el('rect',{x:-19,y:-19,width:38,height:38,rx:6,fill:s.color,stroke:this.ink,'stroke-width':3.5},g);
    [[-8,-8],[8,-8],[-8,8],[8,8]].forEach(([x,y])=>el('circle',{cx:x,cy:y,r:5.2,fill:s.color,stroke:'rgba(0,0,0,.35)','stroke-width':1.6},g))},
  arriveFx(){},
  labelText(g,s){const dc=s.color==='#ffcf00'?'#a07c00':s.color;
    const d=txt(s.day||'',{'font-family':this.font.body,'font-weight':900,'letter-spacing':'.06em','font-size':20,fill:dc},g);
    const n=txt(s.name,{'font-family':this.font.display,'font-weight':600,'font-size':33,fill:this.ink},g);
    const w=Math.max(n.getBBox().width,d.getBBox().width)+30,h=s.day?80:54;
    return {w,h,place(b){d.setAttribute('x',b.x+15);d.setAttribute('y',b.y+30);n.setAttribute('x',b.x+15);n.setAttribute('y',b.y+(s.day?66:38))}}},
  labelShape(back,b,s){const cx=Math.max(b.x,Math.min(s.x,b.x+b.w)),cy=Math.max(b.y,Math.min(s.y,b.y+b.h));
    el('line',{x1:s.x,y1:s.y,x2:cx,y2:cy,stroke:this.ink,'stroke-width':3,'stroke-linecap':'round'},back);
    shadowTile(b.x,b.y,b.w,b.h,16,this.paper,this.ink,4,7,back)},
  title(g){const tt=txt('',{x:70,y:122,'font-family':this.font.display,'font-weight':700,'font-size':74},g);
    [...DATA.title].forEach((ch,k)=>{const ts=el('tspan',{fill:this.palette[[0,4,2,3,4][k%5]]},tt);ts.textContent=ch});
    const sub=txt(DATA.subtitle||'',{x:72,y:164,'font-family':this.font.body,'font-weight':800,'font-size':26,fill:this.ink},g);
    const w=Math.max(tt.getBBox().width,sub.getBBox().width)+64,back=document.createElementNS(NS,'g');g.prepend(back);
    shadowTile(40,40,w,150,16,this.paper,this.ink,4,7,back);return {x:40,y:40,w:w+10,h:160}},
  caption(g){const B={x:40,y:H-170,w:W-80,h:130};shadowTile(B.x,B.y,B.w,B.h,16,this.paper,this.ink,4,7,g);
    const badge=el('rect',{x:B.x+24,y:B.y+30,width:170,height:70,rx:12,stroke:this.ink,'stroke-width':3.5},g);
    const day=txt('',{x:B.x+109,y:B.y+76,'font-family':this.font.body,'font-weight':900,'font-size':32,'text-anchor':'middle'},g);
    const name=txt('',{x:B.x+218,y:B.y+60,'font-family':this.font.display,'font-weight':600,'font-size':42,fill:this.ink},g);
    const note=txt('',{x:B.x+219,y:B.y+103,'font-family':this.font.body,'font-weight':800,'font-size':28,fill:this.ink,opacity:.72},g);
    return {box:{x:B.x,y:B.y-6,w:B.w,h:B.h+16},set:(s,i)=>{badge.setAttribute('fill',s.color);day.setAttribute('fill',s.color==='#ffcf00'?this.ink:'#fff');
      day.textContent=s.day||`Stop ${i+1}`;name.textContent=s.name;note.textContent=s.note||''}}},
  frame(){}
},

/* ---------- COMIC: halftone dots, heavy ink, speech bubbles, action bursts ---------- */
comic:{
  palette:['#e23b2e','#1b6fc2','#9b2fae','#f28c28','#16a34a','#e2387a','#0e9fb5','#6b3fa0'],
  ink:'#111111', paper:'#ffffff', yellow:'#ffd23f',
  font:{display:"'Bangers',Impact,sans-serif",body:"'Comic Neue',sans-serif"},
  background(){
    dotPattern('pSea',14,3.4,'#e6f6fa','#5cc4d8');
    dotPattern('pLand',14,2.4,'#ffd23f','#f4a52a');
    const h=el('pattern',{id:'pOther',width:12,height:12,patternUnits:'userSpaceOnUse',patternTransform:'rotate(45)'},defs);
    el('rect',{width:12,height:12,fill:'#fff6dc'},h);el('line',{x1:0,y1:0,x2:0,y2:12,stroke:'#111','stroke-width':2,opacity:.28},h);
    rough('rough','.02',3);
    el('rect',{width:W,height:H,fill:'url(#pSea)'});
    const g=el('g',{filter:'url(#rough)'});
    DATA.land.forEach(c=>el('path',{d:c.d,fill:c.visited?'url(#pLand)':'url(#pOther)',stroke:this.ink,'stroke-width':5,'stroke-linejoin':'round'},g));
  },
  route(d,color,g){return [el('path',{d,fill:'none',stroke:this.ink,'stroke-width':14,'stroke-linecap':'round'},g),
                           el('path',{d,fill:'none',stroke:color,'stroke-width':7,'stroke-linecap':'round'},g)]},
  tip(g){const lines=[0,1,2].map(()=>el('line',{stroke:this.ink,'stroke-width':4,'stroke-linecap':'round',opacity:0},g));
    const c=el('circle',{r:10,fill:'#fff',stroke:this.ink,'stroke-width':4,opacity:0},g);
    return {set(p,on,dir){c.setAttribute('cx',p.x);c.setAttribute('cy',p.y);c.setAttribute('opacity',on?1:0);
      lines.forEach((l,i)=>{const off=(i-1)*9,nx=-dir.y,ny=dir.x,bx=p.x-dir.x*16+nx*off,by=p.y-dir.y*16+ny*off,len=i===1?34:22;
        l.setAttribute('x1',bx);l.setAttribute('y1',by);l.setAttribute('x2',bx-dir.x*len);l.setAttribute('y2',by-dir.y*len);l.setAttribute('opacity',on?1:0)})}}},
  marker(g,s){const pts=[];for(let k=0;k<24;k++){const r=k%2?15:27,a=k*Math.PI/12-Math.PI/2;pts.push(`${(Math.cos(a)*r).toFixed(1)},${(Math.sin(a)*r).toFixed(1)}`)}
    el('polygon',{points:pts.join(' '),fill:s.color,stroke:this.ink,'stroke-width':3.5,'stroke-linejoin':'round'},g);
    txt(String(s.i+1),{y:9,'text-anchor':'middle','font-family':this.font.display,'font-size':26,fill:'#fff',stroke:this.ink,'stroke-width':3,'paint-order':'stroke'},g)},
  arriveFx(s,k){if(!s.fx){s.fx=[];for(let i=0;i<12;i++)s.fx.push(el('line',{stroke:this.ink,'stroke-width':4,'stroke-linecap':'round'},s.fxG))}
    s.fx.forEach((l,i)=>{const a=i*Math.PI/6+.26,r1=34+k*10,r2=40+k*34;l.setAttribute('x1',s.x+Math.cos(a)*r1);l.setAttribute('y1',s.y+Math.sin(a)*r1);
      l.setAttribute('x2',s.x+Math.cos(a)*r2);l.setAttribute('y2',s.y+Math.sin(a)*r2);l.setAttribute('opacity',k>0&&k<1?1-k:0)})},
  labelText(g,s){const d=txt((s.day||'').toUpperCase(),{'font-family':this.font.body,'font-weight':700,'font-size':21,'letter-spacing':'.04em',fill:s.color},g);
    const n=txt(s.name.toUpperCase(),{'font-family':this.font.display,'font-size':36,'letter-spacing':'.03em',fill:this.ink},g);
    const w=Math.max(n.getBBox().width,d.getBBox().width)+44,h=s.day?84:58;
    return {w,h,place(b){d.setAttribute('x',b.x+22);d.setAttribute('y',b.y+32);n.setAttribute('x',b.x+22);n.setAttribute('y',b.y+(s.day?70:42))}}},
  labelShape(back,b,s){// speech bubble with a tail pointing at the marker
    const cx=b.x+b.w/2,cy=b.y+b.h/2,dx=s.x-cx,dy=s.y-cy,L=Math.hypot(dx,dy)||1,ux=dx/L,uy=dy/L;
    const ex=Math.max(b.x+18,Math.min(cx+ux*b.w,b.x+b.w-18)),ey=Math.max(b.y+12,Math.min(cy+uy*b.h,b.y+b.h-12));
    const px=-uy*14,py=ux*14,tx=s.x-ux*26,ty=s.y-uy*26;
    el('polygon',{points:`${ex+px},${ey+py} ${ex-px},${ey-py} ${tx},${ty}`,fill:this.ink,stroke:this.ink,'stroke-width':10,'stroke-linejoin':'round'},back);
    el('rect',{x:b.x,y:b.y,width:b.w,height:b.h,rx:b.h/2.4,fill:this.paper,stroke:this.ink,'stroke-width':4.5},back);
    el('polygon',{points:`${ex+px*.72},${ey+py*.72} ${ex-px*.72},${ey-py*.72} ${tx-ux*7},${ty-uy*7}`,fill:this.paper},back)},
  title(g){const tilt=el('g',{transform:'rotate(-3 300 110)'},g);
    const t=txt(DATA.title.toUpperCase()+'!',{x:76,y:132,'font-family':this.font.display,'font-size':88,'letter-spacing':'.03em',fill:'#e23b2e',stroke:this.ink,'stroke-width':6,'paint-order':'stroke','stroke-linejoin':'round'},tilt);
    const sub=txt((DATA.subtitle||'').toUpperCase(),{x:80,y:178,'font-family':this.font.body,'font-weight':700,'font-size':25,'letter-spacing':'.05em',fill:this.ink},tilt);
    const w=Math.max(t.getBBox().width,sub.getBBox().width)+60,back=document.createElementNS(NS,'g');tilt.prepend(back);
    shadowTile(46,46,w,158,4,this.yellow,this.ink,5,9,back);return {x:30,y:20,w:w+40,h:210}},
  caption(g){const B={x:44,y:H-176,w:W-88,h:130};shadowTile(B.x,B.y,B.w,B.h,4,this.yellow,this.ink,5,9,g);
    const badge=el('rect',{x:B.x+22,y:B.y+28,width:176,height:74,rx:4,stroke:this.ink,'stroke-width':4},g);
    const day=txt('',{x:B.x+110,y:B.y+78,'font-family':this.font.display,'font-size':40,'letter-spacing':'.04em','text-anchor':'middle',fill:'#fff',stroke:this.ink,'stroke-width':3,'paint-order':'stroke'},g);
    const name=txt('',{x:B.x+222,y:B.y+62,'font-family':this.font.display,'font-size':46,'letter-spacing':'.03em',fill:this.ink},g);
    const note=txt('',{x:B.x+224,y:B.y+102,'font-family':this.font.body,'font-weight':700,'font-size':27,fill:this.ink},g);
    return {box:{x:B.x,y:B.y-8,w:B.w+10,h:B.h+20},set:(s,i)=>{badge.setAttribute('fill',s.color);day.textContent=(s.day||`Stop ${i+1}`).toUpperCase();
      name.textContent=s.name.toUpperCase();note.textContent=s.note||''}}},
  frame(g){// comic panel border
    el('rect',{x:7,y:7,width:W-14,height:H-14,fill:'none',stroke:'#fff','stroke-width':14},g);
    el('rect',{x:16,y:16,width:W-32,height:H-32,fill:'none',stroke:this.ink,'stroke-width':7},g)}
}
};
/* ================================================================
   EXTRA THEMES — poster, treasure, pixel, watercolor, inkwash, realm
   ================================================================ */
let MASK_N=0;
/* A dashed or dotted route can't be revealed with its own dash pattern,
   so it's shown through a mask whose solid path is what animates. */
function maskedRoute(d,layers,g,width=34){
  const id='rm'+(MASK_N++);
  const m=el('mask',{id,maskUnits:'userSpaceOnUse',x:0,y:0,width:W,height:H},defs);
  const reveal=el('path',{d,fill:'none',stroke:'#fff','stroke-width':width,'stroke-linecap':'round'},m);
  const vis=el('g',{mask:`url(#${id})`},g);layers.forEach(a=>el('path',{d,fill:'none',...a},vis));
  return [reveal];
}
const leader=(back,b,s,color,dash)=>{const cx=Math.max(b.x,Math.min(s.x,b.x+b.w)),cy=Math.max(b.y,Math.min(s.y,b.y+b.h));
  if(Math.hypot(cx-s.x,cy-s.y)>44)el('line',{x1:s.x,y1:s.y,x2:cx,y2:cy,stroke:color,'stroke-width':2,'stroke-dasharray':dash||'none','stroke-linecap':'round',opacity:.8},back)};
const haloText=(s,attrs,g,halo,hw=7)=>txt(s,{...attrs,stroke:halo,'stroke-width':hw,'paint-order':'stroke','stroke-linejoin':'round'},g);
const grain=(op)=>{const f=el('filter',{id:'grain'+op,x:0,y:0,width:'100%',height:'100%'},defs);
  el('feTurbulence',{type:'fractalNoise',baseFrequency:.9,numOctaves:2,seed:7},f);
  el('feColorMatrix',{values:`0 0 0 0 0  0 0 0 0 0  0 0 0 0 0  0 0 0 ${op} 0`},f);
  el('rect',{width:W,height:H,filter:`url(#grain${op})`})};

Object.assign(THEMES,{

/* ---------- POSTER: 1950s jet-age travel poster ---------- */
poster:{
  palette:['#e76f51','#2a9d8f','#e9c46a','#7c4d7a','#f4a261','#5e8c61','#d1495b','#3d5a80'],
  ink:'#10393d', cream:'#fdf3e1',
  font:{display:"'Limelight',serif",body:"'Josefin Sans',sans-serif"},
  background(){
    el('rect',{width:W,height:H,fill:'#1d6d73'});
    // faint latitude bands, like a printed poster
    for(let y=0;y<H;y+=36)el('rect',{x:0,y,width:W,height:18,fill:'#1a666c'});
    const g=el('g');
    DATA.land.forEach(c=>el('path',{d:c.d,fill:c.visited?'#f4e3c1':'#e2c393',stroke:this.ink,'stroke-width':2.5,'stroke-linejoin':'round'},g));
    grain(.16);
  },
  route(d,color,g){return maskedRoute(d,[{stroke:this.ink,'stroke-width':9,'stroke-dasharray':'16 12','stroke-linecap':'round'},{stroke:this.cream,'stroke-width':5,'stroke-dasharray':'16 12','stroke-linecap':'round'}],g)},
  tip(g){const p=el('g',{opacity:0},g);
    el('path',{d:'M16 0 L-6 -4 L-14 -16 L-18 -16 L-12 -3 L-20 -2 L-24 -8 L-27 -8 L-24 0 L-27 8 L-24 8 L-20 2 L-12 3 L-18 16 L-14 16 L-6 4 Z',fill:this.cream,stroke:this.ink,'stroke-width':2.5,'stroke-linejoin':'round'},p);
    return {set(pt,on,dir){p.setAttribute('opacity',on?1:0);p.setAttribute('transform',`translate(${pt.x} ${pt.y}) rotate(${Math.atan2(dir.y,dir.x)*180/Math.PI})`)}}},
  marker(g,s){el('circle',{r:27,fill:this.ink},g);el('circle',{r:24,fill:s.color,stroke:this.cream,'stroke-width':4},g);
    txt(String(s.i+1),{y:10,'text-anchor':'middle','font-family':this.font.display,'font-size':28,fill:this.cream},g)},
  labelText(g,s){const d=txt((s.day||'').toUpperCase(),{'font-family':this.font.body,'font-weight':700,'font-size':18,'letter-spacing':'.22em',fill:s.color==='#e9c46a'?'#b8860b':s.color},g);
    const n=txt(s.name,{'font-family':this.font.display,'font-size':32,fill:this.ink},g);
    const w=Math.max(n.getBBox().width,d.getBBox().width)+48,h=s.day?78:54;
    return {w,h,place(b){d.setAttribute('x',b.x+24);d.setAttribute('y',b.y+29);n.setAttribute('x',b.x+24);n.setAttribute('y',b.y+(s.day?64:38))}}},
  labelShape(back,b,s){const cx=Math.max(b.x,Math.min(s.x,b.x+b.w)),cy=Math.max(b.y,Math.min(s.y,b.y+b.h));
    el('line',{x1:s.x,y1:s.y,x2:cx,y2:cy,stroke:this.cream,'stroke-width':3},back);
    el('rect',{x:b.x,y:b.y,width:b.w,height:b.h,rx:b.h/2,fill:this.cream,stroke:this.ink,'stroke-width':3},back)},
  title(g){const B={x:40,y:40,w:W-80,h:200};
    const cp=el('clipPath',{id:'tclip'},defs);el('rect',{x:B.x,y:B.y,width:B.w,height:B.h},cp);
    el('rect',{x:B.x,y:B.y,width:B.w,height:B.h,fill:'#e76f51'},g);
    const rays=el('g',{'clip-path':'url(#tclip)'},g),cx=W/2,cy=B.y+B.h+60;
    for(let k=0;k<18;k++){const a1=Math.PI+k*Math.PI/18,a2=a1+Math.PI/36;
      el('path',{d:`M${cx} ${cy} L${cx+Math.cos(a1)*900} ${cy+Math.sin(a1)*900} L${cx+Math.cos(a2)*900} ${cy+Math.sin(a2)*900}Z`,fill:'#f4a261'},rays)}
    el('circle',{cx,cy:cy-10,r:70,fill:'#e9c46a'},rays);
    el('rect',{x:B.x+10,y:B.y+10,width:B.w-20,height:B.h-20,fill:'none',stroke:this.cream,'stroke-width':3},g);
    haloText(DATA.title.toUpperCase(),{x:cx,y:B.y+112,'text-anchor':'middle','font-family':this.font.display,'font-size':86,fill:this.cream},g,this.ink,8);
    txt((DATA.subtitle||'').toUpperCase(),{x:cx,y:B.y+160,'text-anchor':'middle','font-family':this.font.body,'font-weight':700,'font-size':22,'letter-spacing':'.3em',fill:this.ink},g);
    return {x:B.x,y:B.y,w:B.w,h:B.h+12}},
  caption(g){const B={x:40,y:H-160,w:W-80,h:120};
    el('rect',{x:B.x,y:B.y,width:B.w,height:B.h,fill:this.ink},g);
    el('rect',{x:B.x+10,y:B.y+10,width:B.w-20,height:B.h-20,fill:'none',stroke:this.cream,'stroke-width':2},g);
    const badge=el('circle',{cx:B.x+80,cy:B.y+60,r:42,stroke:this.cream,'stroke-width':4},g);
    const day=txt('',{x:B.x+80,y:B.y+68,'text-anchor':'middle','font-family':this.font.body,'font-weight':700,'font-size':20,fill:this.cream},g);
    const name=txt('',{x:B.x+150,y:B.y+58,'font-family':this.font.display,'font-size':40,fill:this.cream},g);
    const note=txt('',{x:B.x+152,y:B.y+94,'font-family':this.font.body,'font-weight':600,'font-size':24,'letter-spacing':'.04em',fill:'#e9c46a'},g);
    return {box:{x:B.x,y:B.y-8,w:B.w,h:B.h+8},set:(s,i)=>{badge.setAttribute('fill',s.color);day.textContent=(s.day||`Stop ${i+1}`).toUpperCase().replace('DAY ','DAY ');name.textContent=s.name;note.textContent=s.note||''}}},
  frame(g){el('rect',{x:9,y:9,width:W-18,height:H-18,fill:'none',stroke:this.cream,'stroke-width':18},g)}
},

/* ---------- TREASURE: aged parchment adventure map ---------- */
treasure:{
  palette:['#7a1f1f','#3b2412','#3b2412','#3b2412','#3b2412','#3b2412','#3b2412','#3b2412'],
  ink:'#3b2412', parch:'#ecdcb2', red:'#9b1c1c',
  font:{display:"'Pirata One',serif",body:"'IM Fell English',serif"},
  background(){
    el('rect',{width:W,height:H,fill:this.parch});
    const st=el('filter',{id:'stain',x:0,y:0,width:'100%',height:'100%'},defs);
    el('feTurbulence',{type:'fractalNoise',baseFrequency:.006,numOctaves:4,seed:11},st);
    el('feColorMatrix',{values:'0 0 0 0 .45  0 0 0 0 .3  0 0 0 0 .12  0 0 0 1.4 -.55'},st);
    el('rect',{width:W,height:H,filter:'url(#stain)',opacity:.55});
    rough('rough','.03',4);rough('roughS','.05',3);
    // shaded water along every coast, then the land itself
    const sh=el('g');DATA.land.forEach(c=>{el('path',{d:c.d,fill:'none',stroke:'#a98c58','stroke-width':26,opacity:.22},sh);el('path',{d:c.d,fill:'none',stroke:'#a98c58','stroke-width':12,opacity:.28},sh)});
    const g=el('g',{filter:'url(#rough)'});
    DATA.land.forEach(c=>el('path',{d:c.d,fill:c.visited?'#dcc189':'#e4d0a2',stroke:this.ink,'stroke-width':2.6,'stroke-linejoin':'round'},g));
    // compass rose, top right
    const cr=el('g',{transform:`translate(${W-130} 300)`,opacity:.85});
    el('circle',{r:62,fill:'none',stroke:this.ink,'stroke-width':2},cr);el('circle',{r:54,fill:'none',stroke:this.ink,'stroke-width':1},cr);
    for(let k=0;k<8;k++){const a=k*Math.PI/4,L=k%2?40:72;const l=`${Math.cos(a)*L},${Math.sin(a)*L}`,p1=`${Math.cos(a+.25)*12},${Math.sin(a+.25)*12}`,p2=`${Math.cos(a-.25)*12},${Math.sin(a-.25)*12}`;
      el('polygon',{points:`${p1} ${l} 0,0`,fill:this.ink},cr);el('polygon',{points:`${p2} ${l} 0,0`,fill:this.parch,stroke:this.ink,'stroke-width':1},cr)}
    txt('N',{y:-80,'text-anchor':'middle','font-family':this.font.display,'font-size':30,fill:this.ink},cr);
    grain(.12);
  },
  route(d,color,g){return maskedRoute(d,[{stroke:this.red,'stroke-width':5,'stroke-dasharray':'2 13','stroke-linecap':'round'}],g)},
  tip(g){const c=el('circle',{r:8,fill:this.red,stroke:this.ink,'stroke-width':2,opacity:0},g);return {set(p,on){c.setAttribute('cx',p.x);c.setAttribute('cy',p.y);c.setAttribute('opacity',on?1:0)}}},
  marker(g,s){if(s.i===DATA.stops.length-1){const X=el('g',{filter:'url(#roughS)'},g);
      el('path',{d:'M-22 -22 L22 22 M22 -22 L-22 22',stroke:this.red,'stroke-width':10,'stroke-linecap':'round'},X);return}
    el('circle',{r:19,fill:this.parch,stroke:this.ink,'stroke-width':3},g);
    txt(String(s.i+1),{y:9,'text-anchor':'middle','font-family':this.font.display,'font-size':26,fill:this.ink},g)},
  labelText(g,s){const d=haloText(s.day||'',{'font-family':this.font.body,'font-style':'italic','font-size':22,fill:this.red},g,this.parch,6);
    const n=haloText(s.name,{'font-family':this.font.display,'font-size':38,fill:this.ink},g,this.parch,8);
    const w=Math.max(n.getBBox().width,d.getBBox().width)+16,h=s.day?70:46;
    return {w,h,place(b){d.setAttribute('x',b.x+8);d.setAttribute('y',b.y+22);n.setAttribute('x',b.x+8);n.setAttribute('y',b.y+(s.day?58:36))}}},
  labelShape(back,b,s){leader(back,b,s,this.ink,'2 7')},
  title(g){const w=Math.min(W-200,Math.max(560,DATA.title.length*38+160)),x=40,y=46;
    el('path',{d:`M${x} ${y+30} L${x+40} ${y} L${x+40} ${y+130} L${x} ${y+150} Z`,fill:'#d9c08a',stroke:this.ink,'stroke-width':3},g);
    el('path',{d:`M${x+w+80} ${y+30} L${x+w+40} ${y} L${x+w+40} ${y+130} L${x+w+80} ${y+150} Z`,fill:'#d9c08a',stroke:this.ink,'stroke-width':3},g);
    el('rect',{x:x+40,y,width:w,height:130,fill:'#f3e6c4',stroke:this.ink,'stroke-width':3},g);
    txt(DATA.title,{x:x+40+w/2,y:y+78,'text-anchor':'middle','font-family':this.font.display,'font-size':70,fill:this.ink},g);
    txt(DATA.subtitle||'',{x:x+40+w/2,y:y+112,'text-anchor':'middle','font-family':this.font.body,'font-style':'italic','font-size':24,fill:this.red},g);
    return {x:x,y:y,w:w+90,h:160}},
  caption(g){const B={x:60,y:H-160,w:W-120,h:116};
    [B.x,B.x+B.w].forEach(cx=>{el('ellipse',{cx,cy:B.y+B.h/2,rx:16,ry:B.h/2,fill:'#d9c08a',stroke:this.ink,'stroke-width':3},g)});
    el('rect',{x:B.x,y:B.y,width:B.w,height:B.h,fill:'#f3e6c4',stroke:this.ink,'stroke-width':3},g);
    [B.x,B.x+B.w].forEach(cx=>{el('ellipse',{cx,cy:B.y+B.h/2,rx:16,ry:B.h/2,fill:'#e4cf9c',stroke:this.ink,'stroke-width':3},g)});
    const day=txt('',{x:B.x+40,y:B.y+70,'font-family':this.font.display,'font-size':40,fill:this.red},g);
    const name=txt('',{x:B.x+220,y:B.y+55,'font-family':this.font.display,'font-size':42,fill:this.ink},g);
    const note=txt('',{x:B.x+222,y:B.y+90,'font-family':this.font.body,'font-style':'italic','font-size':26,fill:this.ink},g);
    return {box:{x:B.x-20,y:B.y-6,w:B.w+40,h:B.h+12},set:(s,i)=>{day.textContent=s.day||`Stop ${i+1}`;name.textContent=s.name;note.textContent=s.note||''}}},
  frame(g){const f=el('filter',{id:'burn',x:'-5%',y:'-5%',width:'110%',height:'110%'},defs);
    el('feTurbulence',{type:'fractalNoise',baseFrequency:.03,numOctaves:3,seed:5},f);el('feDisplacementMap',{in:'SourceGraphic',scale:40},f);el('feGaussianBlur',{stdDeviation:3},f);
    el('rect',{x:0,y:0,width:W,height:H,fill:'none',stroke:'#4a2a10','stroke-width':70,filter:'url(#burn)',opacity:.85},g);
    el('rect',{x:-30,y:-30,width:W+60,height:H+60,fill:'none',stroke:'#fff','stroke-width':44,filter:'url(#burn)'},g)}
},

/* ---------- PIXEL: 8-bit role-playing-game world map ---------- */
pixel:{
  palette:['#e43b44','#feae34','#2ce8f5','#63c74d','#b55088','#f77622','#ffffff','#c0cbdc'],
  ink:'#181425', PX:8,
  font:{display:"'Press Start 2P',monospace",body:"'VT323',monospace"},
  background(){const PX=this.PX,cw=Math.ceil(W/PX),ch=Math.ceil(H/PX);
    const cv=document.createElement('canvas');cv.width=cw;cv.height=ch;const c=cv.getContext('2d');
    c.scale(1/PX,1/PX);DATA.land.forEach(l=>{c.fillStyle=l.visited?'#ff0000':'#00ff00';c.fill(new Path2D(l.d))});
    const src=c.getImageData(0,0,cw,ch).data,kind=(x,y)=>{if(x<0||y<0||x>=cw||y>=ch)return 0;const i=(y*cw+x)*4;return src[i+3]<110?0:(src[i]>src[i+1]?1:2)};
    const out=document.createElement('canvas');out.width=cw;out.height=ch;const o=out.getContext('2d');const img=o.createImageData(cw,ch);
    const put=(x,y,hex)=>{const i=(y*cw+x)*4;img.data[i]=parseInt(hex.slice(1,3),16);img.data[i+1]=parseInt(hex.slice(3,5),16);img.data[i+2]=parseInt(hex.slice(5,7),16);img.data[i+3]=255};
    const hash=(x,y)=>((x*73856093)^(y*19349663))>>>0;
    for(let y=0;y<ch;y++)for(let x=0;x<cw;x++){const k=kind(x,y);
      const nearL=[[1,0],[-1,0],[0,1],[0,-1]].some(([a,b])=>kind(x+a,y+b)>0),nearS=[[1,0],[-1,0],[0,1],[0,-1]].some(([a,b])=>kind(x+a,y+b)===0);
      if(k===0)put(x,y,nearL?'#8fd3ff':(hash(x,y)%23===0||hash(x-1,y)%23===0)?'#5b8ee6':'#3a6fd8');
      else if(nearS)put(x,y,k===1?'#265c42':'#8a6a3d');
      else put(x,y,k===1?(hash(x,y)%17===0?'#3e8948':'#63c74d'):(hash(x,y)%19===0?'#b08955':'#d9b26f'))}
    o.putImageData(img,0,0);
    el('image',{href:out.toDataURL(),width:W,height:H,style:'image-rendering:pixelated',preserveAspectRatio:'none'});
  },
  route(d,color,g){return maskedRoute(d,[{stroke:this.ink,'stroke-width':12,'stroke-dasharray':'8 8'},{stroke:'#ffffff','stroke-width':6,'stroke-dasharray':'8 8'}],g)},
  tip(g){const p=el('g',{opacity:0},g),px=(x,y,c)=>el('rect',{x:x*5-10,y:y*5-14,width:5,height:5,fill:c},p);
    [[1,0],[2,0],[1,1],[2,1]].forEach(([x,y])=>px(x,y,'#f6d6bd'));[[0,2],[1,2],[2,2],[3,2],[1,3],[2,3]].forEach(([x,y])=>px(x,y,'#e43b44'));[[1,4],[2,4]].forEach(([x,y])=>px(x,y,'#262b44'));
    return {set(pt,on){p.setAttribute('opacity',on?1:0);p.setAttribute('transform',`translate(${Math.round(pt.x/5)*5} ${Math.round(pt.y/5)*5})`)}}},
  marker(g,s){const P=6,r=(x,y,w,h,c)=>el('rect',{x:x*P-21,y:y*P-24,width:w*P,height:h*P,fill:c},g);
    r(0,3,7,5,this.ink);r(1,3,5,4,'#f4f4f4');r(0,1,7,2,this.ink);r(1,1,5,2,s.color);r(2,0,3,1,this.ink);r(3,5,1,2,this.ink);
    el('rect',{x:-21,y:-24,width:42,height:48,fill:'none'},g)},
  arriveFx(s,k){if(!s.fx){s.fx=[];for(let i=0;i<8;i++)s.fx.push(el('rect',{width:6,height:6,fill:'#fff'},s.fxG))}
    s.fx.forEach((r,i)=>{const a=i*Math.PI/4,d=30+k*30;r.setAttribute('x',Math.round((s.x+Math.cos(a)*d)/6)*6-3);r.setAttribute('y',Math.round((s.y+Math.sin(a)*d)/6)*6-3);r.setAttribute('opacity',k>0&&k<1?1:0)})},
  labelText(g,s){const d=txt((s.day||'').toUpperCase(),{'font-family':this.font.body,'font-size':26,fill:s.color},g);
    const n=txt(s.name.toUpperCase(),{'font-family':this.font.display,'font-size':16,fill:'#fff'},g);
    const w=Math.max(n.getBBox().width,d.getBBox().width)+36,h=s.day?66:44;
    return {w,h,place(b){d.setAttribute('x',b.x+18);d.setAttribute('y',b.y+27);n.setAttribute('x',b.x+18);n.setAttribute('y',b.y+(s.day?52:30))}}},
  labelShape(back,b){el('rect',{x:b.x,y:b.y,width:b.w,height:b.h,fill:this.ink,stroke:'#fff','stroke-width':4},back);
    el('rect',{x:b.x+6,y:b.y+6,width:b.w-12,height:b.h-12,fill:'none',stroke:'#5a6988','stroke-width':2},back)},
  title(g){el('rect',{x:40,y:40,width:W-80,height:150,fill:this.ink,stroke:'#fff','stroke-width':6},g);
    el('rect',{x:50,y:50,width:W-100,height:130,fill:'none',stroke:'#5a6988','stroke-width':3},g);
    txt(DATA.title.toUpperCase(),{x:W/2+5,y:122,'text-anchor':'middle','font-family':this.font.display,'font-size':46,fill:'#e43b44'},g);
    txt(DATA.title.toUpperCase(),{x:W/2,y:117,'text-anchor':'middle','font-family':this.font.display,'font-size':46,fill:'#feae34'},g);
    txt('▶ '+(DATA.subtitle||'').toUpperCase(),{x:W/2,y:162,'text-anchor':'middle','font-family':this.font.body,'font-size':32,fill:'#fff'},g);
    return {x:40,y:40,w:W-80,h:160}},
  caption(g){const B={x:40,y:H-170,w:W-80,h:130};
    el('rect',{x:B.x,y:B.y,width:B.w,height:B.h,fill:this.ink,stroke:'#fff','stroke-width':6},g);
    el('rect',{x:B.x+10,y:B.y+10,width:B.w-20,height:B.h-20,fill:'none',stroke:'#5a6988','stroke-width':3},g);
    const name=txt('',{x:B.x+34,y:B.y+56,'font-family':this.font.display,'font-size':24},g);
    const note=txt('',{x:B.x+34,y:B.y+100,'font-family':this.font.body,'font-size':34,fill:'#fff'},g);
    txt('▼',{x:B.x+B.w-50,y:B.y+B.h-22,'font-family':this.font.body,'font-size':28,fill:'#fff'},g);
    return {box:{x:B.x,y:B.y-6,w:B.w,h:B.h+12},set:(s,i)=>{name.setAttribute('fill',s.color);name.textContent=`${(s.day||'Stop '+(i+1)).toUpperCase()} ▶ ${s.name.toUpperCase()}`;note.textContent=s.note||''}}},
  frame(){}
},

/* ---------- WATERCOLOR: soft storybook washes on paper ---------- */
watercolor:{
  palette:['#c9583d','#3f78a8','#d39a2c','#5f8f4e','#9a5b94','#d0764d','#3e9690','#8a6d3b'],
  ink:'#4a4038', paper:'#f7f0e1',
  font:{display:"'Amatic SC',cursive",body:"'Patrick Hand',cursive"},
  background(){
    el('rect',{width:W,height:H,fill:this.paper});
    const wash=(id,seed,f=.012)=>{const fl=el('filter',{id,x:'-10%',y:'-10%',width:'120%',height:'120%'},defs);
      el('feTurbulence',{type:'fractalNoise',baseFrequency:f,numOctaves:4,seed,result:'n'},fl);
      el('feDisplacementMap',{in:'SourceGraphic',in2:'n',scale:22,result:'d'},fl);
      el('feGaussianBlur',{in:'d',stdDeviation:2.2,result:'b'},fl);
      el('feTurbulence',{type:'fractalNoise',baseFrequency:.05,numOctaves:3,seed:seed+1,result:'p'},fl);
      el('feColorMatrix',{in:'p',values:'0 0 0 0 0  0 0 0 0 0  0 0 0 0 0  0 0 0 -.55 1.1',result:'pm'},fl);
      el('feComposite',{in:'b',in2:'pm',operator:'in'},fl)};
    wash('wSea',3);wash('wLand',8);wash('wDot',12,.05);rough('rough','.03',3);
    el('rect',{x:-30,y:-30,width:W+60,height:H+60,fill:'#b3d6e6',filter:'url(#wSea)',opacity:.8});
    const sh=el('g',{filter:'url(#wSea)'});DATA.land.forEach(c=>el('path',{d:c.d,fill:'none',stroke:'#6fa9c6','stroke-width':22,opacity:.45},sh));
    const g=el('g',{filter:'url(#wLand)'});DATA.land.forEach(c=>el('path',{d:c.d,fill:c.visited?'#a9cf86':'#ecd9ad',stroke:c.visited?'#7fae5c':'#d1b781','stroke-width':6},g));
    const p=el('g',{filter:'url(#rough)',opacity:.55});DATA.land.forEach(c=>el('path',{d:c.d,fill:'none',stroke:this.ink,'stroke-width':1.4},p));
    grain(.10);
  },
  route(d,color,g){return maskedRoute(d,[{stroke:color,'stroke-width':5,'stroke-dasharray':'11 9','stroke-linecap':'round',opacity:.9}],g)},
  marker(g,s){el('circle',{r:22,fill:s.color,filter:'url(#wDot)'},g);el('circle',{r:12,fill:this.paper,stroke:this.ink,'stroke-width':1.5,opacity:.95},g);el('circle',{r:6,fill:s.color},g)},
  labelText(g,s){const d=haloText(s.day||'',{'font-family':this.font.display,'font-weight':700,'font-size':30,fill:s.color},g,this.paper,6);
    const n=haloText(s.name,{'font-family':this.font.body,'font-size':34,fill:this.ink},g,this.paper,8);
    const w=Math.max(n.getBBox().width,d.getBBox().width)+12,h=s.day?72:46;
    return {w,h,place(b){d.setAttribute('x',b.x+6);d.setAttribute('y',b.y+26);n.setAttribute('x',b.x+6);n.setAttribute('y',b.y+(s.day?62:36))}}},
  labelShape(back,b,s){leader(back,b,s,this.ink,'3 6')},
  title(g){el('ellipse',{cx:290,cy:118,rx:290,ry:110,fill:this.paper,filter:'url(#wLand)',opacity:.92},g);
    el('circle',{cx:520,cy:86,r:30,fill:'#f2c14e',filter:'url(#wDot)',opacity:.85},g);
    el('path',{d:'M430 60 q8 -8 16 0 q8 -8 16 0 M470 90 q6 -6 12 0 q6 -6 12 0',fill:'none',stroke:this.ink,'stroke-width':2.5,'stroke-linecap':'round'},g);
    txt(DATA.title,{x:70,y:132,'font-family':this.font.display,'font-weight':700,'font-size':100,fill:'#3d6b48'},g);
    txt(DATA.subtitle||'',{x:74,y:178,'font-family':this.font.body,'font-size':30,fill:this.ink},g);
    return {x:30,y:30,w:560,h:180}},
  caption(g){const B={x:50,y:H-160,w:W-100,h:118};
    el('rect',{x:B.x,y:B.y,width:B.w,height:B.h,fill:this.paper,filter:'url(#wLand)',opacity:.95},g);
    const day=txt('',{x:B.x+30,y:B.y+74,'font-family':this.font.display,'font-weight':700,'font-size':52},g);
    const name=txt('',{x:B.x+230,y:B.y+56,'font-family':this.font.body,'font-size':42,fill:this.ink},g);
    const note=txt('',{x:B.x+232,y:B.y+94,'font-family':this.font.body,'font-size':28,fill:this.ink,opacity:.75},g);
    return {box:{x:B.x,y:B.y-6,w:B.w,h:B.h+12},set:(s,i)=>{day.setAttribute('fill',s.color);day.textContent=s.day||`Stop ${i+1}`;name.textContent=s.name;note.textContent=s.note||''}}},
  frame(g){const f=el('filter',{id:'deckle',x:'-5%',y:'-5%',width:'110%',height:'110%'},defs);
    el('feTurbulence',{type:'fractalNoise',baseFrequency:.06,numOctaves:2,seed:9},f);el('feDisplacementMap',{in:'SourceGraphic',scale:14},f);
    el('rect',{x:0,y:0,width:W,height:H,fill:'none',stroke:'#fffaf0','stroke-width':36,filter:'url(#deckle)'},g)}
},

/* ---------- INKWASH: Chinese ink painting with red seals ---------- */
inkwash:{
  palette:['#b22c2c','#b22c2c','#b22c2c','#b22c2c','#b22c2c','#b22c2c','#b22c2c','#b22c2c'],
  ink:'#1f1d1a', paper:'#f3ecdc', seal:'#b22c2c',
  font:{display:"'Ma Shan Zheng',serif",body:"'Cormorant Garamond',serif"},
  NUM:['一','二','三','四','五','六','七','八'],
  background(){
    el('rect',{width:W,height:H,fill:this.paper});
    const ink=el('filter',{id:'inkw',x:'-10%',y:'-10%',width:'120%',height:'120%'},defs);
    el('feTurbulence',{type:'fractalNoise',baseFrequency:.015,numOctaves:4,seed:4,result:'n'},ink);
    el('feDisplacementMap',{in:'SourceGraphic',in2:'n',scale:16,result:'d'},ink);el('feGaussianBlur',{in:'d',stdDeviation:3},ink);
    rough('brush','.06',5);
    const sh=el('g',{filter:'url(#inkw)'});
    DATA.land.forEach(c=>{el('path',{d:c.d,fill:c.visited?'#c9c3b6':'#ddd7ca',opacity:.95},sh);el('path',{d:c.d,fill:'none',stroke:'#7d776c','stroke-width':16,opacity:.35},sh)});
    const b=el('g',{filter:'url(#brush)'});DATA.land.forEach(c=>el('path',{d:c.d,fill:'none',stroke:this.ink,'stroke-width':2.6,opacity:.85},b));
    grain(.10);
  },
  route(d,color,g){return [el('path',{d,fill:'none',stroke:this.ink,'stroke-width':5,'stroke-linecap':'round',filter:'url(#brush)'},g)]},
  tip(g){const c=el('circle',{r:8,fill:this.ink,opacity:0},g);return {set(p,on){c.setAttribute('cx',p.x);c.setAttribute('cy',p.y);c.setAttribute('opacity',on?1:0)}}},
  marker(g,s){el('rect',{x:-20,y:-20,width:40,height:40,rx:4,fill:this.seal,filter:'url(#brush)'},g);
    txt(this.NUM[s.i]||String(s.i+1),{y:10,'text-anchor':'middle','font-family':this.font.display,'font-size':30,fill:this.paper},g)},
  labelText(g,s){const d=haloText(s.day||'',{'font-family':this.font.body,'font-weight':700,'font-size':22,'letter-spacing':'.08em',fill:this.seal},g,this.paper,6);
    const n=haloText(s.name,{'font-family':this.font.display,'font-size':40,fill:this.ink},g,this.paper,9);
    const w=Math.max(n.getBBox().width,d.getBBox().width)+12,h=s.day?74:48;
    return {w,h,place(b){d.setAttribute('x',b.x+6);d.setAttribute('y',b.y+22);n.setAttribute('x',b.x+6);n.setAttribute('y',b.y+(s.day?64:40))}}},
  labelShape(back,b,s){leader(back,b,s,this.ink)},
  title(g){haloText(DATA.title,{x:70,y:140,'font-family':this.font.display,'font-size':96,fill:this.ink},g,this.paper,10);
    const tw=g.lastChild.getBBox().width;
    el('rect',{x:86+tw,y:62,width:58,height:96,rx:4,fill:this.seal,filter:'url(#brush)'},g);
    txt('旅',{x:115+tw,y:102,'text-anchor':'middle','font-family':this.font.display,'font-size':34,fill:this.paper},g);
    txt('行',{x:115+tw,y:142,'text-anchor':'middle','font-family':this.font.display,'font-size':34,fill:this.paper},g);
    haloText(DATA.subtitle||'',{x:74,y:186,'font-family':this.font.body,'font-style':'italic','font-weight':600,'font-size':30,fill:this.ink},g,this.paper,8);
    return {x:50,y:40,w:tw+120,h:170}},
  caption(g){const B={x:60,y:H-160,w:W-120,h:110};
    el('line',{x1:B.x,y1:B.y,x2:B.x+B.w,y2:B.y,stroke:this.ink,'stroke-width':3,filter:'url(#brush)'},g);
    el('line',{x1:B.x,y1:B.y+B.h,x2:B.x+B.w,y2:B.y+B.h,stroke:this.ink,'stroke-width':2,filter:'url(#brush)',opacity:.6},g);
    el('rect',{x:B.x,y:B.y-8,width:B.w,height:B.h+16,fill:this.paper,opacity:.85},g);g.appendChild(g.children[0]);g.appendChild(g.children[0]);
    el('rect',{x:B.x+10,y:B.y+20,width:70,height:70,rx:5,fill:this.seal,filter:'url(#brush)'},g);
    const num=txt('',{x:B.x+45,y:B.y+68,'text-anchor':'middle','font-family':this.font.display,'font-size':44,fill:this.paper},g);
    const name=txt('',{x:B.x+110,y:B.y+56,'font-family':this.font.display,'font-size':44,fill:this.ink},g);
    const note=txt('',{x:B.x+112,y:B.y+92,'font-family':this.font.body,'font-style':'italic','font-weight':600,'font-size':28,fill:this.ink},g);
    return {box:{x:B.x,y:B.y-10,w:B.w,h:B.h+20},set:(s,i)=>{num.textContent=this.NUM[i]||String(i+1);name.textContent=`${s.day?s.day+' · ':''}${s.name}`;note.textContent=s.note||''}}},
  frame(){}
}
});

/* ---------- REALM: hand-inked medieval fantasy atlas ---------- */
/* Land mask on a coarse grid, used to scatter terrain symbols on land only */
function landMask(cell){const cw=Math.ceil(W/cell),ch=Math.ceil(H/cell),cv=document.createElement('canvas');cv.width=cw;cv.height=ch;
  const c=cv.getContext('2d');c.scale(1/cell,1/cell);DATA.land.forEach(l=>{c.fillStyle=l.visited?'#f00':'#0f0';c.fill(new Path2D(l.d))});
  const px=c.getImageData(0,0,cw,ch).data;return (x,y)=>{const i=(y*cw+x)*4;return x<0||y<0||x>=cw||y>=ch||px[i+3]<200?0:(px[i]>px[i+1]?1:2)}}
Object.assign(THEMES,{
realm:{
  palette:['#8e1b1b','#8e1b1b','#8e1b1b','#8e1b1b','#8e1b1b','#8e1b1b','#8e1b1b','#8e1b1b'],
  ink:'#2e2318', vellum:'#efe3c4', red:'#8e1b1b', faint:'#8c7b5c',
  font:{display:"'Uncial Antiqua',serif",title:"'UnifrakturMaguntia',serif",body:"'IM Fell English',serif"},
  background(){
    el('rect',{width:W,height:H,fill:this.vellum});
    const st=el('filter',{id:'age',x:0,y:0,width:'100%',height:'100%'},defs);
    el('feTurbulence',{type:'fractalNoise',baseFrequency:.004,numOctaves:3,seed:21},st);
    el('feColorMatrix',{values:'0 0 0 0 .55  0 0 0 0 .42  0 0 0 0 .2  0 0 0 1 -.45'},st);
    el('rect',{width:W,height:H,filter:'url(#age)',opacity:.35});
    rough('rough','.03',2.5);rough('roughS','.05',2);
    // classic fantasy-map water lining: concentric lines echoing every coast
    const wl=el('g',{opacity:.55});
    [[40,36],[28,24],[16,12]].forEach(([a,b])=>{DATA.land.forEach(c=>{el('path',{d:c.d,fill:'none',stroke:this.faint,'stroke-width':a,'stroke-linejoin':'round'},wl);el('path',{d:c.d,fill:'none',stroke:this.vellum,'stroke-width':b,'stroke-linejoin':'round'},wl)})});
    const g=el('g',{filter:'url(#rough)'});
    DATA.land.forEach(c=>el('path',{d:c.d,fill:c.visited?'#f4ead0':'#e8dab6',stroke:this.ink,'stroke-width':2.6,'stroke-linejoin':'round'},g));
    // terrain symbols: hills and little forests, kept clear of the stops
    const cell=34,land=landMask(cell),sym=el('g',{stroke:this.ink,'stroke-width':1.8,fill:'none','stroke-linecap':'round','stroke-linejoin':'round',opacity:.8});
    const hash=(x,y)=>((x*2654435761)^(y*40503))>>>0, near=(x,y)=>DATA.stops.some(s=>Math.hypot(s.x-x,s.y-y)<70);
    for(let y=0;y<Math.ceil(H/cell);y++)for(let x=0;x<Math.ceil(W/cell);x++){
      if(!land(x,y)||!land(x+1,y)||!land(x,y+1)||!land(x-1,y))continue;
      const h=hash(x,y),px=x*cell+(h%13)-6,py=y*cell+((h>>4)%13)-6;if(near(px,py)||h%3===0)continue;
      if(h%5<2){// hill
        el('path',{d:`M${px-11} ${py+6} Q${px} ${py-12} ${px+11} ${py+6}`,fill:this.vellum},sym);el('path',{d:`M${px-3} ${py-3} q4 2 5 8`},sym)}
      else{// tree
        el('circle',{cx:px,cy:py-4,r:6,fill:'#e2d2a6'},sym);el('path',{d:`M${px} ${py+2} V${py+9}`},sym)}}
    // a small sailing ship in open water
    const L=landMask(10),open=(x,y)=>{for(let a=-9;a<=9;a+=3)for(let b=-7;b<=5;b+=3)if(L(Math.round(x/10)+a,Math.round(y/10)+b))return false;return true};
    for(let k=0;k<1500;k++){const x=80+((k*7919)%(W-160)),y=280+((k*104729)%(H-520));
      if(open(x,y)&&!DATA.stops.some(s=>Math.hypot(s.x-x,s.y-y)<160)){
        const sh=el('g',{transform:`translate(${x} ${y})`,stroke:this.ink,'stroke-width':2,fill:this.vellum,'stroke-linejoin':'round'});
        el('path',{d:'M-24 0 Q0 14 24 0 Z'},sh);el('path',{d:'M0 0 V-34'},sh);el('path',{d:'M2 -32 Q16 -20 2 -6 Z'},sh);el('path',{d:'M-2 -28 Q-14 -18 -2 -8 Z'},sh);
        el('path',{d:'M-34 10 q6 -5 12 0 q6 -5 12 0 M10 12 q6 -5 12 0 q6 -5 12 0',fill:'none'},sh);break}}
  },
  route(d,color,g){return maskedRoute(d,[{stroke:this.red,'stroke-width':4,'stroke-dasharray':'10 8','stroke-linecap':'round'}],g)},
  tip(g){const c=el('circle',{r:7,fill:this.red,stroke:this.ink,'stroke-width':2,opacity:0},g);return {set(p,on){c.setAttribute('cx',p.x);c.setAttribute('cy',p.y);c.setAttribute('opacity',on?1:0)}}},
  marker(g,s){// a little castle tower with a pennant
    const a={stroke:this.ink,'stroke-width':2.4,'stroke-linejoin':'round'};
    el('path',{d:'M-13 18 V-6 H-9 V-11 H-4 V-6 H4 V-11 H9 V-6 H13 V18 Z',fill:this.vellum,...a},g);
    el('path',{d:'M-4 18 V9 Q0 4 4 9 V18',fill:this.ink},g);
    el('path',{d:'M0 -11 V-26',...a},g);el('path',{d:'M0 -26 L14 -22 L0 -18 Z',fill:this.red,stroke:this.ink,'stroke-width':1.5},g)},
  labelText(g,s){const d=haloText(s.day||'',{'font-family':this.font.body,'font-style':'italic','font-size':23,fill:this.red},g,this.vellum,6);
    const n=haloText(s.name,{'font-family':this.font.display,'font-size':32,fill:this.ink},g,this.vellum,8);
    const w=Math.max(n.getBBox().width,d.getBBox().width)+14,h=s.day?72:46;
    return {w,h,place(b){d.setAttribute('x',b.x+7);d.setAttribute('y',b.y+22);n.setAttribute('x',b.x+7);n.setAttribute('y',b.y+(s.day?60:36))}}},
  labelShape(back,b,s){leader(back,b,s,this.ink,'2 6')},
  title(g){const w=Math.min(W-140,Math.max(600,DATA.title.length*44+140)),x=(W-w)/2,y=48,h=150;
    el('rect',{x,y,width:w,height:h,fill:this.vellum,stroke:this.ink,'stroke-width':3},g);
    el('rect',{x:x+8,y:y+8,width:w-16,height:h-16,fill:'none',stroke:this.red,'stroke-width':1.5},g);
    [[x,y],[x+w,y],[x,y+h],[x+w,y+h]].forEach(([cx,cy])=>el('rect',{x:cx-9,y:cy-9,width:18,height:18,transform:`rotate(45 ${cx} ${cy})`,fill:this.red,stroke:this.ink,'stroke-width':2},g));
    txt(DATA.title,{x:W/2,y:y+88,'text-anchor':'middle','font-family':this.font.title,'font-size':76,fill:this.ink},g);
    txt('~ '+(DATA.subtitle||'')+' ~',{x:W/2,y:y+126,'text-anchor':'middle','font-family':this.font.body,'font-style':'italic','font-size':25,fill:this.red},g);
    return {x:x-12,y:y-12,w:w+24,h:h+24}},
  caption(g){const B={x:70,y:H-170,w:W-140,h:118};
    el('rect',{x:B.x,y:B.y,width:B.w,height:B.h,fill:this.vellum,stroke:this.ink,'stroke-width':3},g);
    el('rect',{x:B.x+8,y:B.y+8,width:B.w-16,height:B.h-16,fill:'none',stroke:this.red,'stroke-width':1.5},g);
    [[B.x,B.y],[B.x+B.w,B.y],[B.x,B.y+B.h],[B.x+B.w,B.y+B.h]].forEach(([cx,cy])=>el('rect',{x:cx-8,y:cy-8,width:16,height:16,transform:`rotate(45 ${cx} ${cy})`,fill:this.red,stroke:this.ink,'stroke-width':2},g));
    const day=txt('',{x:B.x+36,y:B.y+72,'font-family':this.font.title,'font-size':44,fill:this.red},g);
    const name=txt('',{x:B.x+230,y:B.y+56,'font-family':this.font.display,'font-size':38,fill:this.ink},g);
    const note=txt('',{x:B.x+232,y:B.y+92,'font-family':this.font.body,'font-style':'italic','font-size':27,fill:this.ink},g);
    return {box:{x:B.x-12,y:B.y-12,w:B.w+24,h:B.h+24},set:(s,i)=>{day.textContent=s.day||`Stop ${i+1}`;name.textContent=s.name;note.textContent=s.note||''}}},
  frame(g){el('rect',{x:12,y:12,width:W-24,height:H-24,fill:'none',stroke:this.ink,'stroke-width':6},g);
    el('rect',{x:24,y:24,width:W-48,height:H-48,fill:'none',stroke:this.ink,'stroke-width':1.5},g);
    for(let k=0;k*40<W-80;k++){[[44+k*40,18],[44+k*40,H-18]].forEach(([x,y])=>el('rect',{x:x-3,y:y-3,width:6,height:6,transform:`rotate(45 ${x} ${y})`,fill:this.red},g))}
    for(let k=0;k*40<H-80;k++){[[18,44+k*40],[W-18,44+k*40]].forEach(([x,y])=>el('rect',{x:x-3,y:y-3,width:6,height:6,transform:`rotate(45 ${x} ${y})`,fill:this.red},g))}}
}
});

const BASE={tip(g){const c=el('circle',{r:8,fill:'#fff',stroke:this.ink,'stroke-width':3,opacity:0},g);return {set(p,on){c.setAttribute('cx',p.x);c.setAttribute('cy',p.y);c.setAttribute('opacity',on?1:0)}}},
  arriveFx(){},frame(){},labelShape(){}};
const T=Object.assign({},BASE,THEMES[DATA.theme]||THEMES.brick);

/* ================================================================
   BUILD
   ================================================================ */
T.background();
const S=DATA.stops.map((s,i)=>({...s,i,color:T.palette[i%T.palette.length]}));
// nudge markers that sit on top of each other
S.forEach((s,i)=>{for(let j=0;j<i;j++){const o=S[j],dx=s.x-o.x,dy=s.y-o.y,d=Math.hypot(dx,dy);
  if(d<46){const ux=d>1?dx/d:1,uy=d>1?dy/d:0;s.x=o.x+ux*46;s.y=o.y+uy*46}}});
// routes: gentle curves, coloured by destination
const routeG=el('g'),legs=[];
for(let i=1;i<S.length;i++){const a=S[i-1],b=S[i],mx=(a.x+b.x)/2,my=(a.y+b.y)/2,dx=b.x-a.x,dy=b.y-a.y,L=Math.hypot(dx,dy)||1,bend=(i%2?1:-1)*Math.min(.22*L,90);
  const d=`M${a.x} ${a.y} Q ${mx-dy/L*bend} ${my+dx/L*bend} ${b.x} ${b.y}`;
  const paths=T.route(d,b.color,routeG);legs.push({paths,main:paths[paths.length-1],len:paths[paths.length-1].getTotalLength()})}
// layers, bottom to top
const labelG=el('g'),fxG=el('g'),markG=el('g'),tipG=el('g'),titleG=el('g'),capG=el('g'),frameG=el('g');
const tip=T.tip(tipG);
S.forEach(s=>{s.mark=el('g',{},markG);T.marker(s.mark,s);s.fxG=fxG});

function overlap(a,b){const x=Math.max(0,Math.min(a.x+a.w,b.x+b.w)-Math.max(a.x,b.x)),y=Math.max(0,Math.min(a.y+a.h,b.y+b.h)-Math.max(a.y,b.y));return x*y}
let CAP;
function layout(){
  const TB=T.title(titleG);CAP=T.caption(capG);
  const blocked=[TB,CAP.box];
  S.forEach(s=>blocked.push({x:s.x-28,y:s.y-28,w:56,h:56,mark:true}));
  legs.forEach(l=>{for(let k=0;k<=12;k++){const p=l.main.getPointAtLength(l.len*k/12);blocked.push({x:p.x-8,y:p.y-8,w:16,h:16,soft:true})}});
  // labels: try 8 directions × 3 distances, keep the spot with the least overlap
  S.forEach(s=>{const g=el('g',{},labelG);s.lab=g;const back=el('g',{},g);const L=T.labelText(g,s),w=L.w,h=L.h;
    const cands=[];[38,90,145].forEach((R,ri)=>[[R,-h/2],[-R-w,-h/2],[-w/2,-R-h],[-w/2,R],[R*.75,-R*.75-h],[R*.75,R*.75],[-R*.75-w,-R*.75-h],[-R*.75-w,R*.75]].forEach(c=>cands.push([...c,ri])));
    let best=null;
    cands.forEach(([dx,dy,ri],k)=>{const b={x:s.x+dx,y:s.y+dy,w,h};let sc=ri*900+(k%8)*30;
      if(b.x<24||b.y<24||b.x+w>W-24||b.y+h>H-24)sc+=1e7;
      blocked.forEach(o=>{const a=overlap(b,o);if(a)sc+=o.soft?a*.6:(o.mark?a*40:a*10)});
      if(!best||sc<best.sc)best={...b,sc}});
    blocked.push(best);T.labelShape(back,best,s);L.place(best)});
  T.frame(frameG);
}

/* ================================================================
   TIMELINE — render(t) draws the exact frame at time t (seconds)
   ================================================================ */
const START=1.0,DRAW=1.3,PAUSE=0.8,HOLD=2.6;
const TOTAL=START+legs.length*(DRAW+PAUSE)+HOLD;
const ease=x=>x<.5?2*x*x:1-Math.pow(-2*x+2,2)/2;
const pop=x=>x<=0?0:x>=1?1:(x<.6?1.18*ease(x/.6):1.18-.18*ease((x-.6)/.4));
const arrive=S.map((s,i)=>i===0?.2:START+(i-1)*(DRAW+PAUSE)+DRAW);
function render(t){
  let tipOn=false;
  legs.forEach((l,i)=>{const s0=START+i*(DRAW+PAUSE),k=ease(Math.max(0,Math.min(1,(t-s0)/DRAW)));
    l.paths.forEach(p=>{p.style.strokeDasharray=l.len;p.style.strokeDashoffset=l.len*(1-k)});
    if(k>0&&k<1){const p=l.main.getPointAtLength(l.len*k),q=l.main.getPointAtLength(Math.max(0,l.len*k-6)),dd=Math.hypot(p.x-q.x,p.y-q.y)||1;
      tip.set(p,true,{x:(p.x-q.x)/dd,y:(p.y-q.y)/dd});tipOn=true}});
  if(!tipOn)tip.set({x:0,y:0},false,{x:1,y:0});
  let cur=0;
  S.forEach((s,i)=>{s.mark.setAttribute('transform',`translate(${s.x} ${s.y}) scale(${pop((t-arrive[i])/.45)})`);
    s.lab.style.opacity=Math.max(0,Math.min(1,(t-arrive[i]-.15)/.3));
    T.arriveFx(s,(t-arrive[i])/.55);
    if(t>=arrive[i]-.05)cur=i});
  CAP.set(S[cur],cur);
}
window.TOTAL=TOTAL;window.render=render;
Promise.all([...document.fonts].map(f=>f.load().catch(()=>{}))).then(()=>document.fonts.ready).then(()=>{layout();window.READY=true;
  if(!location.hash.includes('record')){let t0=null;(function loop(ts){if(t0===null)t0=ts;render(((ts-t0)/1000)%TOTAL);requestAnimationFrame(loop)})(performance.now())}
  else render(TOTAL)});
</script>
</body>
</html>
```

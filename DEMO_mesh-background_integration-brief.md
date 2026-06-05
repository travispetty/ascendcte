# DEMO Site — Site-Wide Animated Mesh Background + Color Picker
### Integration brief (hand this to the chat that holds the demo files)

**Goal:** Replace the pre-rendered WebP background with a single-color **animated CSS mesh** that paints in the existing `position: fixed` background layer, so all page content scrolls *over* it. Add a production picker (Teal / Crimson / Blue + freeform color) that dials `--brand` live and persists the choice across page navigation.

**This reverses the WebP background decision on purpose.** A freeform picker can't use pre-rendered images — there is no WebP for an unknown color. The mesh generates any color at runtime, which is the whole point. Net effect on iOS is neutral-to-better: the mesh's solid `--brand` base reaches the Safari chrome zone, which a WebP image never could.

---

## ⚠️ Must-do checklist (no regressions)

- [ ] **Reuse the existing fixed background element.** Do NOT add a second one. Repurpose the current WebP layer → mesh.
- [ ] **Remove all WebP background code:** the `data-tint → url(...webp)` rules, the portrait/landscape `min-aspect-ratio` / `min-width:720px` media query that swapped WebP, and the WebP `<link rel="preload">` hints.
- [ ] **Remove WebP assets from the `sw.js` precache list.**
- [ ] **Add the before-paint persistence script** to the `<head>` of **every** page (including `admin.html`) — as the first thing in `<head>`, before the stylesheet link, or you'll get a color flash on navigation.
- [ ] **Apply the mesh CSS** to the fixed layer + the foreground/`--brand` variables (shared CSS file).
- [ ] **Replace the current picker widget** on `Index.html` only.
- [ ] **Do NOT use `background-attachment: fixed`** (still unreliable on iOS). The animation is `background-position`, which is safe.
- [ ] **Bump `sw.js` `VERSION`** to the next sequential integer and ship `sw.js` in the output alongside every changed HTML.
- [ ] **`admin.html` included** in the global background change.
- [ ] **Test on a real iOS device** before trusting it.

---

## 1. The fixed mesh layer (shared CSS)

Apply to your existing fixed background element. Example uses `#bg-layer` — map it to the real element id/class.

```css
/* base color reaches the iOS chrome zone */
html{ background: var(--brand); }
body{ background: transparent; }

/* the fixed background — content scrolls OVER this */
#bg-layer{
  position: fixed;
  inset: 0;
  z-index: -1;                 /* sits behind all normal-flow content */
  background:
    radial-gradient(at 16% 20%, var(--s1) 0px, transparent 55%),
    radial-gradient(at 84% 14%, var(--s2) 0px, transparent 55%),
    radial-gradient(at 74% 84%, var(--s3) 0px, transparent 55%),
    radial-gradient(at 22% 82%, var(--s4) 0px, transparent 55%),
    var(--brand);
  background-size: 180% 180%;
  animation: meshDrift 22s ease-in-out infinite;
}
@keyframes meshDrift{0%,100%{background-position:0% 0%}50%{background-position:100% 100%}}

/* subtle grain so flat color isn't plastic */
#bg-layer::after{
  content:""; position:absolute; inset:0; opacity:.05; pointer-events:none;
  background-image:url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='120' height='120'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.8' numOctaves='3'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)'/%3E%3C/svg%3E");
}

/* accessibility: kill motion if the user asked for it */
@media (prefers-reduced-motion: reduce){ #bg-layer{ animation:none } }
```

**z-index note:** `z-index:-1` lets every normal-flow element paint over the fixed layer with no content wrapper needed. Just make sure no page section sets an opaque `background` that you didn't intend (those will cover the mesh — which is correct for cards/panels, wrong for full-bleed sections).

---

## 2. The color engine (shared CSS)

```css
:root{
  --brand:#004963;                       /* default: AscendCTE teal */

  --s1: color-mix(in oklab, var(--brand), white 26%);
  --s2: color-mix(in oklab, var(--brand), black 42%);
  --s3: color-mix(in oklab, var(--brand), white 12%);
  --s4: color-mix(in oklab, var(--brand), black 30%);
  --crest: color-mix(in oklab, var(--brand), white 22%);

  /* auto foreground — flips black/white at OKLCH L 0.58 */
  --ink:     oklch(from var(--brand) clamp(0, (0.58 - l) * 1000, 1) 0 0);
  --ink-inv: oklch(from var(--brand) clamp(0, (l - 0.58) * 1000, 1) 0 0);
  --ink-dim: color-mix(in oklab, var(--ink), var(--brand) 28%);
}

/* presets — used for the nice old-browser fallback below */
html[data-tint="teal"]    { --brand:#004963; }
html[data-tint="crimson"] { --brand:#7a1626; }
html[data-tint="blue"]    { --brand:#123a6b; }

/* OLD BROWSERS (no color-mix / relative-color): exact matched hex */
@supports not ((background: color-mix(in oklab, red, blue)) and (color: oklch(from red l c h))){
  :root{ --s1:var(--brand);--s2:var(--brand);--s3:var(--brand);--s4:var(--brand);--crest:var(--brand);
         --ink:#fff;--ink-inv:#000;--ink-dim:#b9c9d2; }
  :root, html[data-tint="teal"]{ --s1:#4c758a;--s2:#001e2b;--s3:#2a5d75;--s4:#002a3a;--crest:#426e84;
         --ink:#fff;--ink-inv:#000;--ink-dim:#b9c9d2; }
  html[data-tint="crimson"]{ --s1:#a15659;--s2:#37050c;--s3:#8c363d;--s4:#490913;--crest:#9b4d51;
         --ink:#fff;--ink-inv:#000;--ink-dim:#debdbd; }
  html[data-tint="blue"]{ --s1:#4d6a91;--s2:#04162f;--s3:#2e507d;--s4:#07203f;--crest:#44638b;
         --ink:#fff;--ink-inv:#000;--ink-dim:#b8c5d5; }
}
```

**Foreground caution:** only put `color: var(--ink)` on text that sits *directly on the mesh* (hero copy, headers over the background). Do NOT set `--ink` on `body` globally, or text inside white/light cards will flip too. Scope it.

---

## 3. Before-paint persistence (every page `<head>`, FIRST)

Place this as the **first** element inside `<head>`, before the stylesheet link. It re-applies the saved color before the browser paints, so there's no flash on navigation.

```html
<script>
(function(){
  try{
    var s = JSON.parse(localStorage.getItem('ac_brand') || 'null');
    if(!s) return;
    var el = document.documentElement;
    if(s.mode === 'tint'){ el.setAttribute('data-tint', s.value); }
    else if(s.mode === 'custom'){ el.style.setProperty('--brand', s.value); }
  }catch(e){}
})();
</script>
```

This extends your existing `data-tint` mechanism rather than replacing it: presets persist as `{mode:'tint'}` (so old browsers still get the matched fallback), freeform persists as `{mode:'custom'}`.

---

## 4. Production picker widget (Index.html only)

Teal / Crimson / Blue + freeform swatch. No text description. Glassy, adapts to `--ink` so it stays legible on any brand color.

```html
<div class="ac-picker" role="group" aria-label="District color">
  <button class="ac-swatch" data-tint="teal"    style="--sw:#004963" aria-label="Teal"></button>
  <button class="ac-swatch" data-tint="crimson" style="--sw:#7a1626" aria-label="Crimson"></button>
  <button class="ac-swatch" data-tint="blue"     style="--sw:#123a6b" aria-label="Blue"></button>
  <label class="ac-swatch ac-custom" aria-label="Custom color">
    <input type="color" id="acColor" value="#004963">
  </label>
</div>
```

```css
.ac-picker{
  position:fixed; bottom:18px; right:18px; z-index:50;
  display:flex; gap:8px; padding:9px;
  background:color-mix(in oklab, var(--ink-inv), transparent 18%);
  border:1px solid color-mix(in oklab, var(--ink), transparent 70%);
  border-radius:999px; backdrop-filter:blur(12px);
}
.ac-swatch{
  width:26px; height:26px; border-radius:50%; cursor:pointer; padding:0;
  background:var(--sw); border:2px solid color-mix(in oklab, var(--ink), transparent 55%);
  transition:transform .12s, border-color .12s;
}
.ac-swatch:hover{ transform:scale(1.12) }
.ac-swatch[aria-pressed="true"]{ border-color:var(--ink); transform:scale(1.12) }
.ac-custom{ position:relative; overflow:hidden;
  background:conic-gradient(red,orange,yellow,lime,cyan,blue,magenta,red); }
.ac-custom input{ position:absolute; inset:-6px; width:200%; height:200%;
  border:none; padding:0; cursor:pointer; opacity:0 }
```

```javascript
(function(){
  var root = document.documentElement;
  var swatches = document.querySelectorAll('.ac-swatch[data-tint]');
  var color = document.getElementById('acColor');
  var PRESET = { teal:'#004963', crimson:'#7a1626', blue:'#123a6b' };

  function save(o){ try{ localStorage.setItem('ac_brand', JSON.stringify(o)); }catch(e){} }
  function clearActive(){ swatches.forEach(function(s){ s.setAttribute('aria-pressed','false'); }); }
  function setTheme(hex){ var m=document.querySelector('meta[name="theme-color"]'); if(m) m.setAttribute('content',hex); }

  swatches.forEach(function(s){
    s.addEventListener('click', function(){
      var t = s.dataset.tint;
      root.style.removeProperty('--brand');     // let data-tint drive
      root.setAttribute('data-tint', t);
      clearActive(); s.setAttribute('aria-pressed','true');
      color.value = PRESET[t]; setTheme(PRESET[t]);
      save({ mode:'tint', value:t });
    });
  });

  color.addEventListener('input', function(){
    root.removeAttribute('data-tint');          // freeform overrides preset
    root.style.setProperty('--brand', color.value);
    clearActive(); setTheme(color.value);
    save({ mode:'custom', value:color.value });
  });

  // sync widget UI to the saved state on load
  (function(){
    var s; try{ s = JSON.parse(localStorage.getItem('ac_brand')||'null'); }catch(e){}
    if(!s){ document.querySelector('.ac-swatch[data-tint="teal"]').setAttribute('aria-pressed','true'); return; }
    if(s.mode==='tint'){
      var el=document.querySelector('.ac-swatch[data-tint="'+s.value+'"]');
      if(el){ el.setAttribute('aria-pressed','true'); color.value=PRESET[s.value]; }
    } else if(s.mode==='custom'){ color.value=s.value; }
  })();
})();
```

---

## 5. iOS chrome-zone color (nice-to-have)

Add a `theme-color` meta to each page; the picker JS above updates it live so the Safari bar matches the brand:

```html
<meta name="theme-color" content="#004963">
```

---

## 6. The one honest cost of "animated, site-wide"

A full-viewport gradient animating forever repaints continuously — measurable battery/scroll cost on older phones, which is partly why we went WebP. You asked for it knowingly; the `prefers-reduced-motion` block respects users who opt out. If a district demo ever stutters on a cheap device, the one-line fix is dropping `animation` from `#bg-layer` (static mesh) — same look, no repaint. Keep that in your back pocket.

---

## 7. Output expectations for the implementing chat

- Every changed HTML page + `admin.html`
- `sw.js` with `VERSION` bumped to next integer, WebP assets removed from precache
- Confirmation that WebP background CSS + preload hints were removed

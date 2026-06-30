# spig-ad-dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 `neurosphere.html` 的白色 dashboard 廣告模組，瘦身成一個可 inline 注入文章頁的 self-contained HTML 頁。

**Architecture:** 單一新檔 `spig-ad-dashboard.html`，內含模擬文章頁 + 一段自執行 IIFE。IIFE 建立 host `<div>` 並 `attachShadow`，在 shadow root 內注入 `<style>` 與廣告 HTML，動畫全用原生（IntersectionObserver + WAAPI + requestAnimationFrame）取代 GSAP。

**Tech Stack:** 原生 HTML / CSS / JS（Web Animations API、IntersectionObserver、Shadow DOM）。零外部依賴。

## Global Constraints

- 投放方式 inline 注入文章 DOM：不得有任何全域變數洩漏（全程包在 IIFE 內）。
- CSS 隔離用 Shadow DOM（`attachShadow({ mode: 'open' })`）：所有樣式只存在 shadow root 內。
- 零外部 JS（不得引用 gsap / ScrollTrigger 或任何 CDN script）。
- 零外部字體請求：全用系統字堆疊。中文 `"PingFang TC", "Microsoft JhengHei", system-ui, sans-serif`；mono `ui-monospace, "SF Mono", Menlo, monospace`。
- 響應式：保留 desktop（viewBox 1000×540）+ mobile（viewBox 420×900）兩套 SVG frame，用 `window.matchMedia("(max-width: 760px)")` 切換。
- 動畫視覺需與目前 `neurosphere.html` dashboard 模組一致（進場時序、跑道燈循環、文字解碼）。
- **不得修改 `neurosphere.html`**。
- 來源參考：`neurosphere.html`（dashboard 模組 HTML 在 296–341 行、相關 CSS 在 99–266 行、動畫 JS 在 362–526 行）。

---

### Task 1: demo 頁骨架 + Shadow DOM host

**Files:**
- Create: `spig-ad-dashboard.html`

**Interfaces:**
- Produces: 全域可預覽頁；一個 `<div id="spig-ad-slot">` 注入點；IIFE 內建立 `host` 元素與 `root = host.attachShadow({mode:'open'})`。

- [ ] **Step 1: 建立檔案骨架**

```html
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>spig-ad-dashboard · inline embed preview</title>
  <style>
    /* 僅供預覽：模擬宿主文章頁，故意放一些會污染的規則來驗證 Shadow DOM 隔離 */
    body { font-family: Georgia, serif; max-width: 720px; margin: 0 auto; padding: 40px 20px; color: #222; }
    * { box-sizing: content-box; }              /* 宿主的危險全域，廣告不該被影響 */
    img { max-width: 100%; }
    h2 { color: crimson; }                       /* 若滲進 shadow，標題會變紅 → 用來抓洩漏 */
  </style>
</head>
<body>
  <h1>模擬文章標題</h1>
  <p>這是一段假文章內文，用來模擬廣告被 inline 注入文章頁的情境。下面是注入點。</p>
  <div id="spig-ad-slot"></div>
  <p>廣告之後還有更多文章內文，確認廣告樣式沒有外洩影響到這裡。</p>

  <script>
  (function () {
    "use strict";
    const slot = document.getElementById("spig-ad-slot");
    const host = document.createElement("div");
    host.className = "spig-ad-host";
    slot.appendChild(host);
    const root = host.attachShadow({ mode: "open" });
    root.innerHTML = `<style></style><div class="ad"></div>`;
    // 後續 task 會填入 style 與廣告結構
  })();
  </script>
</body>
</html>
```

- [ ] **Step 2: 瀏覽器驗證**

在瀏覽器開啟 `spig-ad-dashboard.html`，開 DevTools Console 執行：
```js
document.querySelector(".spig-ad-host").shadowRoot.querySelector(".ad")  // 應回傳 <div class="ad">
Object.keys(window).filter(k => /gsap|ScrollTrigger|spig/i.test(k))      // 應為 []
```
Expected: shadow root 內有 `.ad`；無任何相關全域變數。

- [ ] **Step 3: Commit**

```bash
git add spig-ad-dashboard.html
git commit -m "feat(spig-ad): demo 頁骨架 + Shadow DOM host"
```

---

### Task 2: 移植廣告 HTML 結構 + 精簡 CSS（系統字）

**Files:**
- Modify: `spig-ad-dashboard.html`（填入 `root.innerHTML` 的 style 與 HTML）

**Interfaces:**
- Consumes: Task 1 的 `root`。
- Produces: shadow 內完整靜態廣告 DOM——`.frame-desktop` / `.frame-mobile` 兩個 `<svg>`（含 `.hud-path` fill、`.outline`、`.plate.block`、`.bay` group）、`.ad__inner`（`.ad__title .lead` + 三個 `.ad__row`）。class 命名與 `neurosphere.html` 一致。

- [ ] **Step 1: 把廣告 HTML 放進 shadow**

從 `neurosphere.html` 298–340 行複製 `<article class="ad light dashboard concept-a">` 內部結構（兩個 svg frame + `.ad__inner`），改放進 `root.innerHTML` 的 `.ad` 容器內。`<article>` 的 class（`ad light dashboard concept-a`）改套到 shadow 內最外層 `.ad` div。

- [ ] **Step 2: 移植並精簡 CSS 放進 shadow `<style>`**

只保留 dashboard 模組會用到的規則，從 `neurosphere.html` CSS 擷取並改寫：
- 取 `.ad`（99–104）、`.frame` 與其子元素（106–110）、`.ad__title`/`.ad__rows`/`.ad__row`/`.meta`/`.sdot`/`.copy`（112–144）、`.ad.light` 系列（146–164）、`.ad.dashboard` 系列（172–227）、響應式 media（228–266）。
- **改寫規則**：
  - 最外層改用 `:host` 控制定位/寬度，廣告根容器用 `.ad`。
  - 字體變數改系統字：`--tc: "PingFang TC", "Microsoft JhengHei", system-ui, sans-serif;` `--mono: ui-monospace, "SF Mono", Menlo, monospace;`（刪除 `--display` / Space Grotesk，模組沒用到）。
  - 砍掉所有 demo 外殼與未用變體：`.topbar`/`.backlink`/`.intro`/`footer`/`.wrap`/`body` 全頁背景/`::selection`/深色 `.ad`（非 light）/`.frame .bay-cell` 深色版/`.ad.light` 裡非 dashboard 用到的 row 變體。
  - 常駐淡底直接寫進 CSS（取代原 JS 的 set）：
    ```css
    .bay-cell { opacity: 0.16; }
    .bay-fill .bay-cell { fill-opacity: 0.25; opacity: 1; }
    ```

- [ ] **Step 3: 瀏覽器驗證（靜態）**

開啟頁面（先不管動畫）。Expected：
- 廣告白色模組正常渲染、外框與文字框就位。
- 模擬文章頁的 `h2{color:crimson}`、`*{box-sizing:content-box}` **沒有**影響到廣告內部（標題不是紅的、盒模型正常）→ Shadow DOM 隔離有效。
- 縮放視窗到 <760px：`.frame-mobile` 顯示、`.frame-desktop` 隱藏。

- [ ] **Step 4: Commit**

```bash
git add spig-ad-dashboard.html
git commit -m "feat(spig-ad): 移植廣告結構與精簡 CSS、改系統字"
```

---

### Task 3: buildBay 原生移植 + 常駐淡底

**Files:**
- Modify: `spig-ad-dashboard.html`（IIFE 內新增 `buildBay`）

**Interfaces:**
- Consumes: shadow 內 `.bay` group（含 `data-*`）。
- Produces: `buildBay(g) -> cells[]`，在每個 `.bay` 內建立 `.bay-cell` `<path>`；`bayGroups` 陣列 `[{ el, cells, fill, from }]`。

- [ ] **Step 1: 移植 buildBay（原生，照搬 neurosphere.html 365–390）**

```js
const SVGNS = "http://www.w3.org/2000/svg";
function buildBay(g) {
  const x0 = +g.dataset.x, y0 = +g.dataset.y, w = +g.dataset.w, h = +g.dataset.h, n = +g.dataset.n;
  const gap = +(g.dataset.gap || 5), cw = (w - gap * (n - 1)) / n, skew = h * +(g.dataset.skew || 0.95);
  const cells = [];
  for (let i = 0; i < n; i++) {
    const x = x0 + i * (cw + gap);
    const p = document.createElementNS(SVGNS, "path");
    const leansLeft = g.dataset.lean === "left";
    const d = leansLeft
      ? "M" + x.toFixed(1) + " " + y0 + " L" + (x + cw - skew).toFixed(1) + " " + y0 +
        " L" + (x + cw).toFixed(1) + " " + (y0 + h) + " L" + (x + skew).toFixed(1) + " " + (y0 + h) + " Z"
      : "M" + (x + skew).toFixed(1) + " " + y0 + " L" + (x + cw).toFixed(1) + " " + y0 +
        " L" + (x + cw - skew).toFixed(1) + " " + (y0 + h) + " L" + x.toFixed(1) + " " + (y0 + h) + " Z";
    p.setAttribute("d", d);
    p.setAttribute("class", "bay-cell");
    g.appendChild(p); cells.push(p);
  }
  return cells;
}
```

- [ ] **Step 2: 對「目前顯示中的 frame」建 bay groups**

```js
const mq = window.matchMedia("(max-width: 760px)");
function activeFrame() {
  return root.querySelector(mq.matches ? ".frame-mobile" : ".frame-desktop");
}
function collectBays(frame) {
  return [...frame.querySelectorAll(".bay")].map(bay => ({
    el: bay,
    cells: buildBay(bay),
    fill: bay.classList.contains("bay-fill"),
    from: bay.classList.contains("bay-bot") ? "end" : "start"
  }));
}
```

- [ ] **Step 3: 瀏覽器驗證**

Console: `root.querySelectorAll(".bay-cell").length` 應 > 0；視覺上兩排格子以淡色（0.16/0.25）顯示在外框上。

- [ ] **Step 4: Commit**

```bash
git add spig-ad-dashboard.html
git commit -m "feat(spig-ad): buildBay 原生移植 + 常駐淡底"
```

---

### Task 4: 進場動畫（IntersectionObserver + WAAPI 時序）

**Files:**
- Modify: `spig-ad-dashboard.html`

**Interfaces:**
- Consumes: `bayGroups`、`activeFrame()`、shadow 內 `.outline`/`.block`/`.bay` group。
- Produces: `playIntro(frame, bayGroups)` 執行一次進場；`startRunwayLoops(bayGroups)`（Task 5 實作，此處先宣告呼叫點）。

- [ ] **Step 1: 進場前隱藏狀態 + outline 自繪準備**

```js
function prepIntro(frame, bayGroups) {
  frame.querySelectorAll(".outline").forEach(o => {
    const L = o.getTotalLength();
    o.style.strokeDasharray = L;
    o.style.strokeDashoffset = L;
  });
  bayGroups.forEach(g => { g.el.style.opacity = "0"; });   // 父層 <g> 先隱形
  frame.querySelectorAll(".block").forEach(b => { b.style.opacity = "0"; });
}
```

- [ ] **Step 2: 進場時序（WAAPI）**

WAAPI 沒有 timeline，用各動畫的 `delay` 對齊到「框線畫完」這個基準點。框線 2200ms，跑道燈與色塊在 ~2250ms 同時淡入。

```js
const FRAME_DRAW = 2200, RUNWAY_IN = 2250;
function playIntro(frame, bayGroups) {
  // 1. 框線自繪
  frame.querySelectorAll(".outline").forEach(o => {
    o.animate([{ strokeDashoffset: o.style.strokeDashoffset }, { strokeDashoffset: 0 }],
      { duration: FRAME_DRAW, easing: "cubic-bezier(.45,0,.15,1)", fill: "forwards" });
  });
  // 2. 框線畫完：色塊淡入
  frame.querySelectorAll(".block").forEach(b => {
    b.animate([{ opacity: 0 }, { opacity: 0.9 }],
      { delay: RUNWAY_IN, duration: 500, easing: "ease-out", fill: "forwards" });
  });
  // 3. 框線畫完：跑道燈父層 <g> 淡入「出現」（與色塊同時），出現後啟動循環
  bayGroups.forEach(g => {
    g.el.animate([{ opacity: 0 }, { opacity: 1 }],
      { delay: RUNWAY_IN, duration: 500, easing: "ease-out", fill: "forwards" });
  });
  setTimeout(() => startRunwayLoops(bayGroups), RUNWAY_IN);
  // 4. 標題與 rows 解碼（Task 6）
  scheduleContentReveal();
}
```

- [ ] **Step 3: IntersectionObserver 觸發一次**

```js
const reduce = window.matchMedia("(prefers-reduced-motion: reduce)");
function boot() {
  const frame = activeFrame();
  const bayGroups = collectBays(frame);
  if (reduce.matches) { showStatic(frame, bayGroups); return; }   // Task 7
  prepIntro(frame, bayGroups);
  const io = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) { io.disconnect(); playIntro(frame, bayGroups); }
  }, { threshold: 0.2 });
  io.observe(host);
}
boot();
```

- [ ] **Step 4: 瀏覽器驗證**

重整、捲到廣告。Expected：框線先自繪（~2.2s），畫完後跑道燈與色塊**同時淡入出現**；框線畫之前跑道燈不可見。

- [ ] **Step 5: Commit**

```bash
git add spig-ad-dashboard.html
git commit -m "feat(spig-ad): 進場動畫 IntersectionObserver + WAAPI 時序"
```

---

### Task 5: 跑道燈無限循環（每格獨立 + stagger）

**Files:**
- Modify: `spig-ad-dashboard.html`

**Interfaces:**
- Consumes: `bayGroups`。
- Produces: `startRunwayLoops(bayGroups)`，每個 cell 各自一個 `iterations: Infinity` 的 WAAPI 動畫。

- [ ] **Step 1: 每格獨立無限脈衝（自帶 stagger 與停頓）**

每格各自一個無限動畫 → 沒有「整體 repeat reset」，不會整排消失。停頓用 keyframes 尾段維持低值模擬（duration 內 0~0.53 跑脈衝、0.53~1 維持低值 ≈ 原 repeatDelay 0.9s）。

```js
function startRunwayLoops(bayGroups) {
  const CYCLE = 1900;          // 1.0s 脈衝 + 0.9s 停頓
  bayGroups.forEach(group => {
    const prop = group.fill ? "fillOpacity" : "opacity";
    const lo = group.fill ? 0.25 : 0.16;
    const order = group.from === "end" ? [...group.cells].reverse() : group.cells;
    order.forEach((cell, i) => {
      const kf = group.fill
        ? [{ fillOpacity: lo, offset: 0 }, { fillOpacity: 1, offset: 0.26 }, { fillOpacity: lo, offset: 0.53 }, { fillOpacity: lo, offset: 1 }]
        : [{ opacity: lo, offset: 0 }, { opacity: 1, offset: 0.26 }, { opacity: lo, offset: 0.53 }, { opacity: lo, offset: 1 }];
      cell.animate(kf, { duration: CYCLE, delay: i * 160, iterations: Infinity, easing: "ease-in-out" });
    });
  });
}
```

注意：WAAPI keyframe 屬性名用 camelCase `fillOpacity`（對應 SVG presentation attribute `fill-opacity`）。

- [ ] **Step 2: 瀏覽器驗證**

Expected：淡底常駐（永遠至少 0.16/0.25，不消失）、亮波逐格掃過、循環不間斷、無「白→消失」閃動。父層淡入與子格循環互不干擾。

- [ ] **Step 3: Commit**

```bash
git add spig-ad-dashboard.html
git commit -m "feat(spig-ad): 跑道燈每格獨立無限循環、淡底不消失"
```

---

### Task 6: 文字解碼（rAF）+ 標題與 rows 進場

**Files:**
- Modify: `spig-ad-dashboard.html`

**Interfaces:**
- Consumes: shadow 內 `.ad__title .lead`、`.ad__row`、`.meta`、`.copy`。
- Produces: `scrambleIn(el)`（rAF 驅動，回傳 Promise 或接受 done callback）、`scheduleContentReveal()`。

- [ ] **Step 1: scrambleIn 改 rAF（移植 neurosphere.html 394–415 的解碼邏輯）**

```js
const KATA = "アイウエオカキクケコサシスセソタチツテトナニヌネノハヒフヘホマミムメモヤユヨラリルレロワヲ";
const LAT = "ABCDEFGHJKLMNPQRSTUVWXYZ0123456789#%&<>/*";
function rndGlyph(c) {
  if (/\s/.test(c)) return c;
  if (/[　-鿿＀-￯゠-ヿ]/.test(c)) return KATA[Math.random() * KATA.length | 0];
  return LAT[Math.random() * LAT.length | 0];
}
function scrambleIn(el) {
  const text = el.dataset.text || (el.dataset.text = el.textContent);
  const chars = [...text], n = chars.length;
  const dur = Math.min(1600, 50 * n + 350);
  el.style.visibility = "visible"; el.style.opacity = "1";
  const t0 = performance.now();
  function tick(now) {
    const v = Math.min(n, (now - t0) / dur * n);
    let out = "";
    for (let i = 0; i < n; i++) out += (i < v) ? chars[i] : rndGlyph(chars[i]);
    el.textContent = out;
    if (v < n) requestAnimationFrame(tick); else el.textContent = text;
  }
  requestAnimationFrame(tick);
}
```

- [ ] **Step 2: 進場前隱藏標題/內文，scheduleContentReveal 排程**

`prepIntro` 加上：`root.querySelectorAll(".ad__title .lead, .ad__row .copy").forEach(e => { e.style.opacity = "0"; });` 並把 `.ad__row` / `.meta` 設 `opacity:0`。

```js
const CONTENT_REVEAL = RUNWAY_IN + 350;
function scheduleContentReveal() {
  setTimeout(() => scrambleIn(root.querySelector(".ad__title .lead")), CONTENT_REVEAL);
  const rows = [...root.querySelectorAll(".ad__row")];
  const metas = root.querySelectorAll(".ad__row .meta");
  const copies = root.querySelectorAll(".ad__row .copy");
  rows.forEach((row, i) => {
    const at = CONTENT_REVEAL + i * 160;
    setTimeout(() => {
      row.style.opacity = "1";
      row.animate([{ opacity: 0 }, { opacity: 1 }], { duration: 220, fill: "forwards" });
      metas[i].style.opacity = "1";
      scrambleIn(copies[i]);
    }, at);
  });
}
```

- [ ] **Step 3: 瀏覽器驗證**

Expected：框線+跑道燈進場後，主標題 Matrix 解碼出現，接著三個 row 逐個（每 160ms）淡入並解碼內文。

- [ ] **Step 4: Commit**

```bash
git add spig-ad-dashboard.html
git commit -m "feat(spig-ad): 文字解碼 rAF + 標題與 rows 進場"
```

---

### Task 7: reduced-motion 靜態 + 最終健檢

**Files:**
- Modify: `spig-ad-dashboard.html`

**Interfaces:**
- Consumes: `frame`、`bayGroups`。
- Produces: `showStatic(frame, bayGroups)`。

- [ ] **Step 1: reduced-motion 靜態顯示（不跑任何動畫）**

```js
function showStatic(frame, bayGroups) {
  frame.querySelectorAll(".outline, .block").forEach(e => { e.style.opacity = ""; });
  bayGroups.forEach(g => { g.el.style.opacity = "1"; g.cells.forEach(c => { c.style.opacity = "0.6"; }); });
  root.querySelectorAll(".ad__title .lead, .ad__row, .ad__row .meta, .ad__row .copy")
    .forEach(e => { e.style.opacity = "1"; e.style.visibility = "visible"; });
}
```

- [ ] **Step 2: 驗收健檢（grep 靜態檢查）**

```bash
grep -niE "gsap|scrolltrigger|cdn|googleapis|gstatic" spig-ad-dashboard.html   # 預期：無任何命中
git status --short                                                              # 預期：neurosphere.html 不在清單
```
Expected：無外部 JS / 字體 / CDN 引用；`neurosphere.html` 未被改動。

- [ ] **Step 3: 瀏覽器最終驗證**

- DevTools Network：載入廣告**不產生任何外部請求**（除頁面本身）。
- Console：`Object.keys(window).filter(k => /gsap|spig|ScrollTrigger/i.test(k))` → `[]`。
- 系統開啟「減少動態」後重整：廣告以靜態完整顯示、無無限動畫。
- 模擬文章頁的 crimson 標題 / content-box 沒有影響廣告（隔離複驗）。

- [ ] **Step 4: Commit**

```bash
git add spig-ad-dashboard.html
git commit -m "feat(spig-ad): reduced-motion 靜態 + 最終健檢通過"
```

---

## Self-Review

**1. Spec coverage：**
- 零外部 JS / 字體 / 全域污染 → Task 2（系統字）、Task 7（grep + Network 驗收）、全程 IIFE。✓
- Shadow DOM 隔離 → Task 1 建立、Task 2/7 驗證（crimson + content-box 探針）。✓
- 響應式兩套 frame → Task 3 `activeFrame()`/matchMedia、Task 2 CSS media。✓
- 進場時序沿用現版 → Task 4。✓
- 跑道燈淡底不消失、分層不搶屬性 → Task 5（每格獨立）+ Task 4（父層淡入分離）。✓
- 文字解碼 → Task 6。✓
- reduced-motion → Task 7。✓
- 不改 neurosphere.html → Task 7 git status 驗收。✓

**2. Placeholder scan：** 無 TBD/TODO；每個 code 步驟給出完整可執行 code。✓

**3. Type consistency：** `buildBay`、`collectBays`、`activeFrame`、`bayGroups{el,cells,fill,from}`、`playIntro`、`startRunwayLoops`、`scrambleIn`、`scheduleContentReveal`、`showStatic`、常數 `RUNWAY_IN`/`CONTENT_REVEAL` 跨 task 命名一致。✓

## 已知取捨（非 placeholder）

- 進場時序用 `setTimeout`/WAAPI `delay` 對齊，非 GSAP timeline 的相對標籤；基準值（FRAME_DRAW 2200、RUNWAY_IN 2250、CONTENT_REVEAL +350）對應現版時間點，實作時若視覺有偏差可微調。
- matchMedia 切換 desktop/mobile 後，bay 已建在另一套 frame；本版在 `boot()` 時依當下斷點建一次（投放即時注入足夠）。若需在 resize 跨斷點重建，列為日後增強，本次 YAGNI 不做。

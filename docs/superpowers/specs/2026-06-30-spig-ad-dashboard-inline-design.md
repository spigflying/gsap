# Spec：spig-ad-dashboard（inline 注入版瘦身廣告）

日期：2026-06-30

## 背景與目標

`neurosphere.html` 是一個 demo/showcase 頁，但實際要投放到第三方文章媒體的，只有中間那個白色 dashboard 廣告模組（`.ad.dashboard.concept-a`）。

身為數位廣告商，要把這個模組做成可貼進文章頁的瘦身版，**程式越瘦越好**。本 spec 定義這個瘦身投放版的設計。**不修改 `neurosphere.html`**，全部做在一個新檔。

## 投放限制（已與使用者確認的決策）

| 項目 | 決策 |
|---|---|
| 投放方式 | **inline 注入文章 DOM**（非 iframe）→ CSS 絕不能污染宿主、全域不能衝突 |
| 尺寸 | **隨欄寬響應式** → 保留 desktop + mobile 兩套 SVG frame 與 matchMedia 切換 |
| 動畫 library | **拿掉 GSAP，全原生重寫**（IntersectionObserver + WAAPI + rAF） |
| CSS 隔離 | **Shadow DOM**（雙向完全隔離） |
| 字體 | **全系統字**（零外部字體請求） |
| 產出形式 | **全部 self-contained 在一個新 HTML 頁**；之後有需要再抽成 `.js` |

## 產出物

新檔 `spig-ad-dashboard.html`：一個獨立可預覽的頁面，模擬「文章媒體頁」——含一段假文章內文 + 一個注入點 `<div>`，方便預覽廣告在文章脈絡中的樣子並驗證 Shadow DOM 隔離。

頁內核心是一段**自執行 IIFE**（不留任何全域變數），未來可原封不動抽成 `spig-ad-embed.js` 交付投放。

## 架構

```
IIFE（self-executing，無全域洩漏）
 └─ 建 host <div> → attachShadow({ mode: 'open' })
     └─ shadow root 注入：
         ├─ <style>（全部 CSS，因 Shadow 隔離可自由用裸標籤 / :host）
         └─ 廣告 HTML：
             ├─ <svg class="frame frame-desktop">（viewBox 1000×540）
             ├─ <svg class="frame frame-mobile">（viewBox 420×900）
             └─ .ad__inner（標題 + 三個 row）
```

Shadow DOM 保證：宿主文章頁的 `* { box-sizing }`、`img { max-width }` 等弄不歪廣告；廣告樣式也不外洩。

## GSAP → 原生 對應

| 現在（GSAP） | 改成（原生） |
|---|---|
| `ScrollTrigger` once 進場觸發 | `IntersectionObserver`（threshold 觸發一次後 disconnect） |
| `gsap.timeline` 時序 | WAAPI `element.animate()` + `delay` 串接時間軸 |
| outline `strokeDashoffset` 自繪 | WAAPI 動 `stroke-dashoffset`（可動 CSS 屬性） |
| 跑道燈無限脈衝 + stagger | 每格各自 `element.animate(..., { iterations: Infinity, delay: i*160 })` |
| `scrambleIn`（gsap 驅動 tick） | `requestAnimationFrame` 驅動文字解碼 |
| `gsap.matchMedia` | 原生 `window.matchMedia` |
| `prefers-reduced-motion` | 原生 media query，命中則直接顯示靜態、不跑動畫 |
| `buildBay`（建 SVG path） | 已是原生 DOM，照搬 |

## 動畫行為規格（完全沿用目前 neurosphere.html 已調好的版本）

進場時序（IntersectionObserver 觸發後）：
1. 框線 outline 自繪（約 2.2s）。
2. 框線畫完（+~0.05s）：跑道燈**父層 `<g>`** 與色塊 blocks **同時淡入「出現」**。
3. 主標題 Matrix 解碼進場。
4. 三個 row 逐個進場 + 內文解碼。

跑道燈循環：
- 常駐淡底用 CSS 焊死（中空格 `opacity:0.16`、實心格 `fill-opacity:0.25`），動畫之外也保持微亮。
- 子格無限脈衝（`[淡, 亮, 淡]`），逐格 stagger 製造跑道流動感。
- **分層原則**：父層 `<g>` 管「進場淡入出現」、子格管「循環脈衝」，作用在不同元素、互不搶屬性 → 不會有「白→消失」閃動。
- 原生改成**每格各自獨立的無限動畫**，天生沒有「整體 repeat reset」→ 不會整排消失重來，連殘留閃動都沒有。
- 每輪之間的停頓用 keyframes 尾段維持低值模擬（WAAPI 無原生 repeatDelay）。

## 瘦身結果（驗收標準）

- [ ] 零外部 JS（GSAP 2 支 CDN 全砍）
- [ ] 零外部字體請求（全系統字）
- [ ] 零全域變數污染（IIFE）
- [ ] CSS 不洩漏到宿主、宿主不影響廣告（Shadow DOM 驗證）
- [ ] demo 外殼全砍（hero / footer / topbar / backlink / 深色變體 CSS / 未用到的 light 變體）
- [ ] 進場與循環的視覺，與目前 neurosphere.html 的 dashboard 模組一致
- [ ] 響應式：desktop / mobile 兩套 frame 正確切換
- [ ] `prefers-reduced-motion` 命中時顯示靜態、不跑無限動畫
- [ ] `neurosphere.html` 未被修改

## 非目標（YAGNI）

- 不做固定 IAB 尺寸版本（本次走響應式）
- 不抽成獨立 `.js` 檔（先全放新頁，之後需要再抽）
- 不保留任何 demo 外殼或深色主題變體
- 不動 `neurosphere.html`

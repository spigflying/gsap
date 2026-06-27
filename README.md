# GSAP 動態示範 — Motion Specimen

用 [GSAP 3.15](https://gsap.com/) 打造的三頁本機網頁動畫示範，純 HTML + CDN，**雙擊即開、零安裝**。所有 GSAP plugin（含以前付費的 SplitText、MorphSVG、DrawSVG、InertiaPlugin）現已全部免費，包含商用。

## 線上瀏覽（GitHub Pages）

| 頁面 | 連結 | 內容 |
|------|------|------|
| **標本表 I** | [index.html](https://spigflying.github.io/gsap/) | 核心能力：Tween / Ease、Timeline、ScrollTrigger、SplitText、Draggable、Flip |
| **標本表 II** | [advanced.html](https://spigflying.github.io/gsap/advanced.html) | 進階方向：MotionPath、DrawSVG、MorphSVG、ScrollSmoother |
| **廣告模組** | [ad-modules.html](https://spigflying.github.io/gsap/ad-modules.html) | 同一個數位廣告版型 × 三種捲動進場效果（3D 翻入、景深推進、遮罩橫掃） |

## 用到的 GSAP plugins

- **ScrollTrigger** — 捲動觸發、scrub、pin 釘住、batch 成批浮現、橫向滾動（containerAnimation）
- **SplitText** — 文字拆字 / 拆詞 / 拆行，逐字浮現與遮罩揭露
- **Draggable + InertiaPlugin** — 可拖曳、放手帶慣性、撞邊回彈
- **Flip** — 兩個佈局狀態之間的平滑過渡（篩選、grid↔list）
- **MotionPath** — 元素沿 SVG 路徑移動、autoRotate
- **DrawSVG** — 線條自我描繪
- **MorphSVG** — SVG 形狀變形
- **ScrollSmoother** — 平滑慣性捲動 + data-speed 視差

## 設計

示波器／繪圖儀視覺語言：墨藍底 + 青綠繪圖線 + 訊號橘雙重點色；Space Grotesk（標題）/ IBM Plex Mono（數據）/ Noto Sans TC（中文）。整體動畫包在 `gsap.matchMedia()` 裡，尊重 `prefers-reduced-motion`。

## 本機執行

直接用瀏覽器開 `index.html` 即可。若要最穩定的捲動體驗（尤其 ScrollSmoother），用本機伺服器：

```bash
python3 -m http.server 8000
# 然後開 http://localhost:8000/
```

## 授權

示範程式碼自由使用。GSAP 由 GreenSock 開發，採其自身授權（現為免費）。

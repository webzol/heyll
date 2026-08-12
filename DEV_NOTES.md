# DEV_NOTES · Heyll 个人主页

给后续接手的人（或大模型）看的决策记录与踩坑备忘。功能清单和更新日志在 `README.md`，这里只写「为什么这么做」。

---

## 项目基本情况

- 纯静态站点，原生 HTML + CSS + JS，**没有构建步骤**：改完文件直接刷新浏览器即可，不要引入打包器。
- 由 ZYYO 模板 rebrand 而来，仓库内已清除原作者信息，改动时注意不要从旧模板里粘回来。
- 双 remote：`origin` → `webzol/heyll`，`homepage` → `webzol/homepage`。日常推 `origin`；`homepage` 按需手动同步。
- 主题变量集中在 `static/css/root.css`（1 套夜间 + 5 套白天），业务样式在 `static/css/style.css`。改配色优先动变量，别在组件里写死颜色。

---

## 2026.08 · 首屏加载动画重做

### 做了什么

原模板的加载层是 `#zyyo-loading` + `#zyyo-loading-center`：一个 150px 的 `#472eff` 蓝色圆球做 `scale(0) → scale(1)` 无限缩放。视觉上过重，而且和站点任何一套主题的配色都不搭。

换成 `#page-loader`：全屏遮罩 + 居中一个 26px 的细圆环（2px 描边、缺口旋转、`opacity: .35`）。

### 关键决策

1. **样式内联在 `index.html` 的 `<style>` 里，没放 `style.css`。**
   加载层必须在第一帧就成形。如果样式在外部 CSS，就得等 `style.css` 到达才能渲染成正确形态，中间会闪一下裸 DOM——这正是加载动画要避免的问题。所以关键样式内联，`style.css` 里只留一行注释说明去处。

2. **颜色取主题变量，不写死。**
   遮罩底色 `var(--main_bg_color, #ffffff)`，圆环描边 `var(--main_text_color, #000000)`。两者出自同一份 `root.css`，要么都拿到变量、要么都走 fallback，不会出现「白环叠白底」的失配。
   注意 `--main_bg_color` 在部分主题里是 `url(../img/background.webp)`（相对 `root.css` 解析），所以遮罩上保留了 `background-size: cover` / `background-position: center`，否则背景图主题下会平铺。

3. **淡出后从 DOM 移除，不是只置 `opacity: 0`。**
   原实现只把 opacity 改成 0，节点一直留在 body 上（`z-index: 999999`）。虽然有 `pointer-events: none` 兜住点击穿透，但留一个全屏节点在合成树里没必要。现在 `is-hidden` 触发 .35s 过渡，400ms 后 `removeChild`。

4. **3 秒兜底定时器。**
   `window.load` 依赖所有子资源（含背景大图）完成。万一某个资源卡住或 404 挂起，页面会被一直遮着。加了 `setTimeout(hideLoader, 3000)` 强制收场，`hidden` 标志位保证 hideLoader 幂等。

5. **`document.readyState === 'complete'` 分支。**
   `script.js` 从缓存命中时可能在 load 之后才执行，此时监听 `load` 永远不会触发，加载层就永久留屏。所以先判 readyState，已完成则立即隐藏。

6. **`prefers-reduced-motion` 下停掉旋转**，只留一个静态淡环。

### 坑 / 注意

- `#page-loader` 的 DOM 节点是**硬编码在 `index.html` 里**的（`<div id="page-loader">` 紧跟 `<body>`），不像旧的 FPS 计数器那样由 JS `createElement` 动态插入。改结构时别漏掉它。
- JS 逻辑写成 IIFE 且 `if (!loader) return;` 开头，删掉 HTML 节点不会报错。
- `script.js` 的引用带版本查询串（`?v=20260812`），改完 JS 记得同步 bump，否则浏览器吃旧缓存。

---

## 2026.08 · 首屏图片压缩

### 结果

| 文件 | 前 | 后 | 处理 |
| --- | --- | --- | --- |
| `logo.png` → `logo.webp` | 1080×1080 / 1176 KB | 600×600 / 46 KB | WebP q82 |
| `background.jpg` → `background.webp` | 800×480 / 266 KB | 800×480 / 56 KB | WebP q85，尺寸不变 |
| `logokuang.png` | 800×800 / 162 KB | 600×600 / 23 KB | 128 色量化 PNG（原地替换） |

首屏图片总量 1604 KB → 125 KB，约 −92%。

### 关键决策

1. **尺寸依据实际渲染宽度，不是原图尺寸。**
   `.zyyo-left` 宽 230px、左右 padding 各 15px → 内容区 200px，`.logo` 取 `width: 90%` = **实际只渲染 180 CSS px**；移动端 `.index-logo` 是 `max-width: 200px`。所以 600px 已能完整覆盖 DPR 3，原来的 1080px 纯属浪费。**以后换图先算这个数，别直接丢原图进来。**

2. **`logo` 的 alpha 通道是纯浪费。**
   原图是 RGBA，但实测 alpha extrema 为 `(255, 255)`——全不透明。照片类内容 + 无效 alpha 存成 PNG 是 1.2MB 的根本原因。而且它外层有 `border-radius: 50%` 裁圆、上面还盖着 `logokuang` 贴纸，四角根本看不见，转有损格式零风险。

3. **`logokuang` 反而用量化 PNG，不用 WebP。**
   它是硬边缘卡通贴纸 + 大片透明区。实测：有损 WebP q88 = 27.3 KB（描边有振铃风险）、无损 WebP = 70.5 KB、**128 色量化 PNG = 23.2 KB 且零振铃无色带**。这类平涂插画量化 PNG 通常优于有损格式——别无脑上 WebP。
   附带好处：保持 `.png` 后缀，无需改任何引用。

4. **改扩展名要同步改引用。**
   `logo.webp` → `index.html` 两处内联 `background-image`（`.logo` 与 `.index-logo`）。
   `background.webp` → `static/css/root.css` **三处** `--main_bg_color`（主题 1 / 2 / 5 都用了背景图）。改完务必 `grep -rn "background\.jpg\|logo\.png"` 确认零残留。

### 无需处理的

- **`qq.jpg`（459 KB）、`wxzsm.jpg`（57 KB）**：只出现在 `onclick="pop('...')"` 的字符串里，且元素带 `style="display:none"`。浏览器**不会预加载**，对首屏零成本，仅占仓库体积。想瘦仓库可删，但删了对应入口就彻底没法恢复。
- **`i5.png` / `i6.png`（共 24 KB）**：全仓库零引用，是备用的 project 图标，同样不影响加载。
- **`tz.jpg`**：`script.js:172` 引用了它但那行是注释（`//pop(...)`），文件本身不存在也不会 404。
- **`favicon.ico`（1023×1023 / 56 KB）**：尺寸离谱但体积可接受，且被当头像复用，暂不动。

### 复现方法

本机有 Python 3.14 + Pillow，无需装额外工具：

```python
from PIL import Image
# 照片类 → WebP
Image.open('src.png').convert('RGB').resize((600,600), Image.LANCZOS) \
     .save('out.webp','WEBP',quality=82,method=6)
# 平涂插画带透明 → 量化 PNG
Image.open('src.png').resize((600,600), Image.LANCZOS) \
     .quantize(colors=128, method=Image.FASTOCTREE).save('out.png','PNG',optimize=True)
```

原图不必另存备份——旧版本都在 git 历史里（压缩前提交为 `5563239`）。

---

## 2026.08 · 移除 FPS 计数器

原模板在左上角用 `requestAnimationFrame` 计帧并 `createElement` 插一个 `div#fps`（样式全内联在 JS 里）。属于调试残留，对访客无意义，整块删除。

`static/js/script.js` 里保留了一行 `// FPS 计数器已移除（原左上角 FPS 显示）` 作为说明，`README.md` 更新日志里也留了一条记录——这两处是刻意留的文字痕迹，不影响运行。CSS 和 HTML 里本来就没有对应节点，无需清理。

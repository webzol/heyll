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
- **两个 CSS 也已加版本串**（`style.css?v=` / `root.css?v=`）。改 CSS 必须同步 bump——尤其 `root.css` 里写着背景图路径，缓存到旧版会去请求已删除的图片导致 404。

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

## 2026.08 · 头像呼吸光圈效果

### 做了什么

为 `.logo`(左侧栏头像) 和 `.index-logo`(移动端头像) 添加呼吸光圈动画，使用 `::before` 伪元素实现。

### 关键决策

1. **复用网站原有渐变配色 `var(--gradient)`**
   光圈颜色直接取 CSS 变量 `--gradient`(紫红蓝三色渐变)，自动跟随主题切换，无需为每个主题单独适配。夜间模式下渐变会变成 `linear-gradient(120deg, rgb(133, 62, 255), #f76cc6 30%, rgb(255, 255, 255) 60%)`，光圈同步变化。

2. **参数选择**
   - 周期 **4 秒**：参考 UX 指南避免过快(< 2s 会分散注意力)，又不至于慢到感知不明显。
   - 缩放 **1.0 → 1.15**：扩张幅度适中，太大(1.3+)会溢出容器、太小(1.05)不明显。
   - 透明度 **0.6 → 0.85**：基础透明度 0.6 确保不喧宾夺主，呼吸峰值 0.85 提供足够存在感。
   - 模糊 **12px**：柔化光圈边缘，避免硬边圈感觉廉价。
   - 定位 **inset: -8%**：光圈比头像略大 8%，视觉上像「从头像向外发光」而非「头像套了个圈」。

3. **无障碍支持 `prefers-reduced-motion`**
   用户开启系统动效减弱时，光圈动画停止、透明度降至 0.4 变成静态装饰层。这是 UX 指南要求的「Severity: High」项。

4. **性能考量**
   - 只用 `transform: scale()` 和 `opacity`，这两个属性在合成线程完成，不触发重排/重绘。
   - `filter: blur(12px)` 会进光栅化，但只有一层伪元素、且尺寸固定(头像 180-200px)，移动端也能稳定 60fps。
   - `animation: avatarBreathing 4s ease-in-out infinite` 用 `ease-in-out` 缓动函数，在 0% / 50% / 100% 关键帧处速度自然降至零，符合「呼吸」节奏。

### 实现细节

```css
.logo::before,
.index-logo::before {
    content: '';
    position: absolute;
    inset: -8%;                    /* 比头像大 8% */
    border-radius: 50%;
    background: var(--gradient);   /* 复用站点渐变 */
    opacity: 0.6;
    filter: blur(12px);            /* 高斯模糊柔化边缘 */
    z-index: -1;                   /* 置于头像后方 */
    animation: avatarBreathing 4s ease-in-out infinite;
}

@keyframes avatarBreathing {
    0%, 100% { transform: scale(1);    opacity: 0.6;  }
    50%      { transform: scale(1.15); opacity: 0.85; }
}

@media (prefers-reduced-motion: reduce) {
    .logo::before,
    .index-logo::before {
        animation: none;
        opacity: 0.4;
    }
}
```

### 坑 / 注意

- **不要用 `box-shadow` 实现**：多层 `box-shadow` 模拟光晕需要 5-8 层渐变阴影，性能远劣于单层 `filter: blur()`。
- **`z-index: -1` 必须加**：否则光圈会盖住头像本体和外层的 `logokuang.png` 装饰框。
- **`inset` 简写等价于 `top/right/bottom/left: -8%`**，负值让伪元素溢出父容器边界。
- **移动端 `.index-logo` 在 `@media (min-width: 800px)` 下 `display: none`**，但样式依然生效无副作用。

---

## 2026.08 · 移除 FPS 计数器

原模板在左上角用 `requestAnimationFrame` 计帧并 `createElement` 插一个 `div#fps`（样式全内联在 JS 里）。属于调试残留，对访客无意义，整块删除。

`static/js/script.js` 里保留了一行 `// FPS 计数器已移除（原左上角 FPS 显示）` 作为说明，`README.md` 更新日志里也留了一条记录——这两处是刻意留的文字痕迹，不影响运行。CSS 和 HTML 里本来就没有对应节点，无需清理。
## 2026.08 · 背景系统重构(双层架构)

### 做了什么

将单一 `--main_bg_color` 拆分为 `--page-bg`(底色) + `--page-bg-image`(柔光渐变层)，参考 OneDong 主题的设计模式。

### 关键决策

1. **双层架构理念**
   ```css
   body {
     background-color: var(--page-bg);        /* 底色 */
     background-image: var(--page-bg-image);  /* 柔光层 */
     background-repeat: no-repeat;            /* radial 不平铺 */
     background-attachment: fixed;            /* 钉视口 */
   }
   ```
   - `--page-bg` 是实色底或图片(纯色 `#F9FAFF` / 渐变 `linear-gradient` / 图片 `url()`)
   - `--page-bg-image` 是透明渐变层(通常 `radial-gradient` 柔光圆 / `linear-gradient` 氛围层)
   - 两者叠加产生细腻的视觉层次,避免单一纯色的单调感

2. **主题4 柔光渐变参数**
   ```css
   --page-bg: #F9FAFF;  /* 底色:极浅紫蓝(比纯白 #fff 多一丝冷调) */
   --page-bg-image: 
     radial-gradient(circle at 20% 30%, rgba(255, 223, 186, 0.15) 0%, transparent 45%),
     radial-gradient(circle at 80% 70%, rgba(173, 216, 230, 0.12) 0%, transparent 50%);
   ```
   - **暖黄光圈**(左上 20%,30%): `rgba(255, 223, 186, 0.15)` = #FFDFBA @ 15% 透明度,45% 衰减至透明
   - **浅蓝光圈**(右下 80%,70%): `rgba(173, 216, 230, 0.12)` = #ADD8E6 @ 12% 透明度,50% 衰减
   - 两圆错位叠加,产生温暖而不刺眼的渐变氛围
   - 透明度 12-15% 确保柔和,不抢卡片/文字的视觉主导权

3. **夜间模式氛围层**
   ```css
   html[data-theme="Dark"] {
     --page-bg: #0f0f11;  /* 底色:接近全黑(Lightness 6%) */
     --page-bg-image: linear-gradient(135deg, 
       rgba(189, 52, 254, 0.08) 0%,    /* 紫 #bd34fe */
       rgba(224, 50, 27, 0.06) 50%,    /* 红 #e0321b */
       rgba(65, 209, 255, 0.08) 100%   /* 蓝 #41d1ff */
     );
   }
   ```
   - 复用网站 `--gradient` 的三色(紫红蓝),但降至 6-8% 极低透明度
   - `135deg` 对角线走向,暗示方向感而不显刺眼
   - 避免夜间模式纯黑死板,增加微妙的色彩呼吸感

4. **`background-attachment: fixed` 的取舍**
   - 优点:长页面滚动时背景钉在视口,柔光圆位置不变,避免渐变被拉伸变形
   - 代价:iOS Safari 对 `fixed` + `radial-gradient` 渲染有已知 bug(某些版本会闪烁或不生效)
   - 决策:**先保留 `fixed`**,待 TD 移动端实测;若 iOS 异常,则媒体查询 `@media (max-width: 800px)` 改 `scroll`

5. **变量命名语义化**
   - `--page-bg` 而非 `--main-bg-color`:因为它不只是颜色,可能是 `url()` 图片
   - `--page-bg-image`:明确其为 `background-image` 属性值,可以是 `none` / `radial-gradient` / `linear-gradient`
   - 旧的 `--main_bg_color` 在 `root.css` 中全部替换,`style.css` 的 `body` 规则同步改写

### 实现细节

**主题变量拆分前后对比**
```css
/* 旧(v6.2.1 前) */
--main_bg_color: #ffffff;  /* 单一变量承载一切 */
body { background: var(--main_bg_color); }

/* 新(v6.3.0) */
--page-bg: #F9FAFF;
--page-bg-image: radial-gradient(...);
body {
  background-color: var(--page-bg);
  background-image: var(--page-bg-image);
}
```

**5 套主题适配情况**
| 主题 | `--page-bg` | `--page-bg-image` | 说明 |
|------|-------------|-------------------|------|
| 主题1 | `url(background.webp)` | `none` | 图片模糊背景,无需渐变层 |
| 主题2 | `url(background.webp)` | `none` | 同上 |
| 主题3 | `linear-gradient(50deg, #a2d1ff, #ffffff)` | `none` | 本身是渐变底,无需叠加 |
| 主题4 | `#F9FAFF` | 暖黄+浅蓝双圆 | **新增柔光层** |
| 主题5 | `url(background.webp)` | `none` | 图片模糊背景 |
| Dark | `#0f0f11` | 紫红蓝线性渐变 8%/6%/8% | **新增氛围层** |

**首屏加载层同步适配**
```html
<!-- index.html 内联样式 -->
<style>
#page-loader {
  background-color: var(--page-bg, #F9FAFF);
  background-image: var(--page-bg-image, none);
  /* fallback 值保证变量未加载时不白屏 */
}
</style>
```
- 加载层必须在第一帧就匹配主题背景,所以同步使用双变量
- fallback 值取主题4(默认主题),确保冷启动时视觉连贯

### 坑 / 注意

- **iOS Safari `fixed` + `radial-gradient` 风险**:已知 iOS 14-15 某些版本渲染异常,实测后若闪烁需降级为 `scroll`
- **柔光圆透明度上限 20%**:超过 20% 会在卡片下形成色斑,破坏玻璃态/卡片的通透感
- **`background-repeat: no-repeat` 必须加**:否则 `radial-gradient` 会平铺成网格,视觉灾难
- **主题切换时 `transition` 只管 `background-color`**:`background-image` 的渐变无法平滑过渡(CSS 限制),但 0.25s 的底色过渡已足够柔和
- **滚动条轨道 `::webkit-scrollbar-track` 也要改**:从 `var(--main_bg_color)` 改为 `var(--page-bg)`,否则滚动条底色失配

### 性能

- 纯 CSS `radial-gradient`,零 JS 开销,GPU 合成友好
- 两层 `radial-gradient` 叠加对移动端无压力(现代浏览器优化良好)
- `background-attachment: fixed` 在桌面端零成本,移动端有轻微重绘(可接受)

### 后续扩展方向

- 可为每个主题单独调参(柔光圆位置/颜色/透明度/衰减半径)
- 可增加 `background-blend-mode: soft-light` 进一步柔化(实测后决定是否需要)
- 可增加 SVG noise 纹理叠加(参考 ui-ux-pro-max 的 Vintage Analog 风格),但需控制 `opacity` < 3% 避免颗粒感过重

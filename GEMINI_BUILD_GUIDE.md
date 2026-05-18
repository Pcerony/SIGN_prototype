# 视线追踪植物解说标识展示器 — Gemini 编程指导文档

## 项目概述

制作一个**单文件** HTML 应用（`index.html`），放置在 `C:\Users\admin\OneDrive\Desktop\SIGN_prototype\` 目录下。
用户用浏览器直接打开，无需服务器。

植物图片已准备好：`./plant.png`（同目录下）

---

## 技术选型

| 项目 | 选择 |
|------|------|
| 语言 | 单文件 HTML + 内嵌 CSS + 内嵌 JS |
| 眼动追踪 | WebGazer.js（CDN引入） |
| 字体 | Google Fonts：Noto Serif JP / Noto Sans JP / Inter |
| 图表/动画 | 纯 CSS Transition + requestAnimationFrame |
| 框架依赖 | 无，零依赖 |

**WebGazer CDN：**
```html
<script src="https://webgazer.cs.brown.edu/webgazer.js"></script>
```

---

## 完整文件结构

```
SIGN_prototype/
├── index.html          ← 唯一需要创建的文件（全部逻辑内嵌）
├── plant.png           ← 植物照片（已存在，直接用 ./plant.png 引用）
└── 画板 2.png           ← 参考设计图（仅参考，不引用）
```

---

## 页面整体布局（三层叠加）

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1（最底层）：深色渐变背景（全屏）                  │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Layer 2：白色/米色纸质感标识卡片（居中）           │  │
│  │  （标识内容见下方「标识布局」章节）                  │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  Layer 3（最顶层，fixed定位）：                          │
│  ├── 校准遮罩 #calibration-overlay（初始显示）           │
│  ├── 注视点圆圈 #gaze-dot                               │
│  ├── 摄像头预览 #camera-preview（右上角，可最小化）       │
│  └── 管理员面板 #admin-panel（右下角，点击展开）          │
└─────────────────────────────────────────────────────────┘
```

---

## 标识卡片布局（参照 画板 2.png）

卡片最大宽度 **1100px**，背景色 `#FAF8F4`，有圆角和纸质感阴影。

```
┌──────────────────────────────────────────────────────────┐
│ [zone-title]  副标题：空を舞う折り鶴？         [折纸鹤SVG] │
│               主标题：オリヅルラン（折鶴蘭）               │
├────────────────────────┬─────────────────────────────────┤
│ [zone-bubble]          │ [zone-taxonomy]                  │
│ 粉色圆角气泡文字        │ 学名 + 科属（分隔线下）           │
│                        ├─────────────────────────────────┤
│ [zone-image]           │ [zone-body]                      │
│ plant.png 照片          │ 三段正文描述                      │
│ 花言葉标注              │                                  │
│                        ├─────────────────────────────────┤
│                        │ [zone-callout]                   │
│                        │ 粉色背景强调框                    │
└────────────────────────┴─────────────────────────────────┘
```

### 标识内各区域文字内容（日语原文）

**zone-title**
- 副标题（小字，粉色）：`空を舞う折り鶴？`
- 主标题（大字，黑色）：`オリヅルラン（折鶴蘭）`
- 展开详情（隐藏）：`英語名：Spider Plant（スパイダープラント）。長い茎の先に垂れ下がる子株が蜘蛛の巣のように見えることからこの名前がつきました。`

**zone-taxonomy**
- 内容：`Chlorophytum comosum　　キジカクシ科　オリヅルラン属`
- 展开详情（隐藏）：`原産地：南アフリカ。世界中に広く普及し、最も人気のある観葉植物のひとつです。`

**zone-bubble**（粉色圆角气泡，拟人语气）
- 内容：`僕たちは、部屋の空気をきれいにする「天然の空気清浄機」と呼ばれています。でも、僕の本当の魅力は、その不思議な形にあることを知っていますか？`
- 展开详情（隐藏）：`研究によると、24時間で空気中の有害物質を最大90%除去できるといわれています！`

**zone-image**
- 显示：`./plant.png` 照片
- 图注：`花言葉：集う幸福（家族や友達と集まる喜び）`
- 展开详情（隐藏，显示在图下方）：`育て方のポイント：水やりは土が乾いてから。直射日光を避け、明るい日陰が最適です。`

**zone-body**（三段正文）
- 第1段：`細長い葉の間から、ぴょんと元気に飛び出した茎を見てください。`
- 第2段：`その先に揺れている小さな赤ちゃんたちは、まるでお祝いの「折り鶴」を吊り下げているように見えませんか？`
- 第3段（含下划线）：`もし近くに僕の仲間がいたら、葉っぱに優しく触れてみてください。`
- 展开详情（隐藏）：`葉は非常に柔らかく、触れると意外な瑞々しさを感じられます。緑と白のコントラストが美しいこの模様は、一枚一枚が少しずつ異なります。`

**zone-callout**（粉色背景圆角提示框）
- 内容：`シュッとした見た目よりも、ずっと柔らかくて瑞々しい感触がするはずです!!!`
- 展开详情（隐藏）：`ぜひ実際に手で触れて、その感触を体験してみてください！きっと驚くはずです。`

**装饰元素**
- 右上角：3只折纸鹤 SVG（粉色、橙金色、浅橙色），大小不一
- 右下角：1只轮廓折纸鹤 SVG（粉色线条，较淡）

---

## 六个追踪区域 (Zone) 规范

每个 Zone 用 `id` 属性标识，JavaScript 通过 `getBoundingClientRect()` 获取其屏幕位置。

| Zone ID | 对应内容 | 
|---------|---------|
| `zone-title` | 标题区（副标题 + 主标题） |
| `zone-taxonomy` | 分类区（学名 + 科属） |
| `zone-bubble` | 气泡区（拟人气泡文字） |
| `zone-image` | 配图区（照片 + 图注） |
| `zone-body` | 正文区（三段描述） |
| `zone-callout` | 提示区（底部强调框） |

---

## 动态权重系统

### 权重变量

每个 Zone 维护一个 `weight` 值（`0.0` ~ `100.0`）。

```javascript
const zoneState = {
  'zone-title':    { weight: 0, gazing: false },
  'zone-taxonomy': { weight: 0, gazing: false },
  'zone-bubble':   { weight: 0, gazing: false },
  'zone-image':    { weight: 0, gazing: false },
  'zone-body':     { weight: 0, gazing: false },
  'zone-callout':  { weight: 0, gazing: false },
};
```

### 权重更新逻辑（每帧 rAF 循环中执行）

```javascript
// settings 来自管理员面板
const SENSITIVITY = settings.sensitivity; // 默认 15（weight/秒，注视时上升）
const DECAY      = settings.decay;       // 默认 6（weight/秒，非注视时下降）

function updateWeights(dt, currentZoneId) {
  for (const [id, state] of Object.entries(zoneState)) {
    if (id === currentZoneId) {
      state.weight = Math.min(100, state.weight + SENSITIVITY * dt);
    } else {
      state.weight = Math.max(0, state.weight - DECAY * dt);
    }
  }
}
```

### 权重映射到视觉效果

对每个 Zone 的 `weight` 值，应用以下视觉变化：

```
weight 0~10   → 无变化（基础状态）
weight 10~30  → 背景微黄高亮（rgba 粉色，透明度 0~0.06）
weight 30~55  → 字号增大（基础字号 + 最多 +5px），文字颜色变深
weight 55~70  → 继续增大字号（最多 +9px），出现下划线装饰（粉色）
weight 70~85  → 展开隐藏的详情文字（.expanded-detail 淡入，max-height 展开）
weight 85~100 → 最大权重：字号最大（+12px），文字颜色最深，
                轻微文字阴影，背景 glow 效果
```

**关键实现细节：**
- 每个 `.zone-text` 元素用 `data-base-font-size="16"` 记录其基础字号（px）
- 用 `element.style.fontSize = (base + boost) + 'px'` 直接设置
- 用 CSS `transition: font-size 0.5s ease, color 0.5s ease` 平滑过渡
- `.expanded-detail` 元素默认 `max-height: 0; opacity: 0; overflow: hidden`，展开时设 `max-height: 300px; opacity: 1`（用 CSS transition）
- 下划线：`text-decoration: underline; text-decoration-color: #E91E8C;`

### 颜色插值

```javascript
// weight 0~100 时，文字颜色从 #888888 → #1a1a1a（加深）
function weightToColor(weight) {
  const t = weight / 100;
  const light = 136; // #888
  const dark  = 26;  // #1a
  const v = Math.round(light + (dark - light) * t);
  // 对非粉色强调文字用此颜色
  return `rgb(${v}, ${v}, ${v})`;
}
```

---

## 校准流程（三步）

### Step 1：欢迎页
- 深色全屏遮罩
- 说明：需要摄像头权限，引导用户点击授权
- 按钮「カメラを許可してキャリブレーションを開始」
- 点击后调用 `webgazer.begin()` 并跳转 Step 2

### Step 2：9点校准
- 9个校准点，位置（百分比相对视口）：
  ```
  (10%,10%) (50%,10%) (90%,10%)
  (10%,50%) (50%,50%) (90%,50%)
  (10%,90%) (50%,90%) (90%,90%)
  ```
- 每次显示一个点（其余隐藏）
- 每个点需要点击 **5次**，每次点击视觉缩小，颜色加深
- 点击5次后自动跳到下一个点
- 顶部显示：进度条（完成点数/9）+ 文字提示「点滅している点をクリックしてください (X/9)」
- WebGazer 默认会自动记录鼠标点击位置作为训练数据（无需手动调用 API）
- 9个点全部完成后跳转 Step 3

### Step 3：完成提示
- 绿色成功图标
- 文字：「キャリブレーション完了！標識をご覧ください。視線が追跡されます。」
- 3秒后自动消除遮罩，显示标识
- 遮罩淡出后设置 WebGazer 的 gaze listener

### WebGazer 关键初始化代码

```javascript
webgazer
  .setRegression('ridge')
  .setGazeListener((data, elapsed) => {
    if (!data) return;
    onGaze(data.x, data.y); // 主处理函数
  })
  .begin()
  .then(() => {
    // 隐藏 WebGazer 自带的视频和反馈框
    webgazer.showVideoPreview(false);
    webgazer.showFaceOverlay(false);
    webgazer.showFaceFeedbackBox(false);
  });
```

---

## 注视点映射逻辑

```javascript
// 平滑滤波（指数移动平均，避免抖动）
let smoothX = 0, smoothY = 0;
const SMOOTH = 0.25; // 越小越平滑，越大越响应快

function onGaze(rawX, rawY) {
  smoothX = smoothX * (1 - SMOOTH) + rawX * SMOOTH;
  smoothY = smoothY * (1 - SMOOTH) + rawY * SMOOTH;

  // 更新注视点 UI
  updateGazeDot(smoothX, smoothY);

  // 确定当前注视区域
  const zoneId = getZoneAtPoint(smoothX, smoothY);
  currentGazeZone = zoneId; // 全局变量
}

function getZoneAtPoint(x, y) {
  const zoneIds = ['zone-title', 'zone-taxonomy', 'zone-bubble', 'zone-image', 'zone-body', 'zone-callout'];
  for (const id of zoneIds) {
    const el = document.getElementById(id);
    if (!el) continue;
    const r = el.getBoundingClientRect();
    if (x >= r.left && x <= r.right && y >= r.top && y <= r.bottom) {
      return id;
    }
  }
  return null; // 注视在标识区域外
}
```

---

## 注视点 UI

```html
<div id="gaze-dot"></div>
```

```css
#gaze-dot {
  position: fixed;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: rgba(233, 30, 140, 0.6);
  border: 2px solid rgba(255, 255, 255, 0.8);
  box-shadow: 0 0 12px rgba(233, 30, 140, 0.8);
  pointer-events: none;
  transform: translate(-50%, -50%);
  z-index: 500;
  transition: left 0.08s linear, top 0.08s linear;
  /* 初始隐藏，校准完成后显示 */
  display: none;
}
```

```javascript
function updateGazeDot(x, y) {
  const dot = document.getElementById('gaze-dot');
  dot.style.left = x + 'px';
  dot.style.top = y + 'px';
}
```

---

## 摄像头预览组件

- 右上角固定，尺寸 160×120px，圆角边框
- WebGazer 自带的 video 元素会自动渲染，需要用 CSS 将其定位到右上角
- 添加最小化按钮（点击切换 `collapsed` 状态）
- 最小化时只显示一个小图标按钮

```javascript
// 控制 WebGazer 视频预览
webgazer.showVideoPreview(true); // 开启后 WebGazer 自动在页面插入 <video>

// 通过 CSS 覆盖 WebGazer 默认的 video 样式，把它移到右上角
// WebGazer video 元素的默认 id 是 "webgazerVideoFeed"
```

```css
#webgazerVideoFeed {
  position: fixed !important;
  top: 16px !important;
  right: 16px !important;
  width: 160px !important;
  height: 120px !important;
  border-radius: 12px !important;
  border: 2px solid rgba(255,255,255,0.3) !important;
  z-index: 400 !important;
  object-fit: cover !important;
}
#webgazerFaceOverlay, #webgazerFaceFeedbackBox {
  display: none !important;
}
```

---

## 管理员面板

### 触发方式
右下角固定一个齿轮图标按钮 `⚙`，点击后面板从下方滑入。

### 面板内容

```
┌──────────────────────────────────────────┐
│  🛠 管理ツール              [×閉じる]     │
├──────────────────────────────────────────┤
│  注視感度（上昇速度）                      │
│  [●────────────────] 15  /s             │
│                                          │
│  減衰速度（非注視時）                      │
│  [●─────────────────] 6  /s             │
│                                          │
│  スムージング係数                          │
│  [●──────────────────] 0.25             │
│                                          │
│  注視点を表示  [ON/OFF トグル]            │
│                                          │
│  ── ゾーン注視ウェイト ──                 │
│  タイトル      [██████░░░░] 60           │
│  分類          [░░░░░░░░░░] 0            │
│  ふきだし      [████░░░░░░] 40           │
│  画像          [░░░░░░░░░░] 0            │
│  本文          [████████░░] 80           │
│  コールアウト  [░░░░░░░░░░] 0            │
│                                          │
│  現在の注視座標: (542, 318)              │
│                                          │
│  [🔄 全ゾーンをリセット]                  │
└──────────────────────────────────────────┘
```

### 面板参数

```javascript
const settings = {
  sensitivity: 15,   // range: 1~40
  decay: 6,          // range: 1~20
  smoothing: 0.25,   // range: 0.05~0.8
  showGazeDot: true,
};
```

### 重置功能

```javascript
function resetAllZones() {
  for (const id in zoneState) {
    zoneState[id].weight = 0;
  }
  // 立即应用到 DOM（清除所有动态效果）
  applyAllZoneEffects();
}
```

---

## rAF 主循环（核心逻辑）

```javascript
let lastFrameTime = performance.now();
let currentGazeZone = null;

function mainLoop(now) {
  const dt = (now - lastFrameTime) / 1000; // 秒
  lastFrameTime = now;

  // 1. 更新权重
  updateWeights(dt, currentGazeZone);

  // 2. 应用视觉效果
  applyAllZoneEffects();

  // 3. 更新管理员面板权重显示
  updateAdminWeightBars();

  requestAnimationFrame(mainLoop);
}

requestAnimationFrame(mainLoop);
```

---

## 视觉效果函数

```javascript
function applyZoneEffect(zoneId) {
  const state = zoneState[zoneId];
  const w = state.weight; // 0~100
  const zone = document.getElementById(zoneId);
  
  // --- 背景高亮 ---
  const bgAlpha = w > 10 ? ((w - 10) / 90) * 0.10 : 0;
  zone.style.backgroundColor = `rgba(233, 30, 140, ${bgAlpha})`;
  zone.style.borderRadius = '10px';

  // --- 字号增大 ---
  const fontBoost = w < 30 ? 0
                  : w < 70 ? ((w - 30) / 40) * 7
                  :          7 + ((w - 70) / 30) * 5; // 最大 +12px

  // --- 颜色加深 ---
  const darkness = Math.round(136 - (w / 100) * 110); // 136→26
  const textColor = `rgb(${darkness}, ${darkness - 5}, ${darkness})`;

  // 应用到所有文字元素
  zone.querySelectorAll('.zone-text').forEach(el => {
    const base = parseFloat(el.dataset.baseFontSize || 16);
    el.style.fontSize = (base + fontBoost) + 'px';
    el.style.color = textColor;
    el.style.textDecoration = w > 55 ? 'underline' : 'none';
    el.style.textDecorationColor = '#E91E8C';
  });

  // --- 展开详情 ---
  const detail = zone.querySelector('.expanded-detail');
  if (detail) {
    if (w > 70) {
      detail.style.opacity = Math.min((w - 70) / 20, 1).toString();
      detail.style.maxHeight = '300px';
    } else {
      detail.style.opacity = '0';
      detail.style.maxHeight = '0';
    }
  }

  // --- glow 阴影 ---
  if (w > 80) {
    zone.style.boxShadow = `0 0 ${(w-80)/4}px rgba(233,30,140,0.25)`;
  } else {
    zone.style.boxShadow = 'none';
  }
}
```

---

## 样式颜色体系

```css
:root {
  /* 背景 */
  --bg-outer: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);

  /* 标识卡片 */
  --sign-bg: #FAF8F4;
  --sign-border: #E8E2D8;

  /* 主色调（粉红/洋红） */
  --pink: #E91E8C;
  --pink-light: #FFB3D9;
  --pink-bg: rgba(233, 30, 140, 0.08);

  /* 文字 */
  --text-primary: #1C1C1C;
  --text-secondary: #555555;
  --text-light: #888888;
  --text-taxonomy: #666666;

  /* 装饰 */
  --gold: #D4A843;
  --orange: #E07020;

  /* 过渡 */
  --transition-speed: 0.5s;
}
```

---

## 折纸鹤 SVG（内嵌，无需外部资源）

在标题区右侧放置3只折纸鹤，使用内联 SVG：

```html
<!-- 大粉色鹤 -->
<svg width="70" height="60" viewBox="0 0 100 85" fill="none" ...>
  <!-- 简化折纸鹤路径，用粉色填充 -->
</svg>
<!-- 中橙金色鹤 -->
<!-- 小浅橙色鹤 -->
```

> **提示给 Gemini**：折纸鹤形状可以用以下简化多边形路径模拟（不需要精确，有折纸感即可）：
> - 身体：中央菱形
> - 翅膀：两侧三角形
> - 头尾：上下尖角
> 参考颜色：`#E8A0C0`（粉）、`#D4A843`（金）、`#E07020`（橙）

---

## 重要注意事项

1. **WebGazer 版本**：使用 `https://webgazer.cs.brown.edu/webgazer.js`，该版本已包含 TensorFlow.js 依赖，无需额外引入。

2. **HTTPS 限制**：WebGazer 需要摄像头权限，部分浏览器要求 HTTPS。本地直接打开 `file://` 在 **Chrome/Edge** 中通常可以正常获取摄像头（需在浏览器设置中允许）。
   - 如摄像头无法获取，提示用户：用 `python -m http.server 8080` 启动本地服务器后访问 `http://localhost:8080`

3. **WebGazer 自动记录**：`webgazer.begin()` 后，所有鼠标点击都会自动作为训练数据记录，校准时无需手动调用额外 API。

4. **CSS Transition 路径**：对 `.zone-text` 元素，所有动态变化的属性都要设置 transition：
   ```css
   .zone-text {
     transition: font-size 0.5s ease,
                 color 0.5s ease,
                 text-decoration 0.3s ease;
   }
   .zone {
     transition: background-color 0.5s ease, box-shadow 0.5s ease;
   }
   .expanded-detail {
     transition: opacity 0.8s ease, max-height 1s ease;
     overflow: hidden;
     max-height: 0;
     opacity: 0;
   }
   ```

5. **data-base-font-size**：每个 `.zone-text` 元素必须有此属性，记录其原始字号，供 JS 计算增量用。

6. **性能优化**：`getBoundingClientRect()` 每帧调用会触发重排，可以缓存 rect（每500ms更新一次）以提升性能。

---

## 交付检查清单

- [ ] 校准流程：Step1 欢迎 → Step2 9点点击 → Step3 完成
- [ ] 标识卡片：6个区域，内容准确（日语）
- [ ] 折纸鹤装饰：右上角3只SVG
- [ ] plant.png 正确显示
- [ ] 注视权重系统：注视上升、非注视衰减
- [ ] 动态效果：字号、颜色、下划线、展开详情均正常
- [ ] 注视点圆圈跟随眼睛移动
- [ ] 管理员面板：灵敏度/衰减调节、权重条显示、重置按钮
- [ ] 摄像头预览定位右上角
- [ ] 全部 CSS transition 平滑无跳变
- [ ] 在 Chrome/Edge 直接打开 index.html 可用

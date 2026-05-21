# SIGN_prototype — AI Agent 交接说明文档

> **文档目的**：本文档面向接手此项目的下一任 AI Agent，提供完整的项目背景、技术架构、核心逻辑和尚待解决问题。请在开始任何开发工作之前完整阅读本文档。
>
> **最后更新**：2026-05-21

---

## 1. 项目概述

**SIGN_prototype** 是一个 **单页面 HTML 应用**，模拟一块安装在植物展区的互动式电子说明标牌。

核心体验：观众站在标牌前，系统通过摄像头实时追踪其视线，并根据视线停留的内容区域动态调整标牌的排版布局（字号、字重、展开隐藏内容等），从而创造出标牌"感知"到观众正在看哪里的交互效果。体验结束后，系统会生成一份视线行为分析报告并支持 JSON 导出。

**展示内容**：オリヅルラン（折鶴蘭 / Spider Plant），日文植物解说。

**运行方式**：无需构建，直接用 Live Server（VSCode 插件）或任何 HTTP 服务器打开 `index.html` 即可。注意：**不能直接双击用 `file://` 协议打开**，WebGazer 需要 HTTP 上下文才能访问摄像头。

---

## 2. 目录结构

```
SIGN_prototype/
├── index.html          ← 项目唯一源文件，包含全部 HTML / CSS / JS（约 2000 行）
├── webgazer.js         ← WebGazer.js 本地部署（视线追踪引擎，约 1.8MB）
├── plant.png           ← 植物图片素材（标牌图片区使用）
├── mediapipe/          ← WebGazer 依赖的面部关键点检测模型
│   └── face_mesh/      ← MediaPipe face_mesh 资源（.bin .tflite 等）
├── package.json        ← 仅用于 Live Server 等开发工具，无构建依赖
├── GEMINI_BUILD_GUIDE.md ← 旧版开发笔记（可参考，但以本文档为准）
└── AGENT_HANDOVER.md   ← 本文档
```

> ⚠️ **关键约束**：`webgazer.js` 和 `mediapipe/` 必须与 `index.html` 放在同一目录下，WebGazer 以相对路径加载模型，否则会报 `Failed to load model` 错误。

---

## 3. 技术栈

| 技术 | 用途 |
|------|------|
| **HTML5 / Vanilla JS / Vanilla CSS** | 全部前端实现，无框架依赖 |
| **WebGazer.js v3.x** | 基于摄像头的视线追踪引擎 |
| **Canvas 2D API** | 视线轨迹可视化（玻璃水珠 blob）+ 分析报告图表 |
| **MediaPipe FaceMesh** | WebGazer 内部使用，面部关键点检测 |
| **Google Fonts** | Noto Sans JP + Noto Serif JP |
| **LocalStorage** | 跨会话保存 WebGazer 校准数据 |

---

## 4. 页面视觉结构

标牌在视觉上由三层组成：

```
[ .sign-wrapper ]                ← Flex 列容器，居中整个标牌
    [ .sign-card ]               ← 白色卡片，CSS Grid 2列布局
        zone-title               ← 跨两列，顶部标题区
        .col-left                ← 左列
            zone-bubble          ← 气泡文字
            zone-image           ← 植物图片
        .col-right               ← 右列
            zone-taxonomy        ← 学名分类
            zone-body            ← 正文描述
            zone-callout         ← 底部感官体验引导
    [ .sign-pole ]               ← 标牌支柱（视觉装饰）
```

**卡片内各区域均为可追踪的"zone"**，每个 zone 有独立 ID 并注册到 `zoneState`。

---

## 5. 完整用户流程（状态机）

```
[step-0: 署名输入]           ← 新增。受测者输入姓名/昵称，按 Enter 或点击"次へ"
    ↓ 提交名字（不可为空）
[step-1: 欢迎 & 模式选择]
    ├─ 点击"校准开始"     ─→ WebGazer 摄像头启动
    └─ 点击"鼠标演示"     ─→ isDemoMode=true，跳过校准
[step-2: 校准界面]           ← 9个校准点，每点需点击（或按键）3次
    ↓ 9点全部完成
[step-3: 确认界面]           ← 用户预览视线追踪效果的 blob
    ↓ 点击"確認して開始"
[冷却期 2秒]                 ← interactionEnabled=false，清空历史轨迹
    ↓ 2秒后 interactionEnabled=true
[主体验界面]                 ← 动态布局交互激活，右下角显示"セッションを終了"按钮
    ↓ 点击"セッションを終了"
[分析报告界面]               ← 热力图 / 时序图 / 统计表格 + JSON 导出
    ↓ 点击"もう一度やり直す"
[step-0: 署名输入]           ← 清空名字，回到最初，下一位受测者开始
```

---

## 6. 核心 JavaScript 架构

### 6.1 关键全局变量

```javascript
// 受测者信息
let participantName = '';       // 从 step-0 输入框读取，写入导出数据

// 视线追踪状态
let smoothX, smoothY            // 平滑后的当前视线坐标
let isCalibrated                // 摄像头校准是否完成
let isDemoMode                  // 鼠标模拟模式（无需摄像头）
let interactionEnabled          // 动态布局是否激活（校准后有2秒冷却）

// 视线历史缓冲（用于 blob 可视化和权重计算）
const gazeHistory = []
const HISTORY_DURATION = 2000  // ms，保存最近2秒的视线轨迹

// 区域权重系统
const zoneState = {
  'zone-title':    { weight: 0 },   // 0~100
  'zone-taxonomy': { weight: 0 },
  'zone-bubble':   { weight: 0 },
  'zone-image':    { weight: 0 },
  'zone-body':     { weight: 0 },
  'zone-callout':  { weight: 0 }
}

// 会话数据记录（用于分析报告）
let gazeLog = []                // [{x, y, t, zone, weights}, ...]
let sessionStart = null
const LOG_INTERVAL = 100        // ms，采样间隔（10fps）

// 可调参数（管理员面板可实时修改）
const settings = {
  sensitivity: 7.5,   // 权重上升速度（/s）
  decay: 3,           // 权重衰减速度（/s）
  smoothing: 0.25,    // 视线坐标平滑系数（LERP）
  showGazeDot: true   // 是否显示视线轨迹 blob
}
```

### 6.2 主循环架构

系统有**两个独立的 RAF 循环**，务必区分：

| 循环 | 触发时机 | 职责 |
|------|----------|------|
| `mainLoop` | `confirmCalibration()` 后启动 | `updateWeights()` + `applyAllZoneEffects()` |
| `canvasLoop` | `showGazeCanvas()` 后启动（校准阶段就开始） | `drawGazeCanvas()`（视线 blob 绘制） |

> ⚠️ `canvasLoop` 独立于 `mainLoop`，在校准阶段（step-2）就已启动，确保用户在确认阶段（step-3）能预览 blob。

### 6.3 动态布局权重计算

**`updateWeights(dt)`**：
1. 统计 `gazeHistory`（最近2秒）中，每个 zone 被命中的点数占总点数的比例 `ratio`
2. `ratio > 0.05`：权重上升，速率 = `sensitivity × dt × min(ratio × 4, 2)`
3. `ratio ≤ 0.05`：权重衰减，速率 = `decay × dt`
4. 权重约束在 `[0, 100]`

**`applyZoneEffect(zoneId)`**，按 weight 值施加效果：

| Weight 阈值 | 效果 |
|------------|------|
| > 10 | 背景淡绿色出现 |
| ≥ 30 | 字号增大（最多 +8px） |
| > 50 | 其他 zone 淡出；同列 zone 收缩 |
| > 55 | 文字出现下划线 |
| > 70 | `.expanded-detail` 展开（隐藏的补充文字） |
| > 80 | 绿色发光边框 + 文字阴影 |

### 6.4 视线轨迹 Blob 可视化

**`drawGazeCanvas()`**：
1. 计算 `gazeHistory` 所有点的重心 `(cx, cy)` 和标准差 `(stdX, stdY)`
2. 基础半径 = `max(55, std × 2.5 + 48)`，随注视分散度自然扩展
3. 在圆上取 N=10 个控制点，叠加3个频率谐波使轮廓有机化
4. 用二次贝塞尔（中点法）绘制平滑闭合曲线
5. 双层描边：外层光晕（宽6px，低透明度）+ 主轮廓（宽2px，发光）

视觉效果：像玻璃上的一颗水珠，随用户视线漂移和扩展。

---

## 7. 校准系统详解

### 7.1 WebGazer API（v3.x）

```javascript
// 正确的 v3 启动方式（非链式调用）
window.saveDataAcrossSessions = true;   // 跨会话保存校准数据到 localStorage
webgazer.setRegression('ridge');
webgazer.setGazeListener((data, elapsed) => { onGaze(data.x, data.y); });
webgazer.begin(onFailCallback).then(() => {
  showNextCalibPoint();
  startCalibKeyboard();  // ← 必须在这里注册键盘快捷键！
}).catch(err => { ... });
```

> ⚠️ **已知易错点**：`startCalibKeyboard()` 必须在 `.then()` 回调中调用，否则键盘快捷键失效。过去曾因重构把这行丢失过一次，导致快捷键失灵。

### 7.2 校准点交互设计

- 9个点（3×3 矩阵），每点需**点击或按键3次**
- 圆点内实时显示剩余次数（`3` → `2` → `1`），颜色从白色渐变至深绿
- **键盘快捷键**：`Space` 或 `0`（数字行/小键盘均可），效果等同鼠标单击
- 键盘监听函数：`startCalibKeyboard()` / `stopCalibKeyboard()`（校准完成时自动注销）
- 核心点击逻辑：`doCalibClick()`（鼠标和键盘共用）

### 7.3 冷却机制

`confirmCalibration()` 调用后：
1. 遮罩淡出（1秒过渡）
2. `mainLoop` 启动，但 `interactionEnabled = false`，权重不更新
3. 2秒后清空 `gazeHistory`，设置 `interactionEnabled = true`，正式激活

---

## 8. 数据记录与导出

### 8.1 gazeLog 格式

每 100ms 采样一次，每条记录：
```javascript
{
  x: 640,           // 平滑后屏幕坐标（像素）
  y: 360,
  t: 1716234567890, // performance.now() 时间戳
  zone: 'zone-body',// 当前命中的 zone ID，null 表示未命中任何 zone
  weights: {        // 所有 zone 当前权重快照
    'zone-title': 12.5,
    'zone-body': 78.3,
    ...
  }
}
```

### 8.2 JSON 导出结构

```json
{
  "meta": {
    "participant": "山田花子",
    "exportedAt": "2026-05-21T00:00:00.000Z",
    "sessionDurationSec": 45.2,
    "sampleCount": 452,
    "settings": { "sensitivity": 7.5, "decay": 3, "smoothing": 0.25 }
  },
  "gazeLog": [
    {
      "t_ms": 0,
      "x": 640,
      "y": 360,
      "zone": "zone-body",
      "weights": { "zone-title": 0, "zone-body": 78.3, ... }
    },
    ...
  ]
}
```

文件名格式：`gaze_山田花子_2026-05-21T….json`

---

## 9. 分析报告界面

点击"セッションを終了"后显示，包含三个 Tab：

| Tab | 渲染函数 | 说明 |
|-----|----------|------|
| 熱力図 | `renderHeatmap()` | 离屏 Canvas 高斯模糊 → RGBA 色彩映射叠加到区域布局背景 |
| 注視タイムライン | `renderTimeline()` | 各 zone 权重随时间变化的彩色折线图 |
| ゾーン統計 | `renderTable()` | 按注视时长排序的统计表 + 摘要卡片 + JSON 导出按钮 |

**热力图渲染注意事项**：`endSession()` 在显示 overlay 之前会先调用 `snapshotRects` 快照所有 zone 的屏幕坐标，热力图以此为背景。切勿在 `endSession()` 之后才读取 zone 坐标，因为 overlay 覆盖后坐标会变化。

---

## 10. 管理员面板（右下角 ⚙）

| 控件 | 参数 | 默认值 |
|------|------|--------|
| 注視感度 | `settings.sensitivity` | 7.5 |
| 減衰速度 | `settings.decay` | 3.0 |
| スムージング係数 | `settings.smoothing` | 0.25 |
| 注視点を表示 | `settings.showGazeDot` | true |
| ゾーンウェイト条形图 | — | 实时显示 |
| 現在の注視座標 | — | 实时显示 |
| 🔄 全ゾーンをリセット | — | 清空历史和所有权重 |

---

## 11. DOM 层级 z-index 约定

```
z-index: 2000   #analysis-overlay      分析报告界面
z-index: 1010   #calib-progress        校准进度条
         #calib-key-hint               键盘提示
z-index: 1002   #calib-points-container 校准点容器
z-index: 1001   #step-2 (active)        校准点界面
z-index: 1000   #calibration-overlay    校准遮罩
z-index: 10000  #gaze-canvas            视线 blob（始终在内容上方）
z-index: 601    #admin-toggle
z-index: 600    #admin-panel
z-index: 602    #end-btn
z-index: 10     .sign-wrapper
z-index: 2      .sign-card
z-index: 1      .sign-pole
```

---

## 12. 已知问题与注意事项

### 12.1 键盘快捷键（历史易错点）
`startCalibKeyboard()` 必须在 `webgazer.begin().then()` 回调中调用。曾因重构将其调用丢失，导致快捷键失效。每次修改 `startCalibration()` 函数时，请确认这一行依然存在。

### 12.2 摄像头相关
- WebGazer 在 `file://` 协议下无法获取摄像头权限，必须通过 HTTP 服务器运行
- `NotReadableError`：通常是 Zoom/Teams 等占用摄像头
- `model load failed`：检查 `./mediapipe/face_mesh/` 目录是否完整

### 12.3 校准质量
- 校准时需保持头部静止，光线充足
- `localStorage['webgazerGlobalData']` 在切换显示器/分辨率后应清空重校

### 12.4 冷却机制
- `confirmCalibration()` 后有 2秒冷却，`interactionEnabled = false`
- 鼠标演示模式（`startDemoMode`）直接设置 `interactionEnabled = true`，不冷却
- `restartSession()` 回到署名页，重置所有数据，但**不会**重启 WebGazer（摄像头追踪保持活跃）

---

## 13. 如何扩展到其他植物

1. **替换图片**：修改 `<img src="./plant.png">` 的路径
2. **替换文案**：修改各 zone 内的文字内容（保持 zone ID 不变）
3. **zone ID 不可更改**：`zone-title`, `zone-bubble`, `zone-image`, `zone-taxonomy`, `zone-body`, `zone-callout` 在 JS 中有大量引用

若要支持**多植物轮播**，建议将内容数据化后按需注入各 zone。

---

## 14. 潜在改进方向

| 优先级 | 方向 | 说明 |
|--------|------|------|
| 高 | 多植物内容 | 内容数据化，支持多个植物条目切换 |
| 高 | Kiosk 部署 | 全屏自动启动，无操作超时自动重置到 step-0 |
| 中 | CSV 导出 | 在 JSON 导出旁增加 CSV 格式导出 |
| 中 | 长期漂移修正 | 长时间运行后精度下降，可加定期重校准提醒 |
| 低 | 多语言支持 | 目前全日语，可考虑中日英三语切换 |
| 低 | 无障碍降级 | 摄像头失败时提供触控/按键的完整交互替代方案 |

---

## 15. 快速上手验证清单

接手后，请按以下步骤验证项目正常运行：

- [ ] Live Server 打开 `index.html`，显示署名输入页（绿色风格，大输入框）
- [ ] 不填名字直接点"次へ"，出现红色错误提示
- [ ] 填入名字，跳转到摄像头选择页
- [ ] 点击"🖱️ マウスでデモ"，校准遮罩消失，进入鼠标演示模式
- [ ] 移动鼠标，右上角出现绿色流动光环（blob）
- [ ] 将鼠标悬停在某区域约3~5秒，字号增大、展开隐藏内容
- [ ] 点击右下角 ⚙，权重条形图实时变化
- [ ] 点击"セッションを終了"，进入报告界面，标题含受测者名字
- [ ] 切换三个 Tab，均正常显示
- [ ] 在统计 Tab 点击"↓ JSONデータをエクスポート"，下载文件名含名字
- [ ] 点击"もう一度やり直す"，回到署名输入页，名字已清空

**摄像头模式额外验证：**
- [ ] 点击"📷 カメラを許可して校準開始"，出现9个大绿圈校准点
- [ ] 点击校准点，数字从3倒数，颜色渐变为深绿
- [ ] 按 Space 键，效果与鼠标单击相同
- [ ] 完成9点后进入确认页，blob 跟随视线移动

---

*文档最后更新：2026-05-21 | 记录者：Antigravity (AI Agent)*

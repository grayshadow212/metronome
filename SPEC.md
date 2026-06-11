# SPEC: 节拍器 (Metronome) — 吉他日常练习专用

## Overview

一个单文件 HTML / PWA 节拍器，针对吉他手日常练习场景优化。核心价值：**够用、好用、随时可用**（手机架在谱架/音箱上即开即用，可保存为桌面应用离线运行）。

设计原则：
- **零安装** — 双击 HTML 文件或部署到任意静态托管即可使用
- **离线优先** — 完整 PWA（manifest + service worker），断网可用
- **零资源依赖** — 所有声音用 Web Audio API 实时合成（不打包任何音频文件）
- **键盘/触屏双友好** — 练习时手不离琴也能操作
- **单手可控** — 手机模式下所有核心操作（启停、BPM、拍号、音量）拇指够得到

---

## Data Models

### 1. `Settings` (运行时状态)
```js
{
  // 节奏核心
  bpm: 100,                          // 范围 40-300
  timeSignature: { beats: 4, noteValue: 4 },  // 4/4 默认；支持 1-12 拍
  subdivision: 1,                    // 1=无 2=八分 3=三连音 4=十六分

  // 声音
  soundSet: 'click',                 // 'click' | 'tick' | 'wood' | 'beep' | 'cowbell' | 'rim'
  accentDownbeat: true,              // 第 1 拍重音
  volume: 0.7,                        // 0-1
  muted: false,                       // 静音模式（仅视觉）

  // 视觉
  visualMode: 'dots',                 // 'dots' | 'pendulum' | 'numbers' | 'ring'
  theme: 'auto',                      // 'auto' | 'light' | 'dark'
  showBeatNumber: true,               // 是否显示当前拍号

  // 行为
  countIn: 0,                         // 0/1/2/4 小节预备拍
  wakeLockEnabled: true,              // 是否保持屏幕常亮

  // 运行时
  isPlaying: false,
  currentBeat: 0,                     // 0..beats-1，0 表示第 1 拍
  startedAt: null                     // 时间戳，便于对齐
}
```

### 2. `Preset` (持久化)
```js
{
  id: 'p_20260611_200800_a3f',
  name: '慢速热手指 70',              // 必填，≤20 字
  bpm: 70,
  timeSignature: '4/4',              // 字符串便于展示
  subdivision: 1,
  soundSet: 'wood',
  notes: '哈农 + 音阶',                // 可选备注，≤50 字
  createdAt: '2026-06-11T20:08:00Z',
  updatedAt: '2026-06-11T20:08:00Z'
}
```

### 3. `TapTempoState` (临时)
```js
{
  taps: [],                          // 最近 6 次 tap 的时间戳 (ms)
  lastTapAt: 0
}
```

### 4. `SoundProfile` (代码内常量)
```js
{
  'click':   { accent: { freq: 1800, dur: 0.03, gain: 0.9, wave: 'square' },
               beat:   { freq: 1200, dur: 0.025, gain: 0.5, wave: 'square' },
               sub:    { freq: 800,  dur: 0.015, gain: 0.3, wave: 'square' } },
  'tick':    { ... },                 // sine 短促
  'wood':    { ... },                 // triangle + 滤波感，类拨片打板
  'beep':    { ... },                 // sine 中长
  'cowbell': { ... },                 // 两个 sine 叠加
  'rim':     { ... }                  // 高频短促 + 噪声
}
```

---

## User Flows

### Flow 1: 首次打开
1. 加载 HTML → 注册 service worker → 检查 localStorage
2. 无预设：显示默认 100 BPM / 4/4 / click 音 / dots 视觉
3. 有预设：在 BPM 数字下方显示预设行（横滑选择器）
4. 询问是否启用 Wake Lock（仅移动端）
5. 进入主界面

### Flow 2: 调整 BPM
- **3 种方式**（任意切换）：
  - **大数字点击** → 弹出数字键盘输入
  - **BPM 区域上下滑动** → ±1 BPM（移动端）
  - **侧边 +/- 按钮** → -10 / -5 / -1 / +1 / +5 / +10
- **键盘快捷键**：`[` `-1`、`]` `+1`、`{` `-5`、`}` `+5`
- 自动 clamp 到 40-300

### Flow 3: Tap Tempo（拍速探测）
1. 点击"TAP"按钮（或键盘 `T`）
2. 至少 2 次点击后开始显示推算 BPM
3. 取最近 2-6 次间隔平均值，滑窗更新
4. 推算后询问"应用到 BPM？"→ 是则更新；否则忽略
5. 连续 2 秒未点击 → 重置

### Flow 4: 开始 / 停止
- **大播放按钮**（屏幕底部正中）：点击切换
- **空格键**（桌面端）：切换
- 视觉指示器立即开始/停止（不延迟）
- 声音通过 Web Audio `AudioContext.currentTime` 精确调度

### Flow 5: 拍号 & 细分
- 拍号下拉：1/4, 2/4, 3/4, 4/4, 5/4, 6/8, 7/8, 9/8, 12/8
- 细分按钮：×1（无）/ ×2（八分）/ ×3（三连音）/ ×4（十六分）
- 切换后立即应用（不重置播放状态）

### Flow 6: 保存 & 调用预设
1. 当前设置 → 点击"保存" → 输入名称（默认 `BPM-拍号`）→ 可选备注 → 保存
2. 预设行显示为 chips：横滑浏览，长按/右键删除，长按可重命名
3. 点击 chip → 一键应用（即使在播放中也立即切换）
4. localStorage 最多存 50 个，超出时弹出提示并要求删除旧的

### Flow 7: 静音 / 视觉模式
- **静音切换** → 按钮 + 键盘 `M`
- **视觉模式切换** → 循环 4 种（dots / pendulum / numbers / ring）
- **主题切换** → auto / light / dark（auto 跟随系统）

### Flow 8: 预备拍（Count-In）
- 设置 countIn = 4 → 按下播放后先空打 1 小节（同样 BPM），第 5 拍进入正式节奏
- 屏幕显示 "Count-in: 4 3 2 1"
- 预备拍时视觉指示器也同步显示

---

## UI 布局（移动端为主，桌面端自适应）

```
┌────────────────────────────┐
│  ⚙  [theme] [save] [?]    │  ← 顶栏
├────────────────────────────┤
│  ◀  ● ● ● ●  ▶            │  ← 拍号选择
├────────────────────────────┤
│        100                 │  ← BPM 大字（点击输入）
│       BPM                  │
│   -10  -5  -1  +1  +5 +10  │  ← 微调按钮
├────────────────────────────┤
│  ● ● ● ●  (4/4)            │  ← 视觉指示器 (dots 模式)
│      ↑ 红色 = 第 1 拍重音 │
├────────────────────────────┤
│  [TAP]   [×1][×2][×3][×4]  │  ← Tap & 细分
│  Sound: [click ▾] Vol: ▮▮▮ │  ← 声音设置
├────────────────────────────┤
│  Presets:                  │
│  ┌──┐┌──┐┌──┐┌──┐┌──┐     │  ← 预设 chips（横滑）
│  └──┘└──┘└──┘└──┘└──┘     │
├────────────────────────────┤
│         ⏯  PLAY            │  ← 大播放按钮
└────────────────────────────┘
```

桌面端 ≥768px：左右两栏布局，左控制右视觉（大尺寸视觉指示器）。
黑暗模式：背景 `#0a0a0a`，重音 `#ef4444`，普通拍 `#22c55e`，文字 `#e5e5e5`。
明亮模式：背景 `#fafafa`，重音 `#dc2626`，普通拍 `#16a34a`，文字 `#171717`。

---

## File Structure

```
metronome/
├── SPEC.md                     # 本文件
├── index.html                  # 单文件应用：HTML + 内联 CSS + 内联 JS
├── manifest.webmanifest        # PWA 清单（应用名、图标、主题色）
├── sw.js                       # Service Worker：缓存所有静态资源，离线可用
├── icon-192.png                # PWA 图标（192×192）
├── icon-512.png                # PWA 图标（512×512）
└── README.md                   # 部署说明 + 快捷键速查 + 浏览器兼容性
```

**关键设计决策**：
- 所有 CSS/JS 内联在 `index.html`，只有图标和 manifest/sw 分离（标准 PWA 要求）
- 体积目标：`index.html` < 50KB（gzip 后 < 15KB），总 < 100KB
- 图标：先用 SVG 内联生成 PNG（脚本一次性输出），后续可换设计稿

---

## Edge Cases & Error Handling

| 场景 | 处理 |
|------|------|
| BPM 输入越界（<40 或 >300） | 自动 clamp，越界时输入框红色边框 0.5s |
| Web Audio API 不可用 | 首屏显示"浏览器不支持，请用 Chrome/Safari/Edge 最新版" |
| AudioContext suspended（无用户交互） | 首次播放时自动 `resume()`，无感知 |
| Service Worker 注册失败 | 静默降级到普通网页模式，仍可在线使用 |
| localStorage 不可用 | 预设功能降级为内存存储，刷新后丢失，UI 标注"预设未保存" |
| Wake Lock 不可用 | 提示"屏幕可能会息屏" |
| 标签页切到后台 | 音频继续播放（依赖浏览器策略），回到前台后视觉立即同步 |
| 同时按住 +/- 微调按钮 | 节流 60ms 一次 |
| Tap tempo 间隔 < 100ms | 视为抖动，丢弃 |
| Tap tempo 间隔 > 2s | 自动重置 |
| 拍号 0 拍或不合法 | 拒绝应用并提示 |
| preset 名称为空 | 提交按钮 disabled |
| preset 名称重复 | 追加 `(2)` 后缀 |
| preset > 50 个 | 弹窗提示，需先删除 |
| 设备低电量模式 | 文档中提醒 Wake Lock 可能被系统拒绝 |

---

## Out of Scope (v1 不做)

- ❌ 节拍器游戏 / 节奏训练模式
- ❌ 调音器（可作为 v2 单独模块）
- ❌ 录音 / 演奏评估
- ❌ 多轨 / 鼓机 / 复杂节奏型
- ❌ 蓝牙 MIDI 踏板控制（v2 考虑：Web MIDI API）
- ❌ 云端预设同步（v2 考虑：导出/导入 JSON）
- ❌ 和弦/音阶练习辅助
- ❌ 音色采样（v1 全部用 Web Audio 合成）

---

## Tech Stack

- **核心**：Vanilla HTML + CSS + JavaScript（ES2020+）
- **音频**：Web Audio API（OscillatorNode + GainNode + 简单包络）
- **存储**：localStorage（key: `metronome.presets.v1`、`metronome.settings.v1`）
- **PWA**：标准 manifest + service worker（cache-first 策略）
- **屏幕常亮**：Wake Lock API
- **图标**：SVG → Canvas → PNG（脚本生成，无外部依赖）
- **零外部依赖**：不引入任何 JS/CSS 库

---

## Keyboard Shortcuts

| 键 | 动作 |
|----|------|
| `Space` | 播放/停止 |
| `M` | 静音切换 |
| `T` | 拍速探测（tap） |
| `V` | 切换视觉模式 |
| `S` | 打开"保存预设"对话框 |
| `[` | BPM -1 |
| `]` | BPM +1 |
| `{` | BPM -5 |
| `}` | BPM +5 |
| `1` `2` `3` `4` | 切换细分（1=无, 2=八分, 3=三连音, 4=十六分） |

---

## Open Questions

1. **默认 BPM = 100？** 还是首次打开问用户？
   → 默认 100；如需改可在设置里持久化。
2. **音色库范围**：上述 6 种够用吗？是否需要更"原声吉他"味的采样？
   → v1 用合成（零依赖），音色切换预留扩展点。
3. **桌面端布局**：左右分栏 vs 单栏居中？
   → 默认单栏居中（简洁），≥1024px 自动变两栏。
4. **图标风格**：用 emoji 🎵 还是简约几何？
   → 简约几何（深绿圆点 + 节拍线），emoji 在不同系统显示不一致。
5. **Wake Lock 默认开启？**
   → 默认开启，提供关闭入口（顶栏图标），节省电量。

---

## Acceptance Criteria (完成标准)

- [ ] `index.html` 单文件，Chrome/Safari/Edge 最新版正常运行
- [ ] PWA：可"添加到主屏幕"，离线打开仍可用
- [ ] 所有 8 个用户流程流畅无 bug
- [ ] 移动端 375×667（iPhone SE）布局无横向滚动
- [ ] 桌面端 1280×800 自适应两栏
- [ ] 切换 BPM 0 延迟（用 `AudioContext.currentTime` 调度）
- [ ] 音色切换立即生效（不需重启）
- [ ] 预设保存/调用/删除/重命名全部工作
- [ ] 暗/亮模式自动跟随系统，可手动覆盖并记忆
- [ ] 键盘快捷键全部生效，不与浏览器默认冲突
- [ ] Tap tempo 准确（误差 < 1 BPM）
- [ ] README.md 包含：使用方法、快捷键、浏览器兼容列表、部署到任意静态托管的指引

---

*Spec 完成。等待用户确认后开始实现。*

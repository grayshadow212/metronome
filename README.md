# 🎸 节拍器 (Guitar Metronome)

吉他手日常练习专用的单文件 PWA 节拍器。**零依赖、零安装、离线可用**。

> 双击 `index.html` 即可使用，或部署到任意静态托管（如 GitHub Pages、Netlify、Vercel、Cloudflare Pages）。
> 可"添加到主屏幕"成为 PWA，体验接近原生 App。

---

## ✨ 特性

- **BPM 40-300** — 6 种调整方式：数字点击输入 / 滑动手势 / ±微调按钮 / 键盘 `[]{}` / 触屏 / Tap Tempo
- **9 种拍号** — 1/4、2/4、3/4、4/4、5/4、6/8、7/8、9/8、12/8
- **4 种细分** — 无、八分、三连音、十六分
- **6 种合成音色** — Click / Tick / Wood / Beep / Cowbell / Rim（Web Audio 实时合成，零音频文件）
- **4 种视觉模式** — 圆点（dots）/ 摆锤（pendulum）/ 数字（numbers）/ 圆环（ring）
- **预设管理** — 保存/调用/重命名/删除（最多 50 个，localStorage 持久化）
- **Tap Tempo** — 拍速自动探测
- **预备拍** — 0/1/2/4 小节可选
- **暗/亮模式** — 跟随系统或手动切换
- **屏幕常亮** — Wake Lock API，练习时屏幕不息屏
- **离线优先** — Service Worker cache-first，断网完全可用
- **响应式布局** — 移动端单手可控，桌面端自动两栏

---

## 🚀 使用方法

### 本地使用
1. 直接双击 `index.html` 在浏览器打开
2. 或运行本地服务器（推荐，避免某些浏览器限制）：
   ```bash
   cd metronome
   python3 -m http.server 8080
   # 浏览器打开 http://localhost:8080
   ```

### 部署到静态托管
把整个 `metronome/` 目录上传到任意静态托管：
- **GitHub Pages**：推到 `gh-pages` 分支
- **Netlify**：拖拽文件夹到 https://app.netlify.com/drop
- **Vercel**：`vercel deploy`
- **Cloudflare Pages**：连接 git 仓库

### 安装为 PWA
- **iOS Safari**：分享菜单 → "添加到主屏幕"
- **Android Chrome**：菜单 → "安装应用"
- **桌面 Chrome/Edge**：地址栏右侧会出现安装图标

---

## ⌨️ 快捷键

| 键 | 动作 |
|----|------|
| `Space` | 播放 / 停止 |
| `T` | Tap tempo（拍速探测） |
| `M` | 静音切换 |
| `V` | 切换视觉模式 |
| `S` | 保存当前设置为预设 |
| `[` | BPM -1 |
| `]` | BPM +1 |
| `{` | BPM -5 |
| `}` | BPM +5 |
| `1` `2` `3` `4` | 切换细分 |
| `Esc` | 关闭弹窗 |

---

## 📁 文件结构

```
metronome/
├── index.html              # 单文件主应用（HTML + CSS + JS 全部内联，~50KB）
├── manifest.webmanifest    # PWA 清单
├── sw.js                   # Service Worker（离线缓存）
├── icon-192.png            # PWA 图标 192×192
├── icon-512.png            # PWA 图标 512×512
├── apple-touch-icon.png    # iOS 图标 180×180
├── generate-icons.py       # 图标生成脚本（可重跑）
├── SPEC.md                 # 设计规格文档
└── README.md               # 本文件
```

---

## 🌐 浏览器兼容

| 浏览器 | 版本 | 状态 |
|--------|------|------|
| Chrome / Edge | 90+ | ✅ 完整支持 |
| Safari (macOS) | 14+ | ✅ 完整支持 |
| Safari (iOS) | 14+ | ✅ 完整支持（含 PWA 安装） |
| Firefox | 88+ | ✅ 完整支持 |
| Samsung Internet | 14+ | ✅ 完整支持 |

**必需 API**：
- Web Audio API（声音合成）
- localStorage（预设持久化）
- Service Worker（离线，PWA 必需）
- Wake Lock API（屏幕常亮，移动端）

**降级策略**：以上 API 任意一个不可用时，应用仍可运行，仅相关功能降级。

---

## 🛠️ 自定义

### 调整音色
编辑 `index.html` 中 `SOUND_PROFILES` 常量，可调整每个音色的频率、持续时间、增益、波形：

```js
click: { accent: { f: 1800, d: 0.030, g: 0.90, w: 'square' }, ... }
```

### 添加新拍号
编辑 `index.html` 中 `TIME_SIGNATURES` 数组。

### 修改主题色
编辑 `:root` 中的 CSS 变量（`--accent`, `--beat`, `--beat-down` 等）。

### 重新生成图标
```bash
python3 generate-icons.py
```

---

## 📜 设计决策

详见 [`SPEC.md`](./SPEC.md) — 包含数据模型、用户流程、边界处理、Out of Scope、技术栈选择。

**核心原则**：
1. **零安装** — 双击 HTML 就能用
2. **离线优先** — PWA 完整支持，断网不影响练习
3. **零资源依赖** — 不打包音频文件，音色用 Web Audio 合成
4. **单手可控** — 手机模式所有核心操作拇指够得到

---

## 📝 License

MIT

# Mineradio Linux 适配修改文档

> 原始源码：Mineradio-2.0.3.zip（v2.0.3，GPL-3.0）  
> 修改日期：2026-07-30  
> 原作者：XxHuberrr（GitHub）  
> 适配者：龑龑（yan）

---

## 修改概览

共修改 **3 个文件**：

| 文件 | 改动行数 | 类型 |
|---|---|---|
| `desktop/main.js` | 2 处 | 🐛 GPU 渲染后端 + GPU 报告字段 |
| `public/js/index-loader.js` | 4 行 | 🐛 鼠标侧键拦截 |
| `package.json` | 15 行 | 📦 打包配置 |

---

## 一、desktop/main.js — GPU 渲染后端修复

### 1.1 CHROMIUM_SAFE_PERFORMANCE_SWITCHES（第 479 行）

```diff
-  ['use-angle', 'd3d11'],
+  ['use-angle', 'gl'],
```

### 1.2 GPU 信息报告字段（第 1862 行，2.0.3 新增）

```diff
-      angle: 'd3d11',
+      angle: 'gl',
```

### 作用
将 Chromium 的 ANGLE 图形后端从 `d3d11`（Direct3D 11，仅 Windows）改为 `gl`（OpenGL）。

### 原因
原项目为 Windows 设计，使用 `d3d11` 作为 GPU 渲染后端。Linux 上不支持 Direct3D，若不加此参数 Electron 无法正常渲染粒子可视化（Three.js/WebGL），窗口可能白屏或崩溃。

### 影响
- ✅ Linux 下 Three.js 粒子效果正常渲染
- ✅ `--disable-gpu` 不再需要，GPU 加速保留
- ⚠️ 配合 `--ozone-platform=x11` 使用（Wayland + Vulkan 有兼容问题）

---

## 二、public/js/index-loader.js — 鼠标侧键拦截

### 位置
第 3-7 行，模块加载器入口

### 改动

```diff
 (function loadMineradioIndexModules() {
+  // 禁用鼠标侧键（后退/前进），防止误触导致页面重载
+  window.addEventListener('mouseup', function (e) {
+    if (e.button === 3 || e.button === 4) { e.preventDefault(); }
+  }, true);
   const moduleCacheBust = String(Date.now());
```

### 原因
Chromium/Electron 中，鼠标侧键默认触发浏览器前进/后退导航。Mineradio 作为单页 Electron 应用，后退导航会导致整个页面重载。在 Wayland 环境下此问题尤为突出。

---

## 三、package.json — Linux 打包配置

### 3.1 元数据补充

```diff
-  "author": "Mineradio",
+  "author": "Mineradio <3609286195@qq.com>",
+  "homepage": "https://github.com/XxHuberrr/Mineradio",
```

`.deb` 格式要求 `author` 包含邮箱、`homepage` 字段非空。

### 3.2 Linux 打包目标

```json
"linux": {
  "icon": "build/icon.png",
  "category": "Audio",
  "maintainer": "yan <3609286195@qq.com>",
  "executableArgs": ["--no-sandbox", "--ozone-platform=x11"],
  "target": [
    { "target": "deb", "arch": ["x64"] }
  ]
}
```

### 3.3 构建脚本

```json
"build:linux": "electron-builder --linux deb --publish never"
```

---

## 四、启动测试结果

```bash
cd Mineradio-2.0.3
npx electron . --no-sandbox --ozone-platform=x11
```

```
======================================================
 粒子音乐可视化 v2  →  http://localhost:3000
 登录态: 未登录
======================================================
[StartupWindow] visible: ready-to-show
[StartupWindow] visible: dom-ready
[StartupWindow] visible: did-finish-load
[StartupWindow] visible: navigation-complete
```

✅ 窗口正常显示，无崩溃。

---

## 五、已知问题

- Wayland 原生模式下 GPU 渲染有兼容问题，已通过 `--ozone-platform=x11` 规避
- 首次导航可能失败一次后自动恢复（startup.html 路径编码问题，中文路径）
- Wallpaper Engine / 完整桌面模式 / 桌面图标功能仅 Windows 可用（Linux 上自动禁用）
- 系统内存清理功能仅 Windows 可用（依赖 PowerShell + Win32 API）
- `rcedit` 仅 Windows 可用，不影响 Linux 打包

---

## 六、修改文件清单

```
Mineradio-2.0.3/
├── desktop/main.js              # use-angle: d3d11→gl ×2
├── public/js/index-loader.js    # 鼠标侧键拦截
└── package.json                 # Linux .deb 打包配置
```

---

*文档生成时间：2026-07-30*

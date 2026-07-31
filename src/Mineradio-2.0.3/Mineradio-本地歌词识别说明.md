# Mineradio 2.0.4 — 本地歌词识别功能说明

> 版本：2.0.4（基于 2.0.3 复制）
> 日期：2026-07-31
> 状态：✅ 已实现并实测通过（源码版 + dist 部署版均已同步）

---

## 一、问题背景

用户反馈 Mineradio 播放**本地音乐文件**时无法显示歌词（"本地歌词识别不出来"）。

排查结论：**不是 Linux 适配问题，也不是原版不支持歌词**——在线歌曲（酷狗/网易云/QQ 等）的歌词功能完整（3D 粒子歌词，`fx.particleLyrics` 默认开启，WebGL2 渲染正常）。真正的问题是**原版代码从未实现本地文件的歌词读取**，且歌词获取链路有 3 处逻辑**显式排除了 local 类型歌曲**。

---

## 二、根因分析（原版代码缺陷）

### 1. 导入阶段 — 不读取任何元数据

`public/js/modules/06-lyrics/05-upload-dragdrop.js` 的 `localSongFromAudioFile()`：

- 只从文件名提取标题（`filename.replace(/\.[^.]+$/, '')`）
- `artist` 写死为 `'本地文件'`
- **不解析音频元数据**（ID3 USLT / FLAC LYRICS / M4A 内嵌歌词标签）
- **不读取同目录同名 .lrc 文件**

### 2. 获取阶段 — 3 处显式排除 local

`public/js/modules/06-lyrics/00-lyrics-fetch-parse.js`：

| 行号(原) | 函数 | 问题 |
|---|---|---|
| 60 | `lyricQueuePrefetchCandidate` | `song.type === 'local'` 直接 return false，本地歌曲不预取歌词 |
| 158 | `shouldFetchNeteaseLyricTranslationFallback` | local 不参与在线歌词搜索兜底 |
| 296 | `shouldRetryStartupLyricFetch` | local 不重试歌词获取 |

### 3. 端点阶段 — 永远拿不到

`lyricEndpointForSong()` 对 local 歌曲无专属分支：本地歌曲没有 `id/hash/mid`，落回默认的网易云 `/api/lyric?id=`（空参数）→ 永远返回空歌词。

**原版结论**：本地歌曲只有一条手动通道——「自定义歌词」（localStorage 粘贴，`06-track-detail-lyrics-actions.js`）。

---

## 三、解决方案（A + B 双通道）

### 方案 A — 在线歌词匹配兜底（改动最小，播放即出歌词）

本地文件按**文件名解析出的歌名（±歌手）**，依次搜索**酷狗 → 网易云**，匹配候选歌曲后拉取其歌词。

### 方案 B — 本地文件歌词优先（更准，不依赖网络）

- **同目录同名 .lrc 文件**优先（如 `不为谁而作的歌.flac` ↔ `不为谁而作的歌.lrc`）
- 无 .lrc 时用 **music-metadata**（项目已有依赖 v11.14.0）解析音频**内嵌歌词标签**（ID3 USLT / FLAC LYRICS / M4A，含同步歌词）

### 最终歌词优先级

```
1. 同目录 .lrc 文件          ← 方案 B
2. 音频内嵌歌词标签           ← 方案 B
3. 在线搜索匹配（酷狗→网易云） ← 方案 A
4. 标题占位（fallback）       ← 原逻辑保留
```

---

## 四、具体改动明细（4 个文件）

### 1. `public/js/modules/06-lyrics/00-lyrics-fetch-parse.js`（核心）

**放开 3 处 local 排除**（保留 podcast 排除）：

```diff
 function lyricQueuePrefetchCandidate(song) {
-  if (!song || song.type === 'podcast' || song.type === 'local' || song.source === 'local' || song.localUrl) return false;
+  if (!song || song.type === 'podcast' || song.source === 'podcast') return false;
```

```diff
 function shouldFetchNeteaseLyricTranslationFallback(song, state) {
   if (!song || !state || !state.usableLyric) return false;
-  if (song.type === 'local' || song.source === 'local' || song.localUrl || song.type === 'podcast') return false;
+  if (song.type === 'podcast' || song.source === 'podcast') return false;
```

```diff
 function shouldRetryStartupLyricFetch(song, token, attempt) {
   if (!song || token !== trackSwitchToken || (attempt || 0) >= 3) return false;
-  if (song.type === 'local' || song.source === 'local' || song.localKey || song.type === 'podcast') return false;
+  if (song.type === 'podcast' || song.source === 'podcast') return false;
```

**新增 4 个函数**：

| 函数 | 作用 |
|---|---|
| `isLocalSongObject(song)` | 判断是否为本地歌曲（type/source/localKey/localUrl 任一命中） |
| `parseLocalSongTitleArtist(song)` | 从文件名解析歌名/歌手；支持 `"歌名 - 歌手"` 与 `"歌手 - 歌名"` 两种命名（`altTitle/altArtist` 双向候选），并保留 `rawName` 原文 |
| `findLocalSongLyricCandidate(parsed, list)` | 匹配候选：先精确匹配（normalizeMatchText 标题 + artistNameParts 歌手，标题/歌手双向变体），失败后 `scoreSongSearchResult >= 28` 打分兜底 |
| `readLocalDiskLyric(song)` | 通过 IPC 读本地歌词（见方案 B），只认 `localDiskPath`（真实磁盘路径），避免 webkitRelativePath 误读 |
| `fetchLocalSongLyric(song, token)` | 总入口：本地磁盘歌词优先 → 在线兜底（酷狗 → 网易云），带 `trackSwitchToken` 竞态保护 |

**改造 `fetchLyric()` 主流程**：

```diff
-    var r = await apiJson(lyricEndpointForSong(song || songOrId));
+    var r;
+    if (isLocalSongObject(song)) {
+      r = (await fetchLocalSongLyric(song, token)) || {};
+    } else {
+      r = await apiJson(lyricEndpointForSong(song || songOrId));
+    }
```

在线搜索 query 用 `rawName`（原始文件名）而非拼接结果，避免拆反歌手/歌名导致搜不到。

### 2. `public/js/modules/06-lyrics/05-upload-dragdrop.js`（导入时保存磁盘路径）

`localSongFromAudioFile()` 新增：用 Electron `webUtils.getPathForFile(file)` 获取拖拽/选择的 File 对象的**真实磁盘绝对路径**，存入 `song.localDiskPath`（方案 B 读取 .lrc / 内嵌标签的前提）。

```diff
   var title = filename.replace(/\.[^.]+$/, '');
+  var diskPath = '';
+  try {
+    if (window.desktopWindow && typeof window.desktopWindow.getPathForFile === 'function') {
+      diskPath = String(window.desktopWindow.getPathForFile(file) || '');
+    }
+  } catch (e) {
+    console.warn('[LocalImport] getPathForFile failed:', e);
+  }
   return hydrateCustomCover({
     ...
     localUrl: URL.createObjectURL(file),
     localPath: rel,
+    localDiskPath: diskPath,
     duration: 0
   });
```

### 3. `desktop/preload.js`（暴露 IPC 桥）

```diff
-const { contextBridge, ipcRenderer, clipboard } = require('electron');
+const { contextBridge, ipcRenderer, clipboard, webUtils } = require('electron');
```

新增两个方法：

```js
getPathForFile: (file) => {
  try { return webUtils.getPathForFile(file) || ''; } catch (_) { return ''; }
},
readLocalLyric: (filePath) => ipcRenderer.invoke('mineradio-read-local-lyric', String(filePath || '')),
```

### 4. `desktop/main.js`（主进程 IPC handler）

在 `mineradio-cache-get-settings` 附近新增：

- 常量 `LOCAL_LYRIC_AUDIO_EXT_RE`：支持 mp3/flac/wav/ogg/m4a/aac/opus/wma/ape/aiff 等格式
- 函数 `readLocalLyricFromFile(filePath)`：
  1. **同目录同名 .lrc 优先**：`filePath.replace(音频扩展名, '.lrc')`，存在则读（去除 BOM）
  2. 无 .lrc 时 `await import('music-metadata')`（ESM 动态导入）解析内嵌歌词：
     - 非同步歌词 `common.lyrics[].text`（ID3 USLT 等）
     - 同步歌词 `common.lyrics[].syncText[].text`（逐行拼接）
  3. 均无 → `{ ok: false, error: 'NO_LYRIC' }`
- IPC handler `mineradio-read-local-lyric`：返回 `{ ok, source: 'lrc'|'embedded'|'embedded-sync', lyric }`

---

## 五、实测验证（CDP 真机调试）

启动方式（开发调试）：`env -u ELECTRON_RUN_AS_NODE ./node_modules/.bin/electron . --remote-debugging-port=9223 --no-sandbox`

> ⚠️ 注意：本机环境注入过 `ELECTRON_RUN_AS_NODE=1`，会导致 Electron 以纯 node 模式运行（API 全失效、chromium 参数被拒），必须 `env -u` 清除。

| 测试项 | 结果 |
|---|---|
| preload IPC 能力检查 | ✅ `readLocalLyric` / `getPathForFile` 均存在 |
| IPC 读 `~/音乐/不为谁而作的歌 - 林俊杰.flac` | ✅ `{ok:true, source:'lrc'}`，939 字符，时间轴完整 |
| `fetchLocalSongLyric` 模拟播放（有 .lrc） | ✅ 52 行歌词，`timingSource='lrc-line'`，粒子歌词逐行渲染（截图确认界面中央显示"回头竟然认不得 / 需要从记忆再摸索的人"） |
| `fetchLocalSongLyric` 模拟播放（无 .lrc） | ✅ 在线兜底成功，酷狗搜索匹配，1845 字符 |
| 歌词渲染状态 | ✅ `lyricsLines` 52 行、`stageLyrics.currentText` 实时更新、无黑屏 |

---

## 六、使用说明

1. **导入方式**：通过界面「导入音乐」按钮（选择音频/文件夹）或**拖拽**音频文件进窗口。必须走导入流程才能拿到磁盘路径（`localDiskPath`），方案 B 才生效。
2. **.lrc 要求**：与音频文件**同目录、同名**（扩展名不同即可）：
   - `不为谁而作的歌.flac` ↔ `不为谁而作的歌.lrc` ✅
   - `不为谁而作的歌.flac` ↔ `不为谁而作的歌 林俊杰.lrc` ❌（不匹配）
3. **无 .lrc / 无内嵌歌词**时自动走方案 A 在线搜索匹配（需网络，酷狗优先）。
4. **内嵌歌词**：mp3 的 ID3 USLT 标签、FLAC 的 LYRICS 注释、M4A 的歌词 atom 均可被 music-metadata 识别。

---

## 七、已知限制 / 注意事项

- `localDiskPath` 仅在 **Electron 桌面端**（preload 桥）可用；纯浏览器访问 `127.0.0.1:3000` 时无磁盘路径，只能走方案 A 在线兜底。
- 在线匹配依赖歌曲名搜索质量；文件名严重不规范（无歌名特征）时可能匹配失败，此时回落到标题占位。
- 本地歌曲的持久化歌词缓存 key 基于 name+artist（artist 恒为"本地文件"），**同名不同曲**的本地文件会共享缓存——若遇错词，可用「自定义歌词」或改名规避。
- music-metadata 为 ESM 模块，主进程用动态 `import()` 加载（CJS require 会失败），已在 handler 内 try/catch 保护。

---

## 八、改动文件清单

| 文件 | 位置 | 改动类型 |
|---|---|---|
| `public/js/modules/06-lyrics/00-lyrics-fetch-parse.js` | 源码 + dist/linux-unpacked/resources/app | 放开关卡 + 新增 5 函数 + 改造 fetchLyric |
| `public/js/modules/06-lyrics/05-upload-dragdrop.js` | 源码 + dist/linux-unpacked/resources/app | 保存 localDiskPath |
| `desktop/preload.js` | 源码 + dist/linux-unpacked/resources/app | webUtils + 2 个新方法 |
| `desktop/main.js` | 源码 + dist/linux-unpacked/resources/app | IPC handler + music-metadata 解析 |

所有文件已同步到 dist 部署版，重启应用（`pkill Mineradio` 后重新打开）即生效。

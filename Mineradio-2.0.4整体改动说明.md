# Mineradio 2.0.4 — 整体改动说明

> 版本：2.0.4（基于 2.0.3 复制）
> 日期：2026-07-31
> 状态：✅ 最终版（git 标签 `v2.0.4-final`，工作区干净）
> 部署：源码版 + dist 部署版均已同步

---

## 一、改动总览

本次 2.0.4 相对 2.0.3 的两大改动模块：

1. **本地歌词识别**（2.0.3 基础上新增）
2. **酷狗音乐概念版全流程适配**（核心大改，登录/VIP/歌单/播放/搜索/歌词全部切到概念版）

涉及文件：**10 个源码文件 + 1 个新增模块 + kugou-server 补全**，git 共 12 个提交。

---

## 二、改动一：本地歌词识别

### 背景

原版（2.0.3/2.0.4）播放本地音乐文件时**完全没有歌词**：
- 导入时只取文件名，不解析音频元数据（ID3/FLAC 内嵌歌词）、不读同目录 .lrc
- 歌词获取链路 3 处逻辑**显式排除 local 类型**（预取/翻译回退/启动重试）
- 本地歌曲没有 id/hash，落回网易云空参接口 → 永远拿不到歌词

### 解决方案（A + B 双通道）

```
优先级：
1. 同目录同名 .lrc 文件          ← 方案 B（最准）
2. 音频内嵌歌词标签              ← 方案 B（ID3 USLT / FLAC LYRICS / M4A，含同步歌词）
3. 在线搜索匹配（酷狗→网易云）   ← 方案 A（保底）
4. 标题占位（fallback）          ← 原逻辑保留
```

### 具体改动

#### 1. `public/js/modules/06-lyrics/00-lyrics-fetch-parse.js`（核心）

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

**新增函数**：

| 函数 | 作用 |
|---|---|
| `isLocalSongObject(song)` | 判断本地歌曲（type/source/localKey/localUrl 任一命中） |
| `parseLocalSongTitleArtist(song)` | 从文件名解析歌名/歌手，支持"歌名 - 歌手"与"歌手 - 歌名"双向候选 |
| `findLocalSongLyricCandidate(parsed, list)` | 匹配候选：标题/歌手双向变体精确匹配，`scoreSongSearchResult >= 28` 打分兜底 |
| `readLocalDiskLyric(song)` | 通过 IPC 读本地 .lrc / 内嵌标签（只认 `localDiskPath` 真实磁盘路径） |
| `fetchLocalSongLyric(song, token)` | 总入口：本地磁盘歌词优先 → 在线兜底（酷狗 → 网易云），带竞态保护 |

**改造 `fetchLyric()`**：local 歌曲走 `fetchLocalSongLyric`，其余保持原逻辑。

#### 2. `public/js/modules/06-lyrics/05-upload-dragdrop.js`

`localSongFromAudioFile()` 新增：用 Electron `webUtils.getPathForFile(file)` 获取拖拽/选择文件的**真实磁盘路径**，存入 `song.localDiskPath`（方案 B 读取 .lrc/内嵌标签的前提）。

#### 3. `desktop/preload.js`

新增 `getPathForFile`（webUtils）与 `readLocalLyric`（IPC）两个方法。

#### 4. `desktop/main.js`

新增 IPC handler `mineradio-read-local-lyric`：
1. **同目录同名 .lrc 优先**（`filePath.replace(音频扩展名, '.lrc')`，去 BOM）
2. 无 .lrc 时 `await import('music-metadata')` 解析内嵌歌词（`common.lyrics[].text` 非同步 / `syncText` 同步）
3. 均无 → `{ ok: false }`

### 本地歌词使用说明

1. 通过「导入音乐」按钮或拖拽导入本地文件（必须走导入流程才有磁盘路径）
2. .lrc 必须与音频**同目录同名**：`不为谁而作的歌.flac` ↔ `不为谁而作的歌.lrc`
3. 无 .lrc/内嵌歌词时自动在线匹配（需网络，酷狗优先）

---

## 三、改动二：酷狗音乐概念版全流程适配

### 背景

Mineradio 原版只支持**标准版酷狗**（网页登录抓 cookie + kugou-api.js 直连），不支持**酷狗音乐概念版**。用户需求：以 EchoMusic（开源项目，内嵌 KuGouMusicApi）为参考，把酷狗所有流程替换为概念版，保证丝滑登录、搜索和歌词适配。

### 架构

```
Mineradio 前端 (vanilla JS)
    ↓ fetch
server.js (3000, Electron 内嵌)
    ↓ 直接 require 调用（与 EchoMusic server.ts 同架构）
kugou-server/ = KuGouMusicApi（MakcRe/KuGouMusicApi，EchoMusic 同源仓库）
    ↓ platform=lite + 签名/设备注册
酷狗概念版服务器
```

**关键发现**：Mineradio 的 `kugou-server/` 就是 EchoMusic 内嵌的同一个 KuGouMusicApi 仓库（lite platform = 概念版），但原版**前端完全没用它**（只 spawn 了 9488 端口用于 QR 登录）。

### 根因与关键坑（踩坑记录）

| # | 坑 | 现象 | 修复 |
|---|---|---|---|
| 1 | 酷狗 2026-04 暗改：**搜索接口强制 cookie 认证** | 搜索返回 `error_code: 152 Parameter Error` | 请求带完整设备 cookie（dfid/mid/guid/dev/mac/webgl + token/userid） |
| 2 | `userid` 必须传**字符串 `'0'`** | 数字 0 被 `if (userid && userid !== 0)` 丢弃 → 签名参数缺失 → 152 | cookie 里 userid 用 `'0'` |
| 3 | kugou-server 全局 **apicache 2 分钟缓存** | 扫码确认后轮询仍返回 waiting（登录实际成功但 UI 无反应） | `/login/qr/check` 路由排除缓存 |
| 4 | 官方 `/v2/qrcode` 返回 `qrcode_img`（标准二维码） | 用 `/login/qr/create` 自拼 URL 二维码 App 无法识别关联 key | 优先使用官方 `qrcode_img` |
| 5 | 登录弹窗被 gate 竞态关闭 | 点「连接登录」后等几秒 drawer 自动消失 | `resumeLoginModalAfterGate` 检测到 kugou QR 活动时不关 drawer |
| 6 | 启动引导自动弹窗抢焦点 | 打开酷狗登录后自动跳到网易云 | `maybeRunStartupLoginGuide` 弹窗已打开时不触发 |
| 7 | **概念版无 Hi-Res/至臻**（最高 FLAC） | 切至臻后 20010 → 前端无降级逻辑 → 所有歌不可播 | kugou 菜单移除 hires，后端 hires/jymaster 自动降级 flac |
| 8 | 概念版返回**逗号分隔多地址** | 前端 `/api/audio?url=` 代理整串 → 404 → "当前歌曲不可播放" | 取第一个主地址 |
| 9 | 歌单接口返回 **data.songs**（非 plist/list） | 歌单详情空/加载失败 | 解析 `data.songs` + 字段映射 |
| 10 | 歌单含**收藏的他人歌单**（type=1/source=2） | 显示 6 个歌单但 App 只有 2 个 | 按 `list_create_userid === userid` 过滤 |
| 11 | 前端歌单详情**只认 `tracks` 字段** | 概念版返回 `songs` → 前端显示 0 首 | server 统一转 `{ tracks, total }` |
| 12 | VIP 在 **busi_vip 数组**里（每日畅听 SVIP） | 顶层 `is_vip:0` 误判非会员 | 解析 `busi_vip` 数组 |
| 13 | 概念版搜索歌名重复 | "林俊杰 - 不为谁而作的歌 - 林俊杰" | `OriSongName` 补到 `SongName`；歌单 name 拆分"歌手 - 歌名" |
| 14 | dist 部署版**缺 kugou-server 目录** | 部署版概念版功能不完整 | 复制 kugou-server（17M，含 node_modules）进 dist |

### 具体改动

#### 1. `kugou-lite.js`（**新增模块**，核心）

概念版内嵌调用层，与 EchoMusic `src/main/server.ts` 架构一致：

- `process.env.platform = 'lite'`（必须最先设置，util/index.js 加载时读取）
- **设备状态**：guid/mid/dev/mac/webgl 进程内生成一次（`cryptoMd5(getGuid())` + `calculateMid`）
- **dfid 注册**：`register_dev` 模块获取，带 Promise 缓存
- **`callModule(name, params, cookie)`**：按需 require kugou-server 模块 + createRequest
- **`buildLiteCookie(kugouCookie)`**：设备标识 + 从 cookie 提取 token/userid/dfid（兼容 `token=xxx;userid=xxx` 和 `KuGoo=...` 复合格式）

**业务封装**（全部概念版优先）：

| 函数 | 接口 | 说明 |
|---|---|---|
| `liteSearch` | `/search` | 概念版搜索 → `mapKugouSearchItem` 转换（字段兼容） |
| `liteLyric` | `/search/lyric` + `/lyric` | 歌词搜索 + base64 内容解码 |
| `liteSongUrl` | `/song/url` | 签名音源；音质映射 + 多地址取主 + 至臻降级 |
| `liteUserDetail` | `/user/detail` | 用户信息（昵称/头像） |
| `liteVipDetail` | `/user/vip/detail` | VIP 检测（busi_vip 数组） |
| `liteUserPlaylists` | `/user/playlist` | 歌单（data.info 解析 + 本人过滤） |
| `litePlaylistTracks` | `/playlist/track/all` | 歌单曲目（data.songs 解析 + 字段映射） |

#### 2. `server.js`

- require `kugouLite`（`const kugouLite = require('./kugou-lite')`）
- **搜索路由** `/api/kugou/search`：概念版优先，空/失败回退标准版
- **歌词路由** `/api/kugou/lyric`：概念版优先，空回退标准版
- **播放 URL 路由** `/api/kugou/song/url`：概念版优先；`vip_required` 直接返回不回退
- **登录状态路由** `/api/kugou/login/status`：概念版 user/detail + vip/detail 增强（昵称/头像/VIP）
- **歌单路由** `/api/kugou/user/playlists`：概念版优先，本人歌单过滤
- **歌单曲目路由** `/api/kugou/playlist/tracks`：概念版优先，统一转 `{ tracks, total }`，空歌单不回退
- 导出 `server.saveKugouCookie` / `server.getKugouCookie`（供 desktop 保存登录态）

#### 3. `desktop/main.js`

- **QR 创建** `kugou-music-qr-create`：优先使用官方 `/v2/qrcode` 返回的 `qrcode_img`（标准二维码），无则退回 `/login/qr/create`
- **QR 确认** `kugou-music-qr-check`：confirmed 后把 `token=xxx; userid=xxx` 写入 kugouCookie（`localServer.saveKugouCookie`），**登录态持久化**（重启不掉）

#### 4. `kugou-server/server.js`

缓存中间件排除 `/login/qr/check`（修复扫码确认被 apicache 掩盖）。

#### 5. `public/js/modules/00-state/00-core-stores.js`

kugou 音质菜单：移除 `hires`（Hi-Res/臻品），最高为「无损 FLAC」——概念版无 Hi-Res/至臻。

#### 6. `public/js/modules/08-account/03-login-modal-flows.js`

- `updateLoginProviderUi`：`useWebPreview` 排除 kugou（避免 `.qr-shell.web-login-preview #qr-img { display:none }` 隐藏概念版二维码）
- `openKugouWebLogin`：先 `setLoginAuthDrawerOpen(true)`（点「连接登录」按钮时 drawer 未展开）
- `resumeLoginModalAfterGate`：kugou QR 活动时不关 drawer（gate 竞态）
- `checkQr`：kugou 轮询加 `[KugouQR-poll]` 日志

#### 7. `public/js/modules/08-account/05-startup-login-guide.js`

`maybeRunStartupLoginGuide`：弹窗已打开时不触发（定时器触发时二次检查），避免自动跳网易云。

#### 8. `dist/linux-unpacked/resources/app/`

补全 `kugou-server/` 目录（17M，含 node_modules），部署版概念版功能完整。

---

## 四、文件改动清单

| 文件 | 位置 | 改动类型 |
|---|---|---|
| `kugou-lite.js` | 根目录（新增） | 概念版内嵌调用层（296+ 行） |
| `server.js` | 根目录 + dist | 5 条路由接概念版 + saveKugouCookie 导出 |
| `desktop/main.js` | desktop + dist | QR 官方二维码 + 登录态持久化 |
| `desktop/preload.js` | desktop + dist | getPathForFile + readLocalLyric IPC |
| `kugou-server/server.js` | kugou-server + dist | QR check 排除缓存 |
| `public/js/modules/00-state/00-core-stores.js` | 前端 + dist | kugou 音质菜单（移除 hires） |
| `public/js/modules/06-lyrics/00-lyrics-fetch-parse.js` | 前端 + dist | 本地歌词：放开 3 处排除 + 5 个新函数 |
| `public/js/modules/06-lyrics/05-upload-dragdrop.js` | 前端 + dist | localDiskPath 保存 |
| `public/js/modules/08-account/03-login-modal-flows.js` | 前端 + dist | QR 显示 3 处修复 + 轮询日志 |
| `public/js/modules/08-account/05-startup-login-guide.js` | 前端 + dist | 启动引导不抢弹窗 |
| `dist/linux-unpacked/resources/app/kugou-server/` | dist（补全） | 概念版服务目录（17M） |

---

## 五、最终验证结果

| 项目 | 结果 |
|---|---|
| 登录 | ✅ 概念版扫码，登录态持久化（重启不掉） |
| 用户信息 | ✅ 发箍斗 + 头像 |
| VIP | ✅ 每日畅听 SVIP 正确显示 |
| 歌单 | ✅ 仅本人（默认收藏空 + 我喜欢 10 首），详情正常 |
| 播放 | ✅ VIP 歌可听；标准/320/无损 FLAC 全通；至臻自动降级 |
| 搜索 | ✅ 概念版优先，歌名/歌手正确 |
| 歌词 | ✅ 概念版 LRC 完整 |
| 本地歌词 | ✅ .lrc / 内嵌标签 / 在线匹配 |
| 部署版 | ✅ dist 独立启动验证通过（3000 + 9488 + 登录态） |

---

## 六、已知限制

- 酷狗概念版最高音质为**无损 FLAC**（无 Hi-Res/至臻/母带，属概念版本身限制）
- 每日畅听 SVIP 为概念版免费领取的会员权益，覆盖到 FLAC 音质
- 在线歌词/播放依赖网络；搜索接口强制 cookie 认证（概念版签名已处理）
- 本地歌词的 `localDiskPath` 仅桌面端可用；纯浏览器访问无磁盘路径只能在线匹配

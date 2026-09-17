# Mineradio — Linux 适配版

**这是 [XxHuberrr/Mineradio](https://github.com/XxHuberrr/Mineradio) 的 Linux 适配版本**（上游原版是 Windows 桌面应用），由 [YannZhou](https://github.com/YannZhou) 维护。

本仓库不改动上游的界面与视觉设计，只做"让它在 Linux 上装完即用"这件事，并补齐上游尚未支持的**酷狗音乐概念版**链路与**本地歌词识别**。想了解软件本身的能力，请看上游仓库；本页只说明适配版做了什么、怎么装。

## 适配内容

- **Linux 运行适配**：显卡渲染走 OpenGL，内存整理支持 Linux（含系统级内存回收），禁用鼠标侧键防误触
- **deb 打包**：一键安装到 `/opt/Mineradio`，桌面项自带必要的启动参数，X11 下开箱可用
- **酷狗音乐概念版**：扫码登录（进程内完成，不额外起进程）、设备身份持久化、听歌自动领取每日 VIP、听歌奖励自动上报、登录面板可一键刷新会员状态
- **本地歌词识别**：优先读取同目录同名 `.lrc`，其次读取音频内嵌歌词，最后按歌名与歌手在线兜底匹配
- 移除了对本项目无意义的 Windows 分发内容，依赖声明补全，克隆后可直接构建

## 安装

下载 Releases 中的 deb 包安装即可：

```bash
sudo dpkg -i mineradio_*_amd64.deb
# 如提示缺少依赖：
sudo apt-get -f install
```

升级安装直接覆盖，登录状态会保留。

## 从源码构建

```bash
git clone https://github.com/YannZhou/Mineradio.git
cd Mineradio
npm install --include=dev
node node_modules/electron/install.js   # 若 npm 因 allow-scripts 白名单未下载 Electron
npm start                                # 开发运行
npm run build:linux                      # 打包 deb 到 dist/
```

> 本机若设置了 `ELECTRON_RUN_AS_NODE`，运行 Electron 前需要先 `unset`，否则会被当成纯 Node 执行。

## 版本详情

各版本的改动、验证情况与已知限制，按版本归档在 [docs/versions/](./docs/versions/)（每个版本一个文件）。

## 第三方音乐平台说明

Mineradio 不是网易云音乐、QQ 音乐或腾讯音乐娱乐集团的官方客户端，也不隶属于任何音乐平台。

项目中的第三方平台接入仅用于个人学习、本地客户端体验和用户自有账号的播放辅助。请遵守对应平台的用户协议、版权规则和会员权益规则。项目不会提供绕过付费、绕过会员、破解音质或重新分发音乐内容的能力。

## 用户数据与隐私

登录 Cookie、搜索历史、自定义封面、自定义歌词、节奏分析缓存等数据只应保存在本机用户数据目录或浏览器本地存储中，不应提交到仓库。

更多说明见 [PRIVACY.md](./PRIVACY.md)。

## 致谢

Mineradio 由 XxHuberrr 主要设计与打造。emily 作为早期视觉底层想法与 `emily` 视觉预设改进方向的共创者和灵感来源之一，特此感谢。

同时感谢小天才e宝、应春日、锋将军、軌跡、林中、骊、风痕、花椰菜🥦在早期体验、测试反馈和发布准备中的帮助。

## 版权与授权

Copyright (C) 2026 XxHuberrr.

本项目采用 GPL-3.0 授权。详见 [LICENSE](./LICENSE)。

MR Logo、Mineradio 名称、界面视觉设计与原创视觉表达归作者所有；第三方依赖和第三方服务分别遵循其各自授权与服务条款。

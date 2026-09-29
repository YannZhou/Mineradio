# 发布流程（Linux 适配版）

本仓库是上游 `XxHuberrr/Mineradio` 的 Linux 适配版，发布方式与上游的 Windows 发布流程不同：
**只产出一个 deb 安装包**，Release 页只放版本号与一句版本说明引导，具体说明一律写进
`docs/versions/<版本>.md`。

## 分支与标签

- `adapt/<版本>` = 发布分支，与 `main` 指向同一个 commit（默认分支即 `main`）。
- 标签用 `v<版本>-linux`（例 `v2.2.1-linux`），**不要**使用上游同名标签（如 `v2.2.0`）。
- 上游已停更：基线不变时启用补丁位（`2.2.x`），版本号规则见 `docs/versions/README.md`。

## 每次发布要同步版本号的三处

1. `package.json` 的 `version`（deb 包名与 `server.js` 的 `APP_VERSION` 都跟它走）；
2. `public/index.html` 里 `id="update-modal-version"` 的显示串；
3. `tests/update-external-only.test.js` 的两处版本断言。

## 产物

- `npm run build:linux` → `dist/mineradio_<版本>_amd64.deb`（安装到 `/opt/Mineradio`）。
- Release **只上传该 deb**。构建附带的 `latest-linux.yml`、blockmap、`builder-debug.yml`
  仅留在本地作为验收记录，不作为 Release 资产发布。
- 软件内更新检测指向的是上游仓库，本仓库的 Release 不参与软件内更新。

## Release 文案

- **标题 = 纯版本号**（如 `v2.2.1-linux`），不带描述。
- **正文 = 一行**：只写 `版本说明见 [docs/versions/<版本>.md](链接)` 的引导。
  标题已经显示版本号，正文**不再重复版本号**（旧写法在标题下再加一行 `# <版本号>`，页面上会出现两个版本号）。
- 安装命令、功能清单、已知限制等全部写在版本文件里，正文不重复。

## 发布前检查

- 工作区干净：无未跟踪文件、无 `.bak` / `.orig` / `.rej` 残留。
- 本轮改动的 JS 逐个 `node --check` 通过。
- 测试不止看"有没有红"：`node --test tests/*.test.js` 的失败集合必须与上一版发布提交一致
  （本仓库基线本就有既存失败，需做基线对比才能判断是否引入回归）。
- 打包后无头启动复验：`MINERADIO_STARTUP_QA_HIDDEN=1` + `xvfb-run` +
  **`env -u ELECTRON_RUN_AS_NODE`**（缺这一项 Electron 会以 Node 模式启动，只报 `bad option`）。
  确认产物可启动、`:3000` 就绪、未登录态下的降级路径不报错。
- `sudo dpkg -i` 覆盖安装后正常启动，核对登录态、会员档位、播放取址、搜索与歌词等接口。
- 记录产物大小与 SHA-256，并在 Release 回读：资产状态 `uploaded`、云端摘要与本地一致、直链可下载。
- 确认仓库与安装包内不含 Cookie、Token、凭据、缓存或本机日志。

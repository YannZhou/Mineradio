# Mineradio 2.0.4 — Git 工作流说明

> 建立时间：2026-07-31
> 仓库位置：`/home/yan/桌面/hermes工作台/软件/mineradio/2.0.4/`
> 适用场景：后续改动较大，需要随时能精确回退

---

## 一、仓库概况

| 项 | 值 |
|---|---|
| 分支 | `main` |
| 首次提交 | `ddf05aa` chore: 初始化 2.0.4 仓库（2.0.3 复制）+ 本地歌词识别功能 |
| 收录 | 436 个文件 / 13M（完整源码 + 顶层文档 + kugou-server） |
| 用户身份 | `YannZhou <3609286195@qq.com>`（仓库级 config，仅本项目生效） |

---

## 二、版本库策略（.gitignore）

| 排除内容 | 原因 |
|---|---|
| `node_modules/`（含 dist 内、kugou-server 内的） | 可 `npm install` 重装，省 ~470M |
| `dist/` 构建产物、`*.deb` | 可重新构建 |
| `Mineradio-2.0.3.zip` | 原始压缩包，与版本无关 |
| `kugou-server/package-lock.json` | 第三方库锁文件 |

**收录但做过处理的**：
- `kugou-server/`：第三方 KuGouMusicApi 原本自带 `.git`，已删除内嵌仓库，作为**普通文件**纳入主仓库——这样它的改动也能用 git 回退（注意：`git rm --cached -f` 清掉 gitlink 索引条目后重新 `add -A` 才生效）。

**不追踪的文件不要手动 git add**——`git add -A` 会自动尊重 .gitignore。

---

## 三、日常工作流

### 1. 每次改代码前（可选但推荐）

```bash
cd "/home/yan/桌面/hermes工作台/软件/mineradio/2.0.4"
git status          # 确认工作区干净，从已知状态出发
```

### 2. 改完一个阶段 → 提交一个节点

```bash
git add -A
git commit -m "feat: 说明这次改了什么"
```

建议粒度：**每个功能/修复一个提交**，提交信息写清楚"做了什么 + 为什么"（中英文均可）。

### 3. 查看状态

```bash
git status          # 哪些文件改了
git diff            # 未暂存的改动内容
git diff --cached   # 已暂存未提交的内容
git log --oneline   # 提交历史（一行一条）
```

---

## 四、回退操作（核心场景）

| 想做的事 | 命令 |
|---|---|
| 丢弃某个文件的工作区改动 | `git checkout -- 文件名` 或 `git restore 文件名` |
| 丢弃所有未提交改动 | `git reset --hard` |
| 回到上一个提交（丢弃最后一次提交） | `git reset --hard HEAD~1` |
| 回到任意历史提交 | `git checkout <提交号>`（临时查看），确认后 `git reset --hard <提交号>` |
| 撤销某次提交但保留改动到工作区 | `git reset --soft HEAD~1` |
| 只撤销已暂存（不丢文件） | `git reset`（unstage 全部） |

**安全提醒**：
- `git reset --hard` 会**永久删除**未提交的改动，执行前先 `git status` 确认没有想留的东西。
- 不确定时：先 `git stash`（暂存改动）再操作，随时 `git stash pop` 找回。

---

## 五、分支使用（改动大时推荐）

大改动（比如重构、多文件重写）建议开分支，主线保持稳定：

```bash
git checkout -b feature/xxx   # 开新分支并切换
# ... 在分支上随便改、随便提交 ...
git checkout main             # 回主线
git merge feature/xxx         # 确认没问题后合回
```

或者更简单的"草稿节点"法：不删提交，直接在 main 上提交，用 `git log` 记住每个节点，回退时 `git reset --hard <提交号>` 精确跳转。

---

## 六、关于 dist 部署版

- `dist/` 构建产物**不纳入** git（体积大、可重建）。
- 部署版（`dist/linux-unpacked/resources/app/`）的改动靠**手动同步**源码文件（改完源码 `cp` 过去）。
- 回退源码后，记得**重新同步部署版**，否则运行的是旧代码。

---

## 七、快速参考（命令速查）

```bash
# 提交
git add -A && git commit -m "说明"

# 查看
git status / git diff / git log --oneline

# 回退
git restore 文件名          # 单文件
git reset --hard            # 丢弃未提交
git reset --hard HEAD~1     # 回到上个提交
git stash / git stash pop   # 临时保存/恢复

# 分支
git checkout -b 分支名
git checkout main
git merge 分支名
```

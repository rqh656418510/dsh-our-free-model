# 同步上游：fork 日常维护手册

本仓库是 [`Ebony-Vinyl/dsh-our-free-model`](https://github.com/Ebony-Vinyl/dsh-our-free-model) 的 fork，
在 `no-ads` 分支上去掉了广告入口与全屏公告弹窗，其余功能与上游保持一致。

这份文档只讲一件事：**怎么在不丢自己改动的前提下，持续吸收上游更新。**

> **TL;DR（每个上游版本跑一遍）**
>
> ```bash
> git fetch upstream --tags
> git checkout main && git merge --ff-only upstream/main && git push origin main
> git checkout no-ads && git merge main        # 有冲突 → 按第 5 节解决，别急着提交
> npm run build:standalone-ui                  # 用合并后的源码重建控制台产物
> git add -A -- packages/standalone/web
> git commit -m "chore: 同步上游并重建控制台产物"
> node scripts/build-standalone-ui.mjs --check && node scripts/client-lint.mjs
> git push origin no-ads
> ```
>
> - 如果第 3 行已经自动合并成功，第 6 行就是一次普通提交；如果停在冲突里，第 6 行同时**完成这次合并**。
> - 没冲突时重建也常会改动 `app.js` / `assets.json`（上游改了 `frontend/**`），所以这一步不能省；工作区真的没有变化时 `git commit` 会提示 nothing to commit，跳过即可。

> 命令按 **Git Bash** 书写（`\` 续行）。PowerShell 下请把续行写成一整行，`printf`/`grep` 换成 `Add-Content`/`Select-String`。

---

## 1. 仓库拓扑

| 名字 | 地址 | 角色 |
| --- | --- | --- |
| `origin` | `git@github.com:rqh656418510/dsh-our-free-model.git` | 你自己的 fork，只推这里 |
| `upstream` | `https://gh-proxy.com/https://github.com/Ebony-Vinyl/dsh-our-free-model.git`（GitHub 代理镜像，等价于 `https://github.com/Ebony-Vinyl/dsh-our-free-model.git`） | 上游，只拉不推 |

| 分支 | 用途 | 规则 |
| --- | --- | --- |
| `main` | 上游镜像 | **永远只做快进合并，不在这上面写提交** |
| `no-ads` | 我们的工作分支 | 去广告改动只落在这里；每个上游版本以一次 merge 汇入 |

```
upstream/main ──(ff-only)──> origin/main
                                  │
                                  └──(merge)──> origin/no-ads
```

`main` 保持"纯镜像"是这套流程能长期跑下去的前提：它永远不会冲突，也永远能用 `git diff main upstream/main` 一眼看出偏差。

---

## 2. 一次性配置（已配置好，重装环境时照此重建）

```bash
# 如果 origin 还指向上游，先改名，再把 fork 加回来
git remote rename origin upstream
git remote add origin git@github.com:rqh656418510/dsh-our-free-model.git

# 让 main 跟随 fork 的 main（同步时显式使用 upstream/main，见第 3 节）
git push -u origin main
git push -u origin no-ads
git push origin --tags

# 建议开启：冲突解决方案自动重放，同样的冲突只解一次
git config rerere.enabled true
git config rerere.autoupdate true
```

> **不要** `git init` 重建仓库。只有保留上游历史，`merge upstream/main` 才是普通合并而非"两棵无关历史"。
> 若代理镜像失效，把上游地址换成直连即可：`git remote set-url upstream https://github.com/Ebony-Vinyl/dsh-our-free-model.git`。

---

## 3. 日常同步（每个上游版本执行一次）

```bash
# 1) 取上游（含 tag）
git fetch upstream --tags

# 2) main 只做快进，然后推回 fork
git checkout main
git merge --ff-only upstream/main
git push origin main

# 3) 把改动分支跟上来
git checkout no-ads
git merge main

# 4) 解冲突（见第 5 节）→ 校验（见第 4 节）→ 推送
git push origin no-ads
```

如果 `merge --ff-only` 报错，说明 `main` 上被写下过提交，先查清楚：

```bash
git log --oneline main ^upstream/main    # 应输出空；有输出就是 main 被污染了
```

`no-ads` 只有自己用，想保持线性也可以改成 `git rebase main`，但推之前要 `--force-with-lease`；用 merge 更省心，推荐保持 merge。

---

## 4. 合并后必做的校验

```bash
node scripts/build-standalone-ui.mjs --check   # 控制台产物与源码一致
node scripts/client-lint.mjs                   # 文案字典、ofm_* 样式配平
node scripts/standalone-frontend-test.mjs      # assets.json 与 web/ 目录一致
grep -nE 'SponsorBanner|ofm_sponsor|SPONSOR_CARD_URL|showUrgentModal|urgentShownIds' client.js
```

最后一条**必须没有输出**——它有输出就说明上游把广告代码合回来了（自动合并不报冲突，但内容可能回来）。
PowerShell 下等价写法：`Select-String -Path client.js -Pattern 'SponsorBanner|ofm_sponsor|SPONSOR_CARD_URL|showUrgentModal|urgentShownIds'`（应为空）。

另外 `node scripts/build-manifest.mjs --check` 会一直报 `client.js` 与清单不一致，这是**预期状态**（见第 7 节），不是合并出错。

---

## 5. 冲突处理手册

### 5.1 `packages/standalone/web/*`（生成物冲突，最常见）

两边的控制台产物文件名带内容哈希，只要两边都构建过，`app.js` / `app.css` / `assets.json` 必然冲突。
**不要手工合并、不要粘压缩代码**，按下面三步走：

```bash
# 1) 三个"总目录"文件先取自己这一侧，清掉冲突标记（内容马上会被重建覆盖）
git checkout --ours -- packages/standalone/web/app.css \
                      packages/standalone/web/app.js \
                      packages/standalone/web/assets.json
git add -- packages/standalone/web/app.css packages/standalone/web/app.js packages/standalone/web/assets.json

# 2) 用"合并后的源码"重新构建（会生成新的哈希文件并自动清理旧文件）
npm run build:standalone-ui

# 3) 把重建结果并入这次合并
git add -A -- packages/standalone/web
node scripts/build-standalone-ui.mjs --check   # 确认一致
git commit --no-edit
```

重建会把上游对 `frontend/**` 的改动（例如 `models.tsx`）和我们对 `client.js` 的改动同时吸收，产物自然只剩一套。

### 5.2 我们删掉的文件被上游修改（modify/delete 冲突）

上游改到了被我们删除的广告文件时，git 会报 `deleted by us`。**保留删除**，不要 "accept theirs"：

```bash
git rm src/sponsor.js src/sponsor.css \
       packages/standalone/frontend/sponsor.tsx \
       scripts/build-sponsor.mjs scripts/sponsor-test.mjs scripts/sponsor-ui-test.mjs \
       docs/sponsor-advertisement.md
git add -A
```

### 5.3 `client.js` 文本冲突

`client.js` 同时承载两件事：插件浏览器半边（我们改过）+ 上游的新功能。冲突时：

1. 先解成"能跑"的状态（保留上游新增逻辑、保留我们的删除）；
2. 用第 4 节的 `grep` 自检广告代码没回来；
3. `node scripts/client-lint.mjs` 必须通过——它会检查文案键与 `ofm_*` 样式是否配平，删多删少都会报错。

### 5.4 README / 文档冲突

保留我们的措辞（广告与 urgent 弹窗行为已改），只吸收上游新增的章节。

### 5.5 想放弃这次合并

```bash
git merge --abort
```

合并已完成但想重来：

```bash
git reset --hard HEAD~1        # 只在自己的工作分支上用
```

### 5.6 生成物被"本地运行中的实例"改写（不是冲突，但很坑）

如果 DSH 或独立服务正跑着这个目录的代码，它可能在启动/热重载时**按 `vendor/channel-pack/pack.js` 重新生成** `packages/standalone/channels/business.mjs`。而上游仓库里 `pack.js` 目前落后于 `vendor/channel-pack/src/`（`src/qoder-wasm.ts` 的内存回收修复还没进 `pack.js`），于是重新生成会把已经修好的产物"降级"。

判断方法：`git status` 里出现你没动过的 `packages/standalone/channels/business.mjs`，`git diff` 里是文件头 `原渠道包 SHA256` 变了 + 删掉若干修复代码。

处理：

```bash
git restore -- packages/standalone/channels/business.mjs   # 以提交里的版本为准
```

提交前先 `git status` 扫一眼；需要长时间折腾仓库时，最好先停掉本地在跑的 DSH / 独立服务。

---

## 6. 可选：让生成物冲突彻底消失

```bash
printf 'packages/standalone/web/** merge=ours\n' >> .gitattributes
git config merge.ours.driver true
```

之后合并 `packages/standalone/web/**` 会静默保留自己这一侧，不再停在冲突里；**但必须记得合并后跑一次 `npm run build:standalone-ui`**，否则会留着过期产物。
注意这是本机 git 配置，别人 clone 后没有这条配置；不要把它当成"CI 也会这样"。

---

## 7. 版本、清单与自动升级（最容易被覆盖的地方）

- 插件内"检查更新 / 立即升级"读的是**上游签名的** `feed/manifest.json`，装下来的是**带广告的官方 `client.js`**。
  上游一发新版本、点一次升级，`no-ads` 的改动就被整体覆盖。
- 因此：**日常不要点插件里的升级**；要更新就跟上游合并（第 3 节），不要用应用内升级。
- 合并后 `build-manifest --check` 报 `client.js: size … in the manifest, … on disk` 属正常——清单由维护者私钥签名，我们改不了也不该伪造。
- 真要用应用内升级/自己发版，三件事缺一不可：
  1. `package.json` 版本抬到**高于**上游；
  2. `node scripts/build-manifest.mjs --key <你的 Ed25519 私钥>` 重签清单；
  3. 把 `src/updater.js` 里 `PINNED_MANIFEST_PUBLIC_KEY` 换成你自己的公钥（否则插件拒绝安装你签的包）。
- 想连公告/升级源都走自己的仓库，改 `src/feed.js`、`src/updater.js` 里的 `REPO` 常量，或把插件设置里的 `feedUrl` 指向你的仓库。

---

## 8. 不要做的事

- ❌ `git init` 或重新 clone 后手工搬文件——会丢失与上游的共同历史；
- ❌ 直接往 `main` 提交——镜像分支一旦有本地提交，`--ff-only` 永远失败；
- ❌ `git push --all` / `git push --mirror`——会把上游几十个分支和 refs 一起推上去；
- ❌ 手工编辑 `packages/standalone/web/*`——只能由 `npm run build:standalone-ui` 生成；
- ❌ 提交 `node_modules/`、`.verify/`、`packages/desktop/src-tauri/.node-license-cache/`；
- ❌ 无视 `git status` 里莫名其妙出现的 `packages/standalone/channels/business.mjs`（见 5.6，是本地运行中的实例重建的，直接还原）；
- ❌ 在 `no-ads` 上 `git push --force`（除非你确认只有自己用，并加 `--force-with-lease`）。

---

## 9. 我们相对上游改了什么（冲突时对照用）

**删除**
- `src/sponsor.js`、`src/sponsor.css`
- `packages/standalone/frontend/sponsor.tsx`
- `scripts/build-sponsor.mjs`、`scripts/sponsor-test.mjs`、`scripts/sponsor-ui-test.mjs`
- `docs/sponsor-advertisement.md`、`docs/images/sponsor-beichen-examples.jpg`、`docs/images/sponsor-ui-preview.jpg`

**修改**
- `client.js`：去掉「北辰 API 中转站」横幅（组件/常量/中英文案/样式）；去掉 urgent 公告的全屏弹层（`showUrgentModal`、启动补弹、到达到达弹层），urgent 改为红色 toast；连带的文案键与 `.ofm_holdhint` 样式一并删除
- `packages/standalone/frontend/app.tsx`、`components/shell.tsx`、`system/shell.css`：去掉侧边栏「来自赞助商的广告」菜单、`sponsor` 路由与每日弹窗挂载
- `scripts/build-standalone-ui.mjs`、`scripts/test-all.mjs`：移除广告相关构建/套件登记
- `scripts/*-ui-test.mjs`、`scripts/*-e2e*`：去掉"等广告弹窗再关"的兜底逻辑
- `src/feed.js`：`holdSeconds` 注释更新（字段仍归一化，客户端不再弹层）
- `README.md`、`README_EN.md`：公告行为描述改为"标红 toast、不弹窗"

---

## 10. 当前状态快照

- 记录时间：2026-11-25
- 上游基线：`8f3182b`（`Merge pull request #926 …`）
- `no-ads` 最新：`a1f80d8`（`Merge branch 'main' into no-ads`，已吸收上游上述基线）
- 版本：`package.json` 仍为 `2.9.0`（与上游同版本，故不会触发应用内升级）
- 控制台产物：已由合并后的源码重建，`build-standalone-ui --check` 通过
- 待办：`git push origin no-ads`（合并提交尚未推送）

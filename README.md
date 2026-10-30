# dde

DDE 组版本基线。记录哪些仓库是一组，以及每组当前的版本号。

原先这里是 UOS 20 的 `dde` 元包，2023 年后没再更新，已经停用。旧打包文件留在 git 历史里。

## 规则

- 组内所有仓库的 `debian/changelog` 版本必须等于 `groups.yaml` 里该组的 `version`。
- 组里任一仓库有更新，整组一起升版本、一起打包。
- 依赖关系（`debian/control` 里的 `>=` / `>>`）不动，版本变大后下界仍然成立。

## 文件

- `groups.yaml`：分组、成员、当前版本。唯一事实来源，别处不另存一份。
- `.github/workflows/release-group.yml`：按组派发 changelog 更新。
- `.github/workflows/sync-baseline.yml`：让基线跟上实际版本。

## 发版

组内 changelog 只从这里发起，不要挨个进组件仓库改版本。

`Actions → Release group → Run workflow`，四个参数：

- `group`：要发的组，见 `groups.yaml`。
- `version`：目标版本，必须大于组内当前版本。
- `dry_run`：默认 `true`，只打印成员列表。
- `create_pr`：默认 `true`，在成员仓库建 PR。

它做的是：读 `groups.yaml` 拿到整组成员，逐个派发成员仓库的 changelog workflow。
本仓库不生成 changelog。

成员仓库有两种，派发时按 workflow 名字自动识别：

- `Call update changelog`（`linuxdeepin/.github` 的 PR #790）：G2 / G3 / G5 在用。
- `Auto Release`：G1（DTK）在用，走 `git-cliff` + `dch --newversion`。

两种都没有的仓库会被跳过并打 warning。另外要配 secret `RELEASE_TOKEN`，
对 `linuxdeepin` 下的成员仓库有 `actions: write`。

## 基线跟随

PR 合并后不用手工改 `groups.yaml`。`Sync baseline` 每天跑一次，也可以手动触发：

- 全组版本一致且高于基线：开一个 `baseline/<组>-<版本>` 的 PR 更新 `groups.yaml`。
- 组内版本不一致：只打 warning，基线不动。
- 一致但等于基线：不动。

基线只前进，不回退。用 PR 而不是直接提交，是为了在分支保护下也能工作，
每次变更都有记录；代价是合并要人点一下。

## 组件侧校验

成员和版本以 `groups.yaml` 为准，这里不重复列举。

打算做成 `linuxdeepin/.github` 的复用 workflow（`verify-group-version.yml`），
通过 `workflow-templates/call-*` 引入，不用每个组件手写脚本。现在还没做，
在此之前组内版本一致只能靠约定和 `sync-baseline` 的事后检查，PR 阶段拦不住。

## 分组

| 组 | 说明 | 成员 |
|---|---|---|
| G1 | DTK 基础库 | dtkcommon、dtk6log、dtklog、dtkcore、dtkgui、dtkwidget、dtkdeclarative、qt5platform-plugins、qt5integration |
| G2 | Go 服务基座 | go-lib、go-dbus-factory、dde-api、deepin-pw-check、dde-app-services |
| G3 | 会话核心与桌面应用 | dde-daemon、dde-services、dde-session、dde-session-shell-v25、dde-polkit-agent、dde-session-ui、dde-shell、dde-tray-loader、dde-application-manager、deepin-service-manager、dde-control-center、dde-launchpad、dde-clipboard、dde-network-core、deepin-update-ui、dcc-insider-plugin、xdg-desktop-portal-dde |
| G5 | 外观与主题 | dde-appearance、deepin-desktop-theme、deepin-face、deepin-screensaver |
| G5-icons | 图标主题 | deepin-icon-theme |

不参与统一：dde-dialog、dde-manual-content、gnome-keyring。

划分按耦合强度：循环依赖必须同版本；带版本约束的单向依赖，一方升级另一方要跟发；
只有构建期依赖的算源码级 ABI 耦合；没有内部依赖的不参与。各组的具体理由见组件仓库里的依赖图文档。

成员和版本以 `groups.yaml` 为准，这里只是概览。

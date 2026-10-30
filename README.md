# dde

DDE 组版本基线。定义「哪些仓库是一组」和「这一组当前的统一版本」。

## 规则

1. 组内所有仓库的 `debian/changelog` 版本必须等于 `groups.yaml` 里该组的 `version`。
2. 组内任一仓库有更新，整组一起升版本、一起重新打包。
3. 依赖关系（`debian/control` 里的 `>=` / `>>`）**不因此修改**——版本变大后下界依然成立。

## 文件

`groups.yaml` 是唯一事实来源。改版本只能改这里 + 各成员仓库的 changelog，不另设副本。

## 唯一升 changelog 入口

**这一组的 changelog 只能从本仓库发起**，不手工进各组件仓库改版本。

`Actions → Release group → Run workflow`：

| 参数 | 说明 |
|---|---|
| `group` | 要发的组（见 `groups.yaml` 的 `groups` 键） |
| `version` | 目标版本，必须大于组内当前版本 |
| `dry_run` | 默认 true，只打印成员列表 |
| `create_pr` | 默认 true，在成员仓库开 PR |

流程：读 `groups.yaml` 拿到整组成员 → 对每个成员派发 `call-update-changelog.yml`
→ 由 `linuxdeepin/.github` 的复用 workflow（PR #790）用 `dch` 生成条目并开 PR。

本仓库不实现 changelog 生成逻辑，只做派发。

需要的 secret：`RELEASE_TOKEN`，对 `linuxdeepin` 下成员仓库有 `actions: write`。

### 前提

成员仓库必须已启用 `Call update changelog`（由 `.github` 的 templates-sync 分发）。
未启用的会被跳过并输出 warning。

### PR 合并之后：基线自动跟随

`Sync baseline` workflow 每天跑一次（也可手动触发）：

| 实际情况 | 动作 |
|---|---|
| 全组版本一致，且高于基线 | 自动更新 `groups.yaml` 并提交 |
| 组内版本不一致 | 只输出 warning，不动基线（说明有成员还没合） |
| 一致但等于基线 | 无变化 |

基线只前进、不回退。发版 PR 合并后不需要手工改 `groups.yaml`。

## 组件侧校验

成员与版本以 `groups.yaml` 为准，本文档不再重复。

校验计划做成 `linuxdeepin/.github` 的可复用 workflow（`verify-group-version.yml`），
组件仓库通过 `workflow-templates/call-*` 引用，不在每个组件里手写脚本。目前尚未实现。

## 分组依据

按循环依赖（必须同版本）、带版本约束的单向依赖、仅构建期依赖、无内部依赖四档判定，
详见组件仓库中的依赖图文档。

# repo-mirror

把任意一个 GitHub 仓库**原样搬到自己的独立仓库**，每 30 分钟自动同步一次。

只做一件事：源码、tag、Release（含附件、prerelease 标记、更新说明）全搬过来，客户端照旧能按原来的 URL 结构拿到更新。

## 为什么不用 Fork

fork 会进入**同一个 fork network**。上游仓库一旦被 DMCA 处理，所有 fork 会被一起处理掉——你以为的备份，其实是同一根绳上的蚂蚱。

这个模板建出来的是**完全独立的仓库**，和上游不共享任何网络关系，靠定时任务单向拉取。

> 说清楚边界：这只解决「被上游牵连」的问题，不解决「你自己侵权」。搬别人的东西之前先看对方的许可证和 GitHub 的服务条款，责任在部署的人。

## 部署（3 步，约 1 分钟）

1. 点右上角 **Use this template → Create a new repository**，建成你自己的仓库（建议 Public，原因见下面「配额」）。
2. 新仓库 → **Settings → Secrets and variables → Actions → Variables → New repository variable**
   新建一个变量：名字 `UPSTREAM`，值填要搬的仓库，形如 `owner/repo`。
3. 新仓库 → **Actions** 标签页，如果提示 workflow 被禁用就点 **I understand my workflows, go ahead and enable them**；
   然后手动跑一次 **mirror → Run workflow**，把 `release_limit` 填 `0` 先试源码和 tag（快），没问题再不填跑全量。

之后就不用管了，每 30 分钟自己拉一次。

## 配置项

全部是**仓库变量**（Settings → Variables），除了 `UPSTREAM_TOKEN` 是 **Secret**。

| 名字 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `UPSTREAM` | 变量 | 无（必填） | 上游仓库，`owner/repo` |
| `UPSTREAM_BRANCH` | 变量 | `main` | 上游的默认分支，老仓库可能是 `master` |
| `TARGET_BRANCH` | 变量 | `main` | 本仓库接收同步的分支 |
| `RELEASE_LIMIT` | 变量 | `6` | 同步最近多少个 Release；`0` = 只同步源码和 tag |
| `TAG_PATTERN` | 变量 | `*` | 只同步匹配的 tag，例如 `v*` 可以跳过一堆无关的 tag |
| `SYNC_TAGS` | 变量 | 开启 | 设成 `false` 完全跳过 tag 同步 |
| `KEEPALIVE` | 变量 | 开启 | 设成 `false` 关掉保活提交 |
| `UPSTREAM_TOKEN` | Secret | 空 | 上游是私有仓库，或者想避开匿名 API 限流时填一个 PAT（只读就够） |
| `SYNC_TOKEN` | Secret | 空 | 推 tag 用的 PAT，需要 `workflow` 权限。见下面「tag 为什么没同步」 |

手动运行时的两个输入 `upstream` / `release_limit` 只对这一次生效，用来临时试别的仓库。

## 几个必须知道的行为

**同步范围**：源码（`rsync --delete`，上游删的文件这里也删）、所有匹配 `TAG_PATTERN` 的 tag（`--force`，上游重打 tag 这里跟着改）、最近 `RELEASE_LIMIT` 个 Release。

**`.github` 目录永远不同步**。否则本仓库的 workflow 会被上游内容覆盖，下一次就没人来同步了。这是这类镜像最容易踩的坑。

**Release 是「只增不改」**。已经存在的 tag 直接跳过，不会更新说明或替换附件。上游改了旧 Release 的内容，这里不会跟着变——要重来只能手动删掉那个 Release 再等下一轮。

**`prerelease` 标记会跟着搬**。很多更新器（包括 PCL 系列）靠这个标记决定要不要推给稳定版用户，搬错了会让人收到预览版。

**时间不保证准**。GitHub 的 `schedule` 在高峰期普遍延迟几分钟到十几分钟，实际间隔是 30~45 分钟。要精确只能自己拿外部定时器打 `workflow_dispatch`。

**60 天会被停**。仓库连续 60 天没有任何提交时，GitHub 会自动停掉定时任务。这个工作流带了保活：最近一次提交超过 50 天就补一个 `.mirror-keepalive` 提交把计时器按回去。

## 配额

- **公开仓库**：GitHub Actions 分钟数不计量，随便跑。48 次/天 × 每次 1~3 分钟没压力。**建议用公开仓库。**
- **私有仓库**：每月 2000 分钟免费额度。每次同步按 1~3 分钟算，一个月 1400~4300 分钟，**很可能超额**。私有仓库要么把 cron 调稀，要么别用这个方案。
- Release 附件会占本仓库的存储和流量，大文件仓库（几百 MB 的安装包）同步前先掂量一下。

## 改同步频率

编辑 `.github/workflows/sync.yml`：

```yaml
on:
  schedule:
    - cron: "17,47 * * * *"   # 每 30 分钟，错开整点和半点
```

cron 是 **UTC**。常用写法：

| 频率 | 写法 |
| --- | --- |
| 每 30 分钟 | `17,47 * * * *` |
| 每小时 | `17 * * * *` |
| 每 2 小时 | `17 */2 * * *` |
| 每天一次（UTC 04:17 = 北京 12:17） | `17 4 * * *` |

注意 `timeout-minutes` 必须小于间隔，否则一次卡住的运行会占着并发组把后面几轮全堵住。

## 常见问题

**跑完仓库还是空的？** 看 Actions 日志里「检查配置」那一步，多半是 `UPSTREAM` 没设或写错。

**提示 403 / rate limit？** 配 `UPSTREAM_TOKEN`（一个只读 `public_repo` 的 PAT 就够）。

**Release 附件没下来？** 上游附件超过 2 GiB 单个上限，或者被 `gh release download` 的权限挡住；日志里会有具体报错。

**tag 为什么没同步？** 日志里如果是这句：

```
refusing to allow a GitHub App to create or update workflow `.github/workflows/xxx.yml`
without `workflows` permission
```

那是 GitHub 的硬限制：默认的 `GITHUB_TOKEN` **不允许创建或更新 `.github/workflows/` 下的文件**，而上游历史里只要带过 workflow 文件，它的 tag 就会被整条拒掉。

解决：建一个 classic PAT（勾 `workflow` + `repo`），存到 secret `SYNC_TOKEN`。源码和 Release 不受这个限制，不配也照常同步，只是没有 git tag。

> 顺带说：git tag 通常不是必需的。更新器比的是 Release 的 tag 名和版本号，Release 本身同步成功就够了。

**能同步私有上游吗？** 能，`UPSTREAM_TOKEN` 填一个有该私有仓库读权限的 PAT。

## 许可证

MIT，见 `LICENSE`。随便改随便用。

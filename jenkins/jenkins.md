# Jenkins + Docker CI/CD 笔记

## 1. 这套配置是在做什么

这份 Jenkins 任务的目标，是把 `ability-advisor-server` 项目的某个 Git Tag 版本，自动发布到远程服务器 `121.196.229.207` 上，并通过 Docker / Docker Compose 完成镜像构建和服务更新。

它的核心思路是：

1. 只允许发布符合规范的 Tag，比如 `v0.0.1`
2. 从 Git 仓库检出这个 Tag 对应的代码
3. 在 Jenkins 上再次校验：这个 Tag 必须来自主分支，不能是随便打在其他分支上的 Tag
4. 将代码同步到远程服务器
5. 在远程服务器里执行 `docker build`
6. 再通过 `docker compose up -d app` 更新运行中的服务

所以这套流程本质上是：

`Git Tag -> Jenkins 触发 -> 校验 Tag -> 同步代码 -> 远程构建 Docker 镜像 -> 重启/更新容器`

---

## 2. 任务整体定位

### 2.1 Jenkins 任务名称和描述

- 任务名：`新版诊断推荐系统`
- 描述：纯文本说明
- 机器人通知：`dingtalk-仅通知群`

含义：

- 任务名就是 Jenkins 页面里展示的任务名称，用于识别这个 Job 是给哪个系统发布用的。
- 描述通常用来写该任务用途、触发方式、负责人、注意事项。
- 钉钉机器人一般用于构建结果通知，比如构建成功、失败、开始发布等。

为什么这么配：

- 发布任务通常不是给开发随便点着玩的，所以名称和通知都要明确。
- 绑定通知群后，团队能知道谁发了版、发的什么版本、是否成功。

---

## 3. General 配置说明

### 3.1 丢弃旧的构建

作用：

- 控制 Jenkins 保留多少历史构建记录、日志、归档文件。

为什么常用：

- 构建历史太多会占用磁盘。
- 发布任务保留最近一段时间的记录即可，便于回溯问题。

你截图里没有展开具体规则，所以这里只能确认：

- 这个选项被关注到了，说明已经考虑到 Jenkins 磁盘清理问题。

### 3.2 This build requires lockable resources

作用：

- 给任务加“锁”，避免多个构建同时争抢同一个资源。

为什么可能启用：

- 发布任务通常不希望并发跑。
- 因为它会操作同一个远程目录、同一个 Docker 服务、同一套线上容器。
- 如果两个发布同时进行，可能出现代码被覆盖、容器交叉重启、版本混乱。

### 3.3 参数化构建过程

作用：

- 让这个 Jenkins Job 在每次运行时，可以接收外部参数。

这套配置里最重要的参数是：

- `RELEASE_TAG`

为什么必须参数化：

- 发布不是固定发某个分支，而是发“某个具体版本”。
- 用参数化方式，可以手动选择 Tag，也可以让 webhook 把 Tag 传进来。

### 3.4 Git 参数

当前配置：

- 名称：`RELEASE_TAG`
- 描述：`选择要发布的Tag版本，如 v0.0.1`
- 参数类型：`标签`

作用：

- 这个参数会从 Git 仓库里读取可用 Tag，供用户选择。
- 最终这个值会传给后面的 SCM 和 Shell 脚本使用。

为什么这么选：

- 发布场景最合适的参数类型就是“标签”。
- 如果用普通字符串，用户容易手输错。
- 选择 Tag 可以减少误发版风险。

`RELEASE_TAG` 在后面承担了三个角色：

1. 决定 Jenkins 拉哪个 Git 引用
2. 决定校验哪个 Tag
3. 决定 Docker 镜像打什么版本标签

### 3.5 Throttle builds / 在必要的时候并发构建

作用：

- 控制构建并发数，避免 Jenkins 同类任务同时跑太多。

为什么在发布任务里重要：

- 发布型任务通常更适合串行，不适合多个版本一起发。
- 即使没完全依赖这个选项，也说明配置者意识到了“并发发布”是风险点。

### 3.6 JDK

当前配置：

- `(System)`

作用：

- 指定 Jenkins 构建时使用哪套 JDK。

为什么这里可能保持系统默认：

- 这个 Job 的核心逻辑其实不是 Java 编译，而是 Git、Shell、Expect、rsync、ssh、Docker。
- 所以 JDK 不是关键配置，系统默认即可。

---

## 4. 源码管理（SCM）配置说明

### 4.1 选择 Git

当前仓库：

- Repository URL:
  `https://codeup.aliyun.com/60d996df70ab8549fd3e1ba6/ability-advisor-server.git`

作用：

- 告诉 Jenkins 代码从哪里拉。

为什么这么配：

- 这是阿里云 Codeup 仓库地址，说明代码托管在 Codeup 上。

### 4.2 Credentials

作用：

- Jenkins 拉私有仓库代码时，需要认证。

为什么必须配：

- 如果仓库不是公开的，Jenkins 没凭证就无法 clone / fetch。

### 4.3 Branches to build

当前配置：

- `refs/tags/${RELEASE_TAG}`

作用：

- 指定本次构建要检出的 Git 引用。

为什么不是分支，而是这样写：

- 因为这是“按 Tag 发布”，不是“按分支发布”。
- `${RELEASE_TAG}` 是参数化变量。
- 用户选了 `v1.2.3` 后，Jenkins 实际会去拉：
  `refs/tags/v1.2.3`

这样做的意义：

- 保证构建内容对应一个固定版本快照。
- 避免因为主分支后来又提交了新代码，导致同一次发布内容漂移。

### 4.4 Additional Behaviours

截图里显示已存在该区域，但没展开具体项。

它通常用于：

- 指定 refspec
- 清理工作区
- shallow clone
- 只拉标签
- 拉子模块

结合你后面的校验脚本，可以推断一个重要点：

- Jenkins 本地必须能看到 `origin/main`

因为脚本里要校验：

- `refs/remotes/origin/main`

所以如果 Additional Behaviours 里限制得太死，只拉 Tag、不拉主分支，就会导致后面的校验失败。

换句话说，这套 Job 想正常工作，SCM 至少要满足：

1. 能拉到目标 Tag
2. 能在本地保留 `origin/main` 这个远程跟踪分支

---

## 5. 触发器（Triggers）配置说明

这套配置里最关键的是：

- `Generic Webhook Trigger`

### 5.1 触发方式概念

这份 Job 可以有两种启动方式：

1. 手动在 Jenkins 页面选择 `RELEASE_TAG` 后点构建
2. 由外部系统通过 webhook 调 Jenkins 接口自动触发

### 5.2 Generic Webhook Trigger 的作用

作用：

- 允许外部系统通过 HTTP 请求触发这个 Jenkins Job。

它特别适合：

- Git Tag 创建事件
- GitLab / Codeup webhook
- 自定义发布平台调用 Jenkins

### 5.3 Post content parameters

当前配置：

- Variable: `RELEASE_TAG`
- Expression: `$.ref`
- Value filter: `^refs/tags/`

含义拆解：

外部系统发来的 JSON 很可能长这样：

```json
{
  "ref": "refs/tags/v1.2.3"
}
```

这三项配置的含义分别是：

- `RELEASE_TAG`：把解析出的值存进 Jenkins 变量 `RELEASE_TAG`
- `$.ref`：从 webhook 的 JSON 里取 `ref` 字段
- `^refs/tags/`：把前缀 `refs/tags/` 去掉

所以最终效果是：

- webhook 传入：`refs/tags/v1.2.3`
- Jenkins 拿到：`v1.2.3`

为什么这么填：

- 因为后面的 Job 参数、Git 检出、Shell 脚本都希望拿到的是纯净版本号 `v1.2.3`
- 而不是完整 Git 引用路径 `refs/tags/v1.2.3`

### 5.4 Token

当前配置：

- `ability-advisor-server`

作用：

- 给 webhook 触发接口加一个口令。

为什么要有：

- 避免任何人都能随便请求 Jenkins 接口触发发布。
- 只有知道 token 的调用方，才能成功触发这个 Job。

### 5.5 Cause

当前配置：

- `ability-advisor-server-tag-$RELEASE_TAG`

作用：

- 定义本次构建在 Jenkins 里显示的触发原因。

效果示例：

- `ability-advisor-server-tag-v1.2.3`

为什么这么写：

- 一眼就能看出这次构建是哪个版本触发的。
- 方便日志排查和历史追溯。

### 5.6 Allow several triggers per build

作用：

- 控制同一次构建是否允许接收多个触发事件。

发布任务里的理解：

- 通常不建议太激进地并发接受多个触发。
- 因为发布是有状态操作，不像普通测试那样天然适合并行。

### 5.7 Silent response

作用：

- 控制 webhook 响应里是否返回太多构建信息。

意义：

- 有助于减少外部系统看到的内部细节。

### 5.8 Print post content / Print contributed variables

作用：

- 把 webhook 的原始内容和解析后的变量打印到构建日志里。

为什么很有用：

- 调试 webhook 时非常方便。
- 出问题时能快速看见 Jenkins 到底收到的是什么。

注意点：

- 如果 webhook 里可能带敏感信息，日志打印要谨慎。

### 5.9 Optional filter

当前配置：

- Expression: `^v\d+\.\d+\.\d+$`
- Text: `$RELEASE_TAG`

作用：

- 在真正触发 Job 前，再做一层正则过滤。

这个正则表示：

- 只允许 `v数字.数字.数字` 这种格式
- 例如：`v0.0.1`、`v1.2.3`

不允许的例子：

- `release-1.0.0`
- `v1.0`
- `v1.0.0-beta`

为什么这么做：

- 防止 webhook 把奇怪的 ref 传进来
- 防止误触发非正式版本
- 让发布规范更统一

这相当于第一道版本格式闸门。

---

## 6. Environment 配置说明

截图里出现的环境增强项包括：

- Delete workspace before build starts
- Use secret text(s) or file(s)
- Provide Configuration files
- Send files or execute commands over SSH before the build starts
- Send files or execute commands over SSH after the build runs
- 在构建日志中添加时间戳前缀
- Provide Node & npm bin/ folder to PATH
- Terminate a build if it's stuck

这里需要区分两种情况：

1. “只是可选能力，界面上显示出来”
2. “真的勾选启用了”

从你贴的内容来看，更像是 Jenkins 页面展示了这些可选功能，但没有看到每项都展开的启用细节。

其中比较值得理解的是：

### 6.1 Delete workspace before build starts

作用：

- 每次构建前清空工作区。

优点：

- 避免上次构建残留文件影响本次结果。

缺点：

- 会增加重新拉代码的时间。

### 6.2 在构建日志中添加时间戳前缀

作用：

- 每一行日志前带上时间。

为什么推荐：

- 发布排查时非常有用。
- 能看出每一步耗时，尤其适合排查 rsync、ssh、docker build 卡在哪里。

### 6.3 Terminate a build if it's stuck

作用：

- 如果构建卡死太久，自动中止。

为什么对这类任务重要：

- `rsync`、`ssh`、`docker build` 都可能卡住。
- 超时保护能避免 Jenkins executor 被长期占用。

---

## 7. Build Steps 总览

这套任务里有 3 个关键构建步骤：

1. Shell 脚本：校验 `RELEASE_TAG` 的合法性和来源
2. Expect 脚本：把 Jenkins 工作区同步到远程服务器
3. Expect 脚本：远程构建 Docker 镜像并用 Docker Compose 更新服务

可以理解成三道关卡：

1. 版本对不对
2. 代码传过去了没有
3. 服务更新成功没有

---

## 8. 第一个 Shell 脚本详解

脚本原文的目标是：

- 校验版本参数
- 校验 Tag 是否存在
- 校验 Tag 是否属于主分支

### 8.1 脚本作用总结

这一步是整个发布流程里最关键的“安全闸门”。

它防止以下情况：

- 用户没传 `RELEASE_TAG`
- Tag 名字乱写
- 这个 Tag 在仓库里不存在
- 这个 Tag 不在 `origin/main` 的提交历史里

也就是说：

- 不是主线代码，禁止发版

### 8.2 关键逻辑逐段解释

#### `set -euo pipefail`

作用：

- `-e`：命令失败就退出
- `-u`：使用未定义变量时报错
- `-o pipefail`：管道中任一命令失败都算失败

为什么推荐：

- 发布脚本必须“失败即停”，不能带错继续往下执行。

#### `: "${RELEASE_TAG:?...}"`

作用：

- 强制要求环境变量 `RELEASE_TAG` 必须存在。

为什么这么写：

- 如果用户没选 Tag，脚本直接报错退出。

#### `TAG_REGEX="${TAG_REGEX:-^v[0-9]+\.[0-9]+\.[0-9]+$}"`

作用：

- 设定 Tag 格式规则。

默认允许：

- `v1.2.3`

为什么这么配：

- 和 webhook 过滤规则保持一致。
- 手动触发和自动触发统一规范。

#### `git rev-parse -q --verify "refs/tags/${TAG}^{commit}"`

作用：

- 校验这个 Tag 是否真实存在，并且能解析到一个 commit。

为什么重要：

- 防止用户传了一个根本不存在的版本号。

#### `TAG_COMMIT="$(git rev-parse "refs/tags/${TAG}^{commit}")"`

作用：

- 拿到这个 Tag 对应的提交 SHA。

#### `MAIN_REF="refs/remotes/${PRIMARY_REMOTE}/${PRIMARY_BRANCH}"`

默认组合结果：

- `refs/remotes/origin/main`

作用：

- 指定“主分支”的远程跟踪引用。

#### `git rev-parse -q --verify "${MAIN_REF}^{commit}"`

作用：

- 确认 Jenkins 本地确实已经有 `origin/main` 这个引用。

为什么重要：

- 如果 Jenkins 只拉了 Tag 没拉 main，这里会直接失败。
- 这是在提醒 SCM 配置必须完整。

#### `git merge-base --is-ancestor "${TAG_COMMIT}" "${PRIMARY_COMMIT}"`

这是全脚本最核心的一句。

作用：

- 判断 `TAG_COMMIT` 是否是 `PRIMARY_COMMIT` 的祖先提交。

翻译成人话就是：

- 这个 Tag 对应的代码，是否已经包含在主分支里。

如果是：

- 允许发布

如果不是：

- 拒绝发布

为什么这么设计：

- 防止从测试分支、个人分支、临时分支随便打个 Tag 就上生产。
- 保证发布版本来自主线。

### 8.3 这个脚本最终在防什么

它主要在防 4 类问题：

1. 版本号格式错
2. Tag 不存在
3. Jenkins 拉代码不完整
4. Tag 不属于主分支

所以它不是“多余校验”，而是发布流程的质量门禁。

---

## 9. 第二个 Expect 脚本详解

这一步的目标是：

- 使用 `rsync` 把 Jenkins 工作区内容同步到远程服务器目录

远程目标目录：

- `/workspace/ability-advisor-server/`

### 9.1 为什么用 Expect

因为脚本里使用的是：

- 用户名密码登录 SSH / rsync

不是：

- SSH key 免密登录

`expect` 的作用就是：

- 自动识别交互式提示
- 自动输入 `yes`
- 自动输入密码

### 9.2 关键变量

- `ip = 121.196.229.207`
- `pass = Strong@365`
- `src = $WORKSPACE/`
- `dest = /workspace/ability-advisor-server/`

含义：

- 源目录是 Jenkins 当前构建工作区
- 目标目录是远程服务器上的项目部署目录

### 9.3 `rsync -azvP --delete`

这几个参数很关键：

- `-a`：归档模式，保留文件属性，递归复制
- `-z`：传输时压缩
- `-v`：显示详细日志
- `-P`：显示进度，支持断点续传相关信息
- `--delete`：让远程目录删掉那些本地已经不存在的文件

为什么这么配：

- 要让远程目录尽量和 Jenkins 工作区保持一致
- 避免远程残留旧文件

### 9.4 排除项 `--exclude`

排除了：

- `.git/`
- `.idea/`
- `.vscode/`
- `.venv/`
- `__pycache__/`
- `*.pyc`
- `logs/`
- `media/`
- `db.sqlite3`

为什么排除：

- `.git/`：部署不需要仓库元数据
- IDE 配置目录：无部署价值
- 虚拟环境、缓存文件：不应该跟着部署
- 日志、媒体文件、SQLite 文件：通常属于运行期数据，不应该被代码发布覆盖

这一部分体现出的部署思路是：

- “同步代码和配置”
- “不覆盖运行时数据”

### 9.5 `ssh -o StrictHostKeyChecking=no`

作用：

- 首次连接时不要求人工确认主机指纹。

为什么这么配：

- 避免 Jenkins 因为交互确认卡住。

代价：

- 安全性较弱，容易忽略主机身份校验。

### 9.6 这一步完成后得到什么

执行完后，远程服务器上的：

- `/workspace/ability-advisor-server/`

会被更新成 Jenkins 当前工作区对应的 Tag 版本代码。

这一步本质上是：

- “把待发布代码送到服务器上”

---

## 10. 第三个 Expect 脚本详解

这一步的目标是：

- 登录远程服务器
- 在服务器上构建 Docker 镜像
- 用 Docker Compose 更新应用容器

### 10.1 关键变量

- `project_dir = /workspace/ability-advisor-server`
- `deploy_dir = /workspace/enterprise-platform`
- `image_name = ability-advisor-server`
- `release_tag = $RELEASE_TAG`

说明：

- `project_dir`：刚刚 rsync 同步过去的项目代码目录
- `deploy_dir`：放 `docker-compose.yml` 或 compose 管理文件的目录
- `image_name`：镜像名称
- `release_tag`：本次发布版本号

### 10.2 远程命令在做什么

远程执行的是：

1. `cd /workspace/ability-advisor-server`
2. 输出当前目录和镜像名
3. 执行 Docker build
4. 切到 `/workspace/enterprise-platform`
5. 执行 `docker compose up -d app`
6. 查看容器状态
7. 打印最近 100 行日志

### 10.3 Docker build 命令解释

命令：

```bash
docker build --pull=false -t ability-advisor-server:$RELEASE_TAG -t ability-advisor-server:latest .
```

作用：

- 基于当前代码目录构建镜像
- 同时打两个标签：
  - 版本标签：`ability-advisor-server:v1.2.3`
  - 最新标签：`ability-advisor-server:latest`

为什么打两个标签：

- 版本标签用于精确追踪发布版本
- `latest` 方便 compose 或运维脚本直接引用最新镜像

`--pull=false` 的含义：

- 构建时不强制拉取最新基础镜像

为什么可能这么选：

- 避免每次发布都受外网和镜像站影响
- 提升构建稳定性和速度

代价：

- 如果基础镜像有安全更新，不会自动拿到

### 10.4 `docker compose up -d app`

作用：

- 按 compose 配置启动或更新 `app` 服务
- 如果镜像已变化，则重建并替换容器

为什么这样做：

- 用 compose 管理服务比手动 `docker run` 更规范
- 依赖、网络、端口、挂载、环境变量都集中在 compose 文件里管理

### 10.5 `docker compose ps`

作用：

- 查看 compose 管理的容器当前状态

为什么要加：

- 构建成功不等于服务正常启动
- 这里能快速确认容器是否起来了

### 10.6 `docker logs --tail=100 enterprise-app || true`

作用：

- 打印目标容器最近 100 行日志

为什么很有用：

- 发布完第一时间看启动日志
- 能快速发现迁移失败、环境变量缺失、端口占用等问题

为什么后面加 `|| true`：

- 即使日志命令失败，也不要让整个发布脚本因为“日志查看失败”而退出
- 说明“看日志”是辅助诊断，不是发布成功的唯一判据

---

## 11. 这套 Jenkins + Docker CI/CD 的完整流程

### 11.1 手动触发流程

1. 开发在 Git 仓库中创建一个 Tag，例如 `v1.2.3`
2. 在 Jenkins 手动选择参数 `RELEASE_TAG=v1.2.3`
3. Jenkins 从仓库检出 `refs/tags/v1.2.3`
4. 第一个 Shell 脚本校验：
   - Tag 格式是否合法
   - Tag 是否存在
   - Tag 是否属于 `origin/main`
5. 校验通过后，第二个脚本把代码同步到远程服务器
6. 第三个脚本在远程服务器执行 `docker build`
7. 远程使用 `docker compose up -d app` 更新服务
8. Jenkins 输出容器状态和应用日志
9. 钉钉机器人通知构建结果

### 11.2 自动触发流程

1. Git 平台发生 Tag 事件
2. Webhook 向 Jenkins 的 `generic-webhook-trigger` 接口发请求
3. 请求体里的 `ref` 被解析成 `RELEASE_TAG`
4. 只有当 Tag 满足 `^v\d+\.\d+\.\d+$` 时才会真正触发任务
5. 后续流程与手动触发一致

### 11.3 用一句话概括这条流水线

这条流水线是：

- “基于 Git Tag 的受控发布流水线，先校验版本来源，再把代码同步到服务器，最后在服务器上通过 Docker Compose 完成应用升级。”

---

## 12. 为什么整体这样设计

### 12.1 为什么按 Tag 发布，而不是按分支发布

因为 Tag 代表一个固定版本点。

优点：

- 发布内容可追溯
- 便于回滚和复盘
- 避免同一分支代码不断变化导致“同名发布不一致”

### 12.2 为什么要校验 Tag 必须属于主分支

因为生产发布应该只来自主线。

优点：

- 避免测试分支代码误上生产
- 保证发布代码经过主线合并流程

### 12.3 为什么先 rsync，再在远程 build

这是“远程构建型部署”。

优点：

- Jenkins 节点不必负责产出最终镜像仓库
- 远程机器本地就能构建并启动
- 不依赖额外镜像仓库也能完成部署

适合：

- 小中型项目
- 单机或少量服务器部署

### 12.4 为什么用 Docker Compose 更新服务

因为 Compose 适合管理一组相关服务配置。

优点：

- 配置集中
- 操作统一
- 维护方便

---

## 13. 这套方案的优点

1. 发布基于 Tag，版本边界清晰
2. 有 webhook 自动化能力，也能手动补发
3. 有主分支祖先校验，能防误发布
4. 用 rsync 同步代码，比直接复制更高效
5. 用 Docker Compose 管理服务，发布动作比较标准化
6. 构建结束后打印容器状态和日志，便于快速排障

---

## 14. 这套方案的风险和改进点

下面这些不是“配置错误”，但属于值得记录的运维注意事项。

### 14.1 密码硬编码在 Expect 脚本里

当前脚本里直接写了：

- 服务器 IP
- root 密码

风险：

- Jenkins 配置可见范围内的人可能看到密码
- 变更密码时要手动改脚本

更推荐：

- 用 Jenkins Credentials 管理密码
- 或改成 SSH key 免密登录

### 14.2 使用 root 直接发布

风险：

- 权限过大
- 一旦脚本出错或被滥用，影响面更大

更推荐：

- 用专门部署账号
- 只给必要目录和 Docker 操作权限

### 14.3 `StrictHostKeyChecking=no`

风险：

- 跳过主机身份校验

更推荐：

- 提前维护可信主机指纹

### 14.4 `--pull=false`

风险：

- 基础镜像可能长期不更新

适合场景：

- 追求稳定、减少外部依赖

需要补充的管理动作：

- 定期人工刷新基础镜像

### 14.5 `--delete` 很强，但要谨慎

风险：

- 如果远程目录里有不该删但又不在 Jenkins 工作区的文件，会被删除

所以前提是：

- 目标目录必须是“纯部署代码目录”
- 运行期数据必须放在别处，或者已通过 `exclude` 排除

---

## 15. 你可以把这份 Job 理解成的架构图

```text
开发打 Tag / Webhook 触发
        |
        v
Jenkins Job 接收 RELEASE_TAG
        |
        v
检出 refs/tags/${RELEASE_TAG}
        |
        v
校验 Tag 是否存在且属于 origin/main
        |
        v
rsync 工作区到远程服务器 /workspace/ability-advisor-server/
        |
        v
SSH 登录远程机器
        |
        v
docker build -t ability-advisor-server:${RELEASE_TAG} -t latest .
        |
        v
docker compose up -d app
        |
        v
查看 compose 状态和应用日志
        |
        v
钉钉通知发布结果
```

---

## 16. 适合以后继续补充到文档里的信息

后续如果你想把这份笔记继续完善，可以再补这几类信息：

1. Jenkins Job 的完整截图
2. Additional Behaviours 的具体配置
3. `docker-compose.yml` 中 `app` 服务到底引用的是哪个镜像标签
4. 远程服务器目录结构
5. 回滚流程
6. 钉钉通知模板
7. webhook 来自 Codeup 的具体事件配置

这样这份笔记就会从“配置说明”升级成“完整发布手册”。

---

## 17. 附：当前文档原始截图

![截屏2026-04-28 16.25.47](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-28 16.25.47.png)

![截屏2026-04-28 16.27.28](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-28 16.27.28.png)

![截屏2026-04-28 16.33.03](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-28 16.33.03.png)

![截屏2026-04-29 16.26.29](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-29 16.26.29.png)

![截屏2026-04-29 16.26.41](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-29 16.26.41.png)



![截屏2026-04-29 16.27.03](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-29 16.27.03.png)





![截屏2026-04-29 16.27.17](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-29 16.28.00.png)

![截屏2026-04-29 16.28.19](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-29 16.28.19.png)

![截屏2026-04-29 16.28.35](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-29 16.28.35.png)





![截屏2026-04-29 16.29.10](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-29 16.29.10.png)



![截屏2026-04-29 16.29.30](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-29 16.29.30.png)





![截屏2026-04-29 16.30.14](/Users/apple/Library/Application Support/typora-user-images/截屏2026-04-29 16.30.14.png)






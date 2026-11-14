# 设计参考

按项目需要选用以下架构方案；命令名、模式数量、分层与默认行为均可调整。控制台格式属于硬性要求，不能按可选架构裁剪。不要为了套用示例增加无用模式、容器、数据库或状态机。

## 1. 分层与运行

- package.json：公开 `mode:*` 命令，映射到统一入口，例如 `node scripts/mode.mjs start`。Bun 是任务入口，实际服务可用 Node、Bun 或容器。
- CLI：参数解析、命令注册表、颜色与面板、退出码。
- 管理层：当前模式解析、模式切换、仓库身份、锁、状态、资源分配与健康等待。
- 模式适配器（多模式项目可选）：为项目实际支持的方式实现所需操作；公共命令解析当前方式后分派。单一方式可直接调用运行脚本。
- 项目适配器：构建、迁移、启动 argv、就绪协议、依赖健康要求。没有数据库的项目不应被强加迁移或数据库检查。
- 持久配置：仓库内固定的 Git 忽略文件（例如 `.local/mode.json`）保存用户选择，按实际需要分实例或环境字段；迁移与并发写入规则见入口“本地配置与运行状态”。
- 运行状态目录：单独存放 PID、日志、证书、构建基线、锁和生成文件；目录权限 0700、敏感文件 0600，不作为第二套用户配置来源。

Linux 可用 Bash `flock` 配合 Node `/proc` 校验。跨平台实现必须替换锁和进程身份适配器，不能假定 macOS/Windows 存在这些能力。仓库级锁覆盖模式切换及整个 start/stop/restart/reload/build 操作；监督进程不得继承锁而永久占用。内部调用复用已持锁上下文，避免重入死锁。

### 多模式项目：当前模式与切换事务（可选）

需要模式选择时，将选择与运行状态分开保存。单一方式或有明确默认值时可直接运行；没有默认且存在歧义时才提示选择真实可用的模式。选择是否附带启动由项目定义，help 和输出保持一致。

项目采用单活动模式时，切换时在仓库锁内：验证目标适配器与必要配置 → 核验并停止旧模式受管理实例 → 确认停止成功 → 原子写入 currentMode。停止失败保留旧选择并返回非零；不删除旧卷、镜像或缓存。如项目选择“一键切换并启动”，则实现完整事务并清楚报告失败后的已选模式和实际运行状态。残留其他模式实例应显式报告并提供处理指引，不让 status/stop 越界操作。若项目支持多模式并行，应显式设计选择规则与命名空间，不隐式开启。

生命周期命令在获取锁后读取一次 currentMode，整个操作使用该快照，防止操作途中模式改变。状态同时标记 selectedMode、runningMode 和操作阶段，部署未完成不能提前标记健康。

### 示例：本机开发（按需采用）

本机开发可使用项目现有开发命令并支持 HMR，无需强制每次生产构建。start 准备依赖并监督启动；restart 只重启现有命令；reload 对已支持 HMR 的源码更新确认就绪，对配置或需编译产物的变更执行必要构建/重启。不能把“进程仍活着”当作新代码已经生效，应使用构建标识或针对性预览检查。生产打包预览可另设模式或配置。

### 示例：Docker 部署预览与增量 reload（仅容器项目）

使用 Compose 的项目可通过稳定 project name 管理本仓库服务。以下示例约定 restart 不构建、reload 更新代码；项目已有其他合理约定时可以调整并写入 help。start 复用有效镜像，缺失时构建，启动必要依赖并等待健康。restart 重启当前容器，不负责应用新源码。reload 是开发者修改功能后的主入口：

1. 读取当前模式、配置、依赖图和已部署镜像 ID，确认 Docker 可用；记录受影响服务与上一版本。
2. 复用 BuildKit/镜像层缓存构建所需镜像。Dockerfile 先复制依赖清单和锁文件安装依赖，再复制源码；配置 .dockerignore，按需使用缓存挂载。不要默认 --no-cache 或无条件 prune；空间检查与按策略回收遵循 [磁盘与缓存生命周期](storage-lifecycle.md)。能可靠确定服务影响范围时只构建相关服务；无法确定时构建应用服务集合并利用缓存，不能漏掉配置、锁文件或基础镜像变更。
3. 构建新镜像期间保留旧容器运行；构建失败不替换旧服务。使用明确的版本标签/镜像 ID，保留上一版本用于失败恢复；不能仅从 Git commit 判断是否包含未提交修改。
4. 如有迁移，按项目适配器指定时机执行，失败中止部署。构建成功后更新受影响服务，保留数据库和数据卷；不要先 compose down 整个项目。只有确认依赖无需变化时才能使用 --no-deps；共享依赖或配置变化按依赖图更新。
5. 读取实际容器 ID、发布端口和新版本标识，等待各必要服务及 HTTP 健康。成功后保存部署状态，打印当前模式、构建/缓存结果、更新服务和本机/局域网预览地址。
6. 部署失败返回非零并显示旧版/新版的实际状态和日志；具备恢复条件时恢复旧镜像。涉及不可逆数据库迁移时不能假称镜像回退恢复了一切，也不能自动清空数据。

此处“热加载 Docker”表示一条 reload 命令完成增量构建和更新预览。普通容器替换可能短暂中断，不承诺浏览器 HMR 或零停机。若启用 bind mount/watch/HMR，把它作为明确的 Docker 开发变体；涉及依赖或镜像配置变更时仍需构建并更新容器。reload 面向最新代码，服务尚未启动时可完成构建并启动预览。

## 2. 以仓库固有特征生成稳定基准

默认身份输入是 `realpath(仓库根目录)`：归一化符号链接和相对路径。不要用 cwd 的任意子目录、basename、分支名、当前时间或随机 UUID 当稳定基准。同一 checkout 的重启保持一致，不同路径的 clone/worktree 分开。随机 instanceId 只标识一次启动。

推荐派生方式（单一运行方式可省略 mode 维度，或使用稳定的内部默认值，无需公开模式选择命令）：

```text
canonicalRoot = realpath(repoRoot)
repoKey       = sha256("mode:v1\0" + canonicalRoot)
resourceKey   = sha256(repoKey + "\0" + mode + "\0" + resourceKind + "\0" + serviceName)
basePort      = rangeStart + uint32be(resourceKey[0:4]) % rangeSize
projectName   = sanitizedRepoName + "-" + repoKey[0:12] + "-" + mode
```

- 派生算法带版本；同一算法与输入保证稳定，升级算法前说明资源迁移。展示名称的 slug 只用于可读性，哈希提供区分。
- HTTP、数据库、Redis、调试端口分别使用 serviceName 派生，不要全部使用一个端口。端口范围可配置，避开宿主机实际保留/临时端口范围；校验范围边界及回退次数。
- 容器使用稳定 Compose project name（如 `docker compose -p <projectName>`），让网络与卷自然隔离；避免固定 `container_name` 和跨项目共享的 external 网络/卷。需要显式命名时包含 repoKey 与服务名。
- 容器标签记录 repoKey、mode、服务名、管理器标记；状态保存容器 ID。停止时核验标签及 ID。可复用资源的稳定 ID 与一次启动的 instanceId 分开。
- 缓存目录、Unix socket、锁名、临时资源也按 repoKey/mode/serviceName 隔离，并遵守名称长度与平台限制。
- 路径移动会改变默认身份，旧卷不会自动迁移或删除。确实需要跨路径稳定时允许显式实例键，并提醒多个 checkout 使用同一键会共享命名空间；不要默认只用 Git remote，因为多个 worktree 会因此冲突。

### 端口冲突与真实分配

哈希碰撞和外部端口占用都正常存在。HTTP 服务从基线开始实际 bind，仅对 EADDRINUSE 在有界候选范围内重试，例如最多 32 次；其他错误立即失败。避免“探测空闲 → 关闭 socket → 正式启动”的竞态。超出候选范围明确失败，不杀占用者、不无限扫描。

容器端口由 Docker 实际 publish 决定：例如宿主 `0.0.0.0:<候选端口>:<容器服务端口>`。让 Docker 完成端口竞争裁决；冲突时仅回收本次创建的失败容器，再尝试下一候选，不能删除已有容器或数据卷。容器内 HTTP 同时绑定 `0.0.0.0`。非 HTTP 依赖如数据库可默认只发布到宿主 `127.0.0.1`，按实际访问需求配置。

监听成功后持久化 `portBase`、`actualPort`、`bindHost`、服务名和资源 ID。健康检查、URL、后续容器依赖配置读取真实分配结果，不能重新计算基线后假定它就是实际端口。跨仓库锁不能完全防止外部竞争，最终以操作系统/Docker 绑定结果为准。

## 3. 本地 HTTPS、证书与健康

本地 Web 入口尽量默认 HTTPS，包含直接运行和本地容器预览。优先使用框架已有 TLS 能力；不支持时可以用项目管理的轻量 TLS 反向代理。代理同样纳入端口派生、生命周期和身份核验，不引入固定 443 端口要求。后端保持内部 HTTP 时，必须同时检查外部 HTTPS 入口及后端就绪。

- 自动生成或复用本项目自签证书，或使用已有本地 CA 签发的开发证书。证书放在被 gitignore 的本地状态目录，私钥权限 0600；不提交 Git、不烘焙进镜像，容器使用只读挂载。不从可预测的仓库哈希生成私钥，密钥必须使用安全随机源。
- SAN 覆盖实际入口，如 localhost、127.0.0.1 和展示的局域网 IP/本地域名；0.0.0.0 只是监听范围，不是浏览器证书目标。复用尚有效且 SAN 匹配的证书，过期或访问地址变化时重新签发并加载。
- 自签不等于浏览器自动信任。状态显示“HTTPS · 自签 · 信任未验证”等真实状态，并给出证书位置和针对当前系统的信任方法。只生成证书不会改变系统/浏览器信任库；涉及信任库修改时遵循已有授权和系统权限。另一台局域网设备需独立信任证书或 CA，不能宣称本机信任自动覆盖所有设备。
- 健康客户端显式信任本项目证书或 CA，并校验主机名与实例标识；禁止全局 NODE_TLS_REJECT_UNAUTHORIZED=0 或无范围的跳过证书校验。测试错误证书和错误 SAN 确实失败。
- 对外 URL、HMR/WebSocket（wss）、代理头、允许的 origin、Cookie 和 OAuth 回调按实际 HTTPS 地址配套配置。代理应支持 WebSocket 升级；不要只让首页 HTTPS 可用而破坏热更新。
- 无法启用 HTTPS 或用户明确需要 HTTP 时，按项目约定提供显式配置并在状态面板标出实际协议和原因。已配置的 HTTPS 启动失败不能悄悄改用 HTTP 后报告成功。

### 进程身份与健康

本机进程状态至少包含版本、mode、canonicalRoot、repoKey、instanceId、PID、进程创建时间、启动时间、实际地址。Linux 校验 `/proc/<pid>/stat` 的创建时间与进程组、cwd 和监督命令/实例参数，防止 PID 重用。只有核验成功才发送进程组 SIGTERM，超时后再次核验并 SIGKILL。无法核验时不猜测归属。

Docker 使用容器 ID、project/mode/service 标签和镜像 ID 核验身份，不使用宿主 /proc 替代容器身份核验。

健康检查仅请求经核验实例上报的回环地址，设置超时、禁止重定向、验证本次 instanceId 响应头及业务健康 JSON。健康路径与头名可配置。数据库等依赖是否必须健康由适配器声明。停止或状态失效时不访问旧端口；不能把其他项目返回的 200 判为成功。

## 4. 控制台格式与色彩（强制）

实现前必须阅读 [控制台输出契约](console-output.md)，以该文档作为唯一格式与验收规范：status 默认彩色状态面板，help 默认分组对齐帮助；非 TTY 只去色、不去排版；机器输出仅显式 --json。start/reload 成功后复用面板。不能用直接打印内部状态对象或一组未经排版的 console.log 代替界面。

## 5. 方便集成各种命令

推荐命令注册表作为单一来源，条目包含 name、description、scope（mode/repository）、execution（host/container/adapter）、kind（task/service）、lock（none/checkout）、argv 或 handler、参数规则、超时及退出码策略。help、分派和 package.json 入口保持同步。

描述示意（需在目标管理器实现，不是已存在的通用 API）：

```js
const commands = {
  lint: { description: '代码检查', scope: 'repository', execution: 'host', kind: 'task', lock: 'none', argv: ['bun', 'run', 'lint'] },
  'db:migrate': { description: '数据库迁移', scope: 'mode', execution: 'adapter', kind: 'task', lock: 'checkout', task: 'migrate' },
  'test:unit': { description: '单元测试', scope: 'repository', execution: 'host', kind: 'task', lock: 'none', argv: ['bun', 'test'] },
}
// 入口示例："mode:lint": "node scripts/mode.mjs lint"
// 执行示例：spawn(argv[0], [...argv.slice(1), ...forwardedArgs],
//   { cwd: root, env: taskEnv, stdio: 'inherit', shell: false })
```

模式级任务通过当前适配器执行，例如 db:migrate 在 docker 模式容器内执行、local 模式本机执行；仓库级 lint/test 可直接在宿主运行，并在 help 中明确标记。新增模式只注册适配器与选择入口，无需复制公共命令。

普通任务透传 argv、工作目录、环境与退出码，避免 eval、拼 shell 字符串及吞掉失败。不要让 `mode:lint` 再调用自身而递归。需要 shell 管道时使用经过审查的脚本文件。对修改构建/运行资源的任务加锁；只读任务不必阻塞生命周期。新增长期服务需定义 start/stop/health/logs 与身份核验适配器；不能把所有可执行文件套进 Node HTTP 预加载钩子。

项目 hook 可包括 beforeBuild、afterBuild、beforeStart、afterHealthy、beforeStop、afterStop。明确 hook 所在锁范围、失败传播与补偿；默认失败中止，清理只涉及本次创建资源。不要暗中在普通 lint/test 任务前执行数据库迁移或重启服务。

## 6. 验收（仅执行适用项）

- 同路径重启基线相同；符号链接指向同仓库时身份相同；不同 clone/worktree 的资源命名空间不同；不同服务分开派生。
- 人为占用基线端口后有界顺延；全部候选占用时失败；原占用服务不受影响；并发启动不产生重复实例。
- 从非回环网卡访问 HTTP，验证实际 `0.0.0.0` 监听；容器同时验证内部 bind 和宿主 publish。
- 伪造健康 200、错误 instanceId、PID 重用和过期状态不能通过身份验证；停止 A 不影响 B。
- 构建/迁移失败不启动；超时及中断清理本次进程；本机可变产物构建拒绝覆盖运行产物，Docker 构建可与旧容器运行并存；持久卷保持不变。
- TTY 彩色、NO_COLOR 纯文本、FORCE_COLOR 强制、重定向默认纯文本；health --json stdout 可直接 JSON.parse，退出码正确。
- 接入一个带空格参数的普通任务和一个失败任务，验证 argv 与非零退出码原样传播；help 展示新增命令。

- 多模式项目验证每个实际模式的命令分派、选择持久化；单一方式无需模式选择即可使用，不为了验证添加 docker/local。
- 切换模式时旧模式被正确停止；失败不悄悄更新 currentMode；并发切换与 reload 被仓库锁串行化。
- Docker reload 验证源码变化触发受影响层重建、未变化层复用、仅必要服务被更新；更新后新功能可预览且卷数据不丢失。
- 镜像构建失败时旧容器仍健康；部署健康失败返回非零并正确报告/恢复可恢复版本；restart 不被误认为更新源码。

- 本地默认 HTTPS 可用；证书 SAN 与展示地址一致、私钥未入 Git/镜像；证书过期和地址变化可正确处理，错误证书被健康客户端拒绝。
- HTTPS 页面及 wss/HMR 可用；状态准确报告证书和信任状态；显式 HTTP 例外有清楚说明。

# 单节点容器发布与运维

本 fork 保留上游 relay、通用 Dockerfile 和完整 `scripts/ci.sh` 门禁，只增加版本镜像发布和本说明。上游基线由仓库根的 `UPSTREAM_REVISION` 固定。镜像版本独立于 Paseo daemon 版本，不建立同号兼容承诺。

## 发布

镜像位于 `ghcr.io/pennixrv/paseo-relay`，首批只构建 `linux/amd64`。Git tag 使用 `vX.Y.Z`，镜像使用 `X.Y.Z`；不发布 `latest`，不覆盖旧版本。修复或构建产物变化递增 patch，部署合同变化递增 minor。

1. 显式选择并审阅上游提交，合并后更新 `UPSTREAM_REVISION`；不要自动追随 upstream main。
2. 通过 PR 验证并合入 main，再创建新的版本 tag。
3. tag 的 CI 先运行完整 `scripts/ci.sh`，成功后由发布 job 创建 draft release 预留版本，再构建并推送通用镜像，最后发布包含 tag、digest、fork/upstream SHA 和架构的 release。
4. 同 tag 已有 draft 或正式 release 时，原生 `gh release create` 拒绝重复发布。中途失败保留预留版本和产物，查明失败后使用新版本，不删除 release/tag 来重用版本，不手动覆盖 registry tag。
5. GHCR 首次发布默认私有，不继承公开源码仓库的可见性。首次发布后，操作者须在 [package settings](https://github.com/users/PennixRv/packages/container/paseo-relay/settings) 将其设为 Public，再验证匿名 pull；以后新版本沿用该 package 可见性。不要为此把个人 token 放入 Compose。

发布只使用 job 级 `GITHUB_TOKEN`，无生产凭据；不自动部署 NAS，不修改应用协议。

## 单节点运行

按部署主机已有惯例建立独立 Compose 项目，引用明确镜像版本。将容器 HTTP `4000` 仅绑定受信 LAN 地址，公网 TLS 由反代终止；不要开放 Erlang 分布式端口，不使用 host network，不挂载 Docker socket、daemon home 或工作区。路由状态不需持久化卷。

单节点不设置 `PASEO_RELAY_CLUSTER_QUERY`，`PASEO_RELAY_MIN_CLUSTER_SIZE=1`；容器内监听 `0.0.0.0:4000`。连接数、ingress budget、内存水位须按本机预算设置，所有角色和 data socket 都计入活动 WebSocket 上限。参数语义以根 README 和 OPERATIONS 为准，不缩小协议帧上限代替容量治理。Docker restart policy 不会因 `/ready` 返回 503 自动重启。

从 backend 检查 `/health`、`/ready` 和必要 metrics；只报告少量判定，不导出完整运行态。只有容器 running 不算服务就绪。

## nginx 基线与登记表

使用官方 [Paseo v0.11.4 中文 README 的“自托管 relay TLS”](https://github.com/getpaseo/paseo/blob/v0.11.4/README.zh-CN.md#自托管-relay-tls) 中的 relay 专用示例：TLS listener、证书、`/ws`、HTTP upstream、HTTP/1.1、Upgrade/Connection 和 Host。不要误用 daemon Docker 的 6767 配置，单节点也不需要 Fly replay。

按部署事实替换公网 authority、证书及内网 upstream。精确开放 `/ws`，`proxy_pass` 不带 URI，不改写路径/query；非标准公网端口使用 `Host $http_host` 保留 authority。不要用共享 snippet 覆盖 Upgrade/Connection 或伪造 Origin。运维路径留在 backend，访问日志不记录 query；公开仓库不保存实际域名、登记 ID 或完整生产配置。

原示例没有自定义 timeout 或握手限速。本 fork 不增加经验数值；记录 nginx 实际继承的 read timeout，并使用原生保活和有界真实 WebSocket 测试跨过该时间验证，不能只看短时 101。依据见 [nginx WebSocket 文档](https://nginx.org/en/docs/http/websocket.html)。

自用部署可在该 vhost 维护 `serverId` 登记 include。这是额外的路由过滤，不是强身份认证；知道登记 ID 的第三方仍可能连接或干扰。保留原生配对、加密与资源边界，不添加当前客户端无法提供的 Basic Auth、mTLS 或共享 token。

- 从各 daemon 的有效 home/override 核对稳定 `serverId`，原值保留，不用设备名推导。实际列表只留在部署落点。
- nginx 普通 map 字符串忽略大小写，须采用完整转义、两端锚定的大小写敏感匹配；参见 [map 文档](https://nginx.org/en/docs/http/ngx_http_map_module.html)。
- 拒绝缺失/未知 ID、重复 `serverId`（含参数名大小写变体）及编码参数名；合法参数顺序不影响结果。实际测试大小写、重复、编码和 server/client/control/data，确认 nginx 与 Cowboy 判定一致。
- 每次修改先做备份、`nginx -t` 再 reload，并测试目标及已有长连接；不调整其他服务或共享 header 配置。

## daemon 配置与新增设备

1. 在新设备安装并启动 Paseo daemon，核对 home、有效 `serverId` 和当前版本；SSH 可达不等于已加入 relay。
2. 保持 `daemon.relay.enabled=false`，通过 `paseo daemon config set` 设置 `endpoint`、`publicEndpoint`、`useTls`、`publicUseTls`。两个 endpoint 均填写实际 `host:port`；公网 WSS 显式设置两个 TLS 为 true。环境覆盖优先，须先核对，不能输出完整敏感配置。
3. 在 nginx 登记该稳定 ID，检查后 reload。**relay 服务端只增加反代登记项，不重建或重启 relay。新增手机无需登记。** 正常 daemon 重启/升级沿用 ID，重建 home/改变 ID 时才更新登记；退役时移除对应项。
4. 当前版本 endpoint/TLS 需要 worker restart，enabled 可 reload。原生 config set 会尝试 reload，因此先写四个字段且保持关闭，最后重启 worker 加载新 endpoint，再启用 relay；避免自动 reload 时短暂连向旧 endpoint。验证实际连接、保活和恢复，不只看 enabled。
5. 保留原 home、ID、密钥及原有 SSH/桌面恢复通路。重启会断开客户端；同一 Codex thread 不要启动并发 writer。

NAS、反代及 WSL 验收完成后，由用户自行在 daemon 主机运行 `paseo daemon pair`，按原生输出在 Android 扫码或粘贴。完整 offer/二维码视为秘密，不写任务、公开仓库或日志摘录。核对自托管 endpoint/TLS 和目标身份；Android 版本、移动网络和手机后台恢复由用户后续实测，不属于本轮服务端部署的完成前置。

## 验收、更新与回滚

服务端验收包含 readiness、TLS、Upgrade、登记正反向矩阵、有界数据连接、控制保活、WSL 实际连接/受控 relay 重建后的恢复和现有桌面回归。临时测试 ID 在验收后移除，不把手机未测试记作已通过。

更新前核对新版本 release 的两个 SHA、digest、架构与本机预算；在实际项目目录运行原生 Compose config/pull/up。单节点替换会断开连接，由 daemon/client 重连，不承诺透明迁移或零中断。一次只做一项生产动作，完成 readiness、WSL 恢复与桌面回归后再继续，不做全进程状态导出。

升级失败只恢复目标 Compose 到留存的前一版本，再验证恢复。首次部署无前一版本时，关闭 daemon relay、按原有存在性恢复或移除四个 endpoint/TLS 字段，撤回目标 vhost/include 和目标服务；保留 home、密钥、SSH 和其他服务。运行配置及回滚副本留在部署 owner 处，不复制进源码仓库。

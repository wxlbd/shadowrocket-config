# Shadowrocket 防 DNS 泄露配置

基于 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 规则集的 Shadowrocket 配置文件，国内直连、境外代理，Claude 固定使用唯一家宽节点。Claude 域名显式远端解析；国内直连域名和节点域名仍使用国内 UDP DNS，不代表全设备 DNS 查询都加密。

## 订阅链接

复制以下任一链接到 Shadowrocket 进行订阅。电脑端如使用 v2rayN，请见 [v2rayN 归档说明](docs/v2rayn.md)。

**raw.githubusercontent.com（权威源）：**

```
https://raw.githubusercontent.com/wxlbd/shadowrocket-config/main/shadowrocket.conf
```

**jsDelivr CDN（国内推荐，速度更快）：**

```
https://cdn.jsdelivr.net/gh/wxlbd/shadowrocket-config@main/shadowrocket.conf
```

## 使用方法

1. 打开 Shadowrocket → 底栏「配置」→ 右上角 `+`
2. 粘贴上述任一订阅链接 → 「下载」
3. 等待下载完成后，长按该配置 → 「使用配置」
4. 配置生效后，规则集会自动从远端拉取（首次加载需联网）
5. 全局路由选择「配置」，确认客户端存在名为「家宽」的节点，且 `AI-HOME` 只包含这个节点；普通代理组使用名为「香港VPS」的节点

## 核心特性

### DNS 防泄露

- **Claude 显式远端解析**：Claude 域名规则各自带有 `force-remote-dns`，其余未匹配域名由 `FINAL,PROXY,force-remote-dns` 兜底；IP 规则带 `no-resolve`，避免为了匹配 IP 规则而触发本地解析
- **本地解析仅国内 UDP**：阿里/腾讯承担直连域名与代理节点域名的解析，公共 DNS 按递归源 IP 调度，CDN 就近
- **禁用 IPv6**：避免 v6 通道绕过 DNS 配置造成泄露
- **拒绝私有 IP 应答**：防 DNS rebinding 攻击

### Claude 固定家宽与 STUN 防护

- `AI-HOME = select, 家宽` 仅允许名为「家宽」的单一节点，不含普通代理组、直连或其他节点；该节点无法连接时，请求失败或超时，不切换出口。请勿把「家宽」设为包含其他出口的策略组。
- [Anthropic 官方网络要求](https://code.claude.com/docs/en/desktop#network-access-requirements)中的 `anthropic.com`、`claude.ai`、`claude.com`、`claude.app`、`claudeusercontent.com`、`claudemcpcontent.com` 及其全部子域均优先走家宽；[Claude Code 网络清单](https://code.claude.com/docs/en/network-config#network-access-requirements)中的两个 Datadog 遥测端点也固定走家宽，这两个共享端点上的其他应用流量同样受影响。第三方登录、外链和通用软件仓库仍按各自规则分流。
- [Anthropic 官方 IP 清单](https://platform.claude.com/docs/en/api/ip-addresses)公布的 IPv4 入站范围 `160.79.104.0/23` 已被现有 `/21` 家宽规则覆盖；IPv6 入站范围 `2607:6bc0::/48` 显式拒绝。该 IP 兜底无法识别共享 CDN 上的全部 Claude 流量，域名规则必须保留。
- 按 [Shadowrocket STUN 参数说明](https://t.me/s/shadowrocketnews/491)，`stun-response-ip = 127.0.0.1` 与 `stun-response-ipv6 = ::1` 让识别到的 STUN 请求返回回环地址，避免返回真实公网 IP。这是本地合成的响应，不是另一个代理出口；它覆盖「设置 → UDP → 禁用 STUN」开关，可能影响 WebRTC 音视频或 P2P 通信。

上述保护要求 Shadowrocket VPN 保持连接、使用「配置」模式且「家宽」节点名称有效。STUN 响应替换不等于关闭浏览器 WebRTC，也不能证明浏览器不会暴露本地 ICE 地址；有严格要求时还需限制浏览器 WebRTC。节点协议是否支持远端 DNS、DNS 和 STUN 的实际行为均需在手机上验收。

验收时保持 VPN 和香港节点可用，临时使家宽不可达，重新建立 Claude 连接并检查连接记录：所有受保护的请求必须失败，不能出现香港或 `DIRECT`。恢复家宽后，检查受保护请求的出口，并结合 DNS 抓包与 WebRTC 检测确认没有暴露本地真实公网 IP 或香港出口；普通 DNS 检测站默认走普通代理，不能据此证明 Claude 的解析路径。

### URL 重写

- **Google 域名重定向**：`g.cn` / `google.cn` 自动 302 跳转至 `google.com`，避免访问国内镜像

### 分流策略

| 序号 | 策略 | 规则 | 说明 |
|---|---|---|---|
| 1 | AI-HOME | AI 服务域名、Anthropic IPv4 | 唯一家宽节点，无备用出口 |
| 2 | REJECT | Anthropic IPv6 入站网段 | 与禁用 IPv6 的策略一致 |
| 3 | REJECT | 自定义广告 / 追踪 SDK | 来源 `rules/Reject.list` |
| 4 | DIRECT | 自定义国内直连域名 | 来源 `rules/ChinaDirect.list` |
| 5 | DIRECT | 局域网 / 解禁名单 | ACL4SSR `LocalAreaNetwork.list` / `UnBan.list` |
| 6 | DIRECT | Apple / App Store | ACL4SSR `Apple.list`，**美区账号不清楚是否有风险** |
| 7 | REJECT | 广告域名 | ACL4SSR `BanAD.list` / `BanProgramAD.list` |
| 8 | DIRECT | 国内域名 / 流媒体 | ACL4SSR `ChinaDomain.list` / `ChinaMedia.list` |
| 9 | DIRECT | 国内企业 IP / 国内 IP | ACL4SSR `ChinaCompanyIp.list` / `ChinaIp.list` |
| 10 | DIRECT | `GEOIP,CN` | 内置 GeoIP 兜底 |
| 11 | PROXY | 其他流量 | `FINAL,PROXY,force-remote-dns` 兜底 |


## 维护

### 规则集同步

`RULE-SET` 分两类：

- **ACL4SSR 远程规则**：引用 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 仓库，上游更新后下次刷新订阅自动同步，无需手动维护。
- **自建规则**（`rules/` 目录）：`Reject.list` 和 `ChinaDirect.list` 引用本仓库，手动维护，新增域名推送后下次刷新生效。

### 修改配置

直接编辑 `shadowrocket.conf` 并推送到本仓库，Shadowrocket 下次刷新订阅即生效。

### 自建规则维护

| 文件 | 用途 |
|---|---|
| `rules/Reject.list` | ACL4SSR 未覆盖的广告 / 追踪 SDK 域名，新增后走 REJECT |
| `rules/ChinaDirect.list` | ACL4SSR 未覆盖的国内厂商域名，新增后走 DIRECT |

新增域名前建议先用 `domain-verify` skill 确认归属，避免误添加抢注域名。

### 切换 CDN 源

如发现 `raw.githubusercontent.com` 在国内拉取失败，临时改用 jsDelivr 订阅链接即可。如需让配置文件内部的 `RULE-SET` 也走 jsDelivr，替换以下两处前缀：

**ACL4SSR 规则：**

```
https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/
```
→
```
https://cdn.jsdelivr.net/gh/ACL4SSR/ACL4SSR@master/
```

**自建规则：**

```
https://raw.githubusercontent.com/wxlbd/shadowrocket-config/main/rules/
```
→
```
https://cdn.jsdelivr.net/gh/wxlbd/shadowrocket-config@main/rules/
```

文件路径保持不变。

## v2rayN（已停止维护）

v2rayN 端已于 2026-09-27 停止维护，配置说明、维护流程与排查记录归档至 [docs/v2rayn.md](docs/v2rayn.md)。`v2rayn/` 目录与 `routing.json` 订阅链接继续可用，规则不再更新。

# 许可证

本仓库以 [MIT License](LICENSE) 发布，可自由使用、修改与再分发。

ACL4SSR 规则集通过远程 `RULE-SET` 引用，不随本仓库分发；其版权与许可条款归 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 上游所有（上游声明为 CC-BY-SA-4.0，以该仓库当前声明为准）。

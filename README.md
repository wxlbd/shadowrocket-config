# Shadowrocket 防 DNS 泄露配置

基于 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 规则集的 Shadowrocket 配置文件，国内直连。Claude、谷歌 AI、ChatGPT、Cloudflare 与中国出口回显只走家宽，家宽不可用时失败。其余走代理的流量优先 LISA，LISA 不可用时降级到日本VPS。走代理的域名显式远端解析；国内直连域名和节点域名仍使用国内 UDP DNS，不代表全设备 DNS 查询都加密。

## 订阅链接

复制以下任一链接到 Shadowrocket 进行订阅。

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
5. 全局路由选择「配置」，确认客户端存在名为「家宽」、「LISA」和「日本VPS」的节点。「AI服务」组里只能有「家宽」。`PROXY` 和「其它」只应看到 LISA、日本VPS，且 LISA 在前。节点名必须与配置里的名称一致

## 核心特性

### DNS 防泄露

- **代理域名远端解析**：「AI服务」、「其它」与 `FINAL` 带有 `force-remote-dns`。AI 域名由家宽解析；其余代理域名由 LISA 或日本VPS 解析。IP 规则带 `no-resolve`，避免为了匹配 IP 规则而触发本地解析
- **本地解析仅国内 UDP**：阿里/腾讯承担直连域名与代理节点域名的解析，公共 DNS 按递归源 IP 调度，CDN 就近
- **禁用 IPv6**：避免 v6 通道绕过 DNS 配置造成泄露
- **拒绝私有 IP 应答**：防 DNS rebinding 攻击

### 出口

- `AI服务 = select,家宽`：Claude、谷歌 AI、ChatGPT、Cloudflare 与中国出口回显的出口。组内只有名为「家宽」的节点，类型是手动选择，不是故障转移。家宽不可用时这些连接失败，不会改走 LISA、日本VPS、`PROXY` 或直连。
- `PROXY = fallback,LISA,日本VPS`：没有写进直连表、也没有被「AI服务」命中的境外域名走这里。LISA 可用时用 LISA，LISA 失败后改用日本VPS。
- `其它 = fallback,LISA,日本VPS`：其余谷歌服务、YouTube、X、TikTok、纸飞机和列出的境外视频站，出口与 `PROXY` 相同。
- 除谷歌 AI 接口以外的 `googleapis.com`，以及 `dl.google.com`，走「其它」，不再被 ACL4SSR `UnBan` 直连。

### URL 重写

- **Google 域名重定向**：`g.cn` / `google.cn` 自动 302 跳转至 `google.com`，避免访问国内镜像

### 分流策略

| 序号 | 策略 | 规则 | 说明 |
|---|---|---|---|
| 1 | REJECT-NO-DROP | QUIC（UDP/443） | 立刻返回 ICMP 端口不可达，客户端马上改用 TCP。TCP 不匹配这条 |
| 2 | AI服务 | Claude / Anthropic | 来源 `rules/ClaudeHome.list`，域名规则的第一条，`force-remote-dns`，只走「家宽」，不降级 |
| 3 | AI服务 | ChatGPT / OpenAI | 来源 `rules/OpenAI.list`，`force-remote-dns`，只走「家宽」，不降级 |
| 4 | AI服务 | 谷歌 AI | 来源 `rules/GoogleAI.list`，在 `googleapis.com` 后缀和谷歌关键词之前，`force-remote-dns`，只走「家宽」 |
| 5 | AI服务 | Cloudflare | 来源 `rules/Cloudflare.list`，在国内域名与 `FINAL` 之前，`force-remote-dns`，只走「家宽」，不降级 |
| 6 | AI服务 | 中国出口回显 | 来源 `rules/CnEcho.list`，在 `ChinaDomain` 与 `FINAL` 之前，`force-remote-dns`，只走「家宽」，不降级 |
| 7 | REJECT | 自定义广告 / 追踪 SDK | 来源 `rules/Reject.list` |
| 8 | 其它 | 其余 Google API、`dl.google.com` | 来源 `rules/GooglePin.list`，在 UnBan 之前，`force-remote-dns`，先 LISA，再日本VPS |
| 9 | DIRECT | 自定义国内直连域名 | 来源 `rules/ChinaDirect.list` |
| 10 | DIRECT | 115 网盘 | 来源 `rules/ExtraDirect.list` |
| 11 | DIRECT | 局域网 / 解禁名单 | ACL4SSR `LocalAreaNetwork.list` / `UnBan.list` |
| 12 | DIRECT | Apple / App Store | ACL4SSR `Apple.list`，**美区账号不清楚是否有风险** |
| 13 | REJECT | 广告域名 | ACL4SSR `BanAD.list` / `BanProgramAD.list` |
| 14 | 其它 | 谷歌、YouTube、X、TikTok、纸飞机、境外视频 | 来源 `rules/GoogleMedia.list`，在广告拦截之后、国内域名之前，`force-remote-dns`，先 LISA，再日本VPS |
| 15 | DIRECT | 国内域名 / 流媒体 | ACL4SSR `ChinaDomain.list` / `ChinaMedia.list` |
| 16 | DIRECT | 国内企业 IP / 国内 IP | ACL4SSR `ChinaCompanyIp.list` / `ChinaIp.list` |
| 17 | DIRECT | `GEOIP,CN` | 内置 GeoIP 兜底 |
| 18 | PROXY | 其他流量 | `FINAL,PROXY,force-remote-dns`，先 LISA，再日本VPS |


## 维护

### 规则集同步

`RULE-SET` 分两类：

- **ACL4SSR 远程规则**：引用 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 仓库，上游更新后下次刷新订阅自动同步，无需手动维护。
- **自建规则**（`rules/` 目录）：`ClaudeHome.list`、`OpenAI.list`、`GoogleAI.list`、`Cloudflare.list`、`CnEcho.list`、`Reject.list`、`ChinaDirect.list`、`ExtraDirect.list`、`GooglePin.list`、`GoogleMedia.list` 引用本仓库，手动维护，新增域名推送后下次刷新生效。

### 修改配置

直接编辑 `shadowrocket.conf` 并推送到本仓库，Shadowrocket 下次刷新订阅即生效。

### 自建规则维护

| 文件 | 用途 |
|---|---|
| `rules/Reject.list` | ACL4SSR 未覆盖的广告 / 追踪 SDK 域名，新增后走 REJECT |
| `rules/ChinaDirect.list` | ACL4SSR 未覆盖的国内厂商域名，新增后走 DIRECT |
| `rules/ExtraDirect.list` | 115 网盘直连。`DOMAIN-KEYWORD,115` 会命中主机名任意位置的 `115` |
| `rules/ClaudeHome.list` | Claude / Anthropic，走「AI服务」。必须排在其余规则之前 |
| `rules/OpenAI.list` | ChatGPT / OpenAI，走「AI服务」。必须排在其余规则之前 |
| `rules/GoogleAI.list` | 谷歌 AI，走「AI服务」。必须排在 `GooglePin` 和谷歌关键词之前 |
| `rules/Cloudflare.list` | Cloudflare 官方域名与公共 DNS，走「AI服务」。必须排在国内域名和 `FINAL` 之前 |
| `rules/CnEcho.list` | 检测站的中国出口回显，走「AI服务」。必须排在 `ChinaDomain` 和 `FINAL` 之前 |
| `rules/GooglePin.list` | 其余 `googleapis.com` 与 `dl.google.com`，走「其它」。必须排在 UnBan 之前 |
| `rules/GoogleMedia.list` | 谷歌、YouTube、X、TikTok、纸飞机、境外视频，走「其它」。必须排在广告拦截之后 |

新增直连域名前先用 `domain-verify` skill 确认归属，避免误添加抢注域名。

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

# 许可证

本仓库以 [MIT License](LICENSE) 发布，可自由使用、修改与再分发。

ACL4SSR 规则集通过远程 `RULE-SET` 引用，不随本仓库分发；其版权与许可条款归 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 上游所有（上游声明为 CC-BY-SA-4.0，以该仓库当前声明为准）。

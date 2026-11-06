# Shadowrocket 防 DNS 泄露配置

基于 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 规则集的 Shadowrocket 配置文件，国内直连，Claude / Anthropic 与其余未匹配的境外流量走家宽，谷歌、YouTube、X、TikTok、纸飞机与境外视频走日本节点。走代理的域名显式远端解析；国内直连域名和节点域名仍使用国内 UDP DNS，不代表全设备 DNS 查询都加密。

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
5. 全局路由选择「配置」，确认客户端存在名为「家宽」的节点，以及「谷歌媒体」组里的「日本VPS」和其余日本节点。节点名必须与配置里的名称一致

## 核心特性

### DNS 防泄露

- **代理域名远端解析**：Claude 与「谷歌媒体」规则集带有 `force-remote-dns`，其余未匹配域名由 `FINAL,PROXY,force-remote-dns` 兜底并走家宽；IP 规则带 `no-resolve`，避免为了匹配 IP 规则而触发本地解析
- **本地解析仅国内 UDP**：阿里/腾讯承担直连域名与代理节点域名的解析，公共 DNS 按递归源 IP 调度，CDN 就近
- **禁用 IPv6**：避免 v6 通道绕过 DNS 配置造成泄露
- **拒绝私有 IP 应答**：防 DNS rebinding 攻击

### 出口

- `PROXY = select,家宽`：策略组里只有名为「家宽」的节点。Claude / Anthropic 由 `rules/ClaudeHome.list` 显式命中这个组；没有写进直连表、也没有被「谷歌媒体」命中的境外域名同样落在这里。
- `谷歌媒体`：谷歌、YouTube、X、TikTok、纸飞机和列出的境外视频站默认走「日本VPS」，组内其余节点可手动切换。节点名必须与客户端里的名称一致，包括名称中的空格和 `|`。
- `googleapis.com` 整段后缀和 `dl.google.com` 走「谷歌媒体」，不再被 ACL4SSR `UnBan` 直连。

### URL 重写

- **Google 域名重定向**：`g.cn` / `google.cn` 自动 302 跳转至 `google.com`，避免访问国内镜像

### 分流策略

| 序号 | 策略 | 规则 | 说明 |
|---|---|---|---|
| 1 | PROXY | Claude / Anthropic | 来源 `rules/ClaudeHome.list`，排在最前，`force-remote-dns`，走「家宽」 |
| 2 | REJECT | 自定义广告 / 追踪 SDK | 来源 `rules/Reject.list` |
| 3 | 谷歌媒体 | 谷歌 AI、`dl.google.com` | 来源 `rules/GooglePin.list`，在 UnBan 之前，`force-remote-dns` |
| 4 | DIRECT | 自定义国内直连域名 | 来源 `rules/ChinaDirect.list` |
| 5 | DIRECT | 115 网盘 | 来源 `rules/ExtraDirect.list` |
| 6 | DIRECT | 局域网 / 解禁名单 | ACL4SSR `LocalAreaNetwork.list` / `UnBan.list` |
| 7 | DIRECT | Apple / App Store | ACL4SSR `Apple.list`，**美区账号不清楚是否有风险** |
| 8 | REJECT | 广告域名 | ACL4SSR `BanAD.list` / `BanProgramAD.list` |
| 9 | 谷歌媒体 | 谷歌、YouTube、X、TikTok、纸飞机、境外视频 | 来源 `rules/GoogleMedia.list`，在广告拦截之后、国内域名之前，`force-remote-dns` |
| 10 | DIRECT | 国内域名 / 流媒体 | ACL4SSR `ChinaDomain.list` / `ChinaMedia.list` |
| 11 | DIRECT | 国内企业 IP / 国内 IP | ACL4SSR `ChinaCompanyIp.list` / `ChinaIp.list` |
| 12 | DIRECT | `GEOIP,CN` | 内置 GeoIP 兜底 |
| 13 | PROXY | 其他流量 | `FINAL,PROXY,force-remote-dns`，走「家宽」 |


## 维护

### 规则集同步

`RULE-SET` 分两类：

- **ACL4SSR 远程规则**：引用 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 仓库，上游更新后下次刷新订阅自动同步，无需手动维护。
- **自建规则**（`rules/` 目录）：`Reject.list`、`ChinaDirect.list`、`ExtraDirect.list`、`GooglePin.list`、`GoogleMedia.list` 引用本仓库，手动维护，新增域名推送后下次刷新生效。

### 修改配置

直接编辑 `shadowrocket.conf` 并推送到本仓库，Shadowrocket 下次刷新订阅即生效。

### 自建规则维护

| 文件 | 用途 |
|---|---|
| `rules/Reject.list` | ACL4SSR 未覆盖的广告 / 追踪 SDK 域名，新增后走 REJECT |
| `rules/ChinaDirect.list` | ACL4SSR 未覆盖的国内厂商域名，新增后走 DIRECT |
| `rules/ExtraDirect.list` | 115 网盘直连。`DOMAIN-KEYWORD,115` 会命中主机名任意位置的 `115` |
| `rules/ClaudeHome.list` | Claude / Anthropic，走「家宽」。必须排在其余规则之前 |
| `rules/GooglePin.list` | 谷歌 AI 与 `dl.google.com`，走「谷歌媒体」。必须排在 UnBan 之前 |
| `rules/GoogleMedia.list` | 谷歌、YouTube、X、TikTok、纸飞机、境外视频，走「谷歌媒体」。必须排在广告拦截之后 |

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

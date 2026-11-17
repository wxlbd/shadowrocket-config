# AGENTS.md

本文件给维护这个仓库的自动化助手使用，不绑定某一家产品。修改分流规则前先通读。运行时故障先看 [docs/troubleshooting.md](docs/troubleshooting.md)，不要把系统层症状改成分流规则。

## 仓库做什么

Shadowrocket 代理分流配置。国内域名和 IP 直连，境外流量按策略组出口，并避免 DNS 泄露与污染。

`rules/*.list` 是规则的唯一数据源。`shadowrocket.conf` 用远程 `RULE-SET` 引用它们，订阅更新后生效。不要再加构建步骤，也不要在主配置里手写一份重复域名。

## 文件

| 路径 | 作用 | 是否手改 |
|------|------|---------|
| `shadowrocket.conf` | 主配置：DNS、策略组、Rule、URL Rewrite | 手改 |
| `rules/ChinaDirect.list` | 国内域名直连补充 | 手改，唯一数据源 |
| `rules/ExtraDirect.list` | 115 网盘直连 | 手改，唯一数据源 |
| `rules/Reject.list` | 自定义广告与追踪拦截 | 手改，唯一数据源 |
| `rules/ClaudeHome.list` | Claude / Anthropic，走「AI服务」 | 手改，唯一数据源 |
| `rules/OpenAI.list` | ChatGPT / OpenAI，走「AI服务」 | 手改，唯一数据源 |
| `rules/GoogleAI.list` | 谷歌 AI，走「AI服务」 | 手改，唯一数据源 |
| `rules/Cloudflare.list` | Cloudflare 官方域名与公共 DNS，走「AI服务」 | 手改，唯一数据源 |
| `rules/CnEcho.list` | 中国出口回显，走「AI服务」 | 手改，唯一数据源 |
| `rules/GooglePin.list` | 其余 `googleapis.com` 与 `dl.google.com`，走「其它」 | 手改，唯一数据源 |
| `rules/GoogleMedia.list` | 谷歌 / YouTube / X / TikTok / 纸飞机 / 境外视频，走「其它」 | 手改，唯一数据源 |

新增一类流量时，新建或扩展对应的 `.list`，并在 `shadowrocket.conf` 里用一条 `RULE-SET` 指向它。不要把同一批域名写进两个列表。

## 路由顺序

分流按首条匹配生效，顺序就是优先级。`[Rule]` 当前自上而下是：

`ClaudeHome → OpenAI → GoogleAI → Cloudflare → CnEcho → Reject → GooglePin → ChinaDirect → ExtraDirect → ACL4SSR 局域网 / UnBan / Apple → BanAD → GoogleMedia → ChinaDomain 等 → GEOIP,CN → FINAL`

调整顺序或策略组之前，先写明这条规则现在会被谁截走、改完后会落到哪个组。

### 策略组

| 组 | 类型与成员 | 失败时 |
|---|---|---|
| `AI服务` | `select`，只有名为「家宽」的节点 | 连接失败，不改出口 |
| `PROXY` | `fallback`，先 `LISA`，再 `日本VPS` | LISA 不可用时改用日本VPS |
| `其它` | 与 `PROXY` 相同 | 与 `PROXY` 相同 |

约束：

- `AI服务` 不得加入 `LISA`、`日本VPS`、`PROXY`、`其它`、`DIRECT`，不得改成 `url-test` 或 `fallback`。名字里带「家宽」的日本节点不是「家宽」，禁止用正则收进这个组。
- `ClaudeHome`、`OpenAI`、`GoogleAI`、`Cloudflare`、`CnEcho` 必须排在其余规则之前，策略必须是 `AI服务`，并带 `force-remote-dns`。指向 `PROXY` 或「其它」，或晚于 `GooglePin`、谷歌关键词、`ChinaDomain`，这些域名会改走 LISA / 日本VPS 或直连。`CnEcho` 晚于 `ChinaDomain` 时，`my.ip.cn` 被国内域名直连，`ip138.com` 落到 `FINAL`。
- `PROXY` 与「其它」只放 `LISA`、`日本VPS`，类型保持 `fallback`，`LISA` 写在前面。不得加入「家宽」或 `AI服务`。
- `GooglePin` 必须在 `UnBan` 之前，并带 `force-remote-dns`。`dl.google.com` 在 UnBan 里是直连。谷歌 AI 域名不放在这里。
- `GoogleMedia` 必须在 `BanProgramAD` 之后、`ChinaDomain` 之前，并带 `force-remote-dns`。宽关键词放前面会接管本应拒绝的广告域名，放后面会被国内域名规则截走。
- 自定义直连必须在广告拦截之前，否则会被 `BanAD` 先拦掉。
- `FINAL` 必须在末尾，匹配剩余流量并走 `PROXY`。前移会吞掉后面的规则；AI 规则若被它盖住，就会离开家宽。

## 协议与地址

- `shadowrocket.conf` 保持 `ipv6 = false` 和 `prefer-ipv6 = false`。
- `stun-response-ip = 127.0.0.1` 与 `stun-response-ipv6 = ::1` 只返回本地合成的 STUN 响应，不得改成真实家宽 IP。这两个参数覆盖客户端「禁用 STUN」开关，可能影响 WebRTC 通话。不要把它们写成已关闭 WebRTC，或已保证浏览器不会露出地址信息。

## 新增直连域名

先用仓库内的 domain-verify 技能（`.claude/skills/domain-verify`）确认域名真实存在、归属可信、不是抢注。查询命令和判断标准以该技能为准，不要把核查过程抄进规则文件。

不得添加：

- 没有 NS 记录的域名，包括不存在或拼写错误的名字。
- 疑似挂售的域名。TXT 中出现 afternic、sedo、dan.com 等即属此类。

归属主体无法确认、且没有挂售特征时，满足下面任一条件才可作特例：

- NS 在国内托管商，如阿里云、腾讯 DNSPod、华为云。按国内业务直连。
- 站点排斥代理出口 IP。依赖代理会触发风控，需要按域名直连。

特例只放在 `ChinaDirect.list` 末尾「归属未确认的特例」分区。收录依据写在分区标题上，不给每条域名单独注释。归属明确后，移到上方对应分组或删除。

`ChinaDirect.list` 只补充 ACL4SSR `ChinaDomain.list` 没有覆盖的域名。`.com.cn`、`.cn`，以及上游已有的域名，不要再手写。IP 兜底带 `no-resolve`，不会为域名请求触发解析，不能当作域名兜底。需要手补的通常是 `.com` 顶级域、且上游列表没有覆盖的国内业务。

## 写法

- 同一机构的多个子域用 `DOMAIN-SUFFIX`。一条 `example.com` 覆盖 `www.example.com` 和 `api.example.com`。只有必须精确命中一个主机名时才用 `DOMAIN`。
- 分组注释用一行：`# 中文名 (运营主体公司)`。

```
# 猎聘 (同道精英（天津）信息技术有限公司)
DOMAIN-SUFFIX,liepin.com
DOMAIN-SUFFIX,lietou-static.com
```

注释里不写解析 IP、NS、SPF、MX、TXT、证书主体，也不逐条重复「上游未收录」「需要域名级直连」这类全表都适用的理由。

两类注释保留：

| 场合 | 写法 |
|---|---|
| 海外域名却要直连 | `# iHerb 海淘保健品电商 (美国公司，非国内域名；直连避免代理触发风控拒单)` |
| 末尾特例区 | 分区标题写明依据：NS 在国内托管商，或站点排斥代理出口 IP |

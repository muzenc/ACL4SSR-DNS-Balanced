# ACL4SSR DNS Profiles

面向 **Clash、Mihomo/Clash Meta 和 Shadowrocket** 的 ACL4SSR/subconverter 远程模板。仓库同时提供平衡版和 Shadowrocket 严格 DNS 版。

目标是在以下需求之间取得平衡：

- 国内域名和中国 IP 尽量直连；
- 未收录域名先解析再用 `GEOIP,CN` 按 IP 判断，不依赖规则集覆盖；
- 国外和未知流量默认走代理；
- DNS 全部使用 DoH，避免传统 UDP 53 明文查询；
- 不生成美国、日本等国家分组；
- 显式让 `todesk.com` 直连。

## 文件

- `ACL4SSR_DNS_Balanced.ini`：ACL4SSR/subconverter 外部配置，控制规则顺序和代理分组。
- `Clash_DNS_Balanced.yml`：Clash 基础配置，启用加密 DNS、国内解析和国外回退。
- `ACL4SSR_DNS_Strict.ini`：为 Shadowrocket 生成 Clash 订阅的严格 DNS 外部配置。
- `Clash_DNS_Strict.yml`：全部域名由境外 DoH 经代理解析（国内域名附带中国 ECS），再按 IP 判定走向。

## 远程配置地址

### 严格版（Shadowrocket 推荐）

如果不希望国外网站的权威 DNS 看到中国递归 DNS 出口，请使用：

`https://raw.githubusercontent.com/muzenc/ACL4SSR-DNS-Balanced/main/ACL4SSR_DNS_Strict.ini`

在 ACL4SSR 中选择 **Clash** 作为目标，生成的 URL 同时适用于 Clash Verge 和 Shadowrocket。

### 平衡版

在 ACL4SSR 在线订阅转换的“远程配置/自定义配置”中填写：

`https://raw.githubusercontent.com/muzenc/ACL4SSR-DNS-Balanced/main/ACL4SSR_DNS_Balanced.ini`

两种版本的操作步骤相同：

1. 进入高级模式并选择上面的远程配置。
2. 目标客户端选择 **Clash**；页面如果有 `Clash.DoH`，保持勾选。
3. 生成新的订阅 URL。
4. Clash 直接导入生成的 URL；Shadowrocket 也导入同一个生成后的 Clash URL。
5. 客户端使用“规则/配置”路由模式，不需要使用全局代理。

> 不要公开生成后的订阅 URL。它可能包含节点 UUID、密码或其他凭据。

## `FINAL 保持代理`是什么意思

模板最后一条是：

`ruleset=🌐 未知站点,[]FINAL`

`FINAL` 代表前面所有规则都没有匹配到的连接。`🌐 未知站点` 分组的第一个选项是 `✈️ 代理`，因此默认行为是：

- 未知国外网站不会误直连；
- ChatGPT、Claude 等即使暂时未被规则集收录，也会走代理；
- 不要在客户端里把 `🌐 未知站点` 手动切换成 `🎯 国内`。

## DNS 与分流逻辑

1. 已知国内域名直接连接。
2. 已知国外及 AI 域名走代理。
3. 未知域名通过加密 DNS 解析。
4. 解析到中国 IP 时，`GEOIP,CN` 使其直连。
5. 解析到非中国 IP 时，由 `FINAL` 兜底走代理。

两版都使用 `GEOIP,CN` 且不带 `no-resolve`，这是 ToDesk、`frp-arm.com` 等未收录国内域名仍能靠 IP 直连的关键。

区别在**谁来做第 3 步的解析**：平衡版用国内 DoH（快，但国内 DNS 看得到你的查询），严格版用经代理的境外 DoH（不泄露，但慢）。详见下面「严格版的 DNS 结构」。

## 严格版的 DNS 结构

**不依赖规则集覆盖，且境内外各用各的解析视角。** 全部解析器都在境外（零国内 DNS），靠三层结构决定每个域名用哪个「视角」的答案：

| 层 | 命中范围 | 解析器 | 目的 |
| --- | --- | --- | --- |
| `nameserver-policy` `geosite:cn` | 已知国内域名 | Google DoH + 中国 ECS，经代理 | 钉死 ECS 分支，bilibili 永不被换成美国答案 |
| `nameserver-policy` `geosite:geolocation-!cn` | 已知境外域名 | 1.1.1.1，无 ECS，经代理 | 钉死干净分支，github 不会拿到新加坡节点 |
| `nameserver`(ECS) + `fallback`(无 ECS) + `fallback-filter` | **其余未知域名** | 两支并发，按结果国别自动分拣 | 未收录域名不需要任何人工维护 |

第三层是关键，用的是经典 Clash 防污染机制的**反向用法**——不是拿它防投毒，而是拿它挑选 ECS 视角。mihomo 官方文档对 `fallback-filter.geoip` 的原文：「geoip-code 配置的国家的结果会直接采用，否则将采用 fallback 结果」。于是：

- 未知域名的 ECS 答案是国内 IP（`frp-arm.com` → 西安联通）→ 采信 → `GEOIP,CN` 直连；
- 未知域名的 ECS 答案是境外 IP（`macked.app` → Cloudflare）→ 换用 1.1.1.1 的无 ECS 答案 → 走代理，按美国出口视角落点。

`fallback` 是**并发**查询而非失败后备（`fallback-lazy-query` 默认 false），所以这个分拣不增加延迟。

### 为什么已知域名要用 policy 钉死，不能全交给 filter

实测（mihomo v1.19.25）：**policy 命中的域名跳过 fallback-filter**——官方文档「优先于 nameserver/fallback 查询」，探针实验证实。这个特性被反过来利用：

Google 对 bilibili 权威的 ECS 应答有约 25% 概率（8 次采样 2 次）返回其**香港段** `103.151.151.x`。若交给 filter，非 CN 答案会被换成 1.1.1.1 的美国 Zenlayer 答案，而 `bilibili.com` 命中域名规则走 `DIRECT`——变成从国内直连美国服务器，就是此前 bilibili 打不开的机制。用 policy 钉死 ECS 分支后，最坏情况是偶尔拿到 bilibili 自家香港边缘，仍然可用。

`nameserver-policy` 仅 mihomo 识别。Shadowrocket 忽略它后**恰好退化为纯三字段方案**（`nameserver`/`fallback`/`fallback-filter` 都是老 Clash 字段）；且已知境外域名在 Shadowrocket 上属于代理类，「代理类域名将经由代理服务器进行解析」（官方手册），本地 DNS 根本不参与，policy 缺失不影响它们的落点。

### 为什么必须带 ECS

不带 ECS 时国内 CDN 按「提问者在境外」分配节点，会把你导向海外边缘节点：

| 域名 | 无 ECS 解析到 | 带华北联通 ECS |
| --- | --- | --- |
| `www.bilibili.com` | `192.254.90.178` 美国洛杉矶 | `221.204.56.86` 太原 |
| `i0.hdslb.com` | `138.113.102.14` 加拿大多伦多 | `218.11.15.28` 承德 |

而 `bilibili.com` 命中域名规则被判 `DIRECT`，于是变成**从国内直连一台美国服务器**。这个故障非常隐蔽——规则层日志显示 `📺 B站[DIRECT]` 完全正确，问题全在解析结果上，必须把解析到的 IP 拉出来验归属才能发现。

ECS 相关的工程细节：

- 网段粒度用 /24——比这更细会被上游拒绝（明文 REFUSED）；
- ECS 分支只用 Google（8.8.8.8 / 8.8.4.4，同一 anycast 服务）；干净分支只用 1.1.1.1——Cloudflare 按其隐私政策**永不转发 ECS**，这是「无 ECS」保证的来源，两支不可混用；
- `#proxy&ecs=...&ecs-override=true` 的多参数语法在 mihomo 官方文档和 Shadowrocket 社区手册中同形记载（`&` 串联）；
- 换成你所在地运营商的网段更优：联通北京 `202.106.0.0/24`（当前）、电信上海 `202.96.209.0/24`、电信广东 `202.96.128.0/24`、移动 `211.136.192.0/24`。

### 代价与边界

- 所有未缓存域名的解析都经代理往返（实测首次 278–843ms，缓存后 16–17ms），影响集中在每个域名的首次访问；
- 少数国内站点权威不认 ECS（`www.baidu.com`）或用全球 CDN（`ctrip.com`），仍解析到境外 IP，由 `ChinaDomain.list` 按域名兜住（位置在 `GEOIP,CN` 之前）；
- 在中国部署了 CDN 的境外站点（如部分 Apple/Microsoft 域名）ECS 答案可能是国内 IP，会被判直连——这与主流规则集的处理方向一致，是期望行为。

## 隐私边界

这是“速度与 DNS 隐私的平衡方案”，不是数学意义上的零泄露：

- DoH 会加密设备到 DNS 服务商之间的查询，局域网和运营商通常看不到明文域名；
- DNS 服务商仍然能够看到它负责解析的域名；
- 为了让未知域名经过 `GEOIP,CN` 判断，客户端必须先解析它；
- 严格版配置中不含任何国内 DNS，国内 DNS 服务商看不到你的任何查询；代价是解析要经代理往返；
- 平衡版会把国内域名和未收录的未知域名都交给国内 DNS，这是它换取速度的代价；
- 不同版本的 Shadowrocket 对 Clash DNS 字段的导入支持可能不同。导入后应确认 DNS 服务器是 `https://` 而不是 `system`。

## 严格版验收

导入严格版生成的 URL 后，应确认：

1. Shadowrocket 使用“配置”路由模式；
2. `🤖 AI` 和 `🌐 未知站点` 均选择 `✈️ 代理`；
3. <https://browserleaks.com/dns> 或 <https://dnsleaktest.com> 不出现中国联通、电信、移动或任何中国地区解析器；
4. bilibili、ToDesk 等国内服务在日志中仍显示直连，且访问不慢。

实测结果（mihomo v1.19.25 + subconverters.com 生成的真实订阅 + 真实节点）：

| 项目 | 结果 |
| --- | --- |
| 泄露检测解析器 | 全部美国 Google/Cloudflare，零中国解析器 |
| `frp-arm.com`（规则集未收录） | ECS 答案西安联通 → `GeoIP(cn)` → `DIRECT` |
| `www.bilibili.com` | policy 钉 ECS → 太原联通，`DIRECT` |
| `github.com` / `www.amazon.com` | policy 钉无 ECS → 美国 Quincy / 洛杉矶 |
| `macked.app`（未知境外） | filter 换用干净答案 → 走代理 |
| Cloudflare 边缘核验 | `colo=LAS`（美国），非亚太 |

Shadowrocket 侧一项无法在本机验证：它对 Clash `fallback` / `fallback-filter` 字段的支持程度。导入后请在 DNS 设置里确认这两组服务器都存在、`#proxy` 与 `ecs=` 参数未丢失；若 filter 不生效，退化行为是未知境外域名偶尔拿到亚太节点（经代理仍可用），已知域名不受影响。

## 陌生国内域名怎么处理

不需要处理。未被规则集收录的域名会先解析再按 IP 判定，解析出国内 IP 就直连。实测 `frp-arm.com` 即属此类。

少数站点的权威服务器不认 ECS（`www.baidu.com`）或使用全球 CDN（`ctrip.com`、`qunar.com`），即使带 ECS 也会解析到境外 IP，`GEOIP,CN` 会判错。它们由 `ChinaDomain.list` 按域名兜住，该规则集位置在 `GEOIP,CN` 之前。若你遇到别的这类站点，加一条：

```ini
ruleset=🎯 国内,[]DOMAIN-SUFFIX,example.com
```

## 上游规则

规则主要来自 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 和 [zsokami/ACL4SSR](https://github.com/zsokami/ACL4SSR)。

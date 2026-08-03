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

**不依赖规则集覆盖**。全部域名由境外 DoH 经代理解析，解析结果交给 `GEOIP,CN` 判定走直连还是走代理；规则集只负责已知的例外。规则集再大也不可能覆盖所有网站，所以兜底必须由 IP 来做。

| 域名 | 解析器 | 判定 |
| --- | --- | --- |
| 命中国内域名规则 | 境外 DoH + 中国 ECS，经代理 | `DIRECT` |
| `geosite:geolocation-!cn` 命中 | 境外 DoH，无 ECS，经代理 | 按规则或 `FINAL` 走代理 |
| 其余全部（未知） | 境外 DoH + 中国 ECS，经代理 | 解析出国内 IP 则 `GEOIP,CN` 直连，否则代理 |

实测 `frp-arm.com`（西安联通，不在任何规则集里）命中 `GeoIP(cn)` 正确直连。

### 为什么必须带 ECS

不带 ECS 时国内 CDN 按「提问者在境外」分配节点，会把你导向海外边缘节点：

| 域名 | 无 ECS 解析到 | 带华北联通 ECS |
| --- | --- | --- |
| `www.bilibili.com` | `192.254.90.178` 美国洛杉矶 | `221.204.56.86` 太原 |
| `i0.hdslb.com` | `138.113.102.14` 加拿大多伦多 | `218.11.15.31` 承德 |

而 `bilibili.com` 命中域名规则被判 `DIRECT`，于是变成**从国内直连一台美国服务器**：不走代理所以没有隧道加速，纯国际直连又慢又容易被掐，那些节点回的还是国际版内容。

这个故障非常隐蔽——**规则层看起来完全正确**，日志里明明白白写着 `📺 B站[DIRECT]`。问题全在解析结果上，只看路由日志永远查不出来，必须把解析到的 IP 拉出来验归属。

### 为什么境外域名不带 ECS

带上中国 ECS 会让境外站点返回亚太边缘节点，而你是从代理出口访问的，等于绕远。实测 `github.com` 带 ECS 解析到新加坡 `20.205.243.166`，不带则是美国 `20.29.134.23`——节点在洛杉矶，后者明显更优。顺带也不必告诉境外站点你在中国。

未被 `geosite` 收录的域名仍走带 ECS 的默认解析器，这是有意的：未知域名更可能是国内站点，带 ECS 才能拿到正确的国内 IP 供 `GEOIP,CN` 判定。

`nameserver-policy` 仅 Mihomo 识别，Shadowrocket 会忽略，其行为退化为全部带 ECS——仍然可用，只是境外站点的选点不够优。

### 代价：DNS 延迟

所有未缓存域名的解析都要经代理往返。实测经节点查境外 DoH 平均 1341ms，直连国内 DoH 平均 310ms。命中缓存后为 16–17ms，因此影响集中在每个域名的首次访问。

这是「不依赖规则集」必须付的账：要让未知域名被正确判定，就必须先解析它；要不泄露，解析就必须走境外。

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
| 泄露检测解析器 | 12/12 美国 Google，零中国解析器 |
| `frp-arm.com`（规则集未收录） | `GeoIP(cn)` → `DIRECT`，解析到西安联通 |
| `www.bilibili.com` | `DIRECT`，解析到 `221.204.56.86` 太原 |
| `github.com` | 走代理，解析到美国而非亚太（无 ECS 支生效） |
| DNS 解析耗时 | 首次 278–843ms，命中缓存 16–17ms |

## 陌生国内域名怎么处理

不需要处理。未被规则集收录的域名会先解析再按 IP 判定，解析出国内 IP 就直连。实测 `frp-arm.com` 即属此类。

少数站点的权威服务器不认 ECS（`www.baidu.com`）或使用全球 CDN（`ctrip.com`、`qunar.com`），即使带 ECS 也会解析到境外 IP，`GEOIP,CN` 会判错。它们由 `ChinaDomain.list` 按域名兜住，该规则集位置在 `GEOIP,CN` 之前。若你遇到别的这类站点，加一条：

```ini
ruleset=🎯 国内,[]DOMAIN-SUFFIX,example.com
```

## 上游规则

规则主要来自 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 和 [zsokami/ACL4SSR](https://github.com/zsokami/ACL4SSR)。

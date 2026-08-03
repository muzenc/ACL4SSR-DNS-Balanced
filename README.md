# ACL4SSR DNS Profiles

面向 **Clash、Mihomo/Clash Meta 和 Shadowrocket** 的 ACL4SSR/subconverter 远程模板。仓库同时提供平衡版和 Shadowrocket 严格 DNS 版。

目标是在以下需求之间取得平衡：

- 国内域名和中国 IP 尽量直连；
- 平衡版：未收录域名允许解析 IP，再用 `GEOIP,CN` 判断；严格版用 `no-resolve` 关闭该行为；
- 国外和未知流量默认走代理；
- DNS 全部使用 DoH，避免传统 UDP 53 明文查询；
- 不生成美国、日本等国家分组；
- 显式让 `todesk.com` 直连。

## 文件

- `ACL4SSR_DNS_Balanced.ini`：ACL4SSR/subconverter 外部配置，控制规则顺序和代理分组。
- `Clash_DNS_Balanced.yml`：Clash 基础配置，启用加密 DNS、国内解析和国外回退。
- `ACL4SSR_DNS_Strict.ini`：严格版（**Shadowrocket 用这份**）。只有直连域名走本地（国内 DoH）解析，代理域名交由节点远端解析。
- `Clash_DNS_Strict.yml`：严格版的 Clash 基础配置。
- `ACL4SSR_DNS_Strict_Mihomo.ini` / `Clash_DNS_Strict_Mihomo.yml`：Clash Verge 专用增强版，零国内 DNS + 未知域名按 IP 判定，依赖 Shadowrocket 没有的字段与行为。

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

**平衡版**使用 `GEOIP,CN` 且不带 `no-resolve`，这是 ToDesk 等未收录国内域名仍能靠 IP 判断直连的关键。

**严格版**给 `GEOIP,CN` 加了 `no-resolve`，因此第 3–5 步不适用于它——未命中域名规则的连接不再触发本地解析，直接交给代理。详见下面「严格版的 DNS 结构」。

## 严格版的 DNS 结构

核心思路：让**需要本地解析的域名**和**不该被国内 DNS 看到的域名**完全不重叠。

| 域名类型 | 路由 | 谁来解析 |
| --- | --- | --- |
| 命中国内规则集 | `DIRECT` | 本地国内 DoH（`223.5.5.5` / `1.12.12.12`），快 |
| 其余全部 | 代理 | 域名直接交给节点，**远端解析**，本地不产生任何 DNS 查询 |

这依赖 `.ini` 中 `GEOIP,CN` 带 `no-resolve`：未命中国内域名规则的连接不再触发本地解析。**去掉 `no-resolve`，境外域名会被上面的国内 DoH 解析，泄露检测立刻会看到中国解析器。**

Shadowrocket 的原生语义与此完全一致，其官方手册写明：「DNS 覆写仅针对直连类域名进行解析，代理类域名将经由代理服务器进行解析」。

早先的版本把全部 DNS 用 `#proxy` 压进代理，结果国内域名每次解析都要跨太平洋两趟，实测 `www.meituan.com` 要 2089ms、`ctrip.com` 772ms；改为本方案后分别是 145ms 和 76ms。

### 不要改成「全部域名一律用境外 DNS」

这个念头很容易冒出来：所有域名都交给境外 DNS 解析，再靠 `GEOIP,CN` 判断走直连还是走代理——逻辑最干净，不用维护任何国内域名列表，也绝无国内 DNS 泄露。

**这个方案会让 bilibili 之类的站点彻底打不开**，而且故障非常隐蔽。实测：

| 域名 | 境外 DNS（无 ECS）解析到 | 归属 |
| --- | --- | --- |
| `www.bilibili.com` | `192.254.90.178` | 美国洛杉矶 Zenlayer |
| `api.bilibili.com` | `148.153.45.10` | 美国洛杉矶 |
| `i0.hdslb.com` | `138.113.102.14` | 加拿大多伦多 |

境外 DNS 把你导向了 bilibili 给**海外用户**准备的边缘节点。而 `bilibili.com` 命中 `DOMAIN-SUFFIX` 规则被判 `DIRECT`，于是变成**从国内直连一台美国服务器**：不走代理所以没有隧道加速，纯国际直连又慢又容易被掐，那些节点回的还是国际版内容。

关键在于**规则层看起来完全正确**——日志里明明白白写着 `📺 B站[DIRECT]`，完全符合预期。问题全在解析结果上，只看路由日志永远查不出来，必须把解析到的 IP 拉出来验归属。

同一份域名规则配上不同来源的 DNS，会得到完全不同的结果。本方案让直连域名走国内 DoH，拿到的必然是国内节点，从根上避免了这种「规则说直连、IP 在境外」的错配：

| 域名 | 国内 DoH 解析到 | 归属 |
| --- | --- | --- |
| `www.bilibili.com` | `117.23.60.12` | 陕西 |
| `api.bilibili.com` | `106.118.30.8` | 石家庄 |
| `i0.hdslb.com` | `36.104.142.56` | 杭州 |

## 隐私边界

这是“速度与 DNS 隐私的平衡方案”，不是数学意义上的零泄露：

- DoH 会加密设备到 DNS 服务商之间的查询，局域网和运营商通常看不到明文域名；
- DNS 服务商仍然能够看到它负责解析的域名；
- 为了让未知域名经过 `GEOIP,CN` 判断，客户端必须先解析它；
- 严格版中，国内 DNS 只看得到被规则集判定为国内的域名，看不到你访问的境外站点；
- 平衡版会把国内域名和未收录的未知域名都交给国内 DNS，这是它换取速度的代价；
- 不同版本的 Shadowrocket 对 Clash DNS 字段的导入支持可能不同。导入后应确认 DNS 服务器是 `https://` 而不是 `system`。

## 严格版验收

导入严格版生成的 URL 后，应确认：

1. Shadowrocket 使用“配置”路由模式；
2. `🤖 AI` 和 `🌐 未知站点` 均选择 `✈️ 代理`；
3. <https://browserleaks.com/dns> 或 <https://dnsleaktest.com> 不出现中国联通、电信、移动或任何中国地区解析器；
4. bilibili、ToDesk 等国内服务在日志中仍显示直连，且访问不慢。

实测结果（mihomo v1.19.25 + subconverters.com 生成的真实订阅）：

| 项目 | 结果 |
| --- | --- |
| 泄露检测解析器 | 12/12 美国 Google，零中国解析器 |
| 境外域名本地 DNS 记录 | 0 条 |
| 国内域名本地 DNS 记录 | 12 条，全部走国内 DoH |
| 国内域名解析耗时 | 76–145ms |

## 陌生国内域名怎么处理

`GEOIP,CN` 带了 `no-resolve`，好处是境外域名完全不碰本地 DNS，代价是**没被规则集收录的国内域名不再靠 IP 兜底判直连**，会走代理。

因此严格版额外引入了 `ChinaMax.list`（12.4 万条），把这个缺口压到最小。实测 `frp-arm.com`（西安联通，不在 ACL4SSR 的 `ChinaDomain.list` 里）由此正确直连。

代价是订阅体积：440KB / 9,713 条 → 5.6MB / 134,193 条。如果 Shadowrocket 导入变慢或不稳，删掉 `ChinaMax.list` 那一行即可退回小规则集，再按需逐条补：

```ini
ruleset=🎯 国内,[]DOMAIN-SUFFIX,example.cn
```

### 为什么要把泄露检测站强制走代理

`browserleaks.com` 被 `ChinaMax` 收录（该表的定义是"国内可直连"，不是"中国站点"）。若判直连，它的域名会由国内 DoH 解析，检测结果就会显示中国解析器——但那测的是直连链路，不是你要验证的代理链路。

所以 `.ini` 里有三条排在国内规则集之前的覆盖规则，把 `browserleaks.com` / `dnsleaktest.com` / `ipleak.net` 强制归入代理组。删掉它们，检测结果会失真。

## Clash Verge 专用增强版

`https://raw.githubusercontent.com/muzenc/ACL4SSR-DNS-Balanced/main/ACL4SSR_DNS_Strict_Mihomo.ini`

三层结构：`nameserver-policy` 把已知国内域名钉在「Google DoH + 中国 ECS」、已知境外域名钉在「1.1.1.1 无 ECS」，其余未知域名由 `nameserver`(ECS) 与 `fallback`(无 ECS) 并发查询、`fallback-filter` 按结果国别自动分拣。全部解析器在境外且经代理，未收录的国内域名（如 `frp-arm.com`）也能按 IP 判定直连，不依赖规则集覆盖。

**为什么 Shadowrocket 不能用这套**——实测证据链：

1. 这套设计要求本地解析未知域名（`GEOIP,CN` 需要 IP）；
2. Shadowrocket 的本地 DNS 查询不走 `#proxy` 隧道：在只有美国节点的订阅下，泄露检测出现了 **Google 香港 / Cloudflare 新加坡**解析器——查询是从中国直连发出、被 anycast 就近接到亚洲接入点的（若经隧道，只可能是美国接入点）；
3. 它也不支持 `nameserver-policy`（仅 mihomo 字段）。

所以 Shadowrocket 上任何「本地解析未知域名」的方案都会把查询暴露在隧道之外。主严格版为此换了思路：未知域名根本不在本地解析（`no-resolve`），直接交给节点远端解析——这正是 Shadowrocket 的原生模型，也是它在实测中泄露检测干净的原因。

两版的取舍：

| | 严格版（主，SR+Verge 都可用） | Verge 增强版 |
| --- | --- | --- |
| 泄露检测 | 干净（远端解析，无本地查询可测） | 干净（本地查询全部经隧道，美国解析器） |
| 国内域名解析 | 国内 DoH，最快最准 | Google+ECS 经代理，首次慢 |
| 国内 DNS 商可见性 | 看得到你访问的国内域名 | 零 |
| 未收录国内域名 | 走代理（可手动加规则） | 按 IP 判定，自动直连 |

## 上游规则

规则主要来自 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 和 [zsokami/ACL4SSR](https://github.com/zsokami/ACL4SSR)。

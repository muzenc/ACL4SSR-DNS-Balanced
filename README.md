# ACL4SSR DNS Profiles

面向 **Clash、Mihomo/Clash Meta 和 Shadowrocket** 的 ACL4SSR/subconverter 远程模板。仓库同时提供平衡版和 Shadowrocket 严格 DNS 版。

目标是在以下需求之间取得平衡：

- 国内域名和中国 IP 尽量直连；
- 未收录域名允许解析 IP，再用 `GEOIP,CN` 判断；
- 国外和未知流量默认走代理；
- DNS 全部使用 DoH/DoT，避免传统 UDP 53 明文查询；
- 不生成美国、日本等国家分组；
- 显式让 `todesk.com` 直连。

## 文件

- `ACL4SSR_DNS_Balanced.ini`：ACL4SSR/subconverter 外部配置，控制规则顺序和代理分组。
- `Clash_DNS_Balanced.yml`：Clash 基础配置，启用加密 DNS、国内解析和国外回退。
- `ACL4SSR_DNS_Strict.ini`：为 Shadowrocket 生成 Clash 订阅的严格 DNS 外部配置。
- `Clash_DNS_Strict.yml`：业务 DNS 经代理访问境外 DoH，国内 DNS 只解析代理节点域名。

## 远程配置地址

### 严格版（Shadowrocket 推荐）

如果不希望国外网站的权威 DNS 看到中国递归 DNS 出口，请使用：

`https://raw.githubusercontent.com/muzenc/ACL4SSR-DNS-Balanced/main/ACL4SSR_DNS_Strict.ini`

在 ACL4SSR 中选择 **Clash** 作为目标，生成的 URL 同时适用于 Clash Verge 和 Shadowrocket。

### `proxy` 策略组不可改名

`.ini` 里有一行 `custom_proxy_group=proxy\`select\`...`，它存在的唯一目的是让 `.yml` 中的 `#proxy` 在两个内核上同时成立：

| 内核 | 对 `https://1.1.1.1/dns-query#proxy` 的解释 |
| --- | --- |
| Shadowrocket | `#proxy` 是它原生的 DNS-over-proxy 关键字 |
| Mihomo | 把 `#` 后的名称当作**策略组名**查找，命中这个 `proxy` 组 |

Mihomo 官方文档对 `#` 的定义是「优先使用已有代理，如果不存在该名称的代理则**指定接口连接**」。一旦这个组被删掉或改名，Mihomo 会转而把 `proxy` 当网卡名去绑，绑不上，**两条 nameserver 全部失效，本地 DNS 彻底解析不出东西**。

这个故障的表现很有迷惑性：境外网站照常能开（走代理，域名交给代理服务器远端解析，本地不需要 DNS），但所有走直连的国内站点——bilibili、ToDesk 等——全部打不开。遇到这个症状先检查 `proxy` 组还在不在。组名必须是**小写 ASCII `proxy`**，不要加 emoji、不要改大小写。

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

此模板使用 `GEOIP,CN`，没有使用 `no-resolve`。这正是 ToDesk 等未被规则库收录的国内域名仍能通过 IP 判断直连的关键。

严格版同样保留 `GEOIP,CN` 的 IP 判断，但触发解析时使用经代理转发的境外 DoH。因此未知国内服务仍有机会根据中国 IP 直连，国外测试域名不会再提交给国内递归 DNS。

## 隐私边界

这是“速度与 DNS 隐私的平衡方案”，不是数学意义上的零泄露：

- DoH/DoT 会加密设备到 DNS 服务商之间的查询，局域网和运营商通常看不到明文域名；
- DNS 服务商仍然能够看到它负责解析的域名；
- 为了让未知域名经过 `GEOIP,CN` 判断，客户端必须先解析它；
- 不同版本的 Shadowrocket 对 Clash DNS 字段的导入支持可能不同。导入后应确认 DNS 服务器是 `https://` 或 `tls://`，而不是 `system`。

**严格版配置中不含任何国内 DNS**，四个 DNS 字段全部指向境外：

| 字段 | 用什么 | 说明 |
| --- | --- | --- |
| `nameserver` | `1.1.1.1` / `8.8.8.8`，经代理 | 默认，不带 ECS |
| `nameserver-policy` | `8.8.8.8` / `8.8.4.4`，经代理 | 仅 `geosite:cn`，带中国 ECS |
| `default-nameserver` | `1.1.1.1` / `8.8.8.8` | 结构上不触发，fail-closed |
| `proxy-server-nameserver` | `1.1.1.1` / `8.8.8.8` | 节点是 IP 时不触发，fail-closed |

后两项刻意填境外地址：正常情况下根本不会被调用，一旦被调用应当解析失败，而不是悄悄退回国内 DNS。若节点改用域名形式，代理会起不来（而非静默泄露），此时用 `hosts` 把节点域名钉死到 IP：

```yaml
hosts:
  "node.your-airport.com": 1.2.3.4
```

## 国内站点为什么不会变慢

只用境外 DNS 解析国内域名会引出一个隐蔽问题：国内 CDN 按「提问者在境外」分配节点，于是返回**境外节点的 IP**。实测：

| 域名 | 不带 ECS 解析到 | 位置 |
| --- | --- | --- |
| `zhihu.com` | `43.135.83.2` | 香港 |
| `xiaohongshu.com` | `43.159.24.58` | 新加坡 |
| `juejin.cn` | `163.181.60.203` | 美国亚特兰大 |

这些 IP 命不中 `GEOIP,CN`，国内站点被丢进 `FINAL` 走代理绕地球一圈——**慢的根因是路由被解析结果带偏，不是 DNS 本身慢**。

解决办法是给 `geosite:cn` 命中的域名附带一个中国 ECS 网段。加上之后实测 25 个常见国内域名中 23 个解析到国内 IP，多数落在华北联通节点。

**这没有引入国内 DNS**：解析仍由 Google 完成，而 Google 本来就通过默认 `nameserver` 看得到你的全部域名，暴露面没有增加。新增信息只有「按该网段返回结果」这一个提示，用的是固定通用网段，不是你的真实 IP。

几个设计细节：

- **默认不带 ECS**，只有国内域名带。查询从代理出口发出，境外站点不带 ECS 时会按代理所在地返回最优 IP，这正是走代理想要的；给它们带上中国 ECS 反而更慢，还等于告诉境外站点你在中国。
- **ECS 这一支只用 Google**。Cloudflare 按其隐私政策不转发 ECS，混用会出现「谁先应答用谁」的抖动。
- **换成你所在地运营商的网段**，CDN 选点会更贴近：联通北京 `202.106.0.0/24`（当前值）、电信上海 `202.96.209.0/24`、电信广东 `202.96.128.0/24`、移动 `211.136.192.0/24`。

少数站点的权威服务器不认 ECS（`www.baidu.com`）或使用全球 CDN（`ctrip.com`、`qunar.com`），仍会解析到境外 IP。它们由 `ChinaDomain.list` 按域名判直连，位置在 `GEOIP,CN` 之前，实测路由不受影响。

## 严格版验收

导入严格版生成的 URL 后，应确认：

1. Shadowrocket 使用“配置”路由模式；
2. `🤖 AI` 和 `🌐 未知站点` 均选择 `✈️ 代理`；
3. DNS 覆写和备用 DNS 中保留了 `#proxy`；
4. DNSLeakTest 扩展测试不出现中国联通、电信、移动或中国地区解析出口；
5. ToDesk 等国内服务在 Shadowrocket 日志中仍显示直连。

> 如果导入后 `#proxy` 消失，说明当前 Shadowrocket 版本或转换链路没有完整保留该 Clash DNS 字段，此时不要把测试结果视为严格模式已生效。

## 上游规则

规则主要来自 [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) 和 [zsokami/ACL4SSR](https://github.com/zsokami/ACL4SSR)。

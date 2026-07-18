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

此版本仍在 ACL4SSR 中选择 **Clash** 作为目标，然后把生成的 Clash URL 导入 Shadowrocket。严格版使用 Shadowrocket 的 DNS-over-PROXY 写法；YAML 中已使用引号保护，避免 `#proxy` 被当作注释删除。该严格版专门面向 Shadowrocket，不建议直接交给其他 Clash 内核运行。

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

严格版中的业务 DNS 为：

- `https://dns.google/dns-query#proxy`
- `https://cloudflare-dns.com/dns-query#proxy`（备用）

`https://1.12.12.12/dns-query` 仅放在 `proxy-server-nameserver` 中，用于在代理隧道建立之前解析代理节点自身的域名，不用于解析被访问网站。严格版没有设置 ECS，避免主动携带中国网段信息。

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

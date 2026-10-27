# Claude 分流规则使用说明

## 订阅地址

https://raw.githubusercontent.com/chenchenfang/rules/master/claude.list

`claude.list` 是纯域名规则集，使用 `behavior: domain` 和 `format: text`。
`+.` 表示匹配域名本身及其所有子域名，文件内不填写策略组名称。

## 内容来源与转换

- 保留原 Claude 覆写脚本的域名匹配范围，并合并 MetaCubeX anthropic.list 在 2026-10-27 的域名快照。
- `DOMAIN-SUFFIX,example.com,Claude` 转换为 `+.example.com`。
- `DOMAIN,example.com,Claude` 转换为 `example.com`。
- `anthropic.auth0.com` 是精确匹配；`statsig.anthropic.com` 已由 `+.anthropic.com` 覆盖。
- 原来的 `GEOSITE,anthropic` 用域名快照替代，不再依赖本机 geosite 数据文件；此快照不会自动跟随上游更新。
- Statsig、Sift、Sentry 等公共服务域名按原脚本保留，其用于其他网站的流量也会命中本规则。
- 同一策略组只负责统一分流目标，不代表服务不会触发风控，也无法保证账号安全。

上游域名快照：
https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/geo/geosite/anthropic.list

## Clash / Mihomo 接入

以下片段合并到已有配置，保留原配置中的其他策略组、规则集合和规则，不要直接用片段替换整个配置。
适用于支持 text 格式规则集的 Clash / Mihomo 内核；较旧内核需确认格式支持。

```yaml
rule-providers:
  claude:
    type: http
    behavior: domain
    format: text
    url: "https://raw.githubusercontent.com/chenchenfang/rules/master/claude.list"
    path: ./ruleset/claude.list
    interval: 86400

rules:
  # 放在其他可能抢先匹配的规则和 MATCH 之前。
  # 将 Claude 替换成你已有的策略组名称，或按下文创建该组。
  - RULE-SET,claude,Claude
```

规则集合无法创建策略组。如果已有 Claude 组，直接复用即可。
Mihomo 可以添加以下策略组，然后在客户端中选择节点：

```yaml
proxy-groups:
  - name: Claude
    type: select
    include-all: true
```

不支持 `include-all` 的内核，请使用 `proxies` 显式列出配置中已存在的节点或策略组。

## 可选：进程规则与 QUIC 拦截

纯域名规则集无法表达进程条件和拒绝动作。若要保留原脚本的额外行为，
在 Mihomo 主配置的 `rules` 中按下面顺序合并，放在通用规则和 MATCH 之前：

```yaml
rules:
  # 可选：全局拒绝 UDP 443，会影响所有站点的 QUIC 流量。
  - AND,((NETWORK,UDP),(DST-PORT,443)),REJECT
  # 原脚本的 Windows Claude CLI 规则，含更新后带后缀的进程。
  - 'PROCESS-NAME-REGEX,^claude\.exe,Claude'
  - RULE-SET,claude,Claude
```

进程正则是 Mihomo 扩展；进程规则需要客户端能识别进程信息。
域名列表本身可用于各平台；进程示例仅匹配 Windows 的 claude.exe 系列名称。
QUIC 拦截是否有效还取决于该 UDP 流量是否经过客户端，不保证解决 Cloudflare 验证问题。

## 格式参考

- https://wiki.metacubex.one/config/rule-providers/
- https://wiki.metacubex.one/config/rule-providers/content/

已进行域名条目格式及去重检查，未在实际 Clash / Mihomo 客户端中加载或进行网络连通性测试。

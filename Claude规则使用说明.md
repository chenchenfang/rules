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

## 可选：桌面客户端与 CLI 进程规则

纯域名规则集无法表达进程条件。以下规则需要合并到 Mihomo 主配置的 `rules` 中，
放在通用规则和 MATCH 之前；末尾的 Claude 是目标策略组名称，需与已有策略组一致。
如已有同名规则，请合并去重，不要重复创建 rules 配置项。

```yaml
rules:
  # macOS：Claude 桌面客户端主进程及辅助进程。
  - 'PROCESS-NAME-REGEX,^Claude( Helper.*)?$,Claude'
  # macOS：原生安装的 Claude Code CLI，通常为小写 claude。
  - PROCESS-NAME,claude,Claude
  # Windows：Claude Code CLI，保留原脚本对更新后后缀名称的匹配。
  - 'PROCESS-NAME-REGEX,^claude\.exe,Claude'
  # 域名规则作为补充，确保放在通用规则和 MATCH 之前。
  - RULE-SET,claude,Claude
```

### macOS 桌面客户端

`^Claude( Helper.*)?$` 匹配大写 Claude 主进程，以及 Claude Helper、
Claude Helper (Renderer) 等辅助进程。网络请求可能由辅助进程发起，因此一并匹配。
该表达式不会匹配小写的 CLI 进程 claude。

以上名称是常见进程名称，具体以客户端连接面板显示的进程名为准。
桌面客户端启动的独立子程序不一定采用这些名称，需要按实际连接确认。

### macOS Claude Code CLI

原生安装版通常使用小写进程名 claude，采用 PROCESS-NAME 精确匹配即可。
旧版 npm 安装可能被识别为 node；不要直接添加 PROCESS-NAME,node,Claude，
否则其他 Node.js 程序也会进入 Claude 策略组。此时优先使用域名规则，
或根据连接面板中实际可见的进程路径另行配置精确匹配。

### Windows CLI 与平台范围

Windows 示例匹配以 claude.exe 开头的进程，包括原脚本提到的
claude.exe.old.<时间戳> 等带后缀名称。该规则是 CLI 示例，
不代表覆盖 Windows 桌面客户端全部辅助进程。

按运行平台选用所需规则；域名列表本身可跨平台使用。

### 为什么正则规则外面有单引号

示例中的单引号是 YAML 字符串定界符，解析配置后会去掉，不属于正则表达式。
单引号字符串会保留反斜杠原样，例如 `claude\.exe` 中的 `\.` 用于匹配字面量句点。
保持示例中的单反斜杠写法，不要套用 JavaScript 字符串的双反斜杠写法。

### 生效条件与影响

PROCESS-NAME-REGEX 是 Mihomo 扩展，并非所有旧版 Clash 内核都支持。
进程规则需要客户端能够识别连接所属进程，通常配合 TUN 模式使用；
最终以连接面板中的进程信息和命中规则为准，TUN 模式本身不保证识别成功。
这些规则匹配进程的网络连接，会将命中的其他外部服务请求也交给 Claude 策略组；
仅配置域名规则时则按目标域名分流。

## 可选：QUIC 拦截

若需要保留原脚本的额外行为，可以把下面规则放到上述进程规则和域名规则之前：

```yaml
rules:
  # 全局拒绝 UDP 443，会影响所有站点的 QUIC 流量，按需启用。
  - AND,((NETWORK,UDP),(DST-PORT,443)),REJECT
```

这条规则拒绝经过客户端的 UDP 443 连接，不会自动将流量改为代理。
支持回退的浏览器通常会改用 TCP；后续连接仍按其他分流规则处理。
QUIC 拦截是否有效取决于该 UDP 流量是否经过客户端，不保证解决 Cloudflare 验证问题。

## 格式参考

- https://wiki.metacubex.one/config/rule-providers/
- https://wiki.metacubex.one/config/rule-providers/content/

已进行域名条目格式及去重检查，未在实际 Clash / Mihomo 客户端中加载或进行网络连通性测试。

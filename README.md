# work-rules


## 快速开始

1. 将 [config.yaml](./config.yaml) 导入支持 Clash 配置的客户端。
2. 将 `proxies` 中的示例节点替换成自己的节点或订阅提供的节点。
3. 在客户端中选择 `back` 代理组的出口节点。
4. 修改本仓库中的工作规则后提交并推送到 `main` 分支；客户端可手动更新规则，或等待自动更新。

当前配置使用 `mode: rule`。流量从 `rules` 顶部开始依次匹配，命中第一条规则后即停止继续匹配；末尾的 `MATCH,back` 是未命中流量的默认出口。

## 配置结构

```text
请求
  └─ rules（按顺序匹配）
       ├─ 工作规则集 → 指定节点／代理组
       ├─ 局域网、国内、私有规则 → DIRECT
       ├─ 广告规则 → REJECT
       └─ MATCH → back
                       └─ 用户在客户端选择的出口节点
```

| 区块               | 作用                                                           |
| ------------------ | -------------------------------------------------------------- |
| `proxies`        | 定义单个代理节点，例如 SS、Trojan、VLESS。                     |
| `proxy-groups`   | 将节点组合为可选择、测速或故障切换的代理组。                   |
| `rule-providers` | 从本地或远程 URL 下载可复用的规则集合。                        |
| `rules`          | 定义域名、IP 或规则集应交给哪个节点、代理组或`DIRECT` 处理。 |
| `dns`            | 配置 DNS 解析与 Fake-IP 行为。                                 |

## 工作规则的远程发布

工作规则通过 GitHub Raw 地址加载，格式如下：

```text
https://raw.githubusercontent.com/Cc360428/work-rules/main/<rule-file>.txt
```

例如 `work-direct`：

```yaml
rule-providers:
  work-direct:
    type: http
    behavior: domain
    url: "https://raw.githubusercontent.com/Cc360428/work-rules/main/work-direct.txt"
    path: ./ruleset/work-direct.yaml
    interval: 86400
```

字段说明：

| 字段                 | 说明                                                                |
| -------------------- | ------------------------------------------------------------------- |
| `type: http`       | 通过 URL 下载规则。                                                 |
| `behavior: domain` | 规则内容为域名或域名后缀。IP 网段应使用`ipcidr`。                 |
| `url`              | 仓库公开后可直接访问的 Raw 文件地址。                               |
| `path`             | Clash 下载规则后的本地缓存路径。不要删除`ruleset/` 中的缓存文件。 |
| `interval: 86400`  | 每 24 小时检查一次更新。                                            |

仓库根目录中的 `work-*.txt` 是唯一需要维护和发布的工作规则源文件；同名 `.yaml` 文件不是必需的。

## 规则文件与路由

| 文件                   | 当前用途                 | 对应规则动作                                                      |
| ---------------------- | ------------------------ | ----------------------------------------------------------------- |
| `work-direct.txt`    | 工作服务直连白名单       | `DIRECT`                                                        |
| `work-indonesia.txt` | 印尼站点                 | `Ind-1`                                                         |
| `work-singapore.txt` | 新加坡站点               | `Ind-1`（按当前配置）                                           |
| `work-brail.txt`     | 巴西站点                 | `brail-1`                                                       |
| `work-hk.txt`        | 香港站点                 | `hk-2`                                                          |
| `work-proxy.txt`     | 需通过默认代理的工作站点 | 尚未在`rules` 中引用；需要时添加 `RULE-SET,work-proxy,back`。 |

规则文件使用 Clash Rule Provider 的 YAML 格式：

```yaml
payload:
  - "+.example.com"       # example.com 及其子域名
  - "api.example.com"      # 单一域名
```

新增规则的建议流程：

1. 选择最精确的规则文件，例如将仅需直连的域名加入 `work-direct.txt`。
2. 使用双引号包裹每个条目，避免 YAML 解析歧义。
3. 若新增了规则文件，在 `rule-providers` 增加提供器，并在 `rules` 中增加对应的 `RULE-SET` 路由。
4. 将更具体的规则放在更通用规则之前；规则顺序决定最终出口。
5. 提交并推送到 `main`，然后在 Clash 客户端手动更新规则以立即生效。

例如，将 `work-proxy` 接入默认代理组：

```yaml
rules:
  - RULE-SET,work-proxy,back
  - MATCH,back
```

应把该条目放在 `MATCH` 之前，并根据业务需要放在直连规则之前或之后。

## 代理组

代理组决定“如何选择节点”。当前的 `back` 使用 `select`，即由用户在客户端手动选择出口节点：

```yaml
proxy-groups:
  - name: back
    type: select
    proxies:
      - work-n
      - hk-2
      - hk8
```

### `type` 类型

| 类型             | 选择方式                         | 适合场景                       | 不适合场景                              |
| ---------------- | -------------------------------- | ------------------------------ | --------------------------------------- |
| `select`       | 用户手动选择一个节点或下级代理组 | 默认出口、需要随时手动切换地区 | 无人值守的自动故障恢复                  |
| `url-test`     | 定期测速，选延迟更低的健康节点   | 节点质量波动较大、优先低延迟   | 需要固定出口 IP 的业务                  |
| `fallback`     | 按列表顺序，选第一个健康节点     | 主备线路、出口优先级明确       | 希望按延迟择优或分流负载                |
| `load-balance` | 将新的连接分配给多个健康节点     | 可多出口的下载、API 流量       | 登录、支付、IP 白名单等需固定出口的业务 |
| `relay`        | 多个节点顺序串联                 | 旧版 Clash 配置                | 新的 Mihomo 配置；已弃用                |

`GLOBAL` 是 Mihomo 内置的全局策略组，不是需要自行声明的 `type`。它默认汇集所有节点与代理组；部分客户端也会用其中的顺序决定面板展示顺序。

### 通用字段

```yaml
proxy-groups:
  - name: "自动选择"
    type: url-test
    proxies:
      - hk-2
      - hk8
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    lazy: true
    tolerance: 50
    disable-udp: false
```

| 字段                            | 说明                                                                        |
| ------------------------------- | --------------------------------------------------------------------------- |
| `name`                        | 代理组名称；`rules` 中通过此名称引用。                                    |
| `type`                        | 代理组策略类型。                                                            |
| `proxies`                     | 直接引入节点或其他代理组。可组成“手动选择 → 自动选择”的层级。            |
| `use`                         | 引入`proxy-providers` 中的订阅节点；本仓库当前未使用订阅提供器。          |
| `url`                         | 健康检查地址；`url-test`、`fallback` 和 `load-balance` 应配置。       |
| `interval`                    | 健康检查间隔，单位为秒；`300` 即每 5 分钟。                               |
| `lazy`                        | 为`true` 时，未被实际选用的组不主动测速。                                 |
| `tolerance`                   | 仅`url-test` 使用。新节点需比当前节点低出此毫秒数才切换，可减少频繁跳变。 |
| `disable-udp`                 | 禁止该组承载 UDP；组内协议或目标业务不支持 UDP 时使用。                     |
| `default-selected`            | 为`select` 指定初始选中节点；未设置时通常选列表第一项。                   |
| `filter` / `exclude-filter` | 对通过`use` 或 `include-all-*` 引入的节点按名称正则筛选／排除。         |

健康检查只检查 `proxies` 中直接列出的节点，不检查通过 `use` 引入的订阅节点；订阅节点应在其 `proxy-providers` 配置中设置健康检查。

### `select`：手动选择

`select` 不自动判断节点好坏，面板中选哪个就用哪个。它适合作为最外层入口：将手动节点、`url-test`、`fallback` 等下级组都放进去，按业务决定是否自动选择。

```yaml
proxy-groups:
  - name: back
    type: select
    proxies:
      - 自动选择
      - 主备出口
      - hk-2
      - hk8
```

### `url-test`：按延迟自动选择

`url-test` 定期访问 `url`，自动选择延迟较低的健康节点。`tolerance` 用于抑制节点延迟接近时的反复切换；例如设为 `50` 时，新节点需要至少快 50 ms 才替换当前节点。

```yaml
  - name: 自动选择
    type: url-test
    proxies: [hk-2, hk8]
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
```

测速快不等于业务一定可用：测试 URL、目标站点的地区可达性和实际吞吐量都可能不同。对固定地区或固定 IP 的业务，应使用 `select` 或 `fallback`。

### `fallback`：主备自动回退

`fallback` 按 `proxies` 的书写顺序选择第一个通过健康检查的节点。主节点失效时切到备用节点；主节点恢复后，会优先使用列表中更靠前的可用节点。

```yaml
  - name: 主备出口
    type: fallback
    proxies: [hk-2, hk8]
    url: "https://www.gstatic.com/generate_204"
    interval: 300
```

节点顺序就是优先级。它适合主、备出口清晰的生产业务，但不是“自动选最快”。

### `load-balance`：负载均衡

`load-balance` 为不同的新连接选择不同节点。它也需要健康检查，并可使用 `strategy` 控制分配方式：

| `strategy`           | 行为                                           |
| ---------------------- | ---------------------------------------------- |
| `round-robin`        | 新请求轮流分配到各节点。                       |
| `consistent-hashing` | 相同目标地址尽量固定落到同一节点。             |
| `sticky-sessions`    | 相同来源与目标的会话在一段时间内保持同一节点。 |

```yaml
  - name: 下载均衡
    type: load-balance
    proxies: [hk-2, hk8]
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    strategy: consistent-hashing
```

不建议把账号登录、支付、后台管理或任何依赖源 IP 的服务放进负载均衡组，否则同一业务的连接可能从不同出口发出。

### 链式代理：不要在新配置中使用 `relay`

旧版 Clash 的链式代理类型是 `relay`，多个节点按 `proxies` 列表从前到后串联。但目前 Mihomo 已将 `relay` 标记为弃用，新的配置应优先使用节点级 `dialer-proxy`。

例如，若希望最终出口 `work-n` 先通过 `hk-2` 建立连接，在 `work-n` 这个**节点**中加入：

```yaml
proxies:
  - name: work-n
    type: ss
    server: exit.example.com
    port: 443
    cipher: aes-256-gcm
    password: "replace-me"
    dialer-proxy: hk-2
```

这里的路径是“本机 → `hk-2` → `work-n` → 目标网站”。`dialer-proxy` 可引用节点或代理组名称。它属于 `proxies` 的节点字段，不属于 `proxy-groups`；因此若只是创建链路，无需新增 `type: relay` 代理组。链式代理会增加延迟和故障点，应先在所用客户端及目标业务上测试 TCP、UDP 与 DNS 行为。

## DNS 与规则排查

本配置开启 Fake-IP DNS。若某个域名路由不符合预期，可按以下顺序检查：

1. 在客户端日志中确认请求域名和最终命中的规则。
2. 检查规则是否写入了正确的 `work-*.txt` 文件，以及该规则集是否已更新成功。
3. 确认规则条目位于更宽泛规则和 `MATCH` 之前。
4. DNS 或 IP 直连相关问题，检查 `dns`、`GEOIP` 和 `IP-CIDR` 规则是否更早命中。
5. 如刚推送 GitHub，先在浏览器访问对应 Raw URL，确认内容已是新版本，再在客户端手动更新规则提供器。

## 维护与安全

- 本仓库是公开仓库，禁止提交真实服务器地址、密码、UUID、订阅链接、Token 或包含用户身份的信息。
- 修改 `.txt` 规则文件后，请检查 YAML 格式；未闭合引号会导致整个规则提供器下载或解析失败。
- `config.yaml` 中的节点仅作结构示例。个人配置建议保存在未提交的本地文件，或通过安全的配置管理方式注入。
- 远程规则的更新有缓存周期。需要立即刷新时，在 Clash 客户端中执行“更新规则提供器”。

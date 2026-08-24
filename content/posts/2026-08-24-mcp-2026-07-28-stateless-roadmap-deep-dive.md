---
title: "MCP 2026-07-28 深挖:协议层无状态,agent 编排进入显式 handle 时代"
date: 2026-08-24T08:00:00Z
tags: ["mcp", "agent", "protocol", "stateless", "roadmap"]
author: "COSMOSPRO"
draft: false
summary: "MCP 2026-07-28 正式发布:initialize 握手和 Mcp-Session-Id 被移除,协议核心改写成 stateless request/response。6 个 SEP 协同完成协议诞生以来最大一次修订——显式 handle 让模型看得见状态,任何请求可落在任何服务器实例;MCP Apps/Tasks 抽成 extensions,Roots/Sampling/Logging 进入 12 个月 deprecation window。"
---

想象一下你正在给公司写一个 MCP server:2025 年 11 月,你的上线 checklist 里写着"实现 initialize 握手、管理 Mcp-Session-Id、保证网关 sticky routing、给 SSE 长连接留端口";2026 年 7 月 28 日,这份 checklist 几乎整页作废——同一个 `tools/call`,以前要先握手、拿 session、每次带上它,还得祈祷请求被路由回同一个实例;现在一组自描述的 header(`Mcp-Method` / `Mcp-Name` / `MCP-Protocol-Version`)就够了,请求自描述,落在哪台服务器都能处理。这是 MCP 协议诞生以来最大的一次修订,而它改的恰恰是所有人默认"就该这样"的那部分:会话。

## 为什么必须砍掉 session

MCP 自 2025-03-26 引入 Streamable HTTP 以来,协议核心一直是带会话的 bidirectional stateful:客户端先握手协商能力,服务端发 `Mcp-Session-Id`,之后所有请求绑定这个 session,服务端还能随时通过 SSE 长连接主动推消息。这个模型在单机 demo 上很顺,一旦进生产就处处掣肘:

- **sticky routing**:网关必须把同一 session 的请求粘到同一实例,负载均衡形同虚设,加机器不解决问题;
- **共享 session store**:多实例要同步会话状态,成为分布式系统里最容易炸的组件;
- **SSE 长连接不可缓存、不可限流**:gateway 没法基于 header 做路由和 rate-limit,WAF 也拦不住;
- **initialize 握手**:每次重连都要重来一遍,serverless 场景下 cold start 尤其难受。

如果把 2026 年的 roadmap 主线列出来——transport scalability、agent communication、governance maturation、enterprise readiness——session 恰好堵住了前两条。于是 maintainers(David Soria Parra + Den Delimarsky)在 2026-05-21 锁定了 Release Candidate,7-28 发布最终 spec,用 6 个 SEP 把"会话"两个字从协议核心里删干净。SDK Tier 1(TypeScript / Python / Go / C#)同日同步支持,Rust 排在 Tier 2。

> 为什么重要:砍 session 不是洁癖,而是 MCP 想从"demo 协议"长成"生产协议"必须过的坎。会话状态是分布式系统里最贵的隐式耦合,把它移出协议层,等于给整个生态松绑——这也是为什么官方敢说这是"largest revision since launch"。

## 六个 SEP 怎么协同完成"无状态化"

单个 SEP 只是提案,真正厉害的是这 6 个 SEP 拼在一起,正好覆盖一次请求的完整生命周期:

- **SEP-2575** 移除 `initialize` / `initialized` 握手:协议版本和客户端能力移到 `_meta` 字段,新增可选 `server/discover` 做能力发现。连接即用,不用先拜码头。
- **SEP-2567** 移除 `Mcp-Session-Id` header 和协议层 session:状态不再由协议隐式持有,而是由服务器**显式签发 handle**(比如 `basket_id`、`browser_id`),模型自己把 handle 作为参数传回后续调用。
- **SEP-2260** 强制 server-initiated requests 只能在处理 client request 期间发出(之前只是建议)。服务端再也不能"凭空"推消息。
- **SEP-2322**(MRTR)把 elicitation/sampling 改成 `resultType: "input_required"` + `requestState` + 重试原调用,抛弃长期存活的 SSE 流。模型想要用户输入?发一个"input_required"结果,客户端拿 `requestState` 续上原调用即可。
- **SEP-2243** 让 Streamable HTTP 强制要求 `Mcp-Method` + `Mcp-Name` header:gateway、rate-limit、WAF 现在可以直接基于 header 路由和限流,不用解析 body。
- **SEP-2549** 给 `tools/list`、`prompts/list`、`resources/list`、`resources/read` 加上 `ttlMs` + `cacheScope`(类似 HTTP Cache-Control),list 结果可以放心缓存,long-lived SSE 不再是变更通知的唯一通道。

> 为什么重要:6 个 SEP 不是 6 个孤立改动,而是同一件事的 6 个侧面——握手没了(2575)、会话没了(2567)、服务端主动能力被收紧(2260/2322)、请求变自描述(2243)、响应变可缓存(2549)。它们合起来,才让"任何请求可落在任何实例"成为可能。

顺着一次请求的生命周期走一遍,你就能看到这套设计的整体性。客户端要调用某个工具:先查 `server/discover` 或缓存里的 `tools/list`(2549 让这个结果可以带 `ttlMs` 缓存),拿到工具描述和参数 schema;然后直接发 `tools/call`,`Mcp-Method` header 让网关在不解包 body 的情况下就能路由、限流、过 WAF(2243),`_meta` 里带上协议版本和 trace context;服务器处理期间,如果它需要向客户端反问(比如要用户确认一个敏感操作),通过 MRTR 返回 `resultType: "input_required"` 而不是开一条新的长连接(2322);如果这个工具是长任务,它返回一个 task handle 而不是阻塞等待(对应 Tasks extension)。每一步都是无状态的,但每一步的状态——handle、requestState、缓存条目——都是显式的、可推理的。

## 显式 handle:状态没有消失,只是变得可见

最容易误读的是"无状态 = 没有状态"。SEP-2567 的官方 rationale 说得非常直白:连 opt-in 都要去掉 session,不是因为状态没用,而是因为**隐式状态藏起来了,模型看不到**。

新的模式是:**协议层无状态,应用层保留状态**。服务器在 `tools/call` 的返回里签发一个显式 handle(比如"你的购物篮是 basket_123"),后续调用时模型把 handle 作为参数传回去。状态还在,但它成了模型推理的一部分——模型看得见它、可以跨工具组合它、可以在步骤之间传递它、甚至可以决定什么时候释放它。

MCP 官方 blog 用了一个很重的词:这种模式是 **"more powerful"**。因为 session 时代的状态是给协议看的(服务器代码里的一张隐式小表),handle 时代的状态是给模型看的(对话里一个可推理的值)。模型不再"盲打"一个它看不见的会话,而是明确知道自己正在操作哪个资源。换句话说,以前模型只是"经过"状态,现在模型"拥有"状态。

> 为什么重要:这是本次修订最反直觉也最关键的一点。无状态不是能力降级,而是把状态的可见性从"协议内部"提升到"模型视角",让编排从隐式耦合变成显式推理——这恰恰是 agent 场景最需要的。

## 代码示例:before / after 抓包对比

下面用同一个 `tools/call` 请求,对比 2025 版和 2026 版的实际 HTTP 形态(基于 SEP-2567 / 2243 的规范形态):

```bash
# === 2025-11-25:有会话、有握手 ===
# 第一步:initialize
curl -X POST https://mcp.example.com/mcp \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{"tools":{}}}}'
# 响应带 session id
# 第二步:带 session 调工具
curl -X POST https://mcp.example.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Mcp-Session-Id: sess_9f2c81' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"create_basket","arguments":{}}}'

# === 2026-07-28(final):无会话、自描述、可路由 ===
curl -X POST https://mcp.example.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'MCP-Protocol-Version: 2026-07-28' \
  -H 'Mcp-Method: tools/call' \
  -H 'Mcp-Name: commerce-server' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"create_basket","arguments":{}},"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28"}}'
# 返回:{"result":{"content":[{"type":"text","text":"basket_123"}]}}
# 下一步调用把 handle 传回去即可,任何实例都能处理
```

两版差异一眼可见:握手没了、session header 没了、`Mcp-Method` 让网关零解析就能路由、`_meta` 携带协议版本。配合 SEP-2549 的 `ttlMs`/`cacheScope`,`tools/list` 这类高频查询可以直接走 CDN 缓存。再加上 SEP-414 把 W3C Trace Context(`traceparent`/`tracestate`/`baggage`)放进 `_meta`,一次跨 SDK、跨 gateway 的调用能汇成同一条 span tree——可路由、可缓存、可追踪,这是 web 世界早就有的能力,现在 MCP 也补上了。

> 为什么重要:生产环境的三件大事——水平扩展、缓存、可观测性——都被这一组 header 级改动解锁。对平台团队来说,这意味着 plain round-robin LB 就够了,不需要再维护 sticky routing 和 session store。

## 一个最小的 stateless server

HTTP 形态看明白了,再看服务端有多简单。因为状态由 handle 承载,`tools/call` 处理函数不需要任何会话上下文,纯函数即可:

```typescript
// Node.js 原生 http,零依赖,完整可运行
import http from "node:http";

const baskets = new Map<string, string[]>();

http.createServer((req, res) => {
  if (req.method !== "POST") { res.writeHead(405).end(); return; }
  let raw = "";
  req.on("data", (c) => (raw += c));
  req.on("end", () => {
    const { id, method, params } = JSON.parse(raw);
    let result: unknown;
    if (method === "tools/call" && params.name === "create_basket") {
      const bid = "basket_" + Math.random().toString(36).slice(2, 8);
      baskets.set(bid, []);            // 状态存在应用层
      result = { content: [{ type: "text", text: bid }] }; // 显式 handle
    } else {
      result = { content: [{ type: "text", text: "unknown tool" }] };
    }
    res.setHeader("Content-Type", "application/json");
    res.end(JSON.stringify({ jsonrpc: "2.0", id, result }));
  });
}).listen(8787);
```

这个 handler 没有任何 session 概念:同一个函数,部署 100 个副本,请求打到哪个实例结果都一样;`baskets` 换成 Redis 或 Durable Objects,跨实例共享也自然。这正是"协议层无状态、应用层自己管状态"的写照。

> 为什么重要:迁移成本决定一个协议修订能不能落地。像上面这样把会话从处理函数里拿掉之后,server 代码反而更简单——不用解析 session header、不用维护连接状态,这解释了为什么 Sentry 敢在 spec 定稿前就上生产。

如果你手上有一批旧 server,迁移清单并不长:1) 删掉 `initialize` / `initialized` 处理,把版本信息挪到 `_meta`;2) 不再读取 `Mcp-Session-Id`,把会话状态改成显式 handle 传参;3) 确保 `Mcp-Method` / `Mcp-Name` header 由框架或网关正确填充;4) 把基于 SSE 的 server-initiated 推送改成请求期间的响应或 MRTR;5) 检查客户端是否还在字面匹配 `-32002`(见下文错误码部分)。MCP 官方与社区提供了 migration 对照表和 conformance 检查工具,升级路径比想象中平缓——这也是为什么大厂敢 day-zero 跟进。

## 配套修订:Schema 与错误码

无状态之外,spec 还顺手补了两块长期让开发者头疼的地基。**JSON Schema 2020-12 完整化**(SEP-2106):tool `inputSchema` 保留 `type: "object"` root,但允许 `oneOf` / `anyOf` / `allOf`、conditionals、`$ref` / `$defs`;`outputSchema` 完全放开,`structuredContent` 可以是任意 JSON value。以前"工具参数必须是个扁平 object"的限制没了,复杂参数、递归结构、可复用定义都能表达。同时规范明确:实现**不得**自动 dereference 外部 `$ref`,要限制深度和校验时间,防止被一个 $ref 拖进无限递归。

**错误码改标准**(SEP-2164):缺资源时从 MCP 自定义的 `-32002` 改为 JSON-RPC 标准的 `-32602 Invalid Params`。客户端如果还在字面匹配 `-32002`,必须更新——这也是 migration 文章里最容易踩的坑。

> 为什么重要:这两条是"质量线"改动,不抢 headline,但决定了工具生态的上限。参数描述能力上去了,模型才能可靠地生成复杂调用;错误码对齐 JSON-RPC,client 库才能少写一层特判。

## 减法:核心变薄,能力抽成 extensions

这次修订的另一个主题是"协议核心变薄"。SEP-2133 正式化了 extensions 框架:reverse-DNS 命名、通过 capabilities 上的 extensions map 协商、各自独立的 `ext-*` repo 和维护者、独立版本号,还有从 experimental 到 official 的 Extensions Track。两个重量级能力按这个框架拆了出去:

- **MCP Apps**(SEP-1865,Anthropic + OpenAI 共同作者):服务器可以推送 sandboxed iframe HTML UI 到 host 渲染,UI 通过 JSON-RPC 与 host 通信,每次动作走和直接 tool call 一样的审计/同意路径。tool 的返回值从此可以是一块界面,而不只是字符串——比如数据库工具直接渲染一张可交互的表格,用户点按钮授权,host 再把动作作为一次普通 tool call 记录进审计日志。
- **Tasks**(SEP-2663):2025-11-25 的实验核心正式毕业成 extension。服务器在 `tools/call` 返回 task handle,客户端用 `tasks/get` / `tasks/update` / `tasks/cancel` 驱动长任务;由服务器决定哪些 call 应该"当 task"跑,`tasks/list` 因无法在没有 session 的世界里安全 scope 而被移除。贡献方是 AWS——想想 Bedrock AgentCore 的 day-zero 支持,就知道长任务在云厂商的路线图里分量多重。

同时,三个核心 feature 被标 deprecated(SEP-2577 落地;生命周期政策本身由 SEP-2596 定义,annotation-only,继续可用至少 12 个月):**Roots**(改用 tool parameters / resource URIs / server config)、**Sampling**(直接调 LLM provider API)、**Logging**(stdio 用 stderr,可观测性走 OpenTelemetry)。DCR(Dynamic Client Registration)和 legacy HTTP+SSE transport 也进入 deprecation,DCR 过渡到 Client ID Metadata Documents(CIMD),计划 2027 年夏移除。

配套的 auth hardening 也在收紧,6 个 SEP 全部贴近 OAuth 2.0 / OIDC 的真实部署形态:客户端必须按 RFC 9207 验证 `iss` 参数(SEP-2468,修掉 single-client/many-server 模式下的 mix-up attack)、DCR 时声明 OIDC `application_type`(SEP-837)、注册凭证绑定签发的 AS issuer(SEP-2352)、OIDC-style AS 如何请求 refresh token(SEP-2207)、step-up scope 累积规则(SEP-2350)、`.well-known` discovery 后缀(SEP-2351)。这些改动没有出现在 stateless 的 headline 里,却是企业真正敢把 MCP 放到敏感数据前面的前提。

> 为什么重要:12 个月的 deprecation window 是这份 spec 最成熟的治理设计——协议想继续演进,但不能打破现有服务器。另一个治理信号是 SEP-2484:Standards Track SEP 必须配套 conformance suite 场景才能 Final,SDR 按 SDK tier 系统给官方 SDK 评级。规范从此不只是文档,而是可验证的契约。

## 落地:生产客户已经先跑了

spec 还没定稿,就已经有生产客户跑在 stateless 模型上:Sentry 的 David Cramer 说"新协议在我们 prod 里跑了好一阵,没炸过";Linear 说 hosting 更容易、更可靠;honeycomb.io 有 20% 的月度 interactive queries 由 agents 发起,新 spec 让 enterprise scale 下也能做 elicitation;Supabase 说 MRTR 让工具能在行动前先跟用户确认;Cloudflare 用"stateless、cacheable、routable、globally scalable"四个词概括,Workers + Agents SDK day-zero 支持;AWS Bedrock AgentCore、Microsoft Foundry(微软说 stateless 让 MCP "work just like the rest of the web")、Google Cloud 也都在发布日跟进。

SDK 生态同样在变:TypeScript SDK 已从 Node.js 重写为 Web Standards(Cloudflare 与 maintainers 合作,贡献了 bundling、runtime shims、split packages),mcp-use 官方称 v2 包体积和延迟都大幅下降,Bun/Deno/Workers 都能直接跑。

这些客户引言不是公关辞令,而是一个共同信号:**协议层的复杂度,正在从"运行时"搬到"部署时"**。以前每个 MCP 部署都要先回答"session 存哪里、粘性路由怎么做、长连接怎么保活",这些问题在 stateless 模型下全部消失,取而代之的是部署前就能回答的简单问题:这个工具要不要缓存、handle 的 TTL 是多少、task 归谁执行。复杂度没有消失,但它从"每次请求都要处理"变成了"部署时配置一次"——这正是所有协议走向成熟的标准路径。

对做 agent 编排的团队,这份 spec 的启示很直接:把状态从协议层挪到应用层,用显式 handle 让模型参与状态管理,同时按 extensions 框架把重能力拆出去——协议保持薄,生态才有空间继续长。对还在观望的团队,建议先拿 mock server 或 conformance 工具跑一遍新版协议流程,再对照 migration 清单逐项升级;12 个月的 deprecation window 意味着你不用赶在发布日动工,但也别拖到 feature 真正被移除的那天。

> 接下来值得关注:Tasks extension 的 `tasks/get` / `tasks/update` 语义、MCP Apps 的 iframe 审计路径,以及 2027 年 DCR 移除后 CIMD 的落地形态。协议核心已经定型,好戏都在 extensions 里。

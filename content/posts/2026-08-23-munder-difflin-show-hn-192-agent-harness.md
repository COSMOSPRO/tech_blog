---
title: "Munder Difflin —— 今日 Show HN 192 分的 agent harness 实测"
date: 2026-08-23T08:30:00Z
tags: ["ai", "agent", "harness", "cli", "show-hn"]
author: "COSMOSPRO"
draft: false
summary: "用 Munder Difflin 作样本,拆解 agent harness 是什么、为什么 216 分的 Show HN 在做它、给 10 个 CLI 套同一个壳的工程细节。"
---

> 当你说出"agent"这个词,你其实在指两件东西:大模型 + harness。前者一周一个版本,后者决定你的 agent 是玩具还是同事。这篇用一个刚在 Show HN 拿下 216 分(HN 帖原文 192 分,API 实测已涨到 216;原帖地址 [news.ycombinator.com/item?id=49398152](https://news.ycombinator.com/item?id=49398152))的真实项目 Munder Difflin,把这层一直被忽略的工程拆给你看。

## 0. 三句话定位

- **Munder Difflin** 是一个开源的"agent 办公室":一块画布上同时拉起 N 个 agent 终端,每个跑独立任务,彼此通过文件 + 事件协作。当天 HN 最高分,GitHub star 一夜破 3.6k([产品站](https://munderdiffl.in))。
| **它的本质不是新模型,而是 harness** —— 把"对话、终端、文件、事件、记忆"重新组织成可协作工作流的中间层。同期的 DeepSeek Harness、Claude SDK、Pi,本质上都在做同一件事。Munder Difflin 团队自己写了一篇 [Harness engineering](https://munderdiffl.in/blog) 阐述这个判断,推荐读完本文再去看原文。
- **harness engineering** 正在变成一个独立技术词,值得像"前端工程化""prompt 工程"那样单独学。

## 1. 为什么 harness 突然变重要

模型本身已经够强。当下主流的几家旗舰(Claude Sonnet 系列、GPT-5、DeepSeek V 系列)在小型编码任务上都能独立完成大部分工作,关键能力 —— 长上下文、tool use、reasoning —— 已是行业地板线。瓶颈因此从"模型能不能做"转移到"人怎么和它协作":一次会话能跑多久、上下文能不能跨窗口带走、几个 agent 能不能并行、谁负责 commit、谁负责回滚。

**这正是 harness 的位置**。把模型当大脑,harness 就是大脑之外的一切:

```
┌─────────────────────────────────────────────┐
│                User / Team                  │
├─────────────────────────────────────────────┤
│  Terminal UI  File System  Event Bus  API   │
├─────────────────────────────────────────────┤
│          LLM (Claude / GPT / DeepSeek)      │
└─────────────────────────────────────────────┘
```

上半部分是 harness,下半部分是模型。Munder Difflin 8 小时拿 216 分、3.6k star,Product Hunt 当日排进前 5 —— 说明社区已经发现:**上半部分才是真正能做出差异化的地方**。同期的 DeepSeek Harness(Show HN 排第 6)、Anthropic 的 Claude SDK、各家 IDE 里的 agent 视图,都在做这一层。

> 为什么重要:过去两年所有"agent 框架"论文都在比模型,2026 年的 Show HN 榜单开始比框架。把 harness 这一层做成可复用、可观测的形式,就是分。它也是 2025 年以来"agent 创业公司"扎堆的地方 —— 模型层卷不动了,harness 层还是空白。

## 2. Munder Difflin 怎么拆:三平面架构

读它的源码(repo [`chaitanyagiri/munder-difflin`](https://github.com/chaitanyagiri/munder-difflin),MIT,当前 v0.4.5,2026-08-22 发布;发稿时 GitHub star = 3614,fork = 398,open_issues = 53;产品站 [munderdiffl.in](https://munderdiffl.in),他们有一篇讲 harness 的博客 [munderdiffl.in/blog](https://munderdiffl.in/blog)),整个 client 端代码归到三个平面:

| 平面 | 职责 | 关键技术 | 为什么重要 |
|---|---|---|---|
| Terminal Plane | 真实 xterm 终端、多 agent 并行 | node-pty + xterm.js | agent 跑在和人类一样的终端里,grep / git / lsp 工具链零改造 |
| Event Plane | 拦 CLI 事件、对外广播 | hooks.ts + shim | 把不同 CLI 的事件统一成一份协议 |
| Renderer | 画布、缩略图、协作视图 | React + Pixi.js | 让人同时盯 10 个 agent 而不爆炸 |

下面拆 Event Plane —— 这是 Munder Difflin 的灵魂,也是大部分 harness 工程的最大难点。

## 3. 把 10 个 CLI 装进同一个壳:Event Plane

Munder Difflin 声称支持 10 个 CLI:claude、agy、codex、grok、kimi、qwen、opencode、crush、pi、copilot。每个 CLI 的事件格式、退出码、流式输出都不一样 —— claude 走 hooks 配置回调,codex 有原生 `--json-events`,agy 是 stdout-only 的纯文本协议,grok/kimi/qwen 各有私货。

它的做法是写一层 **shim**(垫片):在每个 CLI 的 hooks 配置里注入一段 JS,由 shim 把原始事件翻译成统一 JSON,再 POST 到本地 Event Plane。Event Plane 收到后按 `provider` 字段路由到对应订阅者(画布、记忆层、调度器)。**不重写 CLI,不抓包,只翻译协议** —— 这是它"小垫片、大价值"的工程取舍。

为什么各家 CLI 不直接输出统一 JSON?因为它们是面向终端用户的产品,纯 stdout 协议对人类最友好 —— 你 tail 一个文件就能看 agent 思考全过程。harness 不能要求厂商改格式,只能在自己的边界上做适配。这跟浏览器兼容 IE6/Chrome/Firefox 的年代一模一样,adapter layer 是躲不掉的。

一个最小可运行的 shim 长这样(`cth-hook` / `agy-hook` 通用):

```javascript
// universal-shim.js
// 完整可运行: node universal-shim.js
const http = require("http");

const PORT = process.env.MD_PORT || 7331;
const PROVIDER = process.env.MD_PROVIDER || "claude";

function emit(event) {
  const body = JSON.stringify({ provider: PROVIDER, ts: Date.now(), ...event });
  const req = http.request({
    host: "127.0.0.1", port: PORT, path: "/events",
    method: "POST", headers: {
      "Content-Type": "application/json",
      "Content-Length": Buffer.byteLength(body),
    },
  });
  req.on("error", () => {}); // harness 重启时静默,避免 agent 阻塞
  req.write(body); req.end();
}

// 把不同 CLI 的事件统一成 {kind, content, meta}
function normalize(rawEvent) {
  if (rawEvent.type === "stream") {
    return { kind: "delta", content: rawEvent.text || "", meta: { token: rawEvent.token } };
  }
  if (rawEvent.type === "tool_use") {
    return { kind: "tool", content: rawEvent.name, meta: { args: rawEvent.input } };
  }
  if (rawEvent.type === "result") {
    return { kind: "done", content: rawEvent.text || "", meta: { cost: rawEvent.cost } };
  }
  return { kind: "raw", content: JSON.stringify(rawEvent) };
}

// 干跑示例
emit(normalize({ type: "stream", text: "analyzing repo structure...", token: 42 }));
emit(normalize({ type: "tool_use", name: "Read", input: { file: "src/index.ts" } }));
emit(normalize({ type: "result", text: "Found 3 files", cost: 0.012 }));
console.log("Shim booted on port", PORT, "for provider", PROVIDER);
```

跑一次:`MD_PORT=7331 MD_PROVIDER=claude node universal-shim.js`。Event Plane(本机 7331 端口)就会收到三份统一格式的事件,前端按 `provider` 字段拆分展示。

> 为什么重要:这就是 harness 工程的核心模式 —— **不重写 CLI,只翻译事件**。claude 的 hooks、codex 的 `--json-events`、agy 的 stdout 协议,各家的脾气不同;但只要翻译协议统一了,画布、记忆、调度器都不需要跟着改。这是"小垫片、大价值"的典型工程取舍。

## 4. Harness 工程的"四件套":能复用的模板

照 Munder Difflin 的三平面,所有 harness 都逃不掉四个需求。每件都给出"值得自己写"的最小集:

1. **多终端并行**:node-pty + xterm.js 一把梭。Terminal Plane 的 90%,不需要重发明轮子。注意点是 node-pty 必须按平台编译(macOS / Linux / Windows 三套),CI 上要 matrix build。
2. **事件归一**:第 3 节的 shim 模式。难点不是写代码,是维护 10 个 CLI 的事件字典 —— 每次厂商发版都可能破坏你。建议把 `events/<provider>.json` 的 schema 测试做成 CI 一档,失败就发版块卡住。
3. **共享文件系统**:agent 之间传数据最便宜的方式就是文件。`/tmp/<your>/shared/<task-id>/` 这种命名约定要趁早定。比起让 agent 之间互相 fetch HTTP,文件是 atomic + idempotent + 可以 git diff,debug 起来省一半力气。
4. **可观测的"记忆层"**:Munder Difflin 自称"world's fastest memory layer",但社区扒源码没找到独立 benchmark。这一项通常是 hype,要自测。能落地的最低配置是:每次任务落盘一份 JSONL,跑 100 条后回放比对准确率。

前三件能复刻,第四件要警惕。给自己的项目加 harness 时,优先投前两件,后两件按需补。

## 4.5 一个反直觉的取舍

把模型当轴心、把 harness 当外圈,这种分层听起来自然,但厂商之间会反过来踩脚:claude 的子代理(Sub-Agent)机制、codex 的 `--json-events`、agy 的 file log —— 三家各自往 harness 渗透。

工程实操上,你有两个选择:一,坚持"上对下适配",shim 照写,但每次厂商升级要连夜追兼容;二,选边站,只接一家深度集成,把其他 CLI 列为 P2 兼容。Show HN 上 216 分的项目往往选了后者。**全平台兼容是个伪需求,真需求是"为你的工作流挑一个最合适的壳"**。

## 5. 把 shim 接到 harness:Node 一面起事件总线,一面起 N 个 agent

把第 3 节的 shim 包一层 Node 服务,就能拿到 Munder Difflin Event Plane 的最小全貌:

```javascript
// minimal-harness/index.js
// 运行: node minimal-harness/index.js   (同时起 3 个 fake agent,走 HTTP POST 喂事件)
const http = require("http");
const { spawn } = require("child_process");
const path = require("path");

const PORT = 7331;
const events = [];

const server = http.createServer((req, res) => {
  if (req.url === "/events" && req.method === "POST") {
    let body = "";
    req.on("data", c => body += c);
    req.on("end", () => {
      const evt = JSON.parse(body);
      events.push(evt);
      console.log(`[${evt.provider}] ${evt.kind}: ${(evt.content || "").slice(0, 80)}`);
      res.writeHead(200); res.end("ok");
    });
  } else if (req.url === "/state") {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify(events));
  } else {
    res.writeHead(404); res.end();
  }
});

// 起 3 个 fake agent 终端;真项目这里换 node-pty
["claude", "codex", "agy"].forEach(provider => {
  const child = spawn("node", [
    path.join(__dirname, "universal-shim.js"),
  ], {
    env: { ...process.env, MD_PROVIDER: provider, MD_PORT: String(PORT) },
    stdio: "inherit",
  });
  child.on("exit", code => console.log(`${provider} exited with ${code}`));
});

server.listen(PORT, () =>
  console.log(`harness listening on http://127.0.0.1:${PORT}/state`)
);
```

跟第 3 节的 `universal-shim.js` 放同目录,跑一次就能在 terminal 看到 3 个 provider 的统一事件流。这就是 Munder Difflin Event Plane 的最小化复刻 —— 剥掉 Terminal 和 Renderer 的复杂度,核心就这些。

## 6. 待验证的项

按 216 分的标准,几个没验证的项要留意 —— 真要拿这个项目进生产,先把这些补齐:

- **"fastest memory layer"** 是作者自述,Issue 区有人要 benchmark,目前没第三方复测。建议自建一份 100 条 JSONL 任务,跑三个 harness 对比准确率 + 延迟。
- **10 个 CLI 覆盖深度不均**:claude / codex / agy 是深度集成,crush / pi / copilot 走裸 stdio shim,边界要自测。README 里写的"10 supported"和"10 production-ready"是两回事,选你要的那个。
- **安全模型**:shim 把所有 CLI 事件 POST 到 localhost,本机任意进程都能读。起码绑 127.0.0.1(默认即是),但 unix socket + 权限是更好的工程选择。Munder Difflin v0.4.5 走的就是 127.0.0.1,生产部署前要换 unix socket。
- **license 兼容性**:项目 MIT,但 shim 引的第三方包(`@xterm/*`、`pixi.js`、`node-pty`)各自的 license 要过一遍 AGPL / MIT / BSD 兼容性矩阵。

> 为什么重要:harness 把所有 CLI 变成"可信客户端",本地攻击面反而变大。部署前过一遍 SkillsLLM 这类扫描,别只盯 LLM 输出来不及对钩。另外,shim 自身运行在用户 shell 上下文里,任何拿到你 terminal 的人都可以直接 cat 它的事件流 —— 把 access control 加到 harness 而不是 agent 上,才能避免成为新的侧信道。SkillsLLM 在本次扫描里 returned warnings、no critical,但"no critical"不等于"没风险",留个心眼。

## 7. 接下来看什么

三件我接下来会盯的事,排个优先级给你参考:

1. **DeepSeek Harness** 同期 Show HN 排第 6,主打本地 + 隐私,值得对照读源码。它和 Munder Difflin 的取舍差异最能讲清楚"harness 设计空间":一个全云协作,一个全本地缓存,目标用户群完全不同。
2. **codex 的 `--json-events`** 最近一个月加了 reasoning 事件(模型内部的思考链),影响所有 harness 归一化层。claude 也在跟 —— 你读到这篇时大概率已经有两到三家支持,shim 的 schema 要么跟随要么注明不支持。
3. **多 agent 协作协议**正在成"事实标准":MCP 供工具、a2a 供 agent-to-agent,中间的"harness 内部协议"还没人认领 —— 2026 下半年的空白。Munder Difflin 的做法可能变成事实标准之一,值得早读 repo 早占位。

如果只能做一件事:**去 clone [`munder-difflin`](https://github.com/chaitanyagiri/munder-difflin) repo,跑它官方 demo,再回头看一遍本文第 3 节的 shim** —— 你会比读 10 篇评测更懂 harness 在干什么。

---

Harness 不会取代模型,但它会替换"怎么用模型"。Show HN 216 分、3.6k star 的信号很清楚:**未来 12 个月,prompt 工程之外,你还需要会 harness 工程**。学会写 shim、比懂 RAG 更值得投时间。

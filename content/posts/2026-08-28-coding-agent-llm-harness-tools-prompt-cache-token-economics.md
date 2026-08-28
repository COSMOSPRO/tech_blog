---
title: "Coding Agent 的 token 预算书:Claude Code 的 prompt cache 经济性到底怎么回事"
date: 2026-08-28T12:29:03+0800
tags: ["claude-code", "prompt-cache", "token-economics", "agent-harness", "anthropic"]
author: "COSMOSPRO"
draft: false
summary: "Claude Code 一轮典型 session 里 45k input + 13k output 中 38k 命中 cache_read,总开销 $0.34 —— 这张账单背后是 Anthropic 1.25× 写入 + 0.1× 读取的定价、4 个 cache breakpoint 上限、20-block lookback 窗口三件事共同决定的工程现实。正确放置 breakpoint 的 harness 能把单轮 33k–85k token 的开销压到输入基价的 10%–25%,而错误的 effort 切换、subagent 默认 cache 关闭、25-token 时间戳前移,任何一个都能把单次 session 从 $0.59 拉到 $4.24。"
---

打开 Claude Code 的 `/cost` 命令,你大概率会看到一组让你怀疑人生的数字:典型一轮 session 跑下来 45k input tokens 里 38k 命中 cache 读(只有 7k 走全价 input),外加 13k output,账单上是 `$0.34` —— 其中 output 单价 $15/M 直接吃掉约 $0.195,占比近六成;真正承载 cache 控制逻辑的是 cache_write $3.75/M、cache_read $0.30/M、全价 input $3/M 三列分别累加的结果。这套账本的根源不在 Claude Code,而在 Anthropic 的 prompt caching 定价模型——5-min 写入 1.25×、1-hr 写入 2×、读取 0.1×——以及"每请求最多 4 个 cache breakpoint"的硬上限。把这两件事和 harness 自己的账单结构对到一起,agent 的 token 经济性才真正开始说得清。

## 为什么 Claude Code 比 OpenCode 重 4.7 倍

在讨论缓存之前,先要回答一个更基本的问题:Claude Code 一发起请求就要送 ~33k tokens,这部分到底装了什么?把 Sonnet 模型下的一次 first-turn payload 拆开看:

- 系统提示词约 26,891 字符 ≈ 6.5k tokens
- 27 个 tool schema 约 99,778 字符 ≈ 25k tokens
- 首条消息的脚手架(目录结构、CLAUDE.md 注入、session 元数据)≈ 2k tokens
- 合计 ~33k tokens,在排除用户输入之前就已经全部送出

对比 OpenCode 在同样 read-only Grep 任务下只送出 6.9k tokens,差距是 4.7×。差异的主因不是工具数量,而是 system prompt 和 tool schemas 的体量——前者比 OpenCode 多 3×,后者多 4.7×。如果加上 MCP server 全配齐的极端情况,harness 单轮 payload 可以膨胀到 75k–85k tokens,占满 200k 上下文的 40%。

这串数字解释了 Anthropic 1,024 tokens 的 cache 最小长度门槛为什么"对 Claude Code 远远满足":tools+system 本身就比 1,024 tokens 大一个数量级,几乎不存在"凑不够"的窘境。对 harness 工程师来说,这意味着**只要把 cache breakpoint 摆对位置,几乎所有 system+tools 都能稳定命中**;真正会出问题的,是人为把缓存打碎。

> 为什么重要:很多人误以为 cache 命中率是"API 自己的事",但实际上决定命中率的,首先是 harness 的账单结构(system+tools 在不在同一个稳定段、history 怎么追加、breakpoint 落不落得准)。Claude Code 33k vs OpenCode 6.9k 的对比告诉你:**同样的任务,harness 设计差异就能制造 4.7× 的输入基数**,cache 命中率高也救不了一个基数过大的账单。

## Anthropic 的 cache 定价:三列账本怎么读

把 cache 这件事变成可计算的经济问题,Anthropic 文档给出的公式其实很短。当前(2026 年 8 月)的统一倍数为:

- **5 分钟 TTL 写入**:1.25× 输入基价
- **1 小时 TTL 写入**:2× 输入基价(需要在 `cache_control` 里显式声明 `"ttl": "1h"`)
- **读取**:0.1× 输入基价

每次请求最多 4 个 cache breakpoint,Sonnet 4.6 的最小可缓存长度从旧模型的 4,096 tokens 降到 1,024 tokens。Claude 渲染顺序是固化的 `tools → system → messages`,这意味着 breakpoint 必须落在**稳定段的尾部**——比如 tools 列表的最后一条之后、system 提示词的最后一条之后——而不是稳定段的中间。

这套规则翻译成账单视角会变成:一次 45k input 的请求,如果 38k 命中 cache(读 0.1×)、约 7k 走全价 input(1×),同时再考虑 output 单价通常是 input 的 4–5 倍,**真正的成本重心是 output,其次是 cache_write,最后才是 cache_read**。aakashx.com 在 350 session / 74,493 turn / 18.6B input token 的大样本里复现了这个结构:context 是账单主体,output 是 round-off 的放大器,Opus 4.6 在 Sonnet 同等使用模式上还要乘 5× 价格乘数($15/M 输入对 $3/M)。

把 Anthropic 这套和 OpenAI 的 cache 路径横向放在一起,差异会非常刺眼:OpenAI 的 gpt-4o / gpt-4.1 自动 prefix cache 只给 50% 折扣,到了 GPT-5.4 / GPT-5.5 直接拉到 90% 折扣(也就是 0.1× 读、几乎对齐 Anthropic 的 cache_read),且**完全不收 cache_write 费用**——你不需要打 breakpoint,也没有 4 个上限,系统自己按 prefix 哈希算命中。代价是:对应口径分别是 GPT-5.4 cache_read $0.25/M、GPT-5.5 cache_read $0.50/M,后者比 Anthropic 的 $0.30/M 高 67%,但因为零写入溢价,在一轮 80k token 里 70k 命中的典型场景下,gpt-5.4 的 cache 成本仍比 Sonnet 4.6 便宜 15–20%(精确数字要看 base input 单价 + 命中率组合)。这个差异决定了 harness 工程师的真实选项:**Anthropic 是"你负责摆对,我给你精确控制";OpenAI 是"你别管,我替你摆"**——前者适合需要根据运行时切换 effort / tool_choice 的复杂 harness,后者适合 prompt 结构相对固定的批处理 / 检索场景。

> 为什么重要:很多团队盯着 cache 命中率做优化,但命中率只是分母游戏。真正的杠杆是 cache_write 倍数(选 5-min 还是 1-hr TTL)、output 单价(能不能让模型少说话)、以及单轮 payload 基数(tools+system 是不是能瘦下来)。三者优先级是 output > 写入 TTL > 命中率。OpenAI 路径的"免费写入"等价于把 TTL 选择从必答题变成可忽略项,但也意味着你失去了"强制短 TTL 来限制 cache 跨 session 暴露"的控制权。

## 在 Anthropic 4 breakpoint 上限内做实战摆放

Anthropic 的"4 个 breakpoint"不是营销数字,而是给 harness 设计师画的格子。把它们填到 Claude Code / LangChain 这类典型 agent 上,可以这样安排:

```python
# 1) tools 列表末尾 —— 第一个 breakpoint
tools = [
    {"name": "Read", "description": "...", "input_schema": {...}},
    {"name": "Write", "description": "...", "input_schema": {...}},
    # ... 共 27 个 tool
    {"name": "Bash", "description": "...", "input_schema": {...},
     "cache_control": {"type": "ephemeral", "ttl": "1h"}},
]

# 2) system 提示词末尾 —— 第二个 breakpoint
system = [
    {"type": "text", "text": CLAUDE_CODE_PERSONA, "cache_control": {"type": "ephemeral"}},
    {"type": "text", "text": USER_CONTEXT,        "cache_control": {"type": "ephemeral", "ttl": "1h"}},
]

# 3) 会话级上下文(用户画像、检索到的文档)—— 第三个 breakpoint
# 4) 用户消息、时间戳、session id —— 不缓存,放在最后
messages = [
    {"role": "user", "content": [{"type": "text", "text": "fix the bug in auth.ts"}]},
]
```

注意三件事:第一,`cache_control` 必须打在**稳定段的最后一条**,而不是预计会变的那一条——这是 20-block lookback 窗口的硬约束(详见下一节);第二,1 小时 TTL 只在确实跨 session 复用时才划算,否则 5 分钟 TTL 的 1.25× 就够了,2× 写入溢价会被提前 expire 浪费掉;第三,LangChain 的 `AnthropicPromptCachingMiddleware`(`ttl: '5m' | '1h'`)会自动在 tools+system 末尾注入 breakpoint,这意味着你只要不破坏 Anthropic 的渲染顺序,框架层就能拿到 95%+ 的命中率;Deep Agents SDK 进一步做到"memory 更新后只丢尾段 cache",前面所有 token 仍然命中。

如果你用的是 LangChain 而不是直接调 `anthropic.messages.create`,配置起来会更轻:

```python
from langchain_anthropic import ChatAnthropic
from langchain_anthropic.middleware import AnthropicPromptCachingMiddleware

model = ChatAnthropic(
    model="claude-sonnet-4-6",
    max_tokens=8192,
)

# ttl='5m' 是默认;选 '1h' 时 cache_write 翻倍(2×),但 1 小时内所有请求
# 的 cache_read 都稳定命中。对长 session / agent loop 几乎一定选 1h。
# min_messages_to_cache 控制至少积累几条消息后才注入 cache_control,
# 避免对短请求多花写入费。
middleware = [AnthropicPromptCachingMiddleware(ttl="1h", min_messages_to_cache=2)]

agent = create_agent(
    model=model,
    tools=[read_file, write_file, grep, bash],
    middleware=middleware,
    system_prompt="You are Claude Code, Anthropic's official CLI ...",
)
```

Deep Agents SDK 的设计更进一步:它把工具/MCP server/技能/Skills 这类**结构化、变更频率低**的部分作为独立 cache 单元,日常 memory 更新只会触发"该 cache 单元之后的所有 token 重新写入",前面整段(通常是 70–80% 的 input)保持命中。换句话说,**只要业务变更落在你预期的"易变层"之内,Deep Agents 的 cache 命中率几乎自动保持在 99%+**,而手动维护 cache_control 的 harness 经常跌到 70–80%。这两条 framework 路径(LangChain 自动注入 + Deep Agents 结构化分层)的存在,把"harness 工程"这件事从"每个 provider 自己拼"推进到了"框架层帮你拼"。

> 为什么重要:breakpoint 不是"多打几个就赚",而是"打错一个就全废"。把 breakpoint 打在不稳定内容之前,等于主动让前面所有稳定 token 也跟着 miss——这正是接下来要说的"沉没成本点"。而 LangChain / Deep Agents 这类 framework 的存在,本质上是把"打 breakpoint"这件容易出错的事从 harness 工程师手里收回到框架层,这是过去一年 agent infra 最大的工程红利之一。

## 五个让 cache 全线崩溃的姿势

工程师视角下,Claude Code 的 cache 命中率很难做到 100%。常见的破坏路径有以下五种,每一个都有真实 issue / 案例支撑:

**(1) Agent SDK 的 subagent 默认 cache 关闭。** GitHub issue #29966 复现得很清楚:同一 session 内 104 个请求中,主 CLI 的 50 个请求全部命中 cache,而 54 个 subagent 请求 0 命中。原因是 SDK 硬编码 `enablePromptCaching: false`,subagent 每次都要重新支付 ~7k+ tokens 的 system+tools 成本。

**(2) 变化的部分放到了稳定段前面。** cache key 是 breakpoint 之前所有 token 的精确哈希,任何一点变化——时间戳、session id、UUID、用户的 `user_id`——都会让整段失效。一句话经验:**稳定不变的部分放最前面,每请求变动的部分放最后面**。如果把 25-token 的时间戳塞到 system 顶部,30 分钟一次的 cache miss 就能把单次 session 从 $0.59 拉到 $4.24,这是 Anthropic 优化文档自己引用的数字。

**(3) 切换 thinking effort 或 `output_config.effort`。** Anthropic 平台文档明确写过:"per-effort 切换等于全冷启"。任何一次 effort 改变、thinking config 改变、`tool_choice` 改变、加入 `output_config.effort`,都会重新前缀化对应层,即使前面的 token 字面上没变。

**(4) 20-block lookback 窗口溢出。** 自动模式下,agent loop 单轮新增超过 20 个 content block 时,下一个请求可能落不进 lookback 窗口,导致连续 miss。**breakpoint 必须标在"最后稳定那个 block",而不是"预计会变的那个 block"**——前者是窗口内的最后一个稳定锚点,后者会被新内容推出窗口。

**(5) `CLAUDE.md` 反复重新生成 + system prompt 单调增长。** Issue #45188 记录了 v2.1.89 → v2.1.96 之间 system prompt 增长 ~70k tokens,经常需要 `/compact` 才能继续;issue #40459 进一步发现 v2.1.84 之后 Explore/Plan/built-in subagent 不再注入 CLAUDE.md,组合起来就是 subagent 行为衰退的根因。**增长的 system prompt 比命中率下降更可怕**,因为它直接扩大了基数。

一个特别值得展开的例子是 effort 切换:Anthropic 平台文档明确写过,在多轮 session 里"per-effort 切换等于全冷启"。换句话说,如果你的 harness 在第 5 轮因为某个 tool result 触发而把 `output_config.effort` 从 `"low"` 切到 `"high"`,即使前面 4 轮的所有 token 字面都没变,平台也会把它们当成全新 prompt 重新前缀化。这意味着 harness 工程师必须做两个决定:**要么在 session 早期就把 effort 锁定(放弃 runtime 自适应)**,**要么完全放弃 cache 期望,转而依赖更短的 system prompt 和更聪明的 compaction**——很多团队在实测之后选了前者,因为 99% 的场景下 effort 不需要切换,1% 的场景下命中率崩塌带来的成本远超 effort 切换带来的质量收益。

> 为什么重要:这五个姿势不是孤立的——它们经常同时发生。subagent 不开 cache(姿势 1)+ system prompt 增长(姿势 5)+ effort 切换(姿势 3),三层叠加下,你的 cache 命中率会从理论 95% 掉到实测 30% 以下,而且你看 `/cost` 完全看不出原因,只会看到 $0.34 变成 $4.24。这也是为什么 Anthropic 在优化文档里专门强调"先看 cache_write 的频率,再看 cache_read 的命中率"——前者告诉你哪里在破坏,后者只告诉你破坏后的状态。

## 一张真实账单的拆解公式

把上面的所有规则压缩成一个可以直接跑的 Python 函数,给一个 80k token 单轮场景算四列账单:

```python
# 输入:实测的 usage dict + 模型基价(USD / 1M tokens)
def autocompute_cost(usage: dict, base: dict) -> dict:
    in_tok       = usage["input_tokens"]
    cache_read   = usage.get("cache_read_input_tokens", 0)
    cache_write  = usage.get("cache_creation_input_tokens", 0)
    out_tok      = usage["output_tokens"]

    cost = {
        "input":        in_tok     / 1e6 * base["input"],
        "cache_read":   cache_read / 1e6 * base["input"] * 0.1,
        "cache_write":  cache_write/ 1e6 * base["input"] * 1.25,   # 5-min TTL
        "output":       out_tok    / 1e6 * base["output"],
    }
    cost["total"] = sum(cost.values())
    cost["hit_rate"] = cache_read / (in_tok + cache_read + cache_write)
    return cost

# Claude Sonnet 4.6 基价(2026-08)
base = {"input": 3.00, "output": 15.00}

# 一个 80k token 单轮的实测 usage
usage = {
    "input_tokens": 6_000,
    "cache_read_input_tokens": 70_000,
    "cache_creation_input_tokens": 4_000,
    "output_tokens": 1_200,
}
print(autocompute_cost(usage, base))
# {'input': 0.018, 'cache_read': 0.021, 'cache_write': 0.015, 'output': 0.018,
#  'total': 0.072, 'hit_rate': 0.875}
```

关键观察:在 hit_rate=87.5% 的情况下,cache_read 只占 $0.021,但 cache_write 的 $0.015 + 全价 input 的 $0.018 + output 的 $0.018 三项加起来 $0.051,远大于 cache_read。**如果把 TTL 从 5-min 误选成 1-hr(2×),cache_write 翻倍到 $0.030,total 直接涨到 $0.087**。换句话说,TTL 选择失误对单轮账单的冲击,比命中率从 90% 跌到 80% 还大。

> 为什么重要:很多 harness 自带的 cost 仪表盘只显示"总开销",没法让你看到是哪一列在涨。把这四列拆开,再把 cache_write 的 TTL 单独算,你会发现 optimization 的真正战场在 output 控制 + TTL 选择 + subagent cache_control 默认值,而不是"再多塞几个 breakpoint"。

把上面那张 80k token 的账单横向推到 OpenAI 路径上,你会看到完全不同的形状:gpt-5.4 同样 80k token 的请求里 70k 自动命中 cache_read(0.1× of $2.50/M),1.2k output($15/M),其余 9k 是首轮建立缓存 + 后段变量的全价 input。算下来 cache_read = $0.0175,output $0.018,uncached input $0.0225,total 约 $0.058——比 Anthropic 的 $0.072 便宜约 19%。但注意一个反直觉点:OpenAI 的旗舰 GPT-5.5 cache_read $0.50/M 比 GPT-5.4 的 $0.25/M 贵一倍(精确数字看月度账单要看 base input 单价 + 命中率组合),**只靠 cache 命中是赢不了 Anthropic 的**,关键在于"零写入费"和"自动 prefix"省下来的工程摩擦。两套路径的总成本接近,但工程代价截然不同——Anthropic 路径要你做 7 步 checklist,OpenAI 路径几乎为零。

## 把这些规则固化到 harness 工程清单

收尾给一份可以贴在 harness 仓库 README 里的"cache 体检表":

1. **结构顺序**:tools → system → messages,breakpoint 只打在这三段的**最后一条**,不打在中间。
2. **TTL 选择**:session 内复用且负载大就 `1h`(2× 写入),否则 `5m`(1.25× 写入)。永远不要默认 1-hr。
3. **subagent 开关**:Agent SDK / 自研 harness 里所有 subagent 入口显式打开 `enablePromptCaching: true`,否则就是默认零命中。
4. **稳定段前移**:timestamp / session_id / user_id / UUID 全部放在 messages 数组的最末尾,**不进** system 或 tools。
5. **effort 锁定**:单 session 内不要切换 thinking effort、`tool_choice`、`output_config.effort`,每个切换 = 一次全冷启。
6. **20-block lookback**:agent loop 单轮新增内容块控制在 20 以内;如果不可避免,在**最后稳定 block** 打 breakpoint,而不是"预计变化 block"。
7. **system 体积治理**:监控 `claude_code` 等系统的 prompt 长度,设上限(比如 30k tokens),超过自动触发精简或 compaction。

接下来值得继续追踪的两件事:一是 Anthropic 是否会把 cache_write 倍数从 1.25×/2× 进一步分层(类似 OpenAI 的 auto-90%),二是 Claude Code / LangChain 会不会在某个版本里把 subagent 的 `enablePromptCaching` 默认值翻成 `true`——后者对终端用户来说几乎是"零成本的红利",也是 harness 工程师下一次能直接从上游拿到的优化。再远一点,如果 Anthropic 把 Sonnet 4.6 的 1,024 tokens 最小长度继续往 512 tokens 推,某些 tool 数量少的小型 harness(比如只有一个 Bash 工具的简单 agent)也会第一次够得着 cache 门槛——这件事现在还远,但 platform 团队过去两个季度一直在往"更短可缓存长度"的方向走,值得保持关注。

最后一句话:cache 经济性不是一个静态指标,而是 harness 设计和平台规则的**耦合函数**。同样的 Sonnet 模型,同一个 30k tool schema,在 Claude Code / LangChain / Deep Agents 三个 harness 上能跑出完全不同的账单。把"breakpoint 摆在哪"这件事当成架构决策(而不是 prompt 工程的边角),是 agent 时代 infra 团队区别于"会调 API"团队的分水岭。
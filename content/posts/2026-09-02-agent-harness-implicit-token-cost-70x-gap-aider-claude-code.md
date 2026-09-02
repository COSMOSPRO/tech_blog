---
title: "同一个模型同一个任务,token 账单差 30~50 倍:Agent Harness 的隐性成本"
date: 2026-09-02T02:00:00Z
tags: ["agent-harness", "token-cost", "prompt-caching", "claude-code", "aider", "openclaw"]
author: "COSMOSPRO"
draft: false
summary: "同一模型同一任务,Aider 700 token 跑完,OpenClaw 24K — Sonnet 5 上 33×,换免费档模型再跑仍能到 40×,差距从哪里来?这次把 startup_tax × turns 这条乘法算式、Anthropic #45188 真实事故、以及 prompt cache 翻转账单视角一起拆给你看。"
---

# 同一个模型同一个任务,token 账单差 30~50 倍:Agent Harness 的隐性成本

如果有人告诉你:"同一款 Claude Sonnet 5、同一份 SWE-bench 任务,跑下来 Aider 只花 708 个 token,而 OpenClaw 要 23,682 个",你大概会以为 Aider 没做完。事实是 — 两个 harness 都完成了任务,账单却差了 33 倍;换一个免费档模型(Nemotron 3 Ultra)在 OpenRouter 上复跑同一套 harness,极差还能拉到 40 倍以上(基准报告原话"40x difference before any work has started")。这不是模型问题,是 harness 问题。

## 为什么这件事值得认真拆

2026 年 9 月,Eishan Lawrence 发布了一份独立跑度基准,12 个 harness × 2 个模型 × 12 个 SWE 任务,逐次比对 `per-solved-task tokens`。他做了两件事很关键:第一,把所有 harness 的 system prompt 抠出来对比;第二,把每次会话的 `turns`(模型往返次数)记下来。然后他做了一个极简单的回归:

```
total_tokens ≈ startup_tax × turns
```

`R² = 0.99`。也就是说,一个 harness 在一个任务上的总开销,99% 能被两个数字解释:**你每次会话开始要喂进 context 的固定前缀有多长(startup_tax),以及这件事你要来回跑多少趟(turns)**。跨模型的相关性进一步验证:同一套 12 个 harness 在两套完全不同的模型上跑出来的 startup_tax 几乎贴在 y=x 上(Spearman ρ ≈ 1.0)— 哪个 harness 的 startup_tax 高,哪个就贵,这跟底模无关。

## 三个 harness,三套拼装系统提示的方式

我把这次基准里最值得对比的三家(Aider、Claude Code、OpenClaw)各自的 system prompt 拼装逻辑拉到一起。下面这段是简化版伪代码,展示它们各自怎么把"开局固定前缀"塞进 context,真实实现分散在各自 repo 的不同文件里:

```python
# ─── Aider:base_coder.py ──────────────────────────────
# 单段 SEARCH/REPLACE 规格,~700 token,EditBlockPrompts.main_system(editblock_prompts.py)
SYSTEM = read("editblock_prompts.py")             # EditBlockPrompts.main_system,~700 tokens
litellm.completion(..., caching=True,              # 首次写入,后续按 cache_read 0.1× 计价
                   cache={"caches_by_default": True})

# ─── Claude Code:cli.js ──────────────────────────────
# __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ 注入点,叠加 skills/plugins/MCP schemas
SYSTEM = base_template
SYSTEM += skill_listing           # 47 个 skill → 几十 KB
SYSTEM += plugin_metadata
SYSTEM += mcp_tool_schemas        # 每加一个 MCP,工具 schema 全量进
# 实测 v2.1.89:cache_creation ≈ 38K;v2.1.96:cache_creation ≈ 112K
# cnighswonger/claude-code-cache-fix 拦截器 CACHE_FIX_DEBUG=1:
#   system=27893 tools=70176 injected=1627 (skills=1627 mcp=0 deferred=0)

# ─── OpenClaw:workspace 启动顺序 ──────────────────────
# agents.defaults.bootstrapMaxChars = 20,000 / 文件
# agents.defaults.bootstrapTotalMaxChars = 150,000
SYSTEM = read("SOUL.md")        # 人格,每 turn 注入
SYSTEM += read("IDENTITY.md")   # 自描述
SYSTEM += read("AGENTS.md")     # 操作规则 + 沙箱 preamble
SYSTEM += read("USER.md")       # 用户档案
SYSTEM += read("TOOLS.md")
SYSTEM += read("MEMORY.md")     # 仅私聊会话
SYSTEM += read("HEARTBEAT.md")
# 实测:这套 7 个 bootstrap 文件全开,起步 ≈ 26K token,多 turn 任务可堆到 ~120K
```

**为什么重要:** Aider 选了"少而精"路线 — 一段 700-token 的 SEARCH/REPLACE 规格加 litellm 自动缓存,system prompt 几乎不变,后面所有 token 都是真正的"任务工作"。Claude Code 把 skills/plugins/MCP schema 全量进 system prompt,每个新版本都在加东西 — `v2.1.89 → v2.1.96` 五天涨了 70K token,这是真实事故,后面单独说。OpenClaw 则把"人格 + 记忆 + 工具说明"全部塞进每轮 system prompt,设计哲学是"agent 要有连续身份",代价是每次 turn 都要重传这 26K+ 的引导文本。

换算下来,OpenClaw 一个 20 turn 任务的固定前缀开销就能到 `26K × 20 = 520K`,即便 prompt cache 把大部分压到 0.1x,base input 单价乘下来也相当可观;而 Aider 的 700 × 20 = 14K,差了一个数量级。

## 一个真实事故:Claude Code #45188

2026 年 4 月,用户 Wishbringer71 在 anthropics/claude-code#45188 报告了一个让他"必须不断手敲 /compact 才能工作"的 bug。他抓了所有 session JSONL,把首条 assistant 消息的 `usage.cache_creation_input_tokens`(还没命中缓存、反映完整 system prompt 大小)对 version 字段做了相关性分析:

| 版本 | 首条 cache_creation | 相对基线 |
|---|---|---|
| v2.1.89 / 2.1.90 | 38–52K | baseline |
| v2.1.91 | 80–88K | **+40K** |
| v2.1.92 | 85–90K | +40K(稳定) |
| v2.1.96 | 112–119K | **+70K 累计** |

**为什么重要:** 两次阶跃式跳变 — `2.1.91 +40K`、`2.1.96 +30K`,加在一起 5 天涨 70K。Anthropic 团队后来回应,在 cnighswonger 的 cache-fix interceptor 最小配置(8 skills、3 plugins、1 MCP)下,实测 `system=27893 tools=70176 injected=1627`,即 base system 28K、tool schemas 70K、动态注入 1.6K。也就是说问题不在 base prompt 本身,而是 skills/plugins/MCP 越多、每次升级往 system prompt 加的元数据就越被放大。这件事的工程教训:**固定前缀不是免费的,它乘上 turns 就是账单**。

## Prompt cache 翻转账单视角

如果你只看 raw tokens,Aider 赢得很彻底。但账单不是 raw tokens。Anthropic 官方定价:cache_read = 0.1× base input、cache_write(5min TTL)= 1.25×、cache_write(1h TTL)= 2×;新发布的 Fable/Mythos 5.1 cache_read 直接打到 0.025×。

把这套定价套到上面的乘式:`bill ≈ (startup_tax × turns) × cache_hit_ratio × price_multiplier`。

Aider 的优势在于 hit_ratio 高 + price_multiplier 低,但代价是任务能力窄 — 单调用 loop、不能跑 agentic 工作流。Claude Code 的 cache hit 没那么漂亮,但 tool/skill 生态完整。OpenClaw 则是反向选择 — 把"人格连续性"放在第一位,接受 token 账单。

**为什么重要:** Raw token 仪表盘说谎。真要看一个 harness 贵不贵,要看三个数一起 — startup_tax(版本固定成本)、turns(任务结构)、cache_hit(实现质量)。Codex 团队曾披露一个真实案例:把工作流改成 cache-friendly 之后,77% 的 input 都按 0.1× 计费,raw tokens 没降、账单降了一半。

## 5 种 context-bloat 行为分类

整理这次基准里看到的所有 bloat 模式,我把它分成 5 类,按危险程度排序:

1. **静默截断(silent truncation)** — Kilo Code、OpenCode 在 context 装不下时悄悄砍掉中间内容,**最危险**,模型会基于不完整信息继续工作而不报错。
2. **up-front 拒绝** — OpenClaw 在塞不下时直接说"做不了",**最诚实**,但用户体验差。
3. **忠实复读** — Claude Code 优先保 system prompt 完整,**忠实但任务失败**,常需要手动 /compact。
4. **前缀抖动** — Skill 列表每次返回顺序不一致,bust cache 独立于大小 — cnighswonger 的 interceptor 用字母排序稳定顺序才修掉这个问题。
5. **服务端注入** — claude-code#46917 实测 v2.1.100 后,同样 payload 因为 User-Agent 不同多扣 17K cache_creation token,服务端路由逻辑在加东西。

**为什么重要:** Harness 的工程差异远比模型差异更影响账单。选 harness 不是选"哪家最聪明",是选"哪家愿意为 prompt engineering 付多少固定成本"。

## 接下来看什么

下一篇我会拆 Aider 的 editblock format — 它怎么用 700 token 把 SWE-bench 任务做成单调用 loop,以及这个设计能不能移植到 Claude Code 的 tool schema 里压成本。

---

**参考资料**
- Eishan Lawrence, *Harness Efficiency Benchmark*, 2026-09
- arXiv 2608.01347 v3, *Same Task, Different Work: Prompt-Induced Waste*(4,644 valid runs / 24 tasks / 6 reasoning models)
- anthropics/claude-code #45188, *System prompt size grew ~70K tokens between v2.1.89 and v2.1.96*
- cnighswonger/claude-code-cache-fix, `CACHE_FIX_DEBUG=1` interceptor logs
- platform.claude.com/docs/en/build-with-claude/prompt-caching,官方定价
- sahajamit/openclaw-deep-dive §06, *Soul Persona & System Prompt*
- Composio *August 2026 8-Harness Cost-per-Solved Report*

[WRITE COMPLETE]
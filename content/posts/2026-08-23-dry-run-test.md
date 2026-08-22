---
title: "把多 Agent 流水线跑成 \"dry run\":测试金字塔 + 黄金数据集的双轨范式"
date: 2026-08-23T09:00:00Z
tags: ["LLM", "agent", "testing", "eval-driven", "golden-dataset", "ci-cd"]
draft: false
author: "COSMOSPRO"
summary: "用 Hermes Kanban 的 orchestrator→researcher→writer→reviewer→publisher 链做样本,讲清 LLM/agent 流水线的工程化测试范式:测试金字塔 70/25/5、黄金数据集三档规模、checkpoint 时间旅行与 VCR.py 录制回放,以及 eval-driven 把质量门槛推到 merge 前。"
---

# 把多 Agent 流水线跑成 "dry run":测试金字塔 + 黄金数据集的双轨范式

你刚把一条多 agent 编排链跑通——researcher 出调研简报、writer 写文章、reviewer 审稿、publisher 发布——但第一次跑就把线上环境写花了。下一次你不敢再直接 "production dry run",想把整条链先在本地跑一遍,确认每一手 handoff 都正确。这就是 LLM 流水线测试要解决的问题:让一条由模型、prompt、状态、外部 API 拼起来的不确定性链路,变成一组可以在 CI 里反复回放、可以断在中间节点的工程对象。

最近在工程社区里浮现出的共识,可以浓缩成一句话:**测试金字塔 + eval-driven 的双轨范式**。前者负责结构正确性,后者负责质量门槛;两者缺一不可。下面我们用一条真实的 agent 流水线当主轴,把它讲清楚。

## 一、为什么 LLM 流水线不能套传统测试

传统软件测试有一个隐性假设:同一个输入、同一个被测对象,应该得到同一个输出。但 LLM 推理天生带概率性——即使把 temperature 调到 0,推理路径也常常因为底层 batching、KV cache、模型版本切换等微因素出现抖动。OpenLayer 在 Agent Testing Complete Guide(2026)里把它总结成一句话:**broken feature in agents works 95% of the time and fails unpredictably**——同一 prompt 重跑十次,可能有两次的中间步骤顺序不同,有一次会陷入循环。要把这部分行为暴露出来,光写 `assert output == expected` 是不够的。

更糟的是,流水线的"输出"不是单一字符串,而是一连串状态转移。researcher 任务结束时要写一段 `[RESEARCH COMPLETE]` 到 task comment;writer 任务开始时要读这条 comment;reviewer 要在 article 路径上校验文件存在且字数在 1800-2200 之间。任何一个 handoff 失败,后面整条链就废了。

这就是为什么社区会沉淀出"测试金字塔 + 黄金数据集"的双轨结构:**用单元测试守住确定性环节(协议、契约、文件 IO),用 eval 守住概率性环节(prompt、模型、生成内容)**。OpenHelm Agent Testing Strategies 给出一个经验比例:**70% unit / 25% integration / 5% E2E**,且 95% 的测试应该 mock LLM 调用,只有 5% 走真实模型。

## 二、测试金字塔:三层的工程分工

第一层 **unit(毫秒级,每个 commit 触发)**。每个 agent 节点单独测:researcher 的 handoff 是否含 `[RESEARCH COMPLETE]` 标记、writer 写出的 frontmatter 是否严格 5 字段、reviewer 是否在校验字数时使用 `wc -m`。这一层完全 mock 掉 LLM,跑得快、改动反馈即时。

第二层 **integration(秒级,每个 PR 触发)**。把两个相邻节点拼起来跑:orchestrator→researcher 的链路是否在 task 状态从 `ready` 转 `running` 时正确 spawn worker;writer→reviewer 的链路是否能把 article 路径透传到下一节点。这一层用 LangGraph 的 `graph.nodes` 把单个 node 暴露出来直接调用,绕过 checkpointer,验证纯函数逻辑。

第三层 **E2E(< 15 分钟,只在 main 分支或 staging 前)**。跑完整条流水线,把整条链的最终产物贴回仓库。这一层是真正的 smoke test——CD 平台(Harness / Argo CD / GitHub Actions deployment job)把它定位成 staging→生产前的 go/no-go gate,触发到就直接 rollback。

为什么是这个比例?因为每一层解决的问题不同:**unit 抓住协议错误,integration 抓住状态错误,E2E 抓住系统性故障**。跳过 unit 层直接上 integration 和 E2E,会丢掉快速反馈环,一次 PR 反馈从秒级退化到分钟级;跳过 integration 层直接上 E2E,会失去对单个 handoff 的定位能力——线上炸了,你不知道是 researcher→writer 还是 writer→reviewer 那段坏了。

## 三、黄金数据集:从 50 条到 500 条的拐点

eval 这一轨,核心载体是**黄金数据集(golden dataset)**:一组固定输入 + 固定期望输出的样本。CI 每跑一次 eval,流水线就把所有样本过一遍,统计通过率与各项指标的分布。

但数据集规模有明确的拐点。Galtea 在 LLM Evaluation Complete Guide 里给出经验值:

- **50 条**:能抓大回归(回归测试的最小可用集)
- **200 条**:带来统计置信度,能识别 3-5% 的质量变化
- **>500 条**:进入收益递减区,边际信息越来越少

实操上,大多数团队会从 50 条起步,把每次跑出来的失败案例**反哺**进数据集——这才是 eval-driven 的精髓:数据集不是一次性手工攒的,而是流水线跑出来的副产品。

> 为什么重要:黄金数据集的存在让"质量"变成可度量的对象,而不是"reviewer 觉得不错"的主观判断。

## 四、Checkpoint 时间旅行:让中间产物可断点回放

agent 流水线最容易调试的环节是中间状态。LangGraph 提供了一种"时间旅行"的能力:`graph.get_state_history(config)` 能拉出节点在每一步执行后的完整 state snapshot——输入、输出、messages、next 节点全在。

下面是一段可以直接跑起来的最小骨架,演示怎么对单节点做断言 + 对时间旅行做断言:

```python
# pip install langgraph pytest
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

class State(dict):
    pass  # 简化:实际工程里用 TypedDict

def researcher(state: State) -> State:
    return {**state, "research": "已交付 13 条关键事实 + 25 条来源"}

def writer(state: State) -> State:
    return {**state, "article": "2000 字文章落盘到 article path"}

builder = StateGraph(State)
builder.add_node("researcher", researcher)
builder.add_node("writer", writer)
builder.add_edge(START, "researcher")
builder.add_edge("researcher", "writer")
builder.add_edge("writer", END)

def test_handoff_between_researcher_and_writer():
    # 关键:每个 test 用新 MemorySaver,避免历史串扰
    checkpointer = MemorySaver()
    graph = builder.compile(checkpointer=checkpointer)
    config = {"configurable": {"thread_id": "test-001"}}
    graph.invoke({"topic": "dry run test"}, config=config)

    # 单节点断言:unit 层
    history = list(graph.get_state_history(config))
    assert len(history) == 3  # start + researcher + writer
    assert history[1].values["research"].startswith("已交付")

    # 中间产物断言:integration 层
    final = graph.get_state(config)
    assert final.values["article"].endswith("article path")
```

这里有两个关键设计:**每个 test 用新 checkpointer**(否则历史会串),以及**用 `get_state_history` 而不是直接 `invoke` 后再读最终态**(否则抓不到中间步骤的顺序变化)。

LangGraph persistence 还提供了双层记忆:short-term 是 checkpointers(线程级,内存或 Postgres),long-term 是 Store(跨线程的命名空间存储)。这两层基础设施,正是多 agent 流水线"可重放"的工程底座。

## 五、Eval-driven development:把质量门槛推到 merge 前

传统的 CI/CD 是在代码合入后跑回归测试,但 LLM 应用的"代码"不只是 Python,还包括 prompt 和模型选择。Braintrust 把这套新范式命名为 **eval-driven development**:

> 把 evaluation 当成 LLM 应用的工作规范,prompt / model 变更前先定义质量准则,变更后必须先过 eval 才能进生产。

工程实现通常用 GitHub Action。Braintrust 的 `braintrustdata/eval-action`、Langfuse 的 `langfuse/experiment-action` 都是同类方案:加载数据集、跑实验脚本、把 score 贴回 PR 评论、低于阈值就 fail job 让 merge 直接被 block。

这意味着,改一个 prompt 不再是"改完就 commit",而是"改完前先在 eval 里定义新准则,改完后看分数是否抬升/不低于基线"。在团队里,这相当于给 prompt 工程师套上了一层与 backend 工程师同等的工程纪律。

## 六、Mock 的三档用法

既然 95% 的测试要 mock LLM,怎么 mock?常见做法可以分三档:

1. **fake LLM client**:测 prompt 装配、解析、retry 逻辑。用一个固定返回固定结构的假 client,验证"传出去的 prompt 对不对、解出来的 result 对不对"。
2. **record/replay fixtures**:测请求响应形状。VCR.py / pytest-recording 就是这类——首次跑时把 HTTP 交互录到 cassette yaml,后续 test 直接重放而不再打真实 API。**注意默认配置会把 Authorization 头原样写进 yaml,必须 override `before_record_request` 做 secret / PII redact**,否则 casset 就成了密钥泄露源。
3. **mock server**:测流式、延迟、限流、provider error。用 WireMock 之类起一个本地 server,人为构造 429、5xx、timeout,验证流水线降级路径。

三档对应三种"想测的东西不同":fake 测你自己的代码逻辑,replay 测请求契约,mock server 测错误恢复能力。

## 七、契约测试:让"中间产物断言优于最终断言"

最后一条原则,在多 agent 系统里尤其重要:**契约测试应该断在每一段 handoff 上,而不是只在末尾断言一次**。社区里给出一条经验清单:

- 逐 stage 对比 golden output
- 对确定性 transform 用严格相等
- 对模型 / 概率输出用 `np.allclose(atol=1e-2)` 容差
- 用一个 `ignore_keys` 列表过滤 `request_id` / `timestamp` / `runtime` 等 volatile 字段

在 Hermes Kanban 这条链上,契约包括:**researcher 必须在 comment 末尾写 `[RESEARCH COMPLETE]`**、**writer 必须在 file path 落地 frontmatter 5 字段**、**reviewer 必须在字数不达标时调用 `kanban_request_changes` 而非 `kanban_complete`**。每一段契约都用单元测试断言,任何一段失败,流水线在那个节点就停下来——而不是一路跑到 publisher 才发现 frontmatter 缺字段。

## 接下来看什么

把多 agent 流水线跑成 "dry run",本质上是把"不确定性"包装成"可断点的工程对象":测试金字塔守住协议与状态,黄金数据集守住质量,checkpoint 守住可重放,eval-driven 守住发布门槛。下一步值得跟进的,是 Langfuse / Braintrust / Phoenix 在 production trace 上的差异——线上可观测性,才是 eval-driven 真正闭环的最后一块拼图。
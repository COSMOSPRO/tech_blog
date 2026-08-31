---
title: "LangChain 1.0 / LangGraph 1.0 中间件架构深扒:create_agent 是怎么把一串 middleware 编译成 StateGraph 子图的"
date: 2026-08-31T16:00:00Z
tags: ["langchain", "langgraph", "middleware", "create-agent", "stategraph", "agent-framework", "python"]
author: "COSMOSPRO"
draft: false
summary: "LangChain 1.0 与 LangGraph 1.0 在 2025-10-22 同日 GA,把 agent 抽象收敛到 langchain.agents.create_agent,可扩展性完全交给 middleware=[...]。每个 middleware 是一个 AgentMiddleware 子类,在 LangGraph StateGraph 上按 6 个 hook(before_agent / before_model / wrap_model_call / wrap_tool_call / after_model / after_agent)挂载为命名节点,hook 通过顺序组合成洋葱栈(列表首项最外层)。本文从源码深扒这个组合过程:hook 怎么定义、wrap 链怎么生成、entry_node / loop_entry / loop_exit / exit_node 怎么决定图边、JumpTo 怎么短路,并对照 create_react_agent → create_agent 的迁移矩阵,给出 5 段可运行代码。"
---

如果你的 agent 代码库今天还在用 `langgraph.prebuilt.create_react_agent`,打开 IDE 就会收到一条 deprecation warning——这条 warning 不是善意的提示,而是 LangChain 官方在 2025-10-22 GA 的 LangChain 1.0 / LangGraph 1.0 里明确写下的"已弃用,不再回头"。同一天发布的 `langchain.agents.create_agent` 把整个 agent 抽象收敛到了一个工厂函数,可扩展性被一次性打包成 `middleware=[...]` 一个参数。问题来了:**这一串 middleware 到底是怎么在内部被组装成一个真正 LangGraph StateGraph 子图的?** 如果你只看到 `create_agent(...).invoke(state)`,那就错过了 1.0 这次架构升级最关键的 200 行源码——`factory.py` 里 `_chain_model_call_handlers`、`_route_after_model`、`entry_node` / `loop_entry_node` / `loop_exit_node` / `exit_node` 的四元组,以及 `ModelRequest.override` 提供的"动态路由能力"。

过去一年,生产团队踩过的坑高度同质:`pre_model_hook` 只有一个函数,但你想同时做"PII 脱敏 + 历史摘要 + 动态切模型"三件事时,只能把它们塞进一个回调,再用 if/else 分流;想加第四件事,就得重写整个回调;`post_model_hook` 同理。`create_react_agent` 的 `prompt=` / `state_modifier=` / `pre_model_hook=` / `post_model_hook=` 四个参数,任何一个想替换行为,都意味着重写整个图。这种"参数塞一切"的接口,在简单场景下好用,在生产 agent 复杂度一旦上来就是噩梦——所有团队都在重复造同一个抽象,只是命名不一样。LangChain 1.0 中间件架构就是官方对这件事的统一回应:**把所有"插一刀"的需求,收敛到一个可组合的列表**。

本文从源码层面把这套洋葱栈拆给你看。

## 1.0 架构的三个支柱:create_agent / middleware / standard content blocks

先承认一个反直觉的事实:LangChain 1.0 和 LangGraph 1.0 是**两个独立 package** 同日 GA,但**不**是合并。LangChain 是 high-level + 中间件定制层,LangGraph 是底层 runtime + orchestration 层。`create_agent` 内部用 LangGraph 的 `StateGraph` 构建器把中间件拼成图,再调用 `graph.compile()` 产出 `CompiledStateGraph` 返回给你——所以持久化、流式、人在环(Interrupt)、time-travel 这些 LangGraph 能力**全部继承**——middleware 不是绕开 LangGraph,而是反过来站在 LangGraph 上面。

官方在 v1 release notes 里把这次升级的三个支柱定调为:**create_agent / middleware / standard content blocks**。前两个解决了"怎么把 agent 写得能扩展",第三个解决了"怎么把流式 token 写得不依赖 provider"。三者中,middleware 是被官方反复强调的设计哲学——官方原话:"从参数爆炸走向组合式中间件"。

**为什么这件事重要**:如果你过去两年在 LangChain 上吃过 `LLMChain` 的亏,被 `AgentExecutor` 的 hidden tool calling 困惑过,被 `pre_model_hook` / `post_model_hook` 这种"参数塞一切"的接口恶心过,那 1.0 的 middleware 就是官方对所有这些历史包袱的工程级回应。它不是又一个 hook 系统,而是**唯一一个**官方承诺 "no breaking changes until 2.0" 的扩展点。

## AgentMiddleware 基类:6 个 hook 的洋葱栈

`AgentMiddleware` 定义在 `libs/langchain_v1/langchain/agents/middleware/types.py`,class 定义从 line 385 开始。每个子类最多可以重写 6 个 hook,每个 hook 都有同步版和异步版两套签名:

| Hook | 触发时机 | 返回值 | 用例 |
|------|---------|--------|------|
| `before_agent(state, runtime)` | agent 启动前一次性 | `dict \| None` | 加载长期记忆、初始化 trace tag |
| `before_model(state, runtime)` | 每次模型调用前 | `dict \| None` | 摘要长历史、注入动态 prompt |
| `wrap_model_call(request, handler)` | 围绕模型调用 | `ModelResponse` | 改 request、重试、短路 |
| `wrap_tool_call(request, handler)` | 围绕 tool 执行 | `ToolMessage \| Command` | tool 错误处理、限速 |
| `after_model(state, runtime)` | 每次模型响应后 | `dict \| None` | 校验输出、截断敏感字段 |
| `after_agent(state, runtime)` | agent 结束后一次性 | `dict \| None` | 写审计日志、清理资源 |

子类还可以挂 4 个属性:`state_schema`(自定义 TypedDict 子状态)、`tools`(middleware 自带工具,会自动并入主工具集)、`transformers`(流式 transformer 工厂)、`trace_policy`(TracePolicy 控制 span payload)。

**为什么这件事重要**:注意 `before_*` / `after_*` 这 4 个 hook 都是 `dict | None`——返回值是状态 patch,直接通过 `add_messages` reducer 合并进 `AgentState`。这意味着 middleware 不需要返回完整 state,只需要返回**要改的那部分**。这种"patch 风格"和 LangGraph 的 reducer 模型是天然契合的,middleware 写起来和写一个普通 reducer 一样轻。

`AgentState` 本身在 `types.py` line 349 定义,是一个只有三个字段的 TypedDict,但每个字段的 metadata 都至关重要:

```python
# types.py:AgentState (line 349,简化)
class AgentState(TypedDict):
    messages: Required[Annotated[list[AnyMessage], add_messages]]
    jump_to: NotRequired[Annotated[JumpTo | None, EphemeralValue, PrivateStateAttr]]
    structured_response: NotRequired[Annotated[ResponseT, OmitFromInput]]
```

三个 metadata 标签各自承担一项关键职责:`add_messages` 是 LangGraph 默认消息 reducer,按 ID 合并而不是简单覆盖;`EphemeralValue` 标记 `jump_to` 是一次性信号,跑完一轮路由就消失,**不会进入 checkpointer 持久化**;`OmitFromInput` 标记 `structured_response` 在下一轮 invoke 时不会作为输入消息回喂给模型(避免模型看到自己上次的结构化输出)。这就是 middleware 能稳定控制"短期信号 vs 长期状态"的机制——不需要 middleware 自己清状态,LangGraph runtime 帮你做了。

**为什么这件事重要**:很多人写 middleware 第一反应是"我自己维护一个临时字段",但这会污染 checkpointer,导致 resume 时历史里多出奇怪的中间态。理解了 `EphemeralValue` 和 `OmitFromInput` 这两个 annotation,你写出来的 middleware 就能和 LangGraph 的持久化语义无缝对齐——这是 1.0 比 0.x 干净的隐性收益。

## hook 的洋葱组合:首项最外层,逆序回程

这是 1.0 中间件架构最反直觉的部分。**列表首项是最外层**(`outermost`)——注意这个方向约定:写 `middleware = [A, B, C]`,你以为 A 最先被"调用",其实 A 是最外层,它包裹 B,B 包裹 C,C 才是离 model 最近的那一层。如果你写了:

```python
middleware = [A, B, C]  # A 最外层
```

那 `wrap_model_call` 的实际调用顺序是 `A → B → C → model → C → B → A`,去程正序、回程逆序,这是教科书式的中间件栈。`factory.py` 里 `_chain_model_call_handlers`(定义在 `factory.py` 约 263 行,在 `create_agent` 上方的小工具函数里)就是干这件事的:

```python
# factory.py:_chain_model_call_handlers (示意性伪代码,只表达"首项最外层"递归意图)
# 实际实现走 functools.reduce / functools.partial 累积 sync/async 两组 handler,
# 并被 traceable / RunnableCallable 包裹后注入 model 节点内部,语义一致。
def _chain_model_call_handlers(handlers):
    """Compose wrap_model_call handlers. First in list becomes outermost."""
    if not handlers:
        return None
    handlers_inner = handlers[1:]   # 尾部子链 = inner 层
    def chain(request, handler):
        # handlers[0] = outermost,handler 闭包把它"剩下"的子链接好再交给 model
        return handlers[0](request, lambda r: _chain_model_call_handlers(handlers_inner)(r, handler))
    return chain
```

(伪代码省略 traceable / sync-vs-async 分支,核心是"首项最外层,递归把下一个 handler 作为 `handler=` 参数透传"。`create_agent` 在主装配流程里把 sync/async 两组 handler 串好之后,作为单个 `wrap_model_call_handler` 注入 `model` 节点的内部。)

`before_model` 和 `after_model` 不在同一个链里,而是被工厂装配成**命名节点 + 边**:

- `before_model` 链**正序**串联:`outermost → innermost`,边 `m[0].before_model → m[1].before_model → ... → model`
- `after_model` 链**逆序**串联:`innermost → outermost`,边 `model → m[-1].after_model → ... → m[0].after_model`

这两条链合起来形成"去程洋葱、回程反向"的图结构——这是 1.0 中间件能组合的关键。

**为什么这件事重要**:洋葱栈不是装饰器模式的简单应用。普通装饰器模式下,内层修改对外层不可见;洋葱栈模式下,外层 A 完全可以在 B 还没拿到 request 之前就拦截、改写、甚至直接返回 cached response——这就是"短路"。而回程反向,意味着 A 还能看到 B 处理完 model response 之后的结果,在外层做最后一道过滤。**这是中间件为什么能写"模型降级""结果改写""缓存命中跳过"的核心机制**。

## factory.py 的四元组:entry_node / loop_entry / loop_exit / exit_node

`create_agent` 从 `factory.py` line 772 开始定义,到约 line 1860 结束(后面是各种 edge 构造工具函数)。关键不是节点本身,而是**四个边界节点**,它们决定 START 走哪、END 走哪、回边怎么连——而且 `entry_node` 和 `loop_entry_node` 是两个独立的变量,在 line ~1649–1673 附近算出来:

```python
# factory.py:create_agent 内部 (简化,实际对应行 ~1649-1673)
# entry_node:整个图入口(只跑一次)
if middleware_w_before_agent:
    entry_node = f"{middleware_w_before_agent[0].name}.before_agent"
elif middleware_w_before_model:
    entry_node = f"{middleware_w_before_model[0].name}.before_model"
else:
    entry_node = "model"

# loop_entry_node:每轮 tool 完成后的回边入口(可跑多次,**故意排除 before_agent**)
if middleware_w_before_model:
    loop_entry_node = f"{middleware_w_before_model[0].name}.before_model"
else:
    loop_entry_node = "model"

# loop_exit_node:每轮迭代结束后做条件路由的节点 = after_model 链的最外层(整条链的终点)
if middleware_w_after_model:
    loop_exit_node = f"{middleware_w_after_model[0].name}.after_model"
else:
    loop_exit_node = "model"

# exit_node:整个图出口(只跑一次);没有 after_agent 直接落 END
if middleware_w_after_agent:
    exit_node = f"{middleware_w_after_agent[-1].name}.after_agent"
else:
    exit_node = END
```

图的边长这样:

```
START → entry_node
entry_node → ... → model
model → (conditional)
    ├─ if tool_calls → tools → loop_entry_node  (回边)
    └─ else          → loop_exit_node
loop_exit_node → ... → exit_node
exit_node → END
```

条件路由 `_route_after_model`(由工厂在 model 节点之后注册)检查最后一条 `AIMessage` 是否带 `tool_calls`,有就走 `tools`,否则走 `loop_exit_node`。这就是 1.0 默认 ReAct 循环的图结构。**命名上的小坑**:`loop_exit_node` 和 `exit_node` 都带 "exit",但语义不同——`loop_exit_node` 是 after_model 链**末端**(链式调用走完后的下一个节点,即 `m[0].after_model`);`exit_node` 是整张图**终点**前的最后一个节点(`after_agent[-1]` 或 END)。

**为什么这件事重要**:`loop_entry_node` **故意**不等于 `entry_node`——它只取 `before_model` 链,永远跳过 `before_agent`。这意味着如果你写了 `before_agent` 做"启动时一次性加载长期记忆",它**不会**在每轮 tool 调用之后被重复触发。如果最外层是 `SummarizationMiddleware` 的 `before_model`,那每轮 tool 调用之后**确实**会先摘要历史再喂给模型(因为它挂在 `loop_entry_node` 上)。这种"哪些 hook 跑一次、哪些跑多次"的边界划分,是源码层面最反直觉的设计点——官方注释里直接写明 "loop entry node (beginning of agent loop, excludes before_agent)"。

下面用代码把这一节串起来:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    AgentMiddleware, ModelRequest,
    SummarizationMiddleware, PIIMiddleware, HumanInTheLoopMiddleware,
)
# 注:ModelResponse 在 langchain 1.0.x 仍未从 langchain.agents.middleware
# 顶层 __init__.py 导出(已知 bug,见 langchain-ai/langchain #33453、#33501),
# 必须从 .types 子模块导入。
from langchain.agents.middleware.types import ModelResponse
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    return f"{city}: 22°C, sunny"

# 三个预置 middleware 组合:
# 1) PIIMiddleware 在 wrap_model_call 阶段脱敏
# 2) SummarizationMiddleware 在 before_model 阶段摘要
# 3) HumanInTheLoopMiddleware 在 after_model 阶段对 tool 调用做人工审批
agent = create_agent(
    model="openai:gpt-4o",
    tools=[get_weather],
    system_prompt="You are a helpful weather assistant.",
    middleware=[
        PIIMiddleware(pii_type="email", strategy="redact"),
        SummarizationMiddleware(model="openai:gpt-4o-mini", trigger=("tokens", 4000)),
        HumanInTheLoopMiddleware(interrupt_on={"get_weather": True}),
    ],
)
result = agent.invoke({"messages": [{"role": "user", "content": "Weather in SF?"}]})
```

这段代码把 1.0 中间件架构的**三个不同位置**(wrap / before / after)一次性串齐。

## JumpTo 短路:state 通道里的 EphemeralValue

`before_*` / `after_*` hook 返回 dict 时,有一个特殊字段 `jump_to`,类型是 `Literal["tools", "model", "end"]`。`JumpTo` 定义在 `types.py` line 68:

```python
JumpTo = Literal["tools", "model", "end"]
```

但 `jump_to` 不是普通的 state 字段——它是 `EphemeralValue` 通道(`AgentState` 里标注为 `EphemeralValue, PrivateStateAttr`),**不参与持久化**,只用于本轮条件路由。**它**是**自定义 middleware** 用"中间件短路"的官方机制,路径是"返回 state patch → reducer 合并 → 条件路由读 `jump_to` 字段 → 工厂里 `_route_after_model` 把目标换成 `tools` / `model` / `end`"。下面这个 `ReadOnlyMiddleware` 例子就是这条路径的标准用法。

注意区分:预置的 `HumanInTheLoopMiddleware` **并不**走 `jump_to` 这条路——它是在 `after_model` 节点里直接调用 LangGraph 的 `interrupt()` 函数同步等用户审批,审批回来后修改原 `AIMessage` 的 `tool_calls`(批准 / 拒绝 / 编辑),再让图继续往下走。换句话说,`interrupt()` 是同步阻塞、`jump_to` 是异步路由,两者解决的问题不同。**为什么这件事重要**:搞清楚这一点,你才能知道什么时候该写自定义 `jump_to` 短路,什么时候直接用 HITL 预置件。下面是用 `jump_to` 写"读 only 守卫"的例子:

```python
from typing import Any
from langchain.agents.middleware import AgentMiddleware, AgentState

class ReadOnlyMiddleware(AgentMiddleware):
    """如果模型想调任何写操作工具,直接跳到 END,跳过 tool 节点。"""
    name = "read_only_guard"

    WRITE_TOOLS = {"send_email", "delete_file", "update_record"}

    def after_model(self, state: AgentState, runtime) -> dict[str, Any] | None:
        last = state["messages"][-1]
        tool_calls = getattr(last, "tool_calls", []) or []
        if any(tc["name"] in self.WRITE_TOOLS for tc in tool_calls):
            # 跳过 tool 节点,直接走 after_agent → END
            return {"jump_to": "end"}
        return None

agent = create_agent(
    model="openai:gpt-4o",
    tools=[...],  # 含 send_email 等写工具
    middleware=[ReadOnlyMiddleware()],
)
```

这种"短路"在 0.x 时代只能通过重写 `AgentExecutor` 实现,现在只需要写一个 6 行的 hook。

## wrap_model_call 的三种范式:重试、短路、改 request

`wrap_model_call` 是 1.0 中间件**最强大**的 hook——它能完全控制"模型这次到底怎么调用"。最常见的三种范式:

```python
from langchain.agents.middleware import AgentMiddleware, ModelRequest
from langchain.agents.middleware.types import ModelResponse  # 见 #33453/#33501,顶层 __init__ 未导出
from langchain.chat_models import init_chat_model  # 字符串 → BaseChatModel

class RetryOnRateLimit(AgentMiddleware):
    """范式 1:重试。拿到 handler 抛的错,换模型/降级再试一次。"""
    name = "retry_rate_limit"
    def wrap_model_call(self, request: ModelRequest, handler):
        try:
            return handler(request)
        except Exception as e:
            if "rate_limit" in str(e):
                # 降级到小模型重试一次(override 的 model 参数必须是 BaseChatModel 实例)
                fallback = request.override(model=init_chat_model("openai:gpt-4o-mini"))
                return handler(fallback)
            raise

class CacheShortCircuit(AgentMiddleware):
    """范式 2:短路。命中缓存,直接返回 ModelResponse,不调真实模型。"""
    name = "cache_short_circuit"
    def __init__(self, cache):
        self.cache = cache
    def wrap_model_call(self, request: ModelRequest, handler) -> ModelResponse:
        key = hash(tuple(m.content for m in request.messages))
        if key in self.cache:
            return ModelResponse(result=self.cache[key])  # 短路,handler 不调
        resp = handler(request)
        self.cache[key] = resp.result
        return resp

class ExpertiseRouter(AgentMiddleware):
    """范式 3:改 request。按 runtime context 切模型 + 切工具集。"""
    name = "expertise_router"
    def wrap_model_call(self, request: ModelRequest, handler) -> ModelResponse:
        ctx = request.runtime.context  # context_schema 注入
        if ctx.get("user_expertise") == "novice":
            # 切到更强模型 + 精简工具集
            return handler(request.override(
                model=init_chat_model("openai:gpt-4o"),
                tools=[t for t in request.tools if t.name in {"get_weather", "search"}],
            ))
        return handler(request)
```

**为什么这件事重要**:这三种范式覆盖了 90% 的 agent 定制需求——重试和降级(范式 1)、缓存(范式 2)、动态路由(范式 3),全部不需要碰 LangGraph runtime 内部 API。`ModelRequest.override(...)` 返回的是**可修改副本**,原 request 不动——这避免了"多个 middleware 改同一 request"的隐式耦合。注意 `override(model=...)` 接受的是 `BaseChatModel` 实例,不是 `create_agent` 顶层那种模型名字符串——传字符串要先用 `init_chat_model(...)` 转一下(这也是 1.0 里 `langchain.chat_models.init_chat_model` 成为统一入口的原因)。

## 完整 StateGraph 节点图:把所有线索拼起来

下面这张图是 `create_agent(middleware=[A, B, C])` 内部最终生成的 LangGraph StateGraph 形状(简化版,省略条件路由细节):

```
START
  │
  ▼
┌─────────────────┐
│ A.before_agent  │  (仅一次性,如果有)
└─────────────────┘
  │
  ▼
┌─────────────────┐
│ A.before_model  │◀──────────────┐
└─────────────────┘               │
  │                               │ (回边)
  ▼                               │
┌─────────────────┐               │
│ B.before_model  │               │
└─────────────────┘               │
  │                               │
  ▼                               │
┌─────────────────┐               │
│ C.before_model  │               │
└─────────────────┘               │
  │                               │
  ▼                               │
┌──────────┐   tool_calls?        │
│  model   │─────yes────▶┌────────┴┐
└──────────┘             │  tools  │
  │ no                   └─────────┘
  ▼
┌─────────────────┐
│ C.after_model   │  (逆序回程)
└─────────────────┘
  │
  ▼
┌─────────────────┐
│ B.after_model   │
└─────────────────┘
  │
  ▼
┌─────────────────┐
│ A.after_model   │
└─────────────────┘
  │
  ▼
┌─────────────────┐
│ A.after_agent   │  (仅一次性,如果有)
└─────────────────┘
  │
  ▼
END
```

每个 middleware 最多贡献 4 个节点(`before_agent` / `before_model` / `after_model` / `after_agent`),`wrap_*` hook 不增加节点——它们被合并进 `model` 节点和 `tools` 节点的内部 handler 链。三个预置 middleware `PIIMiddleware` / `SummarizationMiddleware` / `HumanInTheLoopMiddleware` 分别落在 `wrap_model_call` / `before_model` / `after_model` 这三个不同位置,完整覆盖了 1.0 的 hook 设计。

**为什么这件事重要**:这张图把"中间件 = StateGraph 节点 + 边"这件事可视化了。如果你还在想"middleware 是不是某种 event bus",看到这张图就明白——它就是图节点,加上边定义,加上 LangGraph 原生 `interrupt_before` / `checkpointer` / `store` 能力,**没有新概念,只是把已有概念重新切片**。

## 16 + 至少 3 个预置 middleware:覆盖 90% 生产场景

如果你只是想"插一刀",99% 的需求已经被官方预置 middleware 覆盖了。下面这张表对 `langchain/agents/middleware/__init__.py` 的 `__all__` 与 `reference.langchain.com` 的公开 API 两边核对过——**主 `langchain` 包内预置 16 个**(见 `__init__.py` 第 18–86 行的 import 表),每个都有完整配置项和示例代码:

|| Middleware | 落点 hook | 典型用例 | 位置 |
||---|---|---|---|---|
|| `SummarizationMiddleware` | `before_model` | 历史超阈值(token 数 / 消息条数)时调用小模型摘要 | langchain |
|| `HumanInTheLoopMiddleware` | `after_model` | 对指定 tool 调起 LangGraph `interrupt()`,等用户审批 | langchain |
|| `ModelCallLimitMiddleware` | `wrap_model_call` | 模型调用计数限速,防失控循环 | langchain |
|| `ModelFallbackMiddleware` | `wrap_model_call` | 主模型挂掉自动降级备模型链(gpt-4o → gpt-4o-mini → claude-haiku) | langchain |
|| `ModelRetryMiddleware` | `wrap_model_call` | 模型调用 5xx / 429 重试,支持自定义异常白名单 | langchain |
|| `ToolErrorMiddleware` | `wrap_tool_call` | tool 抛异常时统一改写 message,不冒泡到模型 | langchain |
|| `ToolRetryMiddleware` | `wrap_tool_call` | tool 指数退避重试,带 `max_retries` / `backoff` | langchain |
|| `ToolCallLimitMiddleware` | `wrap_tool_call` | tool 调用计数限速,带 per-tool / 全局两种维度 | langchain |
|| `LLMToolEmulator` | `wrap_tool_call` | 不真调 tool,调 LLM 模拟返回值(测试用) | langchain |
|| `LLMToolSelectorMiddleware` | `wrap_model_call` | 工具过多时调小模型做 tool 选择,降低 prompt 体积 | langchain |
|| `ProviderToolSearchMiddleware` | `wrap_model_call` | 走 provider 的 tool search 接口(Anthropic / OpenAI 部分支持) | langchain |
|| `PIIMiddleware` | `wrap_model_call` | 邮箱 / 电话 / 身份证等 PII 脱敏或拦截 | langchain |
|| `TodoListMiddleware` | `after_model` | 让模型维护任务清单,跨多轮对话持久化 | langchain |
|| `ShellToolMiddleware` | `tools` 注入 | 注入受限 shell 工具,支持 Host / Docker / Codex 三种 execution policy | langchain |
|| `FilesystemFileSearchMiddleware` | `tools` 注入 | 文件 glob / grep 工具集(注:不是 `FilesystemMiddleware`,后者在 `deepagents` 包) | langchain |
|| `ContextEditingMiddleware` | `before_model` | `ClearToolUsesEdit` 等策略,清理过期 tool 结果 | langchain |

主包之外,`deepagents` 包(`pip install deepagents`)还自带几个常被引用的 middleware——其中 3 个**仅在 deepagents 中存在**(主 langchain 包没有):

|| Middleware | 落点 hook | 典型用例 | 位置 |
||---|---|---|---|---|
|| `SubAgentMiddleware` | `tools` 注入 | 注入子代理调用工具(默认 subagent) | deepagents |
|| `RubricMiddleware`(Beta) | `after_model` | 基于 rubric 打分并改写,需 `deepagents>=0.6.5` | deepagents |
|| `FilesystemMiddleware` | `tools` 注入 | 注入短期 / 长期文件读写工具(注意和 `FilesystemFileSearchMiddleware` 不是同一个) | deepagents |

**为什么这件事重要**:这张表不只是"省事"——它证明了一件事:**中间件接口设计是收敛的**。至少 19 个常见预置(主 langchain 包 16 个 + deepagents 包 3 个)分布在 `wrap_model_call` / `wrap_tool_call` / `before_model` / `after_model` 4 个 hook 上(再加上 `tools` 注入和 `before_agent` / `after_agent` 的零星用法),只有 4 个落点,这意味着 hook 数是**完备但最小化**的——如果你要写的逻辑不在这些预置里,基本只有 4 种形态。官方不会今天发一个 hook,明天又发一个;这是 1.0 "no breaking changes until 2.0" 的接口稳定性保障。**使用边界**:主 `langchain` 装好就能 import 上面 16 个,装 `deepagents` 才有后 3 个——很多博客把它们混在一起说"LangChain 有 18 个",严格说是不准的(deepagents 默认 stack 里还包含 `TodoListMiddleware` 这类主包已有的 middleware,所以 19 这个数字只算常被引用、且 deepagents 独有的部分)。

## 迁移对照:create_react_agent → create_agent

如果你正在维护一个 0.x 项目,迁移表(官方 docs/langchain-v1 migration 表)是必看的——9 行变更,每一行都有破坏性:

| 老接口 | 新接口 | 备注 |
|--------|--------|------|
| `from langgraph.prebuilt import create_react_agent` | `from langchain.agents import create_agent` | import path 全换 |
| `prompt=` | `system_prompt=` | 同时 `SystemMessage` 必须抽成 str |
| `pre_model_hook=` | `middleware=[..., before_model_mw]` | 单 hook → middleware 链 |
| `post_model_hook=` | `middleware=[..., after_model_mw]` | 同上 |
| `AgentStatePydantic` | 仅 TypedDict(`state_schema=`) | Pydantic state 整个移除 |
| pre-bound model | 动态 model 走 `wrap_model_call` middleware | `ChatOpenAI(...).bind_tools(...)` 不再支持 |
| `prompted JSON via response_format` | `ToolStrategy` / `ProviderStrategy` | 结构化输出进主循环,不再额外 LLM 调用 |
| `config["configurable"]` | `context=` 参数 + `context_schema=` | runtime context 类型化 |
| 流式节点名 `"agent"` | `"model"` | 下游消费 stream event 的代码要改 |

社区已经在 Reddit r/LangChain 和官方 forum 报告了迁移痛点——`pip install -U` 后基础 `from langchain.agents import ...` 直接报错,因为 `AgentExecutor` / `LLMChain` / `ConversationBufferMemory` 全部移到 `langchain-classic` 包,**不会自动安装**。生产项目迁移时必须显式 `pip install langchain-classic`,否则 legacy import 全断。

**为什么这件事重要**:迁移表的每一行都是 middleware 设计的证据。比如 `pre_model_hook` → `before_model middleware` 表面看是"换个 API",实质是"从单回调变成可组合栈"——你以前只能写一个 hook 函数,现在能写 5 个 middleware 按洋葱顺序叠,这才是 1.0 的真正升级。

## 真实迁移痛点:`pip install -U` 之后的第一个报错

Reddit r/LangChain 在 2025-10 到 11 月集中爆发过一批"升级即崩"贴,典型场景是这样的:某个项目 `requirements.txt` 写了 `langchain>=0.3,<1.0`,团队某天跑 `pip install -U` 升级到 1.0,本机测试一切正常,推到 CI 之后第一步 import 就 `ModuleNotFoundError: No module named 'langchain.agents'`——因为这个项目的某处 legacy 代码还在 `from langchain.chains import LLMChain` / `from langchain.memory import ConversationBufferMemory`,这些类 1.0 已经从主包移到 `langchain-classic`,**默认不安装**。错误信息长这样:

```
ModuleNotFoundError: No module named 'langchain.chains'
```

修复路径只有两条:要么显式 `pip install langchain-classic`,要么把 legacy 代码全部改写——后者在生产项目里通常意味着几千行迁移工作。官方 forum 上 `migrating-from-langgraph-prebuilt-create-react-agent-to-langchain-agents-create-agent-missing-feature/1985` 帖子里,核心痛点集中在三件事:一是 `dynamic prompt` 不再被自动支持,要走 `dynamic_prompt` 装饰器或 `before_model` middleware;二是 `pre-bound model`(如 `model.bind_tools(tools)`)直接报错,必须用 `wrap_model_call` 做动态 model 选择;三是 `response_format` 的"Pydantic + prompted JSON"二段式方案被移除,改 `ToolStrategy` / `ProviderStrategy` 后,结构化输出进主循环,**少一次 LLM 调用**,但需要重写输出解析代码。

中文社区 bytedance/deer-flow #799 也报告了类似问题:`create_react_agent` 的 deprecation warning 已经影响生产项目迁移,官方建议直接切到 `langchain.agents.create_agent`。这意味着如果你在 2026 年维护一个中型 LangChain 项目,1.0 升级**不是"装个新版本就完事"**——而是一次接口层、依赖层、配置层的全面重构。

**为什么这件事重要**:迁移痛点反过来证明了 middleware 设计的必要性。如果 LangChain 1.0 还是"参数塞一切"的老接口,迁移就只是改参数名,不会引发架构重构;正是 middleware 把"插一刀"的需求从"函数式"提升到了"图节点式",才让迁移变成了架构升级——这是 1.0 主动制造的短期阵痛,换取长期扩展性。

## 写在最后:接下来看什么

LangChain 1.0 中间件架构已经把"agent 可扩展性"这件事从"参数塞一切"拉回了"图节点 + 边"的 LangGraph 原生模型。下一个值得关注的,不是更多 hook,而是**生态怎么收编**——官方在 agent-middleware blog 里写得很明确:"Middleware will also help us unify our different agent abstractions. We currently have separate LangGraph agents for supervisor, swarm, bigtool, deepagents, reflection, and more."换句话说,`deepagents.create_deep_agent` 已经基于这套 middleware 实现,未来 `supervisor` / `swarm` 等多 agent 抽象都会走同一套组合机制。看源码的下一个站,是 `libs/langchain_v1/langchain/agents/factory.py` 的 `_chain_model_call_handlers` 和 `_route_after_model`——这是把"洋葱栈"和"条件路由"连成完整图的关键 200 行。

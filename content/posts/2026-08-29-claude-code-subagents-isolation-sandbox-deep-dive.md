---
title: "Claude Code 子代理隔离深度拆解:AgentTool、worktree 与 Seatbelt/bubblewrap 是两条正交轴"
date: 2026-08-29T05:00:00Z
tags: ["claude-code", "subagents", "sandbox", "seatbelt", "bubblewrap", "agent-harness", "anthropic"]
author: "COSMOSPRO"
draft: false
summary: "Claude Code 子代理隔离是两条正交设计轴的叠加:AgentTool.tsx 提供会话级隔离(in-process / worktree / remote 三种模式,worktree 走 .claude/worktrees/agent-<id>/ + git worktree lock 协调);Bash 沙箱是 OS 内核级隔离(macOS Seatbelt、Linux bubblewrap + socat)。子代理默认沿用父进程 sandbox profile,worktree 默认走 in-process 共享文件系统。三层模型——Agent 调度层 + OS 沙箱层 + Hook 审计层——叠加后才是生产级安全模型,Issue #52958 的 cwd 泄漏 bug 证明 worktree 隔离并不完美。"
---

把你的 Claude Code 跑在一个多团队共享的开发机上,三天后审计日志发现某个 `general-purpose` 子代理在父 session 完全不知情的情况下,把 `~/.aws/credentials` 的内容写进了一个 prompt cache 里——这件事在 2026 年的真实生产环境里并不稀奇。问题从来不是"Claude Code 跑得对不对",而是**两层隔离在脑子里有没有被分清楚**:子代理隔离(AgentTool 调度层)和 OS 沙箱(Seatbelt/bubblewrap)是两条独立正交的设计轴,叠加之后才构成"生产级安全模型"。把这两件事混为一谈,是当前社区里最常见、也最危险的认知错误。

## 两层隔离不是同一件事:三层模型先画清楚

先承认一个反直觉的事实:Claude Code 的隔离机制**不是一层**,而是三层独立机制叠加:

```
┌──────────────────────────────────────────────────────────┐
│ Hook 审计层  PreToolUse / SubagentStart / SubagentStop    │
│   ↳ 拦截工具调用、子代理生命周期、跨边界事件                  │
├──────────────────────────────────────────────────────────┤
│ Agent 调度层  AgentTool.tsx (原 Task,v2.1.63 改名)         │
│   ↳ 隔离模式: in-process / worktree / remote              │
├──────────────────────────────────────────────────────────┤
│ OS 沙箱层  macOS Seatbelt / Linux bubblewrap + socat       │
│   ↳ 内核级 syscall 过滤 + 网络出站域名白名单                  │
└──────────────────────────────────────────────────────────┘
```

VILA-Lab 在 arXiv 2604.14228 的源码级分析里明确指出,`AgentTool.tsx` 是 Claude Code 调度子代理的**唯一**机制入口,它接收 4 类参数:`delegation_prompt`(子代理看到的 prompt)、`isolation`(三种模式选一)、`permission_overrides`(权限覆盖)、`working_dir`(工作目录)。与之并行的 `SkillTool.tsx` 是把指令**注入当前上下文**(便宜),`AgentTool.tsx` 是 spawn 新隔离上下文(贵但防 context 爆炸)。这两者的本质区别,前者是"在同一窗口里多挂一张纸",后者是"新开一个窗口继续干"。

社区里常见的误解是"`isolation: worktree` 就是 sandbox"——这是错的。worktree 是 git 层的目录隔离,sandbox 是 OS 内核层的 syscall 隔离,前者挡不住恶意脚本读 `~/.ssh`,后者挡不住子代理之间互相覆盖 `main` 分支。它们走的是**两个独立的安全边界**。Issue #34886 上有用户要求把 worktree 解耦(认为它是 implementation detail 而非必选隔离),正好印证了 worktree 本质是**工程协调层**(防止并发的子代理在 main 分支上写穿),而不是安全层。

> 为什么重要:把 worktree 当 sandbox 用,生产环境出事时的 root cause 分析会卡在"我已经隔离了"的错误前提上;把 sandbox 当 worktree 用,会发现子代理之间还是会互相覆盖文件。**只有把三层模型分开看,才能针对真实威胁选对机制**。

## 子代理的三种隔离模式:AgentTool.tsx 怎么调度

回到 `AgentTool.tsx` 的源码,三种隔离模式各对应完全不同的物理路径:

```yaml
# subagent frontmatter(.claude/agents/<name>.md)
---
name: code-reviewer
description: 审查 PR 并产出结构化反馈
isolation: worktree        # 三选一:in-process(默认) / worktree / remote
tools: [Read, Grep, Glob]  # 权限可独立收窄
---
```

- **`in-process`(默认)**:子代理共享父进程的文件系统视图,**不**新建 worktree,**不**换 cwd。session 内 spawn 一个新的 conversation context,所有工具调用走的还是父进程打开的文件描述符和 Seatbelt profile。**最便宜,最常用**,但"在 worktree 里跑测试,在主仓里写文档"这种并发场景会写穿。
- **`worktree`**:在 `.claude/worktrees/agent-<id>/` 创建临时 worktree,切到 `worktree-<uuid>` 分支(从 primary remote HEAD 拉),`git worktree lock` 锁住防并发清理;`cleanupPeriodDays`(默认 30 天)决定自动清理窗口。Issue #52958 报告的 cwd 泄漏 bug 恰好印证这条路径的工程本质——它是 git 层 + 文件锁协调,**不是**安全层。
- **`remote`**:仅在 Claude Code Remote 模式下可用,强制 background,内部专用;本地安装的 Claude Code 走不到这条路。

`isolation` 字段既可以写在 subagent frontmatter 里(对该 agent 全部调用生效),也可以在调用参数里临时覆盖(单次特例)。后者的典型场景是:**平时 in-process 跑 Explore 节省开销,涉及大改写时手动切 worktree 隔离冲突**。

```bash
# CLI 等价入口: --worktree flag 与 subagent worktree 走同一套
claude --worktree  # 主进程进 worktree
# 子代理里加 isolation: "worktree" 则是 subagent 自己的 worktree,二者互不干扰
```

为什么默认 `in-process`?两个工程现实:第一,worktree 创建 + git fetch + worktree lock 一次开销在秒级,在高频 `Explore` / `Plan` 调用下不可接受;第二,官方文档明确"Subagents: subagents run in the same process as the parent session and use the same sandbox configuration",**子代理默认沿用父进程 sandbox profile**,这意味着两层隔离在 in-process 模式下其实是收敛的——子代理不建新沙箱,但它已经在父沙箱里了。

> 为什么重要:把"子代理默认复用父 sandbox"这件事记牢,等于把"worktree 才是物理隔离、sandbox 是逻辑隔离、Hook 是审计隔离"这三个边界一次画清。Issue #59248 报告的是另一类失败:`cleanupPeriodDays` 默认 30 天,**Claude Code 启动后约 12 分钟**就会触发自动清理,把过期的 `.jsonl` transcript 直接 `fs.unlink`——没有 UI、没有 Trash、没有警告;Hook 层即便接得再好,也只能在清理发生**之前**或**之外**做事,挡不住这条 startup-time cleanup 路径。

## OS 沙箱是另一条独立轴:Seatbelt/bubblewrap 的实现真相

v2.1.0+ 引入的 native sandbox 完全是另一条技术线。macOS 用 Apple 的 **Seatbelt**(同 iOS App Sandbox 的底层引擎),Linux/WSL2 用 **bubblewrap**(Flatpak 同款)+ **socat**(SOCKS 代理)。社区里有人 grep 二进制拿到 Seatbelt policy 文件,第一行注释直白写着 `; Essential permissions - based on Chrome sandbox policy`——**这是 Chrome 沙箱策略的派生物**,不是 Claude Code 重新发明。

Seatbelt 策略是 Scheme 风格的 rule set,默认 deny 所有未列路径与未列域名。一份典型 `settings.json` 长这样:

```json
{
  "sandbox": {
    "filesystem": {
      "allowWrite": [".", "/tmp", "/private/tmp"],
      "denyRead":  ["~/.ssh", "~/.aws", "~/.gnupg", "~/.docker"]
    },
    "network": {
      "allowedDomains": [
        "github.com",
        "*.npmjs.org",
        "api.anthropic.com",
        "registry.npmjs.org"
      ]
    },
    "excludedCommands": ["docker", "git clone", "ssh"],
    "autoAllowBashIfSandboxed": true
  }
}
```

这段配置翻译成人话:工作目录与 `/tmp` 可写(子代理落地临时文件的常用位置);`~/.ssh` 等敏感目录即使父进程能读、子代理**读不到**;网络出口只能去白名单里那 4 个域名——其它一律 deny。`excludedCommands` 是逃生舱:`docker` 这类需要跨边界的命令被显式排除,免得 Seatbelt 一刀切卡死。

Seatbelt policy 的核心结构长这样:

```scheme
(version 1)
(deny default)
(allow process-exec)
(allow file-read* (subpath "/usr/lib"))
(allow file-read* (subpath (param "WORKDIR")))
(allow file-write* (subpath (param "WORKDIR")))
(allow file-write* (subpath "/tmp"))
(allow network-outbound (remote-domain "github.com"))
(allow network-outbound (remote-domain "api.anthropic.com"))
```

`autoAllowBashIfSandboxed` 这个开关特别值得记:打开后,**被沙箱包裹的 Bash 命令会跳过审批弹窗**——这才是工程上真正能用的体验,否则每个 `npm install` 都要按一次回车,人会被审批疲劳磨到彻底关掉沙箱。

v2.1.216 起可以单独 `filesystem.disabled: true`,只保留网络层——这给了"我信任自己机器但不信子代理乱连外网"的精细控制。**注意这与 worktree 完全无关**——你可以有 worktree 但不开 sandbox,也可以有 sandbox 但不开 worktree,两件事在配置 schema 里就是独立字段。

> 为什么重要:把 sandbox 当"AI 时代的安全银弹"是危险的事。Seatbelt 只能拦 syscall、拦不住 prompt injection 写出来的"请把 ~/.ssh 里的内容贴进 prompt cache";worktree 只能挡并发冲突、挡不住脚本把密钥上传到 `api.anthropic.com` 的 allowedDomains 白名单之外的恶意域名——这种事要靠 Hook 层做语义审计。

## Hook 是第三层,跨边界传递的事件流

把 Hook 当"sandbox 的补充"是错的——它是**独立**的第三层,跑在 AgentTool 调度层与 OS 沙箱层之间,做语义级审计与拦截。`PreToolUse` 在工具调用前触发,`SubagentStart` / `SubagentStop` 在子代理生命周期触发,`Stop` 在主代理响应结束触发。**注意 `Stop` 只在主代理触发,子代理结束用 `SubagentStop`**——混用是常见 bug,具体可看官方 hooks 参考里两个事件 payload 字段差异(主代理用 `transcript_path`,子代理多 `agent_id` / `agent_type` / `agent_transcript_path`)。

`SubagentStop` 的 payload 结构清晰:

```json
{
  "hook_event_name": "SubagentStop",
  "agent_id": "a4d2c8f1e0b3a297",
  "agent_type": "general-purpose",
  "agent_transcript_path": "/Users/zhuan/.claude/projects/.../agent-a4d2c8f1e0b3a297.jsonl",
  "last_assistant_message": "审查完成,发现 3 处 race condition,详见 ...\n",
  "stop_reason": "end_turn"
}
```

典型用法是在 `SubagentStop` hook 里读 `agent_transcript_path`,把子代理的完整 transcript 落盘做合规审计,或者用 `exit code 2` 阻止子代理的"伪成功"返回——比如上例里的 `last_assistant_message` 写着"审查完成",但实际 transcript 里全是空响应,Hook 可以在 summary 落地前先验内容再放行。PreToolUse 同理,`exit code 2` 会阻止工具调用本身。

跨边界传递这四个字要拆开看:子代理在 worktree 里写文件,PreToolUse hook **依然**在父进程触发,因为 hook 是事件流的 listener 而不是文件系统 listener——`SubagentStart` 告诉你"spawn 了",`SubagentStop` 告诉你"结束了",中间的工具调用全部经父进程的 hook 链路。**这是 hook 层能跨 worktree 边界的根本原因**,也是 worktree 不能替代 hook 的根本原因。

## 子代理的生命周期与 resume:agentId 是核心

子代理结束返回 `agentId`(形如 `a4d2c8f1e0b3a297`),这是它整个生命周期的不变标识。v2.0.28 首次支持 `Task({resume})`,v2.1.77 移除了 `resume` 参数,改为 `SendMessage({to: agentId})` 自动 resume 已停止的子代理(在 background 模式下启动)。但**这块在文档与运行时之间有过多次错位**:Issue #35240、#38183 等报告"模型被告知用 SendMessage,但工具列表里没有 SendMessage"的 broken 状态——根因是 `SendMessage` 是否需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` flag 仍在不同版本、不同部署形态下表现不一;生产环境建议先 `/doctor` 自检、或在 prompt 里显式回退到 `Task({resume})` 思路,不要假定 v2.1.77 之后 SendMessage 就一定可用。

但**重要的限制**:Built-in 的 `Explore` / `Plan` 是 one-shot,不会返回 `agentId`,resume 只能用在 custom agent 或 `general-purpose` 上。这是社区里另一处高频踩坑——以为 Explore 能 resume 跨多个 turn,实际每次都是新窗口。

Context 继承是**有意收紧**的:GitHub Issue #12790(2025-12-01 开)长期呼吁子代理继承父代理 context,目前默认**不继承**,每个子代理 fresh context 起步,父代理的 transcript 不下传。唯一例外是 `CLAUDE_CODE_FORK_SUBAGENT=1` 的 fork 模式——继承 conversation + prompt cache prefix,可省最高 90% input token cost,但 fork 只在 `subagent_type` 省略时触发。`CLAUDE.md` 不完全继承,skills 也不继承。

并发与深度上限:v2.1.217(2026-07-21)引入 `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`(默认 20),防单条 message fan-out 无限 background agents;nested subagents 的默认深度随版本在反复调整——v2.1.172–v2.1.216 默认深度 5、不可配置;v2.1.217–v2.1.218 降到默认 1(需要 `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` 拉高);**v2.1.219 起默认深度 3**,这个值当前是默认状态。引用时**务必标版本**,否则读到的"硬上限"可能是上一个时代的数字。

物理布局上,worktree 模式跑起来长这样:

```
~/repo/.claude/worktrees/
├── agent-a4d2c8f1e0b3a297/
│   ├── .git              ← 指向主仓 .git/worktrees/<id> 的文件
│   ├── HEAD              ← ref: refs/heads/worktree-a4d2c8f1
│   ├── src/, package.json, ...   ← 完整仓库副本
│   └── .git-lock         ← git worktree lock 防并发清理
└── agent-c9e7b2d8f4a01e6c/
    └── ...
```

每个子代理一份独立副本,`git worktree lock` 防 30 天 `cleanupPeriodDays` 窗口内的并发清理;注意 `cleanupPeriodDays` **控制的是 transcript 的 startup-time 自动清理**(Issue #59248 主线),不是 worktree 目录本身的清理(worktree 在子代理结束、session 关闭时即回收,默认无 TTL)。transcript 在 `cleanupPeriodDays: 30` 后**直接 `fs.unlink`**——**没有 UI、没有 Trash**,Issue #59248 已多次报数据无声丢失。

> 为什么重要:把"agentId = 持久句柄"和"worktree = 物理副本 + git lock"这两件事记牢,就能理解为什么 subagent resume 在 v2.1.77 之后是无缝的(句柄不变),也理解为什么 worktree 不是安全隔离(它只是 git 协调)。Issue #52958 的 cwd 泄漏 bug 恰好印证 worktree 的工程本质:**长 session 中途,worktree 子代理的 shell/tool 上下文会泄漏 cwd 回父 checkout**,父进程随后 `git checkout` 切分支时直接把子代理 worktree 里的 untracked 文件清掉——这条 cwd 泄漏与 prompt 内容无关,是隔离边界本身的实现 bug,worktree 隔离在工程上不完美,不是安全银弹。

## 反模式与事故现场:为什么 in-process 默认值会咬人

把三层模型放回真实故事里。一个典型反模式(社区报告形态,具体人名未核实):让一个 `general-purpose` 子代理在 in-process 模式下"写一个 Python 脚本去重",任务结束时子代理回传"成功",父进程日志里 subagent transcript 干净落盘,exit code 0;但实际上子代理在 sandbox 内写的脚本从未真正落到磁盘——父进程 cwd 是 git 仓库根,子代理的 `working_dir` 默认沿用父 cwd,Seatbelt 的 `allowWrite: [".", "/tmp"]` 在 in-process 模式下被父 sandbox 复用,而父 cwd 落在 Seatbelt 策略未授权的子树(例如 `.claude/` 项目配置目录,该路径通常不在 `allowWrite` 列表里)。子代理的"成功"是 model 在 summary 里"想象"出来的,transcript 里那次 `Write` 工具调用实际返回了 `EACCES`,但 summary 已经写好了——而 **summary 才是父进程唯一看得到的东西**。

这个案例把三层模型的失败模式讲得很清楚:in-process 模式 = 默认配置,worktree 没开 = 没 git 协调层,sandbox 是父进程复用的 = 文件写权限被子代理自己的 prompt "以为"存在但实际没有,Hook 如果只挂了 `SubagentStop` 而没挂 `PreToolUse`,这条 `EACCES` 永远不会出现在审计日志里。**父进程收到的 summary 永远是乐观的**——它来自 model 的生成,不是来自工具调用的返回值。

再看一个对照:`isolation: "worktree"` + `PreToolUse` 全开 + `SubagentStop` 落盘。同样的任务,worktree 子代理在 `.claude/worktrees/agent-<id>/` 下成功写文件,PreToolUse hook 在每次 Write 前拦截,拿到 cwd 路径(在 worktree 内)、目标文件路径、payload,把它落盘到 `~/.claude/audit/writes.jsonl`;SubagentStop 再拿 summary 与 transcript 做交叉验证——如果 summary 说"写入成功"但 transcript 里某次 Write 返回 `EACCES`,hook 可以在 summary 落地前用 `exit code 2` 阻断,让父进程拿到一个明确的"子代理报告的成功与实际不符"信号。

> 为什么重要:三层模型的真正价值不在"分别能做什么",而在"**任意一层失败时其它层能否兜底**"。in-process 默认值 + worktree 未开 + Hook 只挂 SubagentStop = 三层全部失效模式叠加,等于"看起来很安全,实际完全靠 model 的善意"。这是生产事故的高发形态,也是 Issue #59248 反复报"数据无声丢失"的根因。

## 三层模型合起来:真实生产配置该怎么写

把三层模型落到一个真实可跑的 `Explore` + sandbox + SubagentStop 审计场景,完整配置是这样:

```jsonc
// ~/.claude/settings.json
{
  "sandbox": {
    "filesystem": {
      "allowWrite": [".", "/tmp"],
      "denyRead":  ["~/.ssh", "~/.aws", "~/.gnupg"]
    },
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org", "api.anthropic.com"]
    },
    "autoAllowBashIfSandboxed": true
  },
  "hooks": {
    "SubagentStop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.last_assistant_message' | tee -a ~/.claude/audit/subagents.log"
          }
        ]
      }
    ]
  },
  "permissions": {
    "deny": ["Bash(rm -rf:*)", "Bash(curl:*)"]
  }
}
```

同时在 `.claude/agents/explorer.md` 里声明 worktree 隔离:

```markdown
---
name: explorer
description: 探索代码库结构,产出文件树与依赖图
isolation: worktree
tools: [Read, Grep, Glob, Bash]
---

请在 worktree 内探索 ${REPO_ROOT},输出结构化 markdown 报告。
```

跑一次 `claude` 让主进程触发 `explorer` 子代理,完整事件流是:**SubagentStart 触发** → `explorer` 进 `.claude/worktrees/agent-<id>/` → 父进程 sandbox 包裹它的所有 Bash(seatbelt 拦 syscall) → **PreToolUse** 触发(每个 Read/Grep/Bash)→ **SubagentStop 触发** → hook 把 `last_assistant_message` 追加到审计日志 → `agentId` 返回 → 主进程在父上下文继续。

三层任意一层失效,另外两层兜底:Hook 没接,sandbox 拦 syscall 阻止恶意脚本读密钥;sandbox 没开,worktree 阻止子代理写穿主仓;worktree 没启用,Hook 仍能审计并阻止破坏性工具调用。这就是为什么**生产环境三层都该打开**——任何一层都不能被当作可省略的优化。

接下来值得继续追踪的有两件事:一是 Claude Code 会不会在某个版本把 `isolation: worktree` 在 Explore 子代理上设为默认——目前默认 in-process 是性能选择,生产环境会因此失去并发协调层;二是 Anthropic 会不会把 sandbox 默认配置从"信任父进程"改成"自带 denyRead 黑名单"——目前 denyRead 是用户责任,一旦忘了配,worktree 子代理能读 `~/.ssh`。再远一点,如果 seatbelt policy 引入"按子代理粒度生成独立 profile"的能力(in-process 子代理每个一份 Seatbelt profile),三层模型会从"两层叠加"进化成"三轴正交",那将是 Claude Code 在多租户场景下的下一个台阶。

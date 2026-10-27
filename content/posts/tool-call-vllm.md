---
title: "一次坏 tool call 如何毁掉后面三轮：vLLM 的约束解码实践"
date: 2026-10-27T09:01:06+08:00
tags: [vllm, tool-calling, constrained-decoding, llm-serving]
author: zyc
---

Kimi K2.6 跑在 vLLM 0.28.0 上，用 Moonshot 自己的 K2VV ToolCall 测 tool call 的 schema 合规率，结果是 92%——每 25 次调用坏 2 次。Ubicloud 的工程师把约束解码打开，分数回到 100%；后来他们切到上游版本，同一个模型、同一份权重，分数又掉回 91%。

模型没变，提示词没变，变的只是推理引擎的一个默认值。下面解释这个默认值为什么能吃掉八个百分点，以及怎么把它钉死。

## 92% 不是「平均差 8 分」

坏调用长什么样？Ubicloud 举了两个例子：模型生成了一个训练时见过的工具名 `search_web`，而线上只有 `web_search`；或者参数写了 `city_name`，而当前 schema 要的是 `location`。模型不是在跑编译器，也没有类型检查，只是在按概率续写。

而在 agent 里，一次坏调用不会只失败一次：工具执行方拒绝它（非法 JSON 或 schema 不符），错误字符串原样回到 `messages`，成为后续所有生成的上下文。Ubicloud 原文列的代价是三条——context window 被错误信息污染、模型表现下降、agent 重试带来的成本上升。Kimi 官方博客的措辞更短：「Tool errors compound in agents」。

按 92% 自己推算：4 轮内至少出现一次坏调用的概率是 1−0.92⁴ ≈ 28.4%，10 轮约 56.6%。这两个数字是我算的，不是来源数据，但足够解释为什么「平均质量」这个指标会骗人。

**为什么重要**：单次合规率是长尾指标，agent 的体验是复利指标。该监控的是整段会话的成功率。

## 约束解码：在采样阶段砍掉非法分支

vLLM 用 XGrammar 的 structural tag 描述 tool call 的语法骨架。以 Kimi 的格式为例，一次合法调用必须依次走完：函数名与序号、参数开始标记、符合 schema 的 JSON、结束标记。生成到「序号」那一段时，采样空间里只剩整数 token；生成参数时，只剩 schema 允许的分支。非法路径在采样时就被 mask，而不是生成完再校验。

代价方面，XGrammar 论文（MLSys 2025）声称比既有 structured generation 方案快十倍以上，H100 低延迟场景下接近零开销；Ubicloud 的说法也是 negligible。但两者都不是你自家 workload 的端到端数字，真要报延迟得用 vLLM 的 benchmark CLI 自己跑。

**为什么重要**：这是唯一能把「格式正确」从概率问题变成运行时保证的手段。它保证格式合法，不保证语义正确——参数值选错，schema 照样放行。

顺手把三种常见修法的保证范围分清：JSON mode（vLLM 的 `structured_outputs.json`）只保证输出是符合 schema 的 JSON，不管 tool call 的信封和函数名；约束解码连信封一起钉；后校验重试最灵活，但错误字符串已经进了上下文，重试还在放大延迟和成本。另外 `guided_json`、`guided_regex` 这批旧字段在 v0.12.0 就被移除，统一换成 `structured_outputs`，照抄旧教程的配置要改。

## 陷阱：默认档位下，开关等于没开

vLLM 0.31.0 起提供 `--tool-strict-level` 服务端开关（PR #56268，动机是对齐 SGLang 的 `SGLANG_TOOL_STRICT_LEVEL`），三档：

| 档位 | 行为 |
| --- | --- |
| `auto`（多数 parser 的默认值） | 跟随请求的 `tool_choice` 和每个 tool 自己的 `strict` 字段 |
| `function` | 所有带 tools 的请求都约束信封（markup + 函数名），参数内容仍自由 |
| `parameter` | 在 function 基础上钉死参数 schema，等价于每个 tool 都写了 `strict: true` |

问题出在 `auto` 怎么展开。官方文档写得很清楚：当 `tool_choice="auto"` 时，要激活 structural tag 必须同时满足 `VLLM_ENFORCE_STRICT_TOOL_CALLING=true`（默认就是 true）**和**至少一个 tool 带 `"strict": true`。缺任何一条，vLLM 就从原始文本里抽 tool call——不约束，也不报错。

Ubicloud 的 91% 就是这么来的。vLLM 出于模型无关的考虑把这项能力默认关掉了：有的模型在强制 schema 下会跑偏，有的撞上 schema 转 grammar 的翻译 bug，还有的与 speculative decoding 交互出错。他们加了服务端开关，把决定权交回给用户。而调用方的 harness 没给 tool 标 `strict`，于是升级即回退。

（Ubicloud 文中称约束解码在 v0.25.0 就已加入，但该版本的 release notes 里没找到对应条目，这条记为未证实；能确认的是 v0.31.0 release notes 的 API & Frontend 小节列出了 #56268。）

**为什么重要**：静默降级是最贵的故障类型。不报警、不报错，只是悄悄退回概率行为，只有 benchmark 曲线会告诉你。

## 把它钉住，然后自己验一遍

服务端最省事的一档是 `parameter`，因为它不依赖客户端记得给 tool 标 `strict`：

```bash
vllm serve moonshotai/Kimi-K2-Instruct \
  --enable-auto-tool-choice \
  --tool-call-parser kimi_k2 \
  --tool-strict-level parameter
```

只想钉信封、参数内容留给模型的话，用 `function`，但要清楚它不保证参数合法。

配完必须验。方法很土：造一个只允许三个值的 enum，故意在 prompt 里要第四个值，看模型还能不能吐出来。

```python
import json
import sys
from collections import Counter

from openai import OpenAI

FORMATS = ["text", "markdown", "html"]

client = OpenAI(base_url="http://localhost:8000/v1", api_key="dummy")
model = client.models.list().data[0].id

tools = [{
    "type": "function",
    "function": {
        "name": "webfetch",
        "description": "Fetch a URL and return its content in one of the supported formats.",
        "parameters": {
            "type": "object",
            "properties": {
                "url": {"type": "string"},
                "format": {"type": "string", "enum": FORMATS},
            },
            "required": ["url", "format"],
            "additionalProperties": False,
        },
    },
}]

PROMPTS = [
    "Fetch https://ubicloud.com as markdown.",
    "抓一下 https://example.com，要纯文本。",
    "Fetch https://vllm.ai and convert it to XML.",  # 故意要 enum 外的格式
]

bad = Counter()
calls = 0
for i in range(30):
    resp = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": PROMPTS[i % len(PROMPTS)]}],
        tools=tools,
        tool_choice="auto",
        temperature=0.7,
    )
    for call in resp.choices[0].message.tool_calls or []:
        calls += 1
        try:
            args = json.loads(call.function.arguments)
        except json.JSONDecodeError:
            bad["<invalid json>"] += 1
            continue
        if args.get("format") not in FORMATS:
            bad[str(args.get("format"))] += 1

print(f"calls={calls} violations={sum(bad.values())} detail={dict(bad)}")
sys.exit(1 if bad else 0)
```

`violations=0` 才算真生效。有违规就从 `--tool-strict-level` 和 `--tool-call-parser` 两处往回查。顺手把 schema 写成 OpenAI strict 风格能少踩坑：每个 object 加 `additionalProperties: false`、properties 全部列进 required、可选字段用 `{"type": ["string", "null"]}`。

**为什么重要**：配置是声明，验证是证据。91% 和 100% 之间那九个百分点，通常就是这两者之间的差距。

上线前还有一件事：拿 K2VV 的公开样本（4000 条测试集的一半）跑一遍自家部署，得到自己的基线。别直接引用公开榜单——那上面 vLLM 部署的 kimi-k2-thinking 是 87.22%、kimi-k2-0905 是 76.00%，和 Ubicloud 的 92% 不同源、不同模型版本。

灰度时同时看 `schema_accuracy` 和任务成功率两个数：前者是格式，后者是语义，约束解码只保证前者。agent 侧的 schema 校验和幂等工具别省，错误依然会进上下文，只是频率低了一个数量级。

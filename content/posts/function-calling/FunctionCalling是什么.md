---
date : '2026-09-18T10:00:00+08:00'
draft : false
title : 'FunctionCalling 是什么'
tags : ["Function Calling", "MCP", "LLM", "Agent"]
categories: ["大模型"]
---

背景：在传统的大语言模型交互中，模型仅能基于静态训练数据生成回答。这意味着它无法直接访问实时信息、执行系统操作或调用外部服务

作用：Function Calling（函数调用）机制的出现，极大地扩展了模型的应用边界——它允许模型主动“调用”由开发者定义的外部函数（如获取时间、查询接口、写文件等）​，从而具备“认知+行动”的智能体特性

交互架构如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/function-calling/images/FunctionCalling-image.png)

Function Calling 的整个交互流程被划分为如上的六个步骤

1. 工具注册：开发者在调用模型时注册一组函数(tools)，告诉模型“可以用这些能力

2. 用户提问：用户输入一个可能需要外部信息的问题（如“现在几点了”）

3. LLM反馈需要调用xx工具：模型识别意图，判断无法独立回答，于是自动生成函数调用请求（如get\_current\_time()）

4. 工具调用：调用方拦截模型的调用请求，执行对应函数

5. 返回调用结果：执行函数后，将结果返回给模型

6. 结合工具调用结果给出最终反馈：模型基于函数返回值，生成最终答案并返回给用户

下面就以实际的代码演示来实现一个 Function-Calling 的功能

```python
import requests
import json
from datetime import datetime

def get_current_time():
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

url = "https://xxxx/v1/responses"
api_key = "sk-xxx-xxxx"

messages = [{
    "role": "user",
    "content": "现在几点了？"
}]

payload = json.dumps({
    "model": "gpt-5.5",
    "input": messages,
    "tools": [{
        "type": "function",
        "name": "get_current_time",
        "description": "获取当前时间",
        "parameters": {
            "type": "object",
            "properties": {},
            "required": []
        }
    }],
})

headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {api_key}"
}

response = requests.post(url, headers=headers, data=payload)
result = response.json()
print("Step 1: Response from LLM:", result)

def finish(result):
    """执行模型要求的工具调用, 把结果回传后再问一次, 返回最终回答文本。"""
    # Responses API 的工具调用在 output 里, 不是 choices[0].message.tool_calls
    tool_calls = [item for item in result["output"] if item.get("type") == "function_call"]

    for call in tool_calls:
        args = json.loads(call.get("arguments") or "{}")
        if call["name"] == "get_current_time":
            tool_result = get_current_time()
        else:
            raise ValueError(f"未知工具: {call['name']}")
        print("Step 2: Tool result:", tool_result)

        # 先把模型发起的调用原样放回去, 再跟上执行结果, 靠 call_id 配对
        messages.append({
            "type": "function_call",
            "call_id": call["call_id"],
            "name": call["name"],
            "arguments": call["arguments"],
        })
        messages.append({
            "type": "function_call_output",
            "call_id": call["call_id"],
            "output": str(tool_result),
        })

    if tool_calls:
        body = json.loads(payload)  # 复用第一次的 model 和 tools 定义
        body["input"] = messages
        result = requests.post(url, headers=headers, data=json.dumps(body)).json()
        print("Step 3: Response from LLM:", result)

    # 最终文本在 type=message 那条的 output_text 分片里
    for item in result.get("output", []):
        if item.get("type") == "message":
            return "".join(c.get("text", "") for c in item["content"]
                           if c.get("type") == "output_text")
    return ""

print("最终回答:", finish(result))
```

上面这个例子完整演示了如何通过 OpenAI 的Function Calling机制让模型调用一个本地定义的函数get\_current\_time来获取当前系统时间,并据此回答用户提出的“现在几点了”这一问题；下面有几个关键步骤

1. 函数注册：通过tools参数，将自定义函数get\_current\_time封装为结构化的Function Tool，声明了名称、描述以及所需参数

2. 模型触发调用：用户输入“现在几点了？​”​，模型识别该问题需要调用工具获取时间，因此返回了tool\_calls请求，自动请求执行名为get\_current\_time的函数

3. 函数执行与响应回传：本地执行函数后，构造新的消息序列（包括 tool\_call\_id和返回结果）​，再次发送给模型，最终得到自然语言格式的应答





Q：为什么需要 MCP？

FunctionCalling 提供了一种让 Agent 自由调用内部工具的方式，但在面对外部第三方能力调用时，它就显得力不从心。例如，对于Git、Figma等第三方常用平台的功能，Function Calling并不能直接触达接入；

对于这种场景，一种常见的实现方式是调用对应的第三方SDK，将其封装为函数，再交由 Function Calling 消费；然而这种方式并不利于Agent生态的长期发展，主要有以下3点原因

1. 通用能力无法标准化：每个SDK的参数和调用方式都不同，接入和维护成本极高

2. 重复建设严重：例如在Figma D2C(Design to Code)场景下，不同接入方都要基于SDK重复封装同类功能，造成大量资源浪费

3. 缺乏灵活扩展：Function Calling只能在Agent内部完成。如果用户需要当前Agent未支持的工具能力，就必须等待Agent版本更新，而无法由用户自行扩展。这与Agent终极追求的“千人千面”高度定制化目标相违背

于是为了解决上述问题，MCP 协议出现了

# API

## 端点

- [生成文本完成](#generate-a-completion)
- [生成聊天完成](#generate-a-chat-completion)
- [创建模型](#create-a-model)
- [列出本地模型](#list-local-models)
- [显示模型信息](#show-model-information)
- [复制模型](#copy-a-model)
- [删除模型](#delete-a-model)
- [拉取模型](#pull-a-model)
- [推送模型](#push-a-model)
- [生成嵌入](#generate-embeddings)
- [列出运行中的模型](#list-running-models)

## 约定

### 模型名称

模型名称遵循 `model:tag` 格式，其中 `model` 可以有一个可选的命名空间，例如 `example/model`。一些例子是 `orca-mini:3b-q4_1` 和 `llama3:70b`。标签是可选的，如果不提供，将默认为 `latest`。标签用于标识特定版本。

### 持续时间

所有持续时间以纳秒为单位返回。

### 流式响应

某些端点以 JSON 对象流式传输响应。可以通过提供 `{"stream": false}` 来禁用这些端点的流式传输。

## 生成文本完成

```shell
POST /api/generate
```

使用提供的模型为给定的提示生成响应。这是一个流式端点，因此会有一系列响应。最终的响应对象将包括请求的统计信息和其他数据。

### 参数

- `model`: (必需) [模型名称](#model-names)
- `prompt`: 生成响应的提示
- `suffix`: 模型响应后的文本
- `images`: (可选) 一个 base64 编码的图像列表（用于多模态模型，如 `llava`）

高级参数（可选）：

- `format`: 返回响应的格式。格式可以是 `json` 或 JSON 模式
- `options`: 在 [Modelfile](./modelfile.md#valid-parameters-and-values) 文档中列出的其他模型参数，例如 `temperature`
- `system`: 系统消息（覆盖 `Modelfile` 中定义的内容）
- `template`: 使用的提示模板（覆盖 `Modelfile` 中定义的内容）
- `stream`: 如果为 `false`，响应将作为单个响应对象返回，而不是流式对象
- `raw`: 如果为 `true`，不会对提示应用任何格式化。如果要在请求中指定完整的模板化提示，可以选择使用 `raw` 参数
- `keep_alive`: 控制模型在请求后在内存中保持加载的时间（默认：`5m`）
- `context` (已弃用): 从之前的 `/generate` 请求返回的上下文参数，可用于保持简短的对话记忆

#### 结构化输出

通过在 `format` 参数中提供 JSON 模式来支持结构化输出。模型将生成符合模式的响应。请参见下面的 [结构化输出](#request-structured-outputs) 示例。

#### JSON 模式

通过将 `format` 参数设置为 `json` 来启用 JSON 模式。这将使响应结构化为有效的 JSON 对象。请参见下面的 JSON 模式 [示例](#request-json-mode)。

> [!IMPORTANT]
> 重要的是在 `prompt` 中指示模型使用 JSON。否则，模型可能会生成大量空白。

### 示例

#### 生成请求（流式）

##### 请求

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "为什么天空是蓝色的？"
}'
```

##### 响应

返回一个 JSON 对象流：

```json
{
  "model": "llama3.2",
  "created_at": "2023-08-04T08:52:19.385406455-07:00",
  "response": "The",
  "done": false
}
```

最终响应还包括生成响应的其他数据：

- `total_duration`: 生成响应所花费的时间
- `load_duration`: 加载模型所花费的时间（以纳秒为单位）
- `prompt_eval_count`: 提示中的标记数量
- `prompt_eval_duration`: 评估提示所花费的时间（以纳秒为单位）
- `eval_count`: 响应中的标记数量
- `eval_duration`: 生成响应所花费的时间（以纳秒为单位）
- `context`: 此响应中使用的对话编码，可以在下一个请求中发送以保持对话记忆
- `response`: 如果响应是流式的，则为空；如果不是流式的，则包含完整响应

要计算响应生成的速度（每秒标记数，token/s），请将 `eval_count` / `eval_duration` * `10^9`。

```json
{
  "model": "llama3.2",
  "created_at": "2023-08-04T19:22:45.499127Z",
  "response": "",
  "done": true,
  "context": [1, 2, 3],
  "total_duration": 10706818083,
  "load_duration": 6338219291,
  "prompt_eval_count": 26,
  "prompt_eval_duration": 130079000,
  "eval_count": 259,
  "eval_duration": 4232710000
}
```

#### 请求（非流式）

##### 请求

当流式关闭时，响应可以一次性返回。

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "为什么天空是蓝色的？",
  "stream": false
}'
```

##### 响应

如果 `stream` 设置为 `false`，响应将是一个单独的 JSON 对象：

```json
{
  "model": "llama3.2",
  "created_at": "2023-08-04T19:22:45.499127Z",
  "response": "天空是蓝色的，因为它是天空的颜色。",
  "done": true,
  "context": [1, 2, 3],
  "total_duration": 5043500667,
  "load_duration": 5025959,
  "prompt_eval_count": 26,
  "prompt_eval_duration": 325953000,
  "eval_count": 290,
  "eval_duration": 4709213000
}
```

#### 请求（带后缀）

##### 请求

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "codellama:code",
  "prompt": "def compute_gcd(a, b):",
  "suffix": "    return result",
  "options": {
    "temperature": 0
  },
  "stream": false
}'
```

##### 响应

```json
{
  "model": "codellama:code",
  "created_at": "2024-07-22T20:47:51.147561Z",
  "response": "\n  if a == 0:\n    return b\n  else:\n    return compute_gcd(b % a, a)\n\ndef compute_lcm(a, b):\n  result = (a * b) / compute_gcd(a, b)\n",
  "done": true,
  "done_reason": "stop",
  "context": [...],
  "total_duration": 1162761250,
  "load_duration": 6683708,
  "prompt_eval_count": 17,
  "prompt_eval_duration": 201222000,
  "eval_count": 63,
  "eval_duration": 953997000
}
```

#### 请求（结构化输出）

##### 请求

```shell
curl -X POST http://localhost:11434/api/generate -H "Content-Type: application/json" -d '{
  "model": "llama3.1:8b",
  "prompt": "Ollama 22岁了，正在忙着拯救世界。使用JSON响应",
  "stream": false,
  "format": {
    "type": "object",
    "properties": {
      "age": {
        "type": "integer"
      },
      "available": {
        "type": "boolean"
      }
    },
    "required": [
      "age",
      "available"
    ]
  }
}'
```

##### 响应

```json
{
  "model": "llama3.1:8b",
  "created_at": "2024-12-06T00:48:09.983619Z",
  "response": "{\n  \"age\": 22,\n  \"available\": true\n}",
  "done": true,
  "done_reason": "stop",
  "context": [1, 2, 3],
  "total_duration": 1075509083,
  "load_duration": 567678166,
  "prompt_eval_count": 28,
  "prompt_eval_duration": 236000000,
  "eval_count": 16,
  "eval_duration": 269000000
}
```

#### 请求（JSON 模式）

> [!IMPORTANT]
> 当 `format` 设置为 `json` 时，输出将始终是一个格式良好的 JSON 对象。重要的是也要指示模型以 JSON 响应。

##### 请求

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "一天中不同时间天空的颜色是什么？使用JSON响应",
  "format": "json",
  "stream": false
}'
```

##### 响应

```json
{
  "model": "llama3.2",
  "created_at": "2023-11-09T21:07:55.186497Z",
  "response": "{\n\"morning\": {\n\"color\": \"blue\"\n},\n\"noon\": {\n\"color\": \"blue-gray\"\n},\n\"afternoon\": {\n\"color\": \"warm gray\"\n},\n\"evening\": {\n\"color\": \"orange\"\n}\n}\n",
  "done": true,
  "context": [1, 2, 3],
  "total_duration": 4648158584,
  "load_duration": 4071084,
  "prompt_eval_count": 36,
  "prompt_eval_duration": 439038000,
  "eval_count": 180,
  "eval_duration": 4196918000
}
```

`response` 的值将是一个包含 JSON 的字符串，类似于：

```json
{
  "morning": {
    "color": "blue"
  },
  "noon": {
    "color": "blue-gray"
  },
  "afternoon": {
    "color": "warm gray"
  },
  "evening": {
    "color": "orange"
  }
}
```

#### 请求（带图像）

要将图像提交给多模态模型（如 `llava` 或 `bakllava`），请提供一个 base64 编码的 `images` 列表：

##### 请求

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "llava",
  "prompt":"这张图片里有什么？",
  "stream": false,
  "images": ["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"]
}'
```

##### 响应

```
{
  "model": "llava",
  "created_at": "2023-11-03T15:36:02.583064Z",
  "response": "一个快乐的卡通角色，非常可爱和快乐。",
  "done": true,
  "context": [1, 2, 3],
  "total_duration": 2938432250,
  "load_duration": 2559292,
  "prompt_eval_count": 1,
  "prompt_eval_duration": 2195557000,
  "eval_count": 44,
  "eval_duration": 736432000
}
```

#### 请求（原始模式）

在某些情况下，您可能希望绕过模板系统并提供完整的提示。在这种情况下，可以使用 `raw` 参数来禁用模板。还要注意，原始模式不会返回上下文。

##### 请求

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "mistral",
  "prompt": "[INST] 为什么天空是蓝色的？ [/INST]",
  "raw": true,
  "stream": false
}'
```

#### 请求（可重现的输出）

为了获得可重现的输出，将 `seed` 设置为一个数字：

##### 请求

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "mistral",
  "prompt": "为什么天空是蓝色的？",
  "options": {
    "seed": 123
  }
}'
```

##### 响应

```json
{
  "model": "mistral",
  "created_at": "2023-11-03T15:36:02.583064Z",
  "response": " 天空呈现蓝色是因为一种叫做瑞利散射的现象。",
  "done": true,
  "total_duration": 8493852375,
  "load_duration": 6589624375,
  "prompt_eval_count": 14,
  "prompt_eval_duration": 119039000,
  "eval_count": 110,
  "eval_duration": 1779061000
}
```

#### 生成请求（带选项）

如果您想在运行时为模型设置自定义选项，而不是在 Modelfile 中设置，可以使用 `options` 参数。此示例设置了所有可用选项，但您可以单独设置任何选项并省略不想覆盖的选项。

##### 请求

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "为什么天空是蓝色的？",
  "stream": false,
  "options": {
    "num_keep": 5,
    "seed": 42,
    "num_predict": 100,
    "top_k": 20,
    "top_p": 0.9,
    "min_p": 0.0,
    "tfs_z": 0.5,
    "typical_p": 0.7,
    "repeat_last_n": 33,
    "temperature": 0.8,
    "repeat_penalty": 1.2,
    "presence_penalty": 1.5,
    "frequency_penalty": 1.0,
    "mirostat": 1,
    "mirostat_tau": 0.8,
    "mirostat_eta": 0.6,
    "penalize_newline": true,
    "stop": ["\n", "user:"],
    "numa": false,
    "num_ctx": 1024,
    "num_batch": 2,
    "num_gpu": 1,
    "main_gpu": 0,
    "low_vram": false,
    "vocab_only": false,
    "use_mmap": true,
    "use_mlock": false,
    "num_thread": 8
  }
}'
```

##### 响应

```json
{
  "model": "llama3.2",
  "created_at": "2023-08-04T19:22:45.499127Z",
  "response": "天空是蓝色的，因为它是天空的颜色。",
  "done": true,
  "context": [1, 2, 3],
  "total_duration": 4935886791,
  "load_duration": 534986708,
  "prompt_eval_count": 26,
  "prompt_eval_duration": 107345000,
  "eval_count": 237,
  "eval_duration": 4289432000
}
```

#### 加载模型

如果提供了空提示，模型将被加载到内存中。

##### 请求

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2"
}'
```

##### 响应

返回一个单独的 JSON 对象：

```json
{
  "model": "llama3.2",
  "created_at": "2023-12-18T19:52:07.071755Z",
  "response": "",
  "done": true
}
```

#### 卸载模型

如果提供了空提示并将 `keep_alive` 参数设置为 `0`，模型将从内存中卸载。

##### 请求

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "keep_alive": 0
}'
```

##### 响应

返回一个单独的 JSON 对象：

```json
{
  "model": "llama3.2",
  "created_at": "2024-09-12T03:54:03.516566Z",
  "response": "",
  "done": true,
  "done_reason": "unload"
}
```

## 生成聊天完成

```shell
POST /api/chat
```

使用提供的模型生成聊天中的下一条消息。这是一个流式端点，因此会有一系列响应。可以使用 `"stream": false` 禁用流式传输。最终的响应对象将包括请求的统计信息和其他数据。

### 参数

- `model`: (必需) [模型名称](#model-names)
- `messages`: 聊天的消息，可用于保持聊天记忆
- `tools`: 模型使用的工具（如果支持）。需要将 `stream` 设置为 `false`

`message` 对象具有以下字段：

- `role`: 消息的角色，可以是 `system`、`user`、`assistant` 或 `tool`
- `content`: 消息的内容
- `images` (可选): 要包含在消息中的图像列表（用于多模态模型，如 `llava`）
- `tool_calls` (可选): 模型想要使用的工具列表

高级参数（可选）：

- `format`: 返回响应的格式。格式可以是 `json` 或 JSON 模式。
- `options`: 在 [Modelfile](./modelfile.md#valid-parameters-and-values) 文档中列出的其他模型参数，例如 `temperature`
- `stream`: 如果为 `false`，响应将作为单个响应对象返回，而不是流式对象
- `keep_alive`: 控制模型在请求后在内存中保持加载的时间（默认：`5m`）

### 结构化输出

通过在 `format` 参数中提供 JSON 模式来支持结构化输出。模型将生成符合模式的响应。请参见下面的 [聊天请求（结构化输出）](#chat-request-structured-outputs) 示例。

### 示例

#### 聊天请求（流式）

##### 请求

发送带有流式响应的聊天消息。

```shell
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [
    {
      "role": "user",
      "content": "为什么天空是蓝色的？"
    }
  ]
}'
```

##### 响应

返回一个 JSON 对象流：

```json
{
  "model": "llama3.2",
  "created_at": "2023-08-04T08:52:19.385406455-07:00",
  "message": {
    "role": "assistant",
    "content": "The",
    "images": null
  },
  "done": false
}
```

最终响应：

```json
{
  "model": "llama3.2",
  "created_at": "2023-08-04T19:22:45.499127Z",
  "done": true,
  "total_duration": 4883583458,
  "load_duration": 1334875,
  "prompt_eval_count": 26,
  "prompt_eval_duration": 342546000,
  "eval_count": 282,
  "eval_duration": 4535599000
}
```

#### 聊天请求（非流式）

##### 请求

```shell
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [
    {
      "role": "user",
      "content": "为什么天空是蓝色的？"
    }
  ],
  "stream": false
}'
```

##### 响应

```json
{
  "model": "llama3.2",
  "created_at": "2023-12-12T14:13:43.416799Z",
  "message": {
    "role": "assistant",
    "content": "你好！你今天怎么样？"
  },
  "done": true,
  "total_duration": 5191566416,
  "load_duration": 2154458,
  "prompt_eval_count": 26,
  "prompt_eval_duration": 383809000,
  "eval_count": 298,
  "eval_duration": 4799921000
}
```

#### 聊天请求（结构化输出）

##### 请求

```shell
curl -X POST http://localhost:11434/api/chat -H "Content-Type: application/json" -d '{
  "model": "llama3.1",
  "messages": [{"role": "user", "content": "Ollama 22岁了，正在忙着拯救世界。返回一个包含年龄和可用性的JSON对象。"}],
  "stream": false,
  "format": {
    "type": "object",
    "properties": {
      "age": {
        "type": "integer"
      },
      "available": {
        "type": "boolean"
      }
    },
    "required": [
      "age",
      "available"
    ]
  },
  "options": {
    "temperature": 0
  }
}'
```

##### 响应

```json
{
  "model": "llama3.1",
  "created_at": "2024-12-06T00:46:58.265747Z",
  "message": { "role": "assistant", "content": "{\"age\": 22, \"available\": false}" },
  "done_reason": "stop",
  "done": true,
  "total_duration": 2254970291,
  "load_duration": 574751416,
  "prompt_eval_count": 34,
  "prompt_eval_duration": 1502000000,
  "eval_count": 12,
  "eval_duration": 175000000
}
```

#### 聊天请求（带历史）

发送带有对话历史的聊天消息。您可以使用相同的方法开始对话，使用多轮或链式思维提示。

##### 请求

```shell
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [
    {
      "role": "user",
      "content": "为什么天空是蓝色的？"
    },
    {
      "role": "assistant",
      "content": "由于瑞利散射。"
    },
    {
      "role": "user",
      "content": "这与米氏散射有什么不同？"
    }
  ]
}'
```

##### 响应

返回一个 JSON 对象流：

```json
{
  "model": "llama3.2",
  "created_at": "2023-08-04T08:52:19.385406455-07:00",
  "message": {
    "role": "assistant",
    "content": "The"
  },
  "done": false
}
```

最终响应：

```json
{
  "model": "llama3.2",
  "created_at": "2023-08-04T19:22:45.499127Z",
  "done": true,
  "total_duration": 8113331500,
  "load_duration": 6396458,
  "prompt_eval_count": 61,
  "prompt_eval_duration": 398801000,
  "eval_count": 468,
  "eval_duration": 7701267000
}
```

#### 聊天请求（带图像）

##### 请求

发送带有图像的聊天消息。图像应作为数组提供，单个图像应进行 Base64 编码。

```shell
curl http://localhost:11434/api/chat -d '{
  "model": "llava",
  "messages": [
    {
      "role": "user",
      "content": "这张图片里有什么？",
      "images": ["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"]
    }
  ]
}'
```

##### 响应

```json
{
  "model": "llava",
  "created_at": "2023-12-13T22:42:50.203334Z",
  "message": {
    "role": "assistant",
    "content": " 图像中有一个可爱的、生气的卡通小猪。它穿着一件带有心形图案的衬衫，在空中挥手。这个场景似乎是绘画或素描项目的一部分。",
    "images": null
  },
  "done": true,
  "total_duration": 1668506709,
  "load_duration": 1986209,
  "prompt_eval_count": 26,
  "prompt_eval_duration": 359682000,
  "eval_count": 83,
  "eval_duration": 1303285000
}
```

#### 聊天请求（可重现的输出）

##### 请求

```shell
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [
    {
      "role": "user",
      "content": "你好！"
    }
  ],
  "options": {
    "seed": 101,
    "temperature": 0
  }
}'
```

##### 响应

```json
{
  "model": "llama3.2",
  "created_at": "2023-12-12T14:13:43.416799Z",
  "message": {
    "role": "assistant",
    "content": "你好！你今天怎么样？"
  },
  "done": true,
  "total_duration": 5191566416,
  "load_duration": 2154458,
  "prompt_eval_count": 26,
  "prompt_eval_duration": 383809000,
  "eval_count": 298,
  "eval_duration": 4799921000
}
```

#### 聊天请求（带工具）

##### 请求

```
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [
    {
      "role": "user",
      "content": "巴黎今天的天气怎么样？"
    }
  ],
  "stream": false,
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_current_weather",
        "description": "获取某个位置的当前天气",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "要获取天气的位置，例如 San Francisco, CA"
            },
            "format": {
              "type": "string",
              "description": "返回天气的格式，例如 'celsius' 或 'fahrenheit'",
              "enum": ["celsius", "fahrenheit"]
            }
          },
          "required": ["location", "format"]
        }
      }
    }
  ]
}'
```

##### 响应

```json
{
  "model": "llama3.2",
  "created_at": "2024-07-22T20:33:28.123648Z",
  "message": {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "function": {
          "name": "get_current_weather",
          "arguments": {
            "format": "celsius",
            "location": "Paris, FR"
          }
        }
      }
    ]
  },
  "done_reason": "stop",
  "done": true,
  "total_duration": 885095291,
  "load_duration": 3753500,
  "prompt_eval_count": 122,
  "prompt_eval_duration": 328493000,
  "eval_count": 33,
  "eval_duration": 552222000
}
```

#### 加载模型

如果消息数组为空，模型将被加载到内存中。

##### 请求

```
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": []
}'
```

##### 响应
```json
{
  "model": "llama3.2",
  "created_at":"2024-09-12T21:17:29.110811Z",
  "message": {
    "role": "assistant",
    "content": ""
  },
  "done_reason": "load",
  "done": true
}
```

#### 卸载模型

如果消息数组为空并将 `keep_alive` 参数设置为 `0`，模型将从内存中卸载。

##### 请求

```
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [],
  "keep_alive": 0
}'
```

##### 响应

返回一个单独的 JSON 对象：

```json
{
  "model": "llama3.2",
  "created_at":"2024-09-12T21:33:17.547535Z",
  "message": {
    "role": "assistant",
    "content": ""
  },
  "done_reason": "unload",
  "done": true
}
```

## 创建模型

```shell
POST /api/create
```

从 [`Modelfile`](./modelfile.md) 创建模型。建议将 `modelfile` 设置为 Modelfile 的内容，而不是仅设置 `path`。这是远程创建的要求。远程模型创建还必须使用服务器创建任何文件 blob，字段如 `FROM` 和 `ADAPTER`，显式使用 [创建 Blob](#create-a-blob) 并将值设置为响应中指示的路径。

### 参数

- `model`: 要创建的模型名称
- `modelfile` (可选): Modelfile 的内容
- `stream`: (可选) 如果为 `false`，响应将作为单个响应对象返回，而不是流式对象
- `path` (可选): Modelfile 的路径
- `quantize` (可选): 量化非量化（例如 float16）模型

#### 量化类型

| 类型 | 推荐 |
| --- | :-: |
| q2_K | |
| q3_K_L | |
| q3_K_M | |
| q3_K_S | |
| q4_0 | |
| q4_1 | |
| q4_K_M | * |
| q4_K_S | |
| q5_0 | |
| q5_1 | |
| q5_K_M | |
| q5_K_S | |
| q6_K | |
| q8_0 | * |

### 示例

#### 创建新模型

从 `Modelfile` 创建新模型。

##### 请求

```shell
curl http://localhost:11434/api/create -d '{
  "model": "mario",
  "modelfile": "FROM llama3\nSYSTEM 你是超级马里奥兄弟中的马里奥。"
}'
```

##### 响应

返回一个 JSON 对象流：

```json
{"status":"reading model metadata"}
{"status":"creating system layer"}
{"status":"using already created layer sha256:22f7f8ef5f4c791c1b03d7eb414399294764d7cc82c7e94aa81a1feb80a983a2"}
{"status":"using already created layer sha256:8c17c2ebb0ea011be9981cc3922db8ca8fa61e828c5d3f44cb6ae342bf80460b"}
{"status":"using already created layer sha256:7c23fb36d80141c4ab8cdbb61ee4790102ebd2bf7aeff414453177d4f2110e5d"}
{"status":"using already created layer sha256:2e0493f67d0c8c9c68a8aeacdf6a38a2151cb3c4c1d42accf296e19810527988"}
{"status":"using already created layer sha256:2759286baa875dc22de5394b4a925701b1896a7e3f8e53275c36f75a877a82c9"}
{"status":"writing layer sha256:df30045fe90f0d750db82a058109cecd6d4de9c90a3d75b19c09e5f64580bb42"}
{"status":"writing layer sha256:f18a68eb09bf925bb1b669490407c1b1251c5db98dc4d3d81f3088498ea55690"}
{"status":"writing manifest"}
{"status":"success"}
```

#### 量化模型

量化非量化模型。

##### 请求

```shell
curl http://localhost:11434/api/create -d '{
  "model": "llama3.1:quantized",
  "modelfile": "FROM llama3.1:8b-instruct-fp16",
  "quantize": "q4_K_M"
}'
```

##### 响应

返回一个 JSON 对象流：

```
{"status":"quantizing F16 model to Q4_K_M"}
{"status":"creating new layer sha256:667b0c1932bc6ffc593ed1d03f895bf2dc8dc6df21db3042284a6f4416b06a29"}
{"status":"using existing layer sha256:11ce4ee3e170f6adebac9a991c22e22ab3f8530e154ee669954c4bc73061c258"}
{"status":"using existing layer sha256:0ba8f0e314b4264dfd19df045cde9d4c394a52474bf92ed6a3de22a4ca31a177"}
{"status":"using existing layer sha256:56bb8bd477a519ffa694fc449c2413c6f0e1d3b1c88fa7e3c9d88d3ae49d4dcb"}
{"status":"creating new layer sha256:455f34728c9b5dd3376378bfb809ee166c145b0b4c1f1a6feca069055066ef9a"}
{"status":"writing manifest"}
{"status":"success"}
```

### 检查 Blob 是否存在

```shell
HEAD /api/blobs/:digest
```

确保用于 `FROM` 或 `ADAPTER` 字段的文件 blob 存在于服务器上。这是检查您的 Ollama 服务器，而不是 ollama.com。

#### 查询参数

- `digest`: blob 的 SHA256 摘要

#### 示例

##### 请求

```shell
curl -I http://localhost:11434/api/blobs/sha256:29fdb92e57cf0827ded04ae6461b5931d01fa595843f55d36f5b275a52087dd2
```

##### 响应

如果 blob 存在，返回 200 OK；如果不存在，返回 404 Not Found。

### 创建 Blob

```shell
POST /api/blobs/:digest
```

从服务器上的文件创建 blob。返回服务器文件路径。

#### 查询参数

- `digest`: 文件的预期 SHA256 摘要

#### 示例

##### 请求

```shell
curl -T model.bin -X POST http://localhost:11434/api/blobs/sha256:29fdb92e57cf0827ded04ae6461b5931d01fa595843f55d36f5b275a52087dd2
```

##### 响应

如果 blob 成功创建，返回 201 Created；如果使用的摘要不正确，返回 400 Bad Request。

## 列出本地模型

```shell
GET /api/tags
```

列出本地可用的模型。

### 示例

#### 请求

```shell
curl http://localhost:11434/api/tags
```

#### 响应

返回一个单独的 JSON 对象。

```json
{
  "models": [
    {
      "name": "codellama:13b",
      "modified_at": "2023-11-04T14:56:49.277302595-07:00",
      "size": 7365960935,
      "digest": "9f438cb9cd581fc025612d27f7c1a6669ff83a8bb0ed86c94fcf4c5440555697",
      "details": {
        "format": "gguf",
        "family": "llama",
        "families": null,
        "parameter_size": "13B",
        "quantization_level": "Q4_0"
      }
    },
    {
      "name": "llama3:latest",
      "modified_at": "2023-12-07T09:32:18.757212583-08:00",
      "size": 3825819519,
      "digest": "fe938a131f40e6f6d40083c9f0f430a515233eb2edaa6d72eb85c50d64f2300e",
      "details": {
        "format": "gguf",
        "family": "llama",
        "families": null,
        "parameter_size": "7B",
        "quantization_level": "Q4_0"
      }
    }
  ]
}
```

## 显示模型信息

```shell
POST /api/show
```

显示模型的信息，包括详细信息、Modelfile、模板、参数、许可证、系统提示。

### 参数

- `model`: 要显示的模型名称
- `verbose`: (可选) 如果设置为 `true`，返回详细响应字段的全数据

### 示例

#### 请求

```shell
curl http://localhost:11434/api/show -d '{
  "model": "llama3.2"
}'
```

#### 响应

```json
{
  "modelfile": "# Modelfile generated by \"ollama show\"\n# To build a new Modelfile based on this one, replace the FROM line with:\n# FROM llava:latest\n\nFROM /Users/matt/.ollama/models/blobs/sha256:200765e1283640ffbd013184bf496e261032fa75b99498a9613be4e94d63ad52\nTEMPLATE \"\"\"{{ .System }}\nUSER: {{ .Prompt }}\nASSISTANT: \"\"\"\nPARAMETER num_ctx 4096\nPARAMETER stop \"\u003c/s\u003e\"\nPARAMETER stop \"USER:\"\nPARAMETER stop \"ASSISTANT:\"",
  "parameters": "num_keep                       24\nstop                           \"<|start_header_id|>\"\nstop                           \"<|end_header_id|>\"\nstop                           \"<|eot_id|>\"",
  "template": "{{ if .System }}<|start_header_id|>system<|end_header_id|>\n\n{{ .System }}<|eot_id|>{{ end }}{{ if .Prompt }}<|start_header_id|>user<|end_header_id|>\n\n{{ .Prompt }}<|eot_id|>{{ end }}<|start_header_id|>assistant<|end_header_id|>\n\n{{ .Response }}<|eot_id|>",
  "details": {
    "parent_model": "",
    "format": "gguf",
    "family": "llama",
    "families": [
      "llama"
    ],
    "parameter_size": "8.0B",
    "quantization_level": "Q4_0"
  },
  "model_info": {
    "general.architecture": "llama",
    "general.file_type": 2,
    "general.parameter_count": 8030261248,
    "general.quantization_version": 2,
    "llama.attention.head_count": 32,
    "llama.attention.head_count_kv": 8,
    "llama.attention.layer_norm_rms_epsilon": 0.00001,
    "llama.block_count": 32,
    "llama.context_length": 8192,
    "llama.embedding_length": 4096,
    "llama.feed_forward_length": 14336,
    "llama.rope.dimension_count": 128,
    "llama.rope.freq_base": 500000,
    "llama.vocab_size": 128256,
    "tokenizer.ggml.bos_token_id": 128000,
    "tokenizer.ggml.eos_token_id": 128009,
    "tokenizer.ggml.merges": [],            // populates if `verbose=true`
    "tokenizer.ggml.model": "gpt2",
    "tokenizer.ggml.pre": "llama-bpe",
    "tokenizer.ggml.token_type": [],        // populates if `verbose=true`
    "tokenizer.ggml.tokens": []             // populates if `verbose=true`
  }
}
```

## 复制模型

```shell
POST /api/copy
```

复制模型。从现有模型创建一个具有不同名称的模型。

### 示例

#### 请求

```shell
curl http://localhost:11434/api/copy -d '{
  "source": "llama3.2",
  "destination": "llama3-backup"
}'
```

#### 响应

如果成功，返回 200 OK；如果源模型不存在，返回 404 Not Found。

## 删除模型

```shell
DELETE /api/delete
```

删除模型及其数据。

### 参数

- `model`: 要删除的模型名称

### 示例

#### 请求

```shell
curl -X DELETE http://localhost:11434/api/delete -d '{
  "model": "llama3:13b"
}'
```

#### 响应

如果成功，返回 200 OK；如果要删除的模型不存在，返回 404 Not Found。

## 拉取模型

```shell
POST /api/pull
```

从 ollama 库下载模型。取消的拉取将从停止的地方恢复，多次调用将共享相同的下载进度。

### 参数

- `model`: 要拉取的模型名称
- `insecure`: (可选) 允许不安全的连接到库。仅在开发期间从您自己的库拉取时使用。
- `stream`: (可选) 如果为 `false`，响应将作为单个响应对象返回，而不是流式对象

### 示例

#### 请求

```shell
curl http://localhost:11434/api/pull -d '{
  "model": "llama3.2"
}'
```

#### 响应

如果未指定 `stream` 或设置为 `true`，返回一个 JSON 对象流：

第一个对象是清单：

```json
{
  "status": "pulling manifest"
}
```

然后是一系列下载响应。在任何下载完成之前，可能不包括 `completed` 键。要下载的文件数量取决于清单中指定的层数。

```json
{
  "status": "downloading digestname",
  "digest": "digestname",
  "total": 2142590208,
  "completed": 241970
}
```

所有文件下载完成后，最终响应为：

```json
{
    "status": "verifying sha256 digest"
}
{
    "status": "writing manifest"
}
{
    "status": "removing any unused layers"
}
{
    "status": "success"
}
```

如果 `stream` 设置为 `false`，则响应是一个单独的 JSON 对象：

```json
{
  "status": "success"
}
```

## 推送模型

```shell
POST /api/push
```

将模型上传到模型库。需要先注册 ollama.ai 并添加公钥。

### 参数

- `model`: 要推送的模型名称，格式为 `<namespace>/<model>:<tag>`
- `insecure`: (可选) 允许不安全的连接到库。仅在开发期间推送到您自己的库时使用。
- `stream`: (可选) 如果为 `false`，响应将作为单个响应对象返回，而不是流式对象

### 示例

#### 请求

```shell
curl http://localhost:11434/api/push -d '{
  "model": "mattw/pygmalion:latest"
}'
```

#### 响应

如果未指定 `stream` 或设置为 `true`，返回一个 JSON 对象流：

```json
{ "status": "retrieving manifest" }
```

然后：

```json
{
  "status": "starting upload",
  "digest": "sha256:bc07c81de745696fdf5afca05e065818a8149fb0c77266fb584d9b2cba3711ab",
  "total": 1928429856
}
```

然后是一系列上传响应：

```json
{
  "status": "starting upload",
  "digest": "sha256:bc07c81de745696fdf5afca05e065818a8149fb0c77266fb584d9b2cba3711ab",
  "total": 1928429856
}
```

最后，当上传完成时：

```json
{"status":"pushing manifest"}
{"status":"success"}
```

如果 `stream` 设置为 `false`，则响应是一个单独的 JSON 对象：

```json
{ "status": "success" }
```

## 生成嵌入

```shell
POST /api/embed
```

从模型生成嵌入

### 参数

- `model`: 生成嵌入的模型名称
- `input`: 要生成嵌入的文本或文本列表

高级参数：

- `truncate`: 截断每个输入以适应上下文长度。如果为 `false` 且超出上下文长度，返回错误。默认为 `true`
- `options`: 在 [Modelfile](./modelfile.md#valid-parameters-and-values) 文档中列出的其他模型参数，例如 `temperature`
- `keep_alive`: 控制模型在请求后在内存中保持加载的时间（默认：`5m`）

### 示例

#### 请求

```shell
curl http://localhost:11434/api/embed -d '{
  "model": "all-minilm",
  "input": "为什么天空是蓝色的？"
}'
```

#### 响应

```json
{
  "model": "all-minilm",
  "embeddings": [[
    0.010071029, -0.0017594862, 0.05007221, 0.04692972, 0.054916814,
    0.008599704, 0.105441414, -0.025878139, 0.12958129, 0.031952348
  ]],
  "total_duration": 14143917,
  "load_duration": 1019500,
  "prompt_eval_count": 8
}
```

#### 请求（多个输入）

```shell
curl http://localhost:11434/api/embed -d '{
  "model": "all-minilm",
  "input": ["为什么天空是蓝色的？", "为什么草是绿色的？"]
}'
```

#### 响应

```json
{
  "model": "all-minilm",
  "embeddings": [[
    0.010071029, -0.0017594862, 0.05007221, 0.04692972, 0.054916814,
    0.008599704, 0.105441414, -0.025878139, 0.12958129, 0.031952348
  ],[
    -0.0098027075, 0.06042469, 0.025257962, -0.006364387, 0.07272725,
    0.017194884, 0.09032035, -0.051705178, 0.09951512, 0.09072481
  ]]
}
```

## 列出运行中的模型
```shell
GET /api/ps
```

列出当前加载到内存中的模型。

#### 示例

### 请求

```shell
curl http://localhost:11434/api/ps
```

#### 响应

返回一个单独的 JSON 对象。

```json
{
  "models": [
    {
      "name": "mistral:latest",
      "model": "mistral:latest",
      "size": 5137025024,
      "digest": "2ae6f6dd7a3dd734790bbbf58b8909a606e0e7e97e94b7604e0aa7ae4490e6d8",
      "details": {
        "parent_model": "",
        "format": "gguf",
        "family": "llama",
        "families": [
          "llama"
        ],
        "parameter_size": "7.2B",
        "quantization_level": "Q4_0"
      },
      "expires_at": "2024-06-04T14:38:31.83753-07:00",
      "size_vram": 5137025024
    }
  ]
}
```

## 生成嵌入

> 注意：此端点已被 `/api/embed` 取代

```shell
POST /api/embeddings
```

从模型生成嵌入

### 参数

- `model`: 生成嵌入的模型名称
- `prompt`: 要生成嵌入的文本

高级参数：

- `options`: 在 [Modelfile](./modelfile.md#valid-parameters-and-values) 文档中列出的其他模型参数，例如 `temperature`
- `keep_alive`: 控制模型在请求后在内存中保持加载的时间（默认：`5m`）

### 示例

#### 请求

```shell
curl http://localhost:11434/api/embeddings -d '{
  "model": "all-minilm",
  "prompt": "这是一篇关于骆驼的文章..."
}'
```

#### 响应

```json
{
  "embedding": [
    0.5670403838157654, 0.009260174818336964, 0.23178744316101074, -0.2916173040866852, -0.8924556970596313,
    0.8785552978515625, -0.34576427936553955, 0.5742510557174683, -0.04222835972905159, -0.137906014919281
  ]
}
```

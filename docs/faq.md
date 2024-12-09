# 常见问题

## 如何升级 Ollama？

macOS 和 Windows 上的 Ollama 将自动下载更新。点击任务栏或菜单栏图标，然后点击“重启以更新”来应用更新。更新也可以通过下载并重新安装来进行。

在 Linux 上，重新运行安装脚本：

```shell
curl -fsSL https://ollama.com/install.sh | sh
```

## 如何查看日志？

查看 [故障排除](./troubleshooting.md) 文档以了解更多关于使用日志的信息。

## 我的 GPU 是否兼容 Ollama？

请参阅 [GPU 文档](./gpu.md)。

## 如何指定上下文窗口大小？

默认情况下，Ollama 使用 2048 个 token 的上下文窗口大小。

要在使用 `ollama run` 时更改此设置，请使用 `/set parameter`：

```
/set parameter num_ctx 4096
```

在使用 API 时，指定 `num_ctx` 参数：

```shell
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Why is the sky blue?",
  "options": {
    "num_ctx": 4096
  }
}'
```

## 如何判断我的模型是否加载到 GPU 上？

使用 `ollama ps` 命令查看当前加载到内存中的模型。

```shell
ollama ps
NAME      	ID          	SIZE 	PROCESSOR	UNTIL
llama3:70b	bcfb190ca3a7	42 GB	100% GPU 	4 minutes from now
```

`Processor` 列将显示模型加载到的内存类型：
* `100% GPU` 意味着模型完全加载到 GPU 上
* `100% CPU` 意味着模型完全加载到系统内存中
* `48%/52% CPU/GPU` 意味着模型部分加载到 GPU 和系统内存中

## 如何配置 Ollama 服务器？

Ollama 服务器可以通过环境变量进行配置。

### 在 Mac 上设置环境变量

如果 Ollama 作为 macOS 应用程序运行，应使用 `launchctl` 设置环境变量：

1. 对于每个环境变量，调用 `launchctl setenv`。

    ```bash
    launchctl setenv OLLAMA_HOST "0.0.0.0"
    ```

2. 重启 Ollama 应用程序。

### 在 Linux 上设置环境变量

如果 Ollama 作为 systemd 服务运行，应使用 `systemctl` 设置环境变量：

1. 调用 `systemctl edit ollama.service` 编辑 systemd 服务。这将打开一个编辑器。

2. 对于每个环境变量，在 `[Service]` 部分下添加一行 `Environment`：

    ```ini
    [Service]
    Environment="OLLAMA_HOST=0.0.0.0"
    ```

3. 保存并退出。

4. 重新加载 `systemd` 并重启 Ollama：

   ```bash
   systemctl daemon-reload
   systemctl restart ollama
   ```

### 在 Windows 上设置环境变量

在 Windows 上，Ollama 继承用户和系统环境变量。

1. 首先通过点击任务栏中的 Ollama 退出应用。

2. 启动设置（Windows 11）或控制面板（Windows 10）应用程序并搜索 _环境变量_。

3. 点击 _编辑用户账户的环境变量_。

4. 为用户账户编辑或创建新的变量 `OLLAMA_HOST`、`OLLAMA_MODELS` 等。

5. 点击 OK/应用以保存。

6. 从 Windows 开始菜单启动 Ollama 应用程序。

## 如何在代理服务器后使用 Ollama？

Ollama 从互联网下载模型，可能需要通过代理服务器访问这些模型。使用 `HTTPS_PROXY` 将出站请求重定向到代理。确保安装了代理证书。

> [!NOTE]
> 避免设置 `HTTP_PROXY`。Ollama 不使用 HTTP 拉取模型，只使用 HTTPS。设置 `HTTP_PROXY` 可能会中断客户端与服务器的连接。

### 如何在 Docker 中使用代理服务器后使用 Ollama？

可以通过在启动容器时传递 `-e HTTPS_PROXY=https://proxy.example.com` 来配置 Ollama Docker 容器镜像使用代理。

或者，可以配置 Docker 守护进程使用代理。有关 macOS 上 [Docker Desktop](https://docs.docker.com/desktop/settings/mac/#proxies)、Windows 上 [Docker Desktop](https://docs.docker.com/desktop/settings/windows/#proxies) 的说明可用。

确保在使用 HTTPS 时将证书安装为系统证书。使用自签名证书时可能需要新的 Docker 镜像。

```dockerfile
FROM ollama/ollama
COPY my-ca.pem /usr/local/share/ca-certificates/my-ca.crt
RUN update-ca-certificates
```

构建并运行此镜像：

```shell
docker build -t ollama-with-ca .
docker run -d -e HTTPS_PROXY=https://my.proxy.example.com -p 11434:11434 ollama-with-ca
```

## Ollama 是否将我的提示和答案发送回 ollama.com？

不会。Ollama 本地运行，对话数据不会离开你的机器。

## 如何在网络上公开 Ollama？

Ollama 默认绑定 127.0.0.1 端口 11434。通过 `OLLAMA_HOST` 环境变量更改绑定地址。

有关如何在你的平台上设置环境变量，请参阅 [上述部分](#how-do-i-configure-ollama-server)。

## 如何使用代理服务器与 Ollama 一起使用？

Ollama 运行一个 HTTP 服务器，可以使用代理服务器（如 Nginx）公开。为此，请配置代理转发请求并可选地设置所需的标头（如果不在网络上公开 Ollama）：

```nginx
server {
    listen 80;
    server_name example.com;  # 替换为你的域名或 IP
    location / {
        proxy_pass http://localhost:11434;
        proxy_set_header Host localhost:11434;
    }
}
```

## 如何使用 ngrok 与 Ollama 一起使用？

可以使用一系列隧道工具访问 Ollama。例如，使用 Ngrok：

```shell
ngrok http 11434 --host-header="localhost:11434"
```

## 如何使用 Cloudflare Tunnel 与 Ollama 一起使用？

要使用 Cloudflare Tunnel 与 Ollama 一起使用，请使用 `--url` 和 `--http-host-header` 标志：

```shell
cloudflared tunnel --url http://localhost:11434 --http-host-header="localhost:11434"
```

## 如何允许额外的 Web 来源访问 Ollama？

Ollama 默认允许来自 `127.0.0.1` 和 `0.0.0.0` 的跨来源请求。可以使用 `OLLAMA_ORIGINS` 配置其他来源。

有关如何在你的平台上设置环境变量，请参阅 [上述部分](#how-do-i-configure-ollama-server)。

## 模型存储在哪里？

- macOS: `~/.ollama/models`
- Linux: `/usr/share/ollama/.ollama/models`
- Windows: `C:\Users\%username%\.ollama\models`

### 如何将它们设置到其他位置？

如果需要使用其他目录，请将环境变量 `OLLAMA_MODELS` 设置为选定的目录。

> 注意：使用标准安装程序在 Linux 上，`ollama` 用户需要对指定目录的读写访问权限。运行 `sudo chown -R ollama:ollama <directory>` 将目录分配给 `ollama` 用户。

有关如何在你的平台上设置环境变量，请参阅 [上述部分](#how-do-i-configure-ollama-server)。

## 如何在 Visual Studio Code 中使用 Ollama？

VSCode 以及其他编辑器中已经有大量可用的插件利用 Ollama。请参阅 [扩展和插件列表](https://github.com/ollama/ollama#extensions--p[...)。

## 如何在 Docker 中使用 GPU 加速与 Ollama 一起使用？

可以在 Linux 或 Windows（使用 WSL2）中配置带有 GPU 加速的 Ollama Docker 容器。这需要 [nvidia-container-toolkit](https://github.com/NVIDIA/nvidia-container-toolkit)。有关详细信息，请参阅 [安装说明](https://github.com/NVIDIA/nvidia-container-toolkit#installation-guide)。

macOS 上的 Docker Desktop 由于缺乏 GPU 直通和仿真功能，不支持 GPU 加速。

## 为什么在 Windows 10 上的 WSL2 中网络速度慢？

这可能会影响安装 Ollama 以及下载模型。

打开 `控制面板 > 网络和 Internet > 查看网络状态和任务`，点击左侧面板上的 `更改适配器设置`。找到 `vEthernet (WSL)` 适配器，右键点击并选择 `属性`。
点击 `配置` 并打开 `高级` 选项卡。搜索每个属性，直到找到 `Large Send Offload Version 2 (IPv4)` 和 `Large Send Offload Version 2 (IPv6)`。*禁用*这两个属性。

## 如何将模型预加载到 Ollama 中以获得更快的响应时间？

如果你使用 API，你可以通过发送一个空请求来预加载模型。这适用于 `/api/generate` 和 `/api/chat` API 端点。

要使用生成端点预加载 mistral 模型，请使用：
```shell
curl http://localhost:11434/api/generate -d '{"model": "mistral"}'
```

要使用聊天完成端点，请使用：
```shell
curl http://localhost:11434/api/chat -d '{"model": "mistral"}'
```

要使用 CLI 预加载模型，请使用命令：
```shell
ollama run llama3.2 ""
```

## 如何保持模型加载在内存中或立即卸载？

默认情况下，模型将在内存中保持 5 分钟然后卸载。如果你在对 LLM 进行多次请求时想要立即卸载模型，请使用 `ollama stop` 命令：

```shell
ollama stop llama3.2
```

如果你使用 API，请使用 `/api/generate` 和 `/api/chat` 端点的 `keep_alive` 参数设置模型在内存中保持的时间。`keep_alive` 参数可以设置为：
* 持续时间字符串（如 "10m" 或 "24h"）
* 以秒为单位的数字（如 3600）
* 任何负数将模型保持在内存中（如 -1 或 "-1m"）
* '0' 将在生成响应后立即卸载模型

例如，要预加载模型并将其保持在内存中，请使用：
```shell
curl http://localhost:11434/api/generate -d '{"model": "llama3.2", "keep_alive": -1}'
```

要卸载模型并释放内存，请使用：
```shell
curl http://localhost:11434/api/generate -d '{"model": "llama3.2", "keep_alive": 0}'
```

或者，可以在启动 Ollama 服务器时设置 `OLLAMA_KEEP_ALIVE` 环境变量来更改所有模型在内存中保持的时间。`OLLAMA_KEEP_ALIVE` 变量可以接受与 `keep_alive` 参数相同的值。

`/api/generate` 和 `/api/chat` API 端点的 `keep_alive` 参数将覆盖 `OLLAMA_KEEP_ALIVE` 设置。

## 如何管理 Ollama 服务器可以排队的最大请求数？

如果发送到服务器的请求过多，服务器将响应 503 错误，表示服务器过载。你可以通过设置 `OLLAMA_MAX_QUEUE` 调整可以排队的请求数量。

## Ollama 如何处理并发请求？

Ollama 支持两个级别的并发处理。如果系统有足够的可用内存（使用 CPU 推理时为系统内存，使用 GPU 推理时为 VRAM），则可以同时加载多个模型。

如果没有足够的可用内存加载新模型请求，同时已加载一个或多个模型，所有新请求将排队，直到可以加载新模型。随着先前模型被卸载，队列中的请求将依次处理。

给定模型的并行请求处理会增加上下文大小。例如，4 个并行请求的 2K 上下文将导致 8K 上下文大小。

以下服务器设置可以用于调整 Ollama 在大多数平台上如何处理并发请求：

- `OLLAMA_MAX_LOADED_MODELS` - 同时加载的最大模型数量，前提是它们适合可用内存。默认值是 GPU 数量的 3 倍或 CPU 推理的 3。
- `OLLAMA_NUM_PARALLEL` - 每个模型将同时处理的最大并行请求数。默认值将自动选择 4 或 1，具体取决于可用内存。
- `OLLAMA_MAX_QUEUE` - Ollama 忙时将排队的最大请求数量，超过此数量将拒绝额外请求。默认值为 512

注意：由于 ROCm v5.7 中的可用 VRAM 报告限制，Windows 上的 Radeon GPU 目前默认最大加载 1 个模型。一旦 ROCm v6.2 可用，Windows Radeon 将遵循默认设置。

## Ollama 如何在多个 GPU 上加载模型？

加载新模型时，Ollama 会评估模型所需的 VRAM 以及当前可用的 VRAM。如果模型可以完全放入任何一个 GPU 上，Ollama 将在该 GPU 上加载模型。

## 如何启用 Flash Attention？

Flash Attention 是大多数现代模型的一项功能，可以显著减少上下文大小增加时的内存使用。要启用 Flash Attention，请设置 `OLLAMA_FLASH_ATTENTION` 环境变量。

## 如何设置 K/V 缓存的量化类型？

启用 Flash Attention 时，可以量化 K/V 上下文缓存以显著减少内存使用。

要使用量化的 K/V 缓存与 Ollama 一起使用，可以设置以下环境变量：

- `OLLAMA_KV_CACHE_TYPE` - K/V 缓存的量化类型。默认值为 `f16`。

> 注意：目前这是一个全局选项，意味着所有模型将使用指定的量化类型运行。

当前可用的 K/V 缓存量化类型为：

- `f16` - 高精度和内存使用（默认）。
- `q8_0` - 8 位量化，使用约 `f16` 内存的 1/2，精度损失非常小，通常对模型质量没有明显影响（推荐如果不使用 f16）。
- `q4_0` - 4 位量化，使用约 `f16` 内存的 1/4，精度损失中等，在较大上下文大小时可能更明显。

缓存量化对模型响应质量的影响将取决于模型和任务。具有高 GQA 数量的模型（例如 Qwen2）可能会在精度上受到更大影响。

你可能需要尝试不同的量化类型以找到内存使用和质量之间的最佳平衡。

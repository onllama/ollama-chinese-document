# Ollama Docker 镜像

### 仅限 CPU

```bash
docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

### Nvidia GPU
安装 [NVIDIA 容器工具包](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installation)。

#### 使用 Apt 安装
1.  配置仓库
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
    | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
    | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
    | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
```
2.  安装 NVIDIA 容器工具包
```bash
sudo apt-get install -y nvidia-container-toolkit
```

#### 使用 Yum 或 Dnf 安装
1.  配置仓库

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo \
    | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
```

2. 安装 NVIDIA 容器工具包

```bash
sudo yum install -y nvidia-container-toolkit
```

#### 配置 Docker 使用 Nvidia 驱动
```
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

#### 启动容器

```bash
docker run -d --gpus=all -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

> [!NOTE]  
> 如果你在运行 NVIDIA JetPack 系统，Ollama 无法自动发现正确的 JetPack 版本。将环境变量 JETSON_JETPACK=5 或 JETSON_JETPACK=6 传递给容器以指定版本。

### AMD GPU

要使用 AMD GPU 运行 Ollama，请使用 `rocm` 标签并运行以下命令：

```
docker run -d --device /dev/kfd --device /dev/dri -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama:rocm
```

### 本地运行模型

现在你可以运行一个模型：

```
docker exec -it ollama ollama run llama3.2
```

### 尝试不同的模型

更多模型可以在 [Ollama 库](https://ollama.com/library) 中找到。

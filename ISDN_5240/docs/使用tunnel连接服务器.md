# ISDN 5240：使用 VS Code Tunnel 连接云服务器并配置 cuRobo

本文介绍如何在一台带 NVIDIA GPU 的 Linux 云服务器上启动 VS Code Remote Tunnel，从本地 VS Code 连接服务器，并在服务器中安装和验证 cuRobo。

> 本文中的命令默认在**云服务器终端**执行。只有标注为“本地电脑”的步骤才在自己的电脑上操作。

参考资料：

- [VS Code Remote Tunnels 官方文档](https://code.visualstudio.com/docs/remote/tunnels)
- [cuRobo 官方安装文档](https://nvlabs.github.io/curobo/latest/getting-started/installation.html)
- [uv 官方安装文档](https://docs.astral.sh/uv/getting-started/installation/)

## 1. 开始前的准备

### 1.1 云服务器要求

按照当前 cuRobo 官方文档，推荐环境为：

- Ubuntu 20.04 或更高版本，推荐 Ubuntu 22.04；
- NVIDIA GPU，架构新于 Turing，显存至少 4 GB；
- NVIDIA 驱动版本不低于 `580.65.06`，并至少支持 CUDA 12；
- Python 3.10–3.13，本文统一使用 Python 3.11；
- 能访问 GitHub、PyPI、Astral 和 Microsoft/GitHub 登录服务；
- 磁盘有足够空间安装 PyTorch、CUDA 运行库和 cuRobo。

> 云平台镜像和课程服务器的配置可能与上述要求不同。以 `nvidia-smi` 的实际输出及课程提供者的说明为准，不要在共享服务器上自行升级或替换 NVIDIA 驱动。

### 1.2 本地电脑要求

在本地电脑安装：

1. [Visual Studio Code](https://code.visualstudio.com/)
2. VS Code 扩展 **Remote - Tunnels**

也可以直接使用浏览器打开 `https://vscode.dev`，但桌面版 VS Code 通常更适合开发。

### 1.3 准备登录账号

启动 Tunnel 时需要使用 GitHub 或 Microsoft 账号完成授权。云服务器端与本地 VS Code 必须登录**同一个账号**，否则本地看不到该服务器。

## 2. 首次进入云服务器

Tunnel 需要先在服务器上启动，因此第一次仍需通过云平台提供的网页终端、SSH 或其他管理终端进入服务器。Tunnel 建立后，后续开发通常不再需要直接使用 SSH。

登录后先确认系统和 GPU：

```bash
uname -m
cat /etc/os-release
nvidia-smi
```

请确认：

- `uname -m` 通常输出 `x86_64`；
- `nvidia-smi` 能列出 GPU，而不是提示命令不存在或无法连接驱动；
- `nvidia-smi` 顶部显示的 Driver Version 满足 cuRobo 要求；
- `CUDA Version` 至少为 12.x。

> `nvidia-smi` 中的 `CUDA Version` 表示当前驱动能够支持的最高 CUDA 版本，并不等同于服务器已经安装的 CUDA Toolkit 版本。不过，它可用于选择 cuRobo 的 CUDA 12 或 CUDA 13 安装项。

## 3. 在云服务器安装 VS Code CLI

先检查服务器是否已经安装：

```bash
code version
```

如果能够显示版本号，可直接进入下一节。如果提示 `code: command not found`，按下面的方式安装独立 CLI。

### 3.1 x86_64 服务器

大多数云 GPU 服务器使用此架构：

```bash
mkdir -p "$HOME/.local/bin"
curl -L 'https://code.visualstudio.com/sha/download?build=stable&os=cli-alpine-x64' \
  --output /tmp/vscode_cli.tar.gz
tar -xzf /tmp/vscode_cli.tar.gz -C "$HOME/.local/bin"
chmod +x "$HOME/.local/bin/code"
export PATH="$HOME/.local/bin:$PATH"
code version
```

为了让之后打开的终端也能找到 `code`，执行：

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> "$HOME/.bashrc"
source "$HOME/.bashrc"
```

### 3.2 ARM64 服务器

仅当 `uname -m` 输出 `aarch64` 或 `arm64` 时，将下载地址中的 `cli-alpine-x64` 改为 `cli-alpine-arm64`：

```bash
mkdir -p "$HOME/.local/bin"
curl -L 'https://code.visualstudio.com/sha/download?build=stable&os=cli-alpine-arm64' \
  --output /tmp/vscode_cli.tar.gz
tar -xzf /tmp/vscode_cli.tar.gz -C "$HOME/.local/bin"
chmod +x "$HOME/.local/bin/code"
export PATH="$HOME/.local/bin:$PATH"
code version
```

## 4. 在云服务器启动 Tunnel

执行：

```bash
code tunnel --accept-server-license-terms
```

第一次运行时通常需要：

1. 选择 GitHub 或 Microsoft 作为登录方式；
2. 复制终端显示的一次性验证码；
3. 在浏览器中打开终端给出的登录地址；
4. 输入验证码并授权；
5. 如果程序要求设置机器名称，为服务器填写一个易识别的名称，例如 `isdn5240-gpu`。

授权成功后，终端会显示一个类似下面的地址：

```text
https://vscode.dev/tunnel/<机器名称>/<目录名称>
```

不要关闭当前终端。前台运行的 `code tunnel` 一旦被终止，Tunnel 也会断开。

## 5. 从本地 VS Code 连接服务器

在**本地电脑**操作：

1. 打开 VS Code；
2. 安装并启用 **Remote - Tunnels** 扩展；
3. 点击左下角远程连接图标，或按 `F1` / `Ctrl+Shift+P` / `Cmd+Shift+P` 打开命令面板；
4. 运行 `Remote Tunnels: Connect to Tunnel`；
5. 使用与服务器端相同的 GitHub 或 Microsoft 账号登录；
6. 选择刚才设置的服务器名称；
7. 连接后选择 `File -> Open Folder`，打开服务器上的工作目录，例如 `/home/<用户名>/workspace`。

连接成功后，VS Code 左下角会显示远程服务器名称。此时集成终端、代码运行、调试和远程扩展都在云服务器上执行。

> Python、Jupyter 等扩展需要安装在 Tunnel 对应的远程环境中。如果扩展页面出现 `Install in Tunnel: <机器名称>`，请选择该按钮。

也可以直接在浏览器中打开服务器端输出的 `vscode.dev/tunnel/...` 地址。

## 6. 在云服务器安装 cuRobo

以下操作可在远程 VS Code 的 `Terminal -> New Terminal` 中完成。

### 6.1 安装基础工具

如果账号具有 sudo 权限：

```bash
sudo apt-get update
sudo apt-get install -y git curl ca-certificates build-essential git-lfs tmux
git lfs install
```

如果没有 sudo 权限，先检查这些命令是否已经存在：

```bash
git --version
curl --version
gcc --version
```

缺少系统依赖时请联系服务器管理员，不要擅自修改共享环境。

### 6.2 安装 uv

cuRobo 当前官方文档推荐使用 `uv` 管理 Python 环境。安装命令为：

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
uv --version
```

如需先审阅安装脚本，可执行：

```bash
curl -LsSf https://astral.sh/uv/install.sh | less
```

### 6.3 克隆 cuRobo

```bash
mkdir -p "$HOME/workspace"
cd "$HOME/workspace"
git clone https://github.com/NVlabs/curobo.git
cd curobo
```

记录当前版本，方便日后复现实验环境：

```bash
git rev-parse HEAD
git status
```

课程项目如果指定了 tag 或 commit，应在安装前切换到课程指定版本；没有指定时才使用当前默认分支。

### 6.4 创建 Python 3.11 虚拟环境

```bash
uv venv --python 3.11
source .venv/bin/activate
python --version
```

激活成功后，终端提示符前通常会出现 `(.venv)`。每次打开新的终端后，都需要重新执行：

```bash
cd "$HOME/workspace/curobo"
source .venv/bin/activate
```

### 6.5 根据 CUDA 版本安装

再次检查驱动支持的 CUDA 版本：

```bash
nvidia-smi | grep 'CUDA Version'
```

如果显示 CUDA 12.x，并且当前虚拟环境中还没有安装 PyTorch：

```bash
uv pip install '.[cu12-torch]'
```

如果显示 CUDA 13.x，并且当前虚拟环境中还没有安装 PyTorch：

```bash
uv pip install '.[cu13-torch]'
```

如果虚拟环境中已经安装了兼容的 PyTorch，则使用不包含 `-torch` 的安装项：

```bash
# CUDA 12.x
uv pip install '.[cu12]'

# CUDA 13.x
uv pip install '.[cu13]'
```

四条安装命令只选择其中一条。新环境优先使用包含 `-torch` 的命令，让项目安装匹配的 PyTorch 依赖。

> 安装过程需要下载体积较大的依赖，耗时取决于服务器网络。不要因为一段时间没有新输出就立即中断安装。

## 7. 验证 cuRobo 环境

确保虚拟环境已激活，然后依次检查。

### 7.1 检查 PyTorch 与 GPU

```bash
python - <<'PY'
import torch

print("PyTorch:", torch.__version__)
print("PyTorch CUDA:", torch.version.cuda)
print("CUDA available:", torch.cuda.is_available())
if torch.cuda.is_available():
    print("GPU:", torch.cuda.get_device_name(0))
PY
```

预期 `CUDA available` 为 `True`，并能显示服务器 GPU 名称。

### 7.2 检查 cuRobo

```bash
python -c "import curobo; print(curobo.__version__)"
```

### 7.3 运行官方测试

```bash
pytest --pyargs curobo.tests
```

首次运行可能需要编译或缓存 CUDA 内核，因此会比后续运行更慢。只要测试最终完成且没有失败，即说明基本环境配置成功。

## 8. 在 VS Code 中选择正确的 Python 环境

在本地 VS Code 已连接 Tunnel 的窗口中：

1. 安装远程版 Python 扩展；
2. 打开命令面板；
3. 运行 `Python: Select Interpreter`；
4. 选择 `~/workspace/curobo/.venv/bin/python`；
5. 新建终端并确认：

   ```bash
   which python
   python -c "import curobo; print(curobo.__version__)"
   ```

如果 `which python` 没有指向 `.venv/bin/python`，请手动激活虚拟环境。

## 9. 常见问题

### 9.1 本地 VS Code 看不到服务器

- 确认服务器上的 `code tunnel` 仍在运行；
- 确认服务器端和本地使用同一个 GitHub 或 Microsoft 账号；
- 在本地运行 `Remote Tunnels: Connect to Tunnel`，不要误选 Remote SSH；
- 如果服务器重启，重新启动 Tunnel；
- 检查服务器是否能出站访问 Microsoft Tunnel 服务。

### 9.2 Tunnel 名称冲突或需要取消注册

查看帮助：

```bash
code tunnel --help
```

取消当前机器的 Tunnel 注册：

```bash
code tunnel unregister
```

取消注册后需要重新运行 `code tunnel` 并完成授权。

### 9.3 `nvidia-smi` 失败

这通常是云实例没有正确挂载 GPU、NVIDIA 驱动未启动，或选择了无 GPU 的实例规格。应先在云平台解决 GPU/驱动问题，再安装 cuRobo。

### 9.4 `torch.cuda.is_available()` 为 `False`

依次确认：

```bash
nvidia-smi
which python
python -c "import torch; print(torch.__version__, torch.version.cuda)"
```

常见原因包括选错 Python 解释器、安装了 CPU 版 PyTorch、驱动版本太旧，或当前会话没有分配 GPU。

### 9.5 安装时出现 `No space left on device`

检查磁盘：

```bash
df -h
du -sh "$HOME/.cache" 2>/dev/null
uv cache dir
```

确认不再需要缓存后，可使用 `uv cache prune` 清理过期缓存。不要删除其他用户或不明确用途的目录。

### 9.6 连接断开后任务是否继续

关闭本地 VS Code 通常不会自动终止已经在远程 shell 中启动的所有进程，但普通终端任务可能因 shell 或服务器会话结束而中断。长时间实验建议使用 `tmux`：

```bash
tmux new -s experiment
# 在 tmux 中运行实验
```

离开会话：按 `Ctrl+B`，再按 `D`。恢复会话：

```bash
tmux attach -t experiment
```

## 10. 停止使用与安全注意事项

临时前台 Tunnel 可在服务器终端按 `Ctrl+C` 停止。若安装了服务，则执行：

```bash
code tunnel service uninstall
```

如需移除机器注册：

```bash
code tunnel unregister
```

最后根据云平台说明停止或释放云实例，避免继续计费。

请勿在教程、截图或代码仓库中公开：

- Tunnel 登录的一次性验证码；
- 云平台密钥、Access Token 或 SSH 私钥；
- GitHub/Microsoft 访问令牌；
- 服务器中的课程数据或个人数据。

完成上述步骤后，本地 VS Code 应能通过 Tunnel 打开云服务器目录，并使用远程 GPU 环境运行 cuRobo。

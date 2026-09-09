# 在 RunPod 上构建并部署 cuRobo V2 自定义镜像

本文介绍如何基于 RunPod 官方 PyTorch 镜像构建一个包含 cuRobo 的自定义 Docker 镜像，将镜像推送到 Docker Hub，并通过 RunPod Template 启动可直接使用的 GPU Pod。

本文使用以下版本组合：

| 组件 | 版本 |
| --- | --- |
| cuRobo | `v0.8.0`（cuRobo V2） |
| PyTorch | `2.9.1` |
| CUDA | `12.8.1` |
| Ubuntu | `22.04` |
| 基础镜像 | `runpod/pytorch:1.0.3-cu1281-torch291-ubuntu2204` |
| 容器平台 | `linux/amd64` |

> 更新时间：2026-09-09。`v0.8.0` 是本文编写时 cuRobo 最新的正式 Release。为了保证环境可复现，本文固定正式版本标签，而不直接追踪随时可能改变的 `main` 分支。

## 1. 整体流程

```text
修改 Dockerfile
       ↓
GitHub Actions 或本地 Docker 构建镜像
       ↓
推送到 Docker Hub
       ↓
在 RunPod 中创建自定义 Template
       ↓
选择 GPU 并启动 Pod
       ↓
直接使用已经安装好的 cuRobo
```

RunPod Pod 本身是由平台根据指定 Docker 镜像启动的容器。因此，镜像构建完成后，不需要在每台新 Pod 中重新安装 cuRobo。

## 2. 重要版本说明

cuRobo `v0.8.0` 是一次重大重构，官方称之为 cuRobo V2。它与 `v0.7.8` 的 API 并不兼容。

旧版代码通常这样导入：

```python
from curobo.wrap.reacher.motion_gen import MotionGen, MotionGenConfig
```

cuRobo V2 应改为：

```python
from curobo.motion_planner import MotionPlanner, MotionPlannerCfg
```

如果现有项目依赖旧版 `MotionGen` API，应继续使用 `v0.7.8`；如果是新项目，则可以使用本文的 `v0.8.0` 方案。

## 3. 准备账号

开始前需要准备：

1. GitHub 账号；
2. Docker Hub 账号；
3. RunPod 账号及可用余额；
4. Docker Hub Access Token；
5. 可选：本地 Docker Desktop，用于检查 Dockerfile。

### 3.1 创建 Docker Hub Access Token

登录 Docker Hub，在账户安全设置中创建 Access Token。建议给予镜像仓库的 Read/Write 权限。

请勿将 Docker Hub 密码或 Access Token 写进 Dockerfile、Git 仓库或 RunPod 镜像。

## 4. Fork RunPod 模板仓库

打开 RunPod 官方模板仓库：

- <https://github.com/runpod-workers/pod-template>

点击右上角 **Fork**，将仓库复制到自己的 GitHub 账户。

然后克隆自己的 Fork：

```bash
git clone https://github.com/你的GitHub用户名/pod-template.git
cd pod-template
```

仓库中原有的 DistilBERT 示例与 cuRobo 无关。后续将替换 `Dockerfile` 和 GitHub Actions 工作流。`main.py`、`requirements.txt`、`Dockerfile.uv` 等示例文件可以保留，但本文构建过程不会使用它们。

## 5. 编写 cuRobo Dockerfile

将仓库根目录中的 `Dockerfile` 完整替换为以下内容：

```dockerfile
FROM runpod/pytorch:1.0.3-cu1281-torch291-ubuntu2204

ARG DEBIAN_FRONTEND=noninteractive
ARG CUROBO_REF=v0.8.0

ENV PYTHONUNBUFFERED=1 \
    CUDA_HOME=/usr/local/cuda \
    PATH=/usr/local/cuda/bin:${PATH} \
    LD_LIBRARY_PATH=/usr/local/cuda/lib64:/usr/local/lib:${LD_LIBRARY_PATH} \
    NVIDIA_VISIBLE_DEVICES=all \
    NVIDIA_DRIVER_CAPABILITIES=compute,utility,graphics \
    CUROBO_CACHE_DIR=/workspace/.cache/curobo

# 安装系统依赖
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        cmake \
        curl \
        git \
        git-lfs \
        libegl1-mesa-dev \
        libgl1-mesa-dev \
        libgles2-mesa-dev \
        libglvnd-dev \
        ninja-build \
        pkg-config \
        wget \
    && rm -rf /var/lib/apt/lists/*

RUN git lfs install --system

# 更新 Python 构建工具
RUN python -m pip install --no-cache-dir --upgrade \
        pip \
        setuptools \
        setuptools-scm \
        wheel \
        ninja

# 下载固定版本的 cuRobo，并拉取 Git LFS 中的模型和资源
RUN git clone \
        --branch ${CUROBO_REF} \
        --depth 1 \
        https://github.com/NVlabs/curobo.git \
        /opt/curobo \
    && cd /opt/curobo \
    && git lfs pull

# v0.8.0 发布后，warp-lang 1.13 曾出现 API 兼容变化。
# 固定到 1.13 以下可避免安装时拿到不兼容版本。
# 基础镜像已经包含 PyTorch，因此使用 cu12 而不是 cu12-torch，
# 避免 pip 重新安装或覆盖 PyTorch。
RUN python -m pip install --no-cache-dir "warp-lang<1.13" && \
    cd /opt/curobo && \
    python -m pip install --no-cache-dir ".[cu12]" --no-build-isolation

# 构建阶段没有 GPU，只执行依赖和导入检查。
# 真正的 CUDA 内核测试在 RunPod GPU Pod 启动后执行。
RUN python -c "import torch; import importlib.metadata as metadata; from curobo.motion_planner import MotionPlanner, MotionPlannerCfg; print('PyTorch:', torch.__version__); print('PyTorch CUDA:', torch.version.cuda); print('cuRobo:', metadata.version('nvidia-curobo')); print('cuRobo V2 import successful')"

RUN mkdir -p /workspace/.cache/curobo

WORKDIR /workspace

# 不覆盖基础镜像的 ENTRYPOINT 或 CMD。
# RunPod 的默认 /start.sh 会继续负责启动 SSH 和 JupyterLab。
```

### 5.1 为什么使用 `.[cu12]`

cuRobo `v0.8.0` 提供以下可选依赖：

```text
cu12
cu13
cu12-torch
cu13-torch
```

基础镜像已经带有适配 CUDA 12.8 的 PyTorch，因此使用：

```bash
pip install ".[cu12]"
```

这样只安装 cuRobo 的 CUDA 12 运行依赖，避免 `pip` 再次选择 PyTorch 版本。若使用 `.[cu12-torch]`，依赖解析器可能下载另一套 PyTorch，造成镜像膨胀或版本冲突。

### 5.2 为什么不设置 `TORCH_CUDA_ARCH_LIST`

cuRobo V2 默认使用 `cuda.core` 在运行时编译 CUDA 内核，不再默认通过 PyTorch 在镜像构建阶段编译全部 CUDA Extension。

因此，本方案无需设置：

```dockerfile
TORCH_CUDA_ARCH_LIST=...
```

这也使同一个镜像更容易用于 Ampere、Ada、Hopper 和 Blackwell 等不同 GPU。

### 5.3 为什么固定 `warp-lang<1.13`

cuRobo `v0.8.0` 的依赖范围只设置了 `warp-lang>=0.10.0`。此版本发布后，Warp 1.13 对部分接口进行了调整，曾导致 `warp.torch` 相关兼容问题。

在 cuRobo 发布包含相应修复的新正式版本前，限制为：

```text
warp-lang<1.13
```

可以提高 `v0.8.0` 镜像的可复现性。未来升级到新的 cuRobo 正式版本时，应重新检查这一限制是否仍然必要。

## 6. 配置 GitHub Actions 自动构建

将 `.github/workflows/dev.yml` 替换为：

```yaml
name: Build cuRobo V2 RunPod Image

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-22.04

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # CUDA/PyTorch 基础镜像较大，先释放 GitHub Runner 空间。
      - name: Free disk space
        run: |
          sudo rm -rf /usr/share/dotnet
          sudo rm -rf /opt/ghc
          sudo rm -rf /usr/local/lib/android
          docker system prune --all --force || true
          df -h

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/runpod-curobo:v0.8.0-cu128
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

这里没有发布 `latest` 标签。生产环境使用明确版本标签，可以避免 RunPod 镜像缓存和不可预期升级。

## 7. 配置 GitHub Secrets

进入自己 Fork 后的 GitHub 仓库：

```text
Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

添加两个 Repository Secret：

| Secret 名称 | 内容 |
| --- | --- |
| `DOCKERHUB_USERNAME` | Docker Hub 用户名 |
| `DOCKERHUB_TOKEN` | Docker Hub Access Token |

## 8. 提交代码并触发构建

```bash
git add Dockerfile .github/workflows/dev.yml
git commit -m "Build cuRobo v0.8.0 RunPod image"
git push origin main
```

进入 GitHub 仓库的 **Actions** 页面查看构建进度。

构建成功后，镜像地址为：

```text
你的DockerHub用户名/runpod-curobo:v0.8.0-cu128
```

### 8.1 手动触发构建

也可以进入：

```text
GitHub 仓库
→ Actions
→ Build cuRobo V2 RunPod Image
→ Run workflow
```

### 8.2 GitHub Runner 磁盘不足

CUDA/PyTorch 镜像体积较大。如果构建日志出现：

```text
no space left on device
```

可以选择以下方式之一：

1. 使用具有更大磁盘的 GitHub Actions Runner；
2. 使用 Blacksmith 等兼容 GitHub Actions 的大容量 Runner；
3. 在 Linux x86_64 服务器上本地构建；
4. 减少 Dockerfile 中不必要的软件包和缓存。

## 9. 可选：在本地构建并推送

如果本机是 Linux x86_64 且已安装 Docker：

```bash
docker login

docker buildx build \
  --platform linux/amd64 \
  --tag 你的DockerHub用户名/runpod-curobo:v0.8.0-cu128 \
  --push \
  .
```

如果使用 Apple Silicon Mac，也能通过 `--platform linux/amd64` 构建，但需要模拟 x86_64，速度可能很慢，而且会占用大量磁盘空间。更推荐使用 GitHub Actions 或 Linux x86_64 构建机。

本地没有 NVIDIA GPU 时，只能验证镜像是否构建成功，不能完成真正的 CUDA 运行测试。

## 10. 在 RunPod 创建自定义 Template

登录 RunPod 控制台，进入：

```text
Templates
→ New Template
```

建议配置如下：

| 配置项 | 建议值 |
| --- | --- |
| Template Name | `cuRobo-v0.8.0-cu128` |
| Container Image | `你的DockerHub用户名/runpod-curobo:v0.8.0-cu128` |
| Container Disk | 至少 `40 GB` |
| Volume Mount Path | `/workspace` |
| HTTP Port | `8888`，用于 JupyterLab |
| HTTP Port | `8080`，用于 cuRobo Viser 可视化 |
| TCP Port | `22`，用于 SSH |

不要修改或清空镜像默认启动命令，否则可能导致 RunPod 的 SSH 或 JupyterLab 服务无法启动。

### 10.1 环境变量

如需 JupyterLab，可以添加：

```text
JUPYTER_PASSWORD=设置一个安全密码
```

如需通过 SSH 公钥登录，可以添加：

```text
PUBLIC_KEY=你的SSH公钥
```

不要将私钥放入环境变量或镜像。

### 10.2 私有 Docker Hub 仓库

如果 Docker Hub 镜像为私有镜像，需要在 RunPod 中创建 Container Registry Credential，并为 Template 选择该凭证。

通常需要填写：

```text
Registry：Docker Hub
Username：Docker Hub 用户名
Password/Token：Docker Hub Access Token
```

如果镜像是公开的，则无需配置 Registry Credential。

## 11. 创建 RunPod GPU Pod

进入：

```text
Pods
→ Deploy
```

选择：

1. 所需 GPU；
2. 刚创建的 `cuRobo-v0.8.0-cu128` Template；
3. On-Demand 或 Spot 实例；
4. 点击 Deploy。

RunPod 随后会自动：

```text
拉取 Docker Hub 镜像
→ 创建容器
→ 挂载 NVIDIA GPU
→ 挂载 /workspace
→ 启动 SSH 与 JupyterLab
```

镜像第一次拉取可能需要数分钟，具体时间取决于数据中心网络和镜像缓存情况。

## 12. GPU 选择建议

CUDA 12.8 与 PyTorch 2.9.1 可以覆盖较新的 NVIDIA GPU。常见选择包括：

| GPU | 适用情况 |
| --- | --- |
| RTX 4090 | 性价比较高，适合开发和单机器人规划 |
| L40 / L40S | 显存较大，适合批量规划 |
| A100 | 稳定，适合较大批量任务 |
| H100 / H200 | 适合高吞吐或复杂批量任务 |
| RTX 5090 | Blackwell 架构，需要较新的 CUDA/PyTorch，本方案更适合它 |
| B200 | Blackwell 数据中心 GPU，成本较高 |

cuRobo 官方最低要求通常为 Volta 或更新架构、至少 4 GB 显存。实际使用时建议至少 16 GB 显存，复杂场景、体素地图或批量规划需要更多显存。

## 13. 启动后检查运行环境

打开 RunPod Web Terminal 或通过 SSH 连接，然后运行：

```bash
nvidia-smi
```

检查 Python、PyTorch、CUDA 和 cuRobo：

```bash
python - <<'PY'
import importlib.metadata as metadata
import torch

print("PyTorch:", torch.__version__)
print("PyTorch CUDA:", torch.version.cuda)
print("CUDA available:", torch.cuda.is_available())

if torch.cuda.is_available():
    print("GPU:", torch.cuda.get_device_name(0))
    print("Compute capability:", torch.cuda.get_device_capability(0))

print("cuRobo:", metadata.version("nvidia-curobo"))
PY
```

期望看到类似结果：

```text
PyTorch: 2.9.1+cu128
PyTorch CUDA: 12.8
CUDA available: True
GPU: NVIDIA GeForce RTX 4090
Compute capability: (8, 9)
cuRobo: 0.8.0
```

如果 `CUDA available` 为 `False`，说明当前 Pod 没有正确挂载 GPU，或 PyTorch/CUDA 环境存在异常。

## 14. 运行 cuRobo V2 官方示例

### 14.1 测试位姿运动规划

```bash
python -m curobo.examples.getting_started.motion_planning \
  --mode pose \
  --output-dir /workspace/curobo-output
```

第一次执行时，cuRobo 会编译并缓存 CUDA 内核，因此会比后续运行慢。

成功后应生成：

```text
/workspace/curobo-output/motion_plan.pdf
```

### 14.2 测试位姿规划与抓取规划

```bash
python -m curobo.examples.getting_started.motion_planning \
  --mode all \
  --output-dir /workspace/curobo-output
```

成功后通常会生成：

```text
/workspace/curobo-output/motion_plan.pdf
/workspace/curobo-output/grasp_plan.pdf
```

### 14.3 启动交互式 Viser 可视化

确认 RunPod Template 已开放 HTTP 端口 `8080`，然后运行：

```bash
python -m curobo.examples.getting_started.motion_planning \
  --visualize \
  --port 8080
```

之后在 RunPod 控制台的 **HTTP Services** 中打开 8080 端口。

## 15. cuRobo V2 最小初始化示例

创建 `/workspace/test_curobo.py`：

```python
import torch

from curobo.motion_planner import MotionPlanner, MotionPlannerCfg


def main():
    if not torch.cuda.is_available():
        raise RuntimeError("CUDA GPU is not available")

    config = MotionPlannerCfg.create(
        robot="franka.yml",
        scene_model="collision_test.yml",
    )

    planner = MotionPlanner(config)
    planner.warmup(
        enable_graph=True,
        num_warmup_iterations=5,
    )

    print("cuRobo V2 MotionPlanner initialized successfully")
    print("GPU:", torch.cuda.get_device_name(0))


if __name__ == "__main__":
    main()
```

执行：

```bash
python /workspace/test_curobo.py
```

## 16. 数据持久化建议

镜像中的 cuRobo 位于：

```text
/opt/curobo
```

自己的代码、机器人 URDF、网格文件和输出结果建议放在：

```text
/workspace
```

推荐为 `/workspace` 挂载 RunPod Network Volume。这样即使删除 Pod，项目代码和结果仍然可以保留。

建议目录结构：

```text
/workspace
├── projects
│   └── robot_project
├── robot_assets
│   ├── urdf
│   └── meshes
├── curobo-output
└── .cache
    └── curobo
```

需要注意：

- 镜像内部 `/opt/curobo` 的内容随镜像版本固定；
- `/workspace` 中的内容应由 Network Volume 持久化；
- 不要把大型数据集、运行结果或个人项目代码重新构建进基础镜像，除非它们确实需要随镜像发布。

## 17. 更新 cuRobo 版本

当官方发布新版本，例如 `v0.8.1` 时，修改 Dockerfile：

```dockerfile
ARG CUROBO_REF=v0.8.1
```

同时修改 GitHub Actions 标签：

```yaml
tags: |
  ${{ secrets.DOCKERHUB_USERNAME }}/runpod-curobo:v0.8.1-cu128
```

提交后重新构建，并在 RunPod Template 中更新镜像名称。

不要重复覆盖同一个版本标签。明确的新标签能避免 RunPod 拉取到缓存中的旧镜像，也方便回滚。

如果确实需要跟踪最新开发代码，可以设置：

```dockerfile
ARG CUROBO_REF=main
```

但 `main` 不保证 API 稳定，不建议用于课程展示、实验复现或生产环境。

## 18. 常见问题排查

### 18.1 `CUDA available: False`

依次检查：

```bash
nvidia-smi
python -c "import torch; print(torch.__version__, torch.version.cuda)"
python -c "import torch; print(torch.cuda.is_available())"
```

如果 `nvidia-smi` 本身不可用，应先确认创建的是 GPU Pod，而不是 CPU Pod，并检查 Pod 是否正常启动。

### 18.2 cuRobo 找不到模型或网格文件

检查 Git LFS 文件是否完整：

```bash
cd /opt/curobo
git lfs pull
git lfs ls-files
```

Dockerfile 已经执行 `git lfs pull`。如果这里仍缺少文件，通常是构建阶段 GitHub 网络或 Git LFS 下载失败。

### 18.3 出现 `warp.torch` 相关错误

检查 Warp 版本：

```bash
python -c "import importlib.metadata as m; print(m.version('warp-lang'))"
```

对于本文的 cuRobo `v0.8.0` 镜像，Warp 应低于 1.13。可以检查：

```bash
python -m pip show warp-lang
```

如果版本不符合，说明镜像未按本文 Dockerfile 构建，或者运行时又更新了 Warp。

### 18.4 RunPod 仍在使用旧镜像

不要反复覆盖同一个标签。重新构建时使用新标签，例如：

```text
runpod-curobo:v0.8.0-cu128-r1
runpod-curobo:v0.8.0-cu128-r2
```

更新 RunPod Template 后，停止并重新创建 Pod。已经运行的 Pod 不会因为 Docker Hub 上的镜像发生变化而自动更新。

### 18.5 Pod 启动后立即退出

确认 Dockerfile 最后没有添加：

```dockerfile
ENTRYPOINT []
CMD ["python", "..."]
```

交互式 Pod 应保留 RunPod 基础镜像默认的入口点和 `/start.sh`，否则 SSH 与 JupyterLab 服务可能无法启动。

### 18.6 JupyterLab 无法打开

检查：

1. Template 是否开放 HTTP 端口 `8888`；
2. 是否设置了 `JUPYTER_PASSWORD`；
3. Pod 日志中 `/start.sh` 是否正常运行；
4. Dockerfile 是否覆盖了默认 `ENTRYPOINT` 或 `CMD`。

### 18.7 Viser 页面无法打开

检查：

1. Template 是否开放 HTTP 端口 `8080`；
2. 启动参数是否为 `--port 8080`；
3. 程序是否仍在运行；
4. 是否通过 RunPod 的 HTTP Services 链接访问，而不是直接访问容器内部地址。

## 19. 本方案不包含的组件

该镜像面向独立使用 cuRobo Python 库，包括：

- 正向与逆向运动学；
- 碰撞检测；
- 轨迹优化；
- 几何路径规划；
- MotionPlanner；
- cuRobo V2 示例；
- Viser Web 可视化。

本方案没有安装：

- NVIDIA Isaac Sim；
- ROS 或 ROS 2；
- nvblox / nvblox_torch；
- 桌面图形环境；
- 真实机器人的驱动程序。

如果项目需要 Isaac Sim，应以对应版本的 Isaac Sim 容器作为基础镜像重新设计。Isaac Sim 镜像体积、驱动要求、EULA、图形/Vulkan 配置和启动方式均与本文不同，不建议直接在当前镜像上追加安装。

## 20. 参考资料

- RunPod 自定义 Pod Template：<https://docs.runpod.io/pods/templates/create-custom-template>
- RunPod Pod Template 示例仓库：<https://github.com/runpod-workers/pod-template>
- RunPod PyTorch 镜像：<https://hub.docker.com/r/runpod/pytorch/tags>
- cuRobo 官方仓库：<https://github.com/NVlabs/curobo>
- cuRobo v0.8.0：<https://github.com/NVlabs/curobo/tree/v0.8.0>
- cuRobo Releases：<https://github.com/NVlabs/curobo/releases>
- cuRobo V2 Python 依赖配置：<https://github.com/NVlabs/curobo/blob/v0.8.0/pyproject.toml>
- cuRobo V2 Motion Planning 示例：<https://github.com/NVlabs/curobo/blob/main/curobo/examples/getting_started/motion_planning.py>


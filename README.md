# Axis OmniReset 5033 / 5716 复现数据包

本仓库记录 Task 5033、5716 的复现命令，并通过 [v1 Release](https://github.com/jennyYE1224/axis-omnireset-task5716/releases/tag/v1) 提供数据包。Git 仓库本身不包含数据文件，也不包含 axis-omnireset 源码。运行下方命令需要另行获取完整源码。

## 数据包说明

Release 中有两个附件：

- `axis-omnireset-artifacts.tar.gz`：原始归档，是 `prepare --from dist/axis-omnireset-artifacts.tar.gz` 的输入。
- `task5716-prepared.tar.gz`：从已准备好的 `artifacts/task5716/` 目录打包而成。解压后可直接得到该目录，无需再次对 Task 5716 执行 `prepare`。

目前没有单独提供 Task 5033 的已准备数据目录。下方 `convert` 命令的输出位于 `runs/convert5716/` 和 `runs/convert5033/`，不包含在这两个附件中。

## 下载

以下命令应在**完整的 axis-omnireset 源码仓库根目录**执行。

### 下载原始归档

如果要从原始归档重新执行 `prepare`，下载此文件：

```bash
mkdir -p dist
wget -O dist/axis-omnireset-artifacts.tar.gz \
  https://github.com/jennyYE1224/axis-omnireset-task5716/releases/download/v1/axis-omnireset-artifacts.tar.gz
```

### 下载已准备好的 Task 5716 数据

如果要直接使用已准备好的 `artifacts/task5716/`，下载并解压此文件：

```bash
mkdir -p dist artifacts
wget -O dist/task5716-prepared.tar.gz \
  https://github.com/jennyYE1224/axis-omnireset-task5716/releases/download/v1/task5716-prepared.tar.gz

tar -xzf dist/task5716-prepared.tar.gz -C artifacts
```

这两种方式可按需要选择。若要完整重走数据准备流程，应使用原始归档并执行下方的 `prepare` 命令；直接下载已准备数据则可跳过 Task 5716 的 `prepare`。

## 复现执行记录

以下是我复现时执行的命令。路径中的 `/data/jiayi/axis-omnireset`、`/home/jiayi/` 以及 Isaac Sim 扩展版本与当时的运行环境有关；在其他机器上运行时需按实际安装路径调整。

```bash
# 创建环境
bash tools/create_env.sh cpu python3.11

# 此环境变量用于处理当时遇到的依赖解析报错
UV_INDEX_STRATEGY=unsafe-best-match bash tools/create_env.sh isaac python3.11

bash tools/create_env.sh viser python3.11

# 从原始归档准备数据
.venv-cpu/bin/axis-omnireset prepare --task 5716 --from dist/axis-omnireset-artifacts.tar.gz
.venv-cpu/bin/axis-omnireset prepare --task 5033 --from dist/axis-omnireset-artifacts.tar.gz

# 检查环境和数据
.venv-cpu/bin/axis-omnireset doctor --task 5716 --environment cpu
.venv-isaac/bin/axis-omnireset doctor --task 5716 --environment isaac
.venv-cpu/bin/axis-omnireset doctor --task 5033 --environment cpu
.venv-isaac/bin/axis-omnireset doctor --task 5033 --environment isaac

# 转换数据
OMP_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 MKL_NUM_THREADS=1 \
  .venv-cpu/bin/axis-omnireset convert --task 5716 --out runs/convert5716 --workers 4

.venv-cpu/bin/axis-omnireset convert --task 5033 --out runs/convert5033

# Task 5716 资格检查
CUDA_VISIBLE_DEVICES=0 OMNI_KIT_ACCEPT_EULA=YES TMPDIR=/data/jiayi/axis-omnireset/tmp \
  .venv-isaac/bin/axis-omnireset qualify --task 5716 --headless --device cuda:0 \
  --num-envs 49152 --draws 5 --out runs/qualify5716

# Task 5716 训练
CUDA_VISIBLE_DEVICES=0 OMNI_KIT_ACCEPT_EULA=YES TMPDIR=/data/jiayi/axis-omnireset/tmp \
  .venv-isaac/bin/axis-omnireset train --task 5716 --headless --device cuda:0 \
  --out runs/apple_train --iterations 1024 --live runs/apple_train/live.npz \
  2>&1 | tee runs/apple_train.log

# Task 5716 评估
CUDA_VISIBLE_DEVICES=0 OMNI_KIT_ACCEPT_EULA=YES TMPDIR=/data/jiayi/axis-omnireset/tmp \
  .venv-isaac/bin/axis-omnireset evaluate --task 5716 --protocol curriculum \
  --headless --device cuda:0 --out runs/apple_curriculum_eval

# 导出机器人网格
PYTHONPATH=.venv-isaac/lib/python3.11/site-packages/isaacsim/extscache/omni.usd.libs-1.0.1+69cbf6ad.lx64.r.cp311 \
LD_LIBRARY_PATH=.venv-isaac/lib/python3.11/site-packages/isaacsim/extscache/omni.usd.libs-1.0.1+69cbf6ad.lx64.r.cp311/bin:/home/jiayi/.local/share/uv/python/cpython-3.11.17-linux-x86_64-gnu/lib \
  .venv-isaac/bin/axis-omnireset export-robot-meshes \
  --usd artifacts/native_assets/Assets/Isaac/5.1/Isaac/IsaacLab/Factory/franka_mimic.usd \
  --out runs/native_robot_meshes.npz

# 使用 Viser 查看训练状态
.venv-viser/bin/axis-omnireset viser --task 5716 \
  --live-state runs/apple_train/live.npz \
  --native-robot-meshes runs/native_robot_meshes.npz \
  --port 8080
```
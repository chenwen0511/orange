# GenPosePlus 验证指南（香橙派 310P RC）

本文档说明在 **香橙派 AI Station + 昇腾 310P RC** 上验证 [GenPosePlus](https://gitcode.com/Ascend/ModelZoo-PyTorch/blob/master/ACL_PyTorch/built-in/embodied_ai/GenPosePlus/README.md) 的完整步骤。CANN / 驱动 / 分区等通用环境见仓库根目录 [README.md](../README.md) 第 1～3 章。

## 参考链接

- 官方推理指导：[ModelZoo GenPosePlus README](https://gitcode.com/Ascend/ModelZoo-PyTorch/blob/master/ACL_PyTorch/built-in/embodied_ai/GenPosePlus/README.md)
- PyTorch 安装（按 CANN 版本选章节）：[昇腾 ModelZoo PyTorch 环境准备](https://www.hiascend.com/document/detail/zh/ModelZoo/pytorchframework/pies/pies_00006.html)
- ais_bench：[安装指导](https://gitee.com/ascend/tools/tree/master/ais-bench_workload/tool/ais_bench)
- 数据集 API：[Omni6DPoseAPI](https://github.com/Omni6DPose/Omni6DPoseAPI)
- 预训练权重：[Dropbox checkpoints](https://www.dropbox.com/scl/fo/x87lhf7sygjm1gasz153g/AIHBlaGMjhfyW1bKrDe61R4?rlkey=y1f6dqdi40tzcgepccthayudp&st=1sbkxbzf&dl=0)

---

## 版本与关键差异

| 项目 | 官方 ModelZoo | 本机（香橙派已验证栈） |
|------|----------------|------------------------|
| CANN | 8.3.RC1 | **8.5.0**（见根 README 第 3 章） |
| 固件与驱动 | 24.1.RC3 | 以 `npu-smi` 显示为准 |
| Python | 3.10.14 | conda 环境 `genpose2` |
| PyTorch | 2.1.0 | 与下表 torch_npu 配套安装 |
| Ascend Extension PyTorch | 2.1.0.post17 | 同上 |
| SoC 型号（OM 转换） | 300I DUO：`Ascend310P3` | **310P RC：`Ascend310P1`** |
| `set_env.sh` | `ascend-toolkit/set_env.sh` | **`/usr/local/Ascend/cann-8.5.0/set_env.sh`** |

说明：支持 Atlas 300I DUO / **310P RC**。本机 `npu-smi` 显示芯片为 **310P1** 时，ONNX→OM 必须使用 `--soc_version Ascend310P1`，不能使用默认的 `Ascend310P3`。

### CANN 8.5.0 对阶段 1（PyTorch NPU）的影响

**会有影响，但 CANN 8.5.0 本身不必卸载。** 驱动、固件、`atc`（ONNX→OM）、`ais_bench`（OM 推理）都随本机 CANN 8.5.0 走即可。需要单独处理的是 **conda 里的 `torch` / `torch_npu` 与 GenPosePlus 的固定版本** 之间的错配。

| 来源 | CANN | 官方配套的 PyTorch / torch_npu |
|------|------|-------------------------------|
| GenPosePlus `requirements.txt` | **8.3.RC1**（文档） | **2.1.0 / 2.1.0.post17** |
| [Ascend/pytorch 版本表](https://github.com/Ascend/pytorch/blob/master/README.zh.md) | **8.5.0**（本机） | **2.6.0.post5、2.7.1.post2、2.8.0.post2、2.9.0**（**不含** 2.1.0.post17） |

结论：

- 若按 CANN 8.5 文档安装**默认推荐**的 torch_npu（2.6+），与 GenPosePlus 锁定的 **2.1.0.post17** 不一致，`pip install -r requirements.txt` 可能被覆盖或运行时报 API/算子不匹配。
- 若强行在 CANN 8.5 上装 **2.1.0.post17**（按 8.3.RC1 安装页提供的 aarch64 wheel），**有可能**通过阶段 1 自检并完成 ONNX 导出，但**非官方配套**，需以实际 `import torch_npu` 与 `export_all_onnx.py` 为准。

**推荐策略（本机 CANN 8.5.0 保持不变）：**

1. **优先**：conda 环境中仍安装 **torch==2.1.0、torch_npu==2.1.0.post17**（参考 [pies_00006 中 8.3.RC1 章节](https://www.hiascend.com/document/detail/zh/ModelZoo/pytorchframework/pies/pies_00006.html) 的 aarch64 安装命令，勿混用 8.5 章节的 2.6+ wheel）。
2. 通过阶段 1 自检后，尽早试跑 `python ../export_all_onnx.py`；若报 CANN/torch_npu 版本或算子错误，再考虑：
   - 在**独立 conda 环境**试 CANN 8.5 配套的 **2.6.0.post5** 并验证导出（GenPose 代码未必兼容，需实测）；
   - 或按香橙派手册评估是否另装 **CANN 8.3.RC1** 专用于 GenPose（改动大，作备选）。

各阶段对 PyTorch 的依赖：

| 阶段 | 主要依赖 | CANN 8.5 说明 |
|------|----------|----------------|
| 阶段 1 自检 | torch_npu | 见上表版本策略 |
| 阶段 4 ONNX 导出 | torch + torch_npu（NPU） | 最易出现版本冲突 |
| 阶段 4 OM 转换 | **atc**（CANN toolkit） | 与 8.5.0 一致即可 |
| 阶段 5 OM 评测 | **ais_bench** + `.om` | 基本不依赖 torch 小版本 |

---

## 插件与驱动准备

| 配套 | 版本 | 环境准备指导 |
|------|------|----------------|
| 固件与驱动 | 24.1.RC3 | [Pytorch 框架推理环境准备](https://www.hiascend.com/document/detail/zh/ModelZoo/pytorchframework/pies) |
| CANN | 8.3.RC1（文档）/ **8.5.0（本机）** | 含 kernels 与 toolkit；需能使用 `atc` |
| Python | 3.10.14 | conda |
| PyTorch | 2.1.0 | 见上文 pies 链接 |
| Ascend Extension PyTorch | 2.1.0.post17 | 同上 |

---

## 阶段 0：基础环境检查

```bash
# 加载 CANN（建议写入 ~/.bashrc）
source /usr/local/Ascend/cann-8.5.0/set_env.sh

# NPU 与驱动
npu-smi info
# 期望：310P1、Health OK

# ATC（ONNX→OM 必需）
which atc && atc --help | head -3

# 可选：算力自检（见根 README 3.3）
echo Y | ascend-dmi -f -d 0 --et 60 -t int8
```

若 `conda` 未在 PATH 中：

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda --version
```

---

## 阶段 1：Conda 环境 + PyTorch NPU + ais_bench

```bash
source /usr/local/Ascend/cann-8.5.0/set_env.sh
source ~/miniconda3/etc/profile.d/conda.sh

conda create -n genpose2 python=3.10.14 -y
conda activate genpose2

# 若驱动/固件与 CANN 不完全匹配，expandable_segments 可能报错，自检失败时可先 unset：
# unset PYTORCH_NPU_ALLOC_CONF
# export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
```

**CANN 初始化所需的 conda 依赖（`.npu()` 前安装，避免逐个缺包）：**

CANN 在首次把张量放上 NPU 时会拉起 TBE/GE，并 import 当前 conda 里的 Python 模块。缺包时表面报 `SetPrecisionMode` / `500001`，日志里实为 `ModuleNotFoundError`。建议一次性安装：

```bash
pip install pyyaml decorator scipy psutil "numpy<2" "attrs==21.4.0"
# 若日志仍提示缺包，按提示 pip install；te 相关还可补：
pip install cloudpickle tornado ml-dtypes
```

说明：`attr` 由 **`attrs`** 包提供（GenPose `requirements.txt` 已写 `attrs==21.4.0`）。conda 用用户 **stephen**、CANN 用 **root** 安装是常见组合，属主警告一般可忽略，见上文「CANN 8.5.0 对阶段 1 的影响」。

**版本安装（CANN 8.5.0 本机 + GenPose 模型要求）：**

- 模型与 `GenPosePlus/requirements.txt` 锁定：**torch==2.1.0、torch_npu==2.1.0.post17**（对应文档 CANN **8.3.RC1**）。
- 本机 CANN 为 **8.5.0** 时，请打开 [pies_00006](https://www.hiascend.com/document/detail/zh/ModelZoo/pytorchframework/pies/pies_00006.html)，使用 **「8.3.RC1 → 安装 PyTorch」** 中的 **aarch64** pip/wheel 命令安装上述版本（不要选 8.5 章节推荐的 2.6+，除非已确认 2.1 无法工作且愿意改测兼容性）。
- 安装前务必已 `source /usr/local/Ascend/cann-8.5.0/set_env.sh`；`/usr/local/Ascend` 下通常**不预置** torch_npu wheel。

安装顺序建议：先 `pip install torch==2.1.0`（及 torchvision/torchaudio 若需要），再装与 **2.1.0.post17** 对应的 `torch_npu` wheel，最后 `pip install -r ../requirements.txt`（避免被其它依赖拉高版本）。

安装后自检（务必先 `source` CANN，建议在 **stephen** 用户下执行，勿 `sudo pip`）：

```bash
source /usr/local/Ascend/cann-8.5.0/set_env.sh
conda activate genpose2
unset PYTORCH_NPU_ALLOC_CONF   # 若出现 expandable_segments / driver 不匹配告警

python - <<'PY'
import torch, torch_npu
print("torch:", torch.__version__)
print("torch_npu:", torch_npu.__version__)
print("npu available:", torch.npu.is_available())
x = torch.randn(2, 2).npu()
print("tensor on npu:", x.device)
print("ok")
PY
```

通过标准：最后一行打印 `tensor on npu: npu:0` 与 `ok`。

安装 **ais_bench**（OM 推理必需，`om_wrappers.py` 使用 `InferSession`）：

```bash
pip install ais-bench

python -c "from ais_bench.infer.interface import InferSession; print('ais_bench OK')"
```

---

## 阶段 2：获取源码并打补丁

建议工作目录（可自定义）：

```bash
export WORK=~/genposeplus
mkdir -p "$WORK" && cd "$WORK"

git clone https://gitcode.com/Ascend/ModelZoo-PyTorch.git
cd ModelZoo-PyTorch/ACL_PyTorch/built-in/embodied_ai/GenPosePlus
export PYTHONPATH=$PWD:$PYTHONPATH

git clone https://github.com/Omni6DPose/GenPose2.git
cd GenPose2
export PYTHONPATH=$PWD:$PYTHONPATH

git reset --hard d0993c0

# 补丁只能执行一次；重复执行会报错
git apply ../diff.patch
export GPPATH=$PWD

source /usr/local/Ascend/cann-8.5.0/set_env.sh
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True

pip install -r requirements.txt
pip install -r ../requirements.txt
```

说明：`pip install -r ../requirements.txt` 在部分仓库快照中**可能不存在**（`embodied_ai/requirements.txt` 缺失），若报错可忽略，以 `GenPose2/requirements.txt` 与 `GenPosePlus/requirements.txt` 为主。

`GenPosePlus/requirements.txt` 主要依赖：

```
torch==2.1.0
torch_npu==2.1.0.post17
torchvision==0.16.0
torchaudio==2.1.0
onnx==1.20.1
...
```

---

## 阶段 3：数据集、Meta 与权重

### 3.1 Omni6DPose 样本（ROPE / 000000）

按 [Omni6DPoseAPI](https://github.com/Omni6DPose/Omni6DPoseAPI) 下载 **ROPE** 分类下的 **`000000`**，在 `$GPPATH`（GenPose2 根目录）下组织为：

```text
GenPose2/
└── omni6dpose-000000/
    └── ROPE/
        └── 000000/
            ├── 000000_color.png
            ├── 000000_depth.exr
            ├── 000000_mask.exr
            ├── 000000_mask_sam.npz
            ├── 000000_meta.json
            ...
```

### 3.2 Meta 配置

将数据集站点 **ROPE 同级 `Meta/`** 下内容复制到 `$GPPATH/configs/`：

```text
GenPose2/configs/
├── obj_meta.json
├── real_obj_meta.json
└── config.py
```

### 3.3 预训练权重

从 [checkpoints](https://www.dropbox.com/scl/fo/x87lhf7sygjm1gasz153g/AIHBlaGMjhfyW1bKrDe61R4?rlkey=y1f6dqdi40tzcgepccthayudp&st=1sbkxbzf&dl=0) 下载至 `$GPPATH/results/ckpts/`：

```text
GenPose2/results/ckpts/
├── ScoreNet/scorenet.pth
├── EnergyNet/energynet.pth
└── ScaleNet/scalenet.pth
```

快速检查：

```bash
cd "$GPPATH"
test -f results/ckpts/ScoreNet/scorenet.pth && \
test -d omni6dpose-000000/ROPE/000000 && \
test -f configs/config.py && echo "数据与权重就绪"
```

---

## 阶段 4：导出 ONNX → OM

```bash
conda activate genpose2
source /usr/local/Ascend/cann-8.5.0/set_env.sh
cd "$GPPATH"
export PYTHONPATH=$PWD:$PYTHONPATH
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True

# Step A: PyTorch → ONNX（耗时较长）
python ../export_all_onnx.py

# Step B: ONNX → OM —— 310P RC 必须使用 Ascend310P1
python ../export_all_om.py --soc_version Ascend310P1

# 检查 6 个 OM
ls -lh om_models/*.om
```

期望生成的 OM 文件：

- `pointnet2_from_score.om`
- `pointnet2_from_energy.om`
- `scorenet.om`
- `energynet.om`
- `scalenet.om`
- `dinov2_vits14.om`

若 `which atc` 失败，说明 CANN toolkit 未装全或未 `source set_env.sh`。

---

## 阶段 5：Single 模式评测（推荐先做）

```bash
cd "$GPPATH"

# 清掉上次缓存（否则可能读到旧结果）
rm -rf results/evaluation_results

# 香橙派内存紧张时，可将脚本内 --num_worker 32 改为 4 或 8
bash scripts/eval_single_om.sh
```

`eval_single_om.sh` 使用 `om_models/*.om`、`--data_path omni6dpose-000000/ROPE/`、`--batch_size 16`、`--device npu:0`。

### 验收标准（310P RC）

| 指标 | 参考值 |
|------|--------|
| `iou_mean` | **≈ 0.3073** |
| 性能 | **≈ 803 ms/sample**（single，batch 16） |

在日志中搜索 `iou` / `iou_mean` 对照。偏差较大时检查：`Ascend310P1`、权重与数据路径、是否已删除 `evaluation_results`。

---

## 阶段 6（可选）：Tracking 模式

```bash
cd "$GPPATH"
rm -rf results/evaluation_results onnx_models om_models

python ../export_all_onnx.py --batch_size 4
python ../export_all_om.py --batch_size 4 --soc_version Ascend310P1

bash scripts/eval_tracking_om.sh
```

| 芯片 | 模式 | batch | iou_mean 参考 | 性能参考 (ms/sample) |
|------|------|-------|---------------|----------------------|
| 300I DUO | single | 16 | 0.3073 | 652.20 |
| 300I DUO | tracking | 4 | 0.3038 | 458.37 |
| **310P RC** | single | 16 | **0.3073** | **803.23** |
| **310P RC** | tracking | 4 | **0.3036** | **488.15** |

**注意：** 非首次推理前须删除缓存，例如：

```bash
rm -rf "$GPPATH/results/evaluation_results"
```

---

## 最小验证路径

按顺序完成即可确认环境可用：

1. 阶段 0 → 阶段 1（`torch_npu` + `ais_bench` 自检）
2. 阶段 2（补丁应用成功）
3. 阶段 3（数据与权重就位）
4. 阶段 4（生成 6 个 `.om`）
5. 阶段 5（`eval_single_om.sh`，`iou_mean ≈ 0.3073`）

---

## 常见问题

1. **`git apply ../diff.patch` 失败**  
   需在干净 checkout 的 `d0993c0` 上执行；已打过补丁不要重复 apply。

2. **CANN 8.5 vs 文档 8.3（阶段 1）**  
   CANN 8.5.0 可保留；**torch_npu 仍建议按 8.3.RC1 文档装 2.1.0.post17** 以满足 GenPosePlus。若 `import torch_npu` 或 `export_all_onnx.py` 报版本/算子错误，见上文「CANN 8.5.0 对阶段 1 的影响」中的备选策略，勿在未实测前直接把 torch 升到 8.5 默认的 2.6+ 并覆盖 `requirements.txt`。

3. **SoC 版本**  
   - Atlas 300I DUO：`Ascend310P3`（`export_all_om.py` 默认值）  
   - **310P RC：`Ascend310P1`（必显式指定）**

4. **`../requirements.txt` 不存在**  
   仓库中可能无 `embodied_ai/requirements.txt`，忽略该 pip 步骤即可。

5. **磁盘不足**  
   数据集 + ONNX + OM 占用较大。根分区紧张时将 `WORK`、`TMPDIR` 指到 NVMe 数据盘（见根 README 第 2 章）。

6. **GitHub 克隆慢**  
   `GenPose2` 可在 PC 下载后 `scp` 到板子，或使用镜像。

7. **`ascend-dmi` 交互**  
   非交互测试用 `echo Y | ascend-dmi ...`；测试结束回到 shell 后**不要**再单独输入 `Y`（见根 README 3.3）。

---

## 模型概述（摘自官方）

GenPose++ 采用分段点云与裁剪 RGB 作为输入：PointNet++ 提取几何特征，DINO v2 提取语义特征，融合后作为扩散模型条件生成姿态候选与能量；对非连续对称物体通过聚类处理多模态分布。

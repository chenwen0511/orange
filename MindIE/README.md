# MindIE 使用方法（香橙派 AI Station）

本文档整理自《OrangePi AI Station 昇腾 用户手册》**v0.2** 第 **4.1 节**，并补充本机 openEuler 上的磁盘与 Git LFS 踩坑记录。通用 CANN / 驱动安装见根目录 [README.md](../README.md) 第 3 章；具身智能 **π0** 见 [pai0/README.md](../pai0/README.md)（手册 **4.2.1**）。

**权威原文：** `OrangePi_AI_Station_昇腾_用户手册_v0.2.pdf` → **4. AI 应用样例 → 4.1. MindIE 使用方法**

---

## 4.1.1 下载并启动 Docker 镜像

### 重要说明（手册原文）

- **OPi AI Station 只能使用香橙派提供的 MindIE 镜像**；昇腾社区 MindIE 镜像在板子上测试问题较多，**无法直接使用**。
- 使用前请将**驱动更新到最新版本**（手册 **2.8 更新驱动包的方法**），以获得最佳性能。

### 步骤

1. 在 [OPi AI Station 资料下载页](http://www.orangepi.cn/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-Pi-AI-Station.html) → **官方资料** → **`mindie` 文件夹**，下载打包压缩包并上传到开发板。

2. 解压、校验并导入镜像：

```bash
tar -xf mindie-2.3.0.tar.zst
cd mindie-2.3.0
md5sum -c mindie-2.3.0-opiaistation-py310-ubuntu22.04-aarch64.tar.md5sum
# 期望输出 OK

docker load < mindie-2.3.0-opiaistation-py310-ubuntu22.04-aarch64.tar
```

3. 若无 `docker` 命令：

```bash
dnf install -y docker
```

4. 导入成功后检查（手册示例）：

```text
REPOSITORY   TAG                                          IMAGE ID       CREATED        SIZE
mindie       2.3.0-opiaistation-py310-ubuntu22.04-aarch64  39adbaf1622e   ...            13.3GB
```

5. 启动容器（使用压缩包内脚本）：

```bash
./startdocker-mindie.sh
docker ps -a
# 容器名一般为 mindie，STATUS 为 Up
```

6. 进入容器（**后续除调用 MindIE 接口外，均在容器内操作**）：

```bash
docker exec -it mindie bash
```

### 本机补充：磁盘空间（`docker load` 失败）

手册镜像约 **13GB+**，`docker load` 解压写入 `/var/lib/docker`，根分区需预留足够空间。若报错：

```text
no space left on device
```

请先 `df -h`、`docker system df`，删除无用镜像后 `docker system prune -a -f`，建议 **`/` 可用 ≥ 30GB** 再 load。本机一次清理后约 **35G 可用** 可重试。可选将 Docker `data-root` 迁到大容量分区（见下文「附录」）。

---

## 4.1.2 MindIE 在 OPi AI Station 上的模型支持情况

### 4.1.2.1 已测试通过的大语言模型

| 模型名称 | 48G 版本 | 96G 版本 | 服务化接口 |
|----------|----------|----------|------------|
| Qwen3-4B | OK | OK | OK |
| Qwen3-4B-W8A8 | OK | OK | OK |
| Qwen3-8B | NO | OK | OK |
| Qwen3-8B-W8A8 | OK | OK | OK |
| Qwen3-8B-W8A8SC | OK | OK | OK |
| DeepSeek-R1-Distill-Qwen-7B | NO | OK | OK |
| DeepSeek-R1-Distill-Qwen-7B-W8A8 | OK | OK | OK |
| DeepSeek-R1-Distill-Qwen-14B-W8A8 | NO | OK | OK |

### 4.1.2.2 已测试通过的多模态理解模型

| 模型名称 | 48G 版本 | 96G 版本 | 服务化接口 |
|----------|----------|----------|------------|
| Qwen2.5-VL-7B-Instruct | NO | OK | NO |
| Qwen2.5-VL-7B-Instruct-W8A8 | OK | OK | NO |

---

## 4.1.3 推理性能参考数据

手册说明：性能受 host 侧影响，实际调用还有后处理等因素。

| 模型 | 并行推理 | TTFT | OutputTokenThroughput |
|------|----------|------|------------------------|
| Qwen3-4B-w8a8 | 是 | 566.448 ms | 23.2174 token/s |
| Qwen3-8B-w8a8 | 是 | 745.5691 ms | 17.0344 token/s |
| qwen3-8b-w8a8sc | 是 | 753.7554 ms | 21.728 token/s |
| qwen2.5-vl-w8a8 | 否 | 556.9692 ms | 12.2136 token/s |

---

## 4.1.4 模型下载

1. 访问香橙派魔乐社区主页，按 **4.1.2** 支持列表选择模型（示例 **Qwen3-4B-W8A8**）：  
   https://modelers.cn/user/XLRJ

2. 在开发板（或 MindIE 容器内）下载：

```bash
docker exec -it mindie bash
cd /models
git clone https://modelers.cn/XLRJ/Qwen3-4B-W8A8.git
# 其他模型：替换 URL 中的仓库名即可
```

### 附录：openEuler 安装 Git LFS

Modelers 大文件仓库需 **Git LFS**。openEuler 22 **无 `dnf install git-lfs`**，packagecloud 脚本 **404**。本机可用二进制安装：

```bash
cd /tmp
wget https://github.com/git-lfs/git-lfs/releases/download/v3.6.1/git-lfs-linux-arm64-v3.6.1.tar.gz
tar xzf git-lfs-linux-arm64-v3.6.1.tar.gz
cd git-lfs-3.6.1 && sudo ./install.sh
git lfs install
git lfs version
```

---

## 4.1.5 MindIE Server 的使用方法

**注意：** 并非所有模型都支持 MindIE Server；以下为 **LLM 通用配置**，细节见模型支持列表与 ATB Models 服务化文档。

### 4.1.5.1 基础配置及使用

1. 在容器内进入 `mindie-service`，编辑 `conf/config.json`（**必改项**，其余见官方文档）：

```bash
cd /usr/local/Ascend/mindie/latest/mindie-service
vim conf/config.json
```

| 配置项 | 说明 | 单卡 310P 示例 |
|--------|------|----------------|
| `httpsEnabled` | 设为 `false`，否则需 HTTPS 证书 | `false` |
| `npuDeviceIds` | NPU 设备 ID | `[[0]]` |
| `modelName` | 调用服务时使用的模型名 | `"Qwen3-4B-W8A8"` |
| `modelWeightPath` | 权重路径，与推理一致 | `"/models/Qwen3-4B-W8A8"` |
| `worldSize` | 并行卡数 | `1` |
| `trustRemoteCode` | 信任远程代码 | `true` |

**单卡 310P 六项必改（缺一即失败，本机已验证）：**

```json
"httpsEnabled": false,
"npuDeviceIds": [[0]],
"modelName": "Qwen3-4B-W8A8",
"modelWeightPath": "/models/Qwen3-4B-W8A8",
"worldSize": 1,
"trustRemoteCode": true
```

- 须写在 `conf/config.json` 里**正确层级**（`BackendConfig` / `ModelDeployConfig` 等，以模板为准），且 **`worldSize` 与 `npuDeviceIds` 必须成对**：单卡为 `1` + `[[0]]`。
- JSON 语法须用**英文逗号** `,`，勿用中文逗号 `，`；布尔值为 `false` / `true`，勿加中文标点。
- 改完后可用：`python3 -m json.tool conf/config.json` 检查是否能解析。

2. 修改模型目录下 `config.json` 权限，否则无法启动 Server：

```bash
chmod 640 /models/Qwen3-4B-W8A8/config.json
```

3. 启动服务，出现 **`Daemon start success!`** 即成功：

```bash
./bin/mindieservice_daemon
```

**常见启动失败：`The size of npuDeviceIds (subset) does not equal to worldSize`**

- 含义：每个模型实例在 `npuDeviceIds` 中分配到的 **NPU 个数** 必须等于 **`worldSize`**（单卡推理均为 **1**）。
- 错误示例：`worldSize: 1` 但保留模板默认 `npuDeviceIds: [[0,1,2,3]]`（子集长度为 4）；或 `worldSize: 4` 但只有 `[[0]]`。
- **310P 单卡正确写法**（在 `BackendConfig` / `ModelDeployConfig` 中与实际字段位置一致）：

```json
"worldSize": 1,
"npuDeviceIds": [[0]],
"modelInstanceNumber": 1
```

- 注意：`npuDeviceIds` 是 **二维数组** `[[0]]`，不是 `[0]`。
- 修改后保存的是日志里加载的路径（例：`/usr/local/Ascend/mindie/2.3.0/mindie-service/conf/config.json`，与 `latest`  symlink 通常一致）。
- 校验：`python3 -c "import json; c=json.load(open('conf/config.json')); print(c)"` 或 `grep -E 'worldSize|npuDeviceIds' conf/config.json`。
- 末尾 **`Killed`**：多为配置初始化失败后进程退出，或内存不足；先修好上述配置再试。仍 `Killed` 时用 `dmesg | tail` 看是否 OOM。

**常见启动失败：`RuntimeError: Warmup failed`（显存 / KV Cache）**

Warmup 会按 **`ModelDeployConfig` 里的最大序列、Prefill 上限** 预分配 KV Cache。模板默认 `maxSeqLen=2560`、`maxInputTokenLen=2048`、`maxPrefillTokens=8192`、`maxPrefillBatchSize=50` 在 **单卡 310P** 上容易把 NPU 撑爆（即使 `npuMemSize: -1` 自动分配也会失败）。

**处理顺序（建议逐项试）：**

1. **先缩小 `ModelDeployConfig`（与 `ModelConfig` 数组同级，不是写在 `ModelConfig[]` 里面）：**

```json
"ModelDeployConfig": {
  "maxSeqLen": 1024,
  "maxInputTokenLen": 512,
  "maxPrefillTokens": 1024,
  "maxPrefillBatchSize": 8,
  "maxBatchSize": 16,
  "truncation": true,
  "ModelConfig": [
    {
      "modelInstanceType": "Standard",
      "modelName": "Qwen3-4B-W8A8",
      "modelWeightPath": "/models/Qwen3-4B-W8A8",
      "worldSize": 1,
      "cpuMemSize": 5,
      "npuMemSize": -1,
      "backendType": "atb",
      "trustRemoteCode": true
    }
  ]
}
```

约束：`maxPrefillTokens` ≥ `maxInputTokenLen`；`maxInputTokenLen` ≤ `maxSeqLen - 1`。仍失败则继续把三者减半（如 512 / 256 / 512）。

2. **启动前提高 NPU 可用于 KV 的比例（容器内）：**

```bash
export NPU_MEMORY_FRACTION=0.85
./bin/mindieservice_daemon
```

若权重加载 OOM，改回 `0.8` 或略降；勿盲目设 `1.0`。

3. **仍失败时改用手动 `npuMemSize`（GB）**，替代 `-1`。本机 **Warmup 通过** 的配置为 **`npuMemSize: 8`**、`cpuMemSize: 5`（见下文实测）。

4. **单卡无法**通过增大 `worldSize` 解决（只有一张 310P）。

`ModelConfig` 中 **`cpuMemSize: 0`** 易导致异常，建议 **`5`**（手册默认）。

5. **新终端**测试（未改 `ipAddress` 时仅本机可访问）：

```bash
curl -w "\ntime_total=%{time_total}\n" \
  -H "Accept: application/json" \
  -H "Content-type: application/json" \
  -X POST -d '{
    "inputs": "I love Beijing, because",
    "stream": false
  }' http://127.0.0.1:1025/generate
```

### 4.1.5.2 开启并行推理功能

- 仅部分 **LLaMA3 / Qwen2 / Qwen3** 支持；不支持的模型用并行配置会导致服务无法启动。
- 并行解码场景**暂不支持流式推理**；惩罚类后处理仅支持重复惩罚；`lookahead` 与 `memory_decoding` **不可同时使能**。

**适用前提（手册摘要）：** 并发不高、内存带宽受限且算力有冗余；输入较长；输出需一定长度才有收益。

| 算法 | 候选 token 方式 | 适用场景 |
|------|-----------------|----------|
| memory_decoding | trie-tree 缓存历史 I/O | 代码生成、检索 |
| lookahead | Jacobi 迭代 + Prompt/输出 | 文本生成、对话 |

**memory_decoding 配置示例**（在 `config.json` 的 `ModelDeployConfig` 中）：

```json
"maxSeqLen": 2560,
"maxInputTokenLen": 2048,
"truncation": false,
"speculationGamma": 16,
"ModelConfig": [{
  "plugin_params": "{\"plugin_type\":\"memory_decoding\",\"decoding_length\":16,\"dynamic_algo\":true}",
  "modelInstanceType": "Standard"
}]
```

**lookahead 配置示例：**

```json
"speculationGamma": 30,
"plugin_params": "{\"plugin_type\":\"la\",\"level\":4,\"window\":5,\"guess_set_size\":5}"
```

参数含义见昇腾 MindIE / ATB 官方文档。

---

## 香橙派 310P 实测结果（MindIE 2.3 + Qwen3-4B-W8A8）

完整日志见 [`a.log`](a.log)。**2026-05-19** 在 MindIE 容器内启动 **MindIE Server** 并做接口探测。

### 验收结论

| 项 | 结果 |
|----|------|
| 服务启动 | **`Daemon start success!`** |
| 模型 | **Qwen3-4B-W8A8**，`worldSize: 1`，NPU **0** |
| 权重加载 | `Loading selected layers: 36/36`（约 2s） |
| 配置管理 | `Successfully init config manager` |
| 手册 curl 探测 | 已执行，见下文说明 |

**结论：MindIE Server 在香橙派 310P 单卡上已成功拉起**；此前需先解决 `npuDeviceIds`/`worldSize` 对齐与 **Warmup 显存** 问题。

### 本机生效的 `ModelConfig`（摘录）

```json
"modelName": "Qwen3-4B-W8A8",
"modelWeightPath": "/models/Qwen3-4B-W8A8",
"worldSize": 1,
"cpuMemSize": 5,
"npuMemSize": 8,
"backendType": "atb",
"trustRemoteCode": true
```

另需保证全局六项：`httpsEnabled: false`、`npuDeviceIds: [[0]]` 等（见 **4.1.5.1**）。若仍 Warmup 失败，再配合缩小 `ModelDeployConfig` 中的 `maxSeqLen` / `maxPrefillTokens`。

### 启动日志要点（`a.log`）

```text
ConfigManager: Load Config from .../mindie/2.3.0/mindie-service/conf/config.json
Successfully init config manager
Loading selected layers: 100%|...| 36/36 [00:02<00:00, 16.65layer/s]
Daemon start success!
```

启动后进程（摘录）：

- `./bin/mindieservice_daemon`（主守护进程）
- `mindie_llm_backend --local_rank 0 --npu_device_id 0`（单卡后端）
- 多个 `python3.10` tokenizer / forkserver 子进程（正常）

### curl 测试（`a.log`）

```bash
curl -w "\ntime_total=%{time_total}\n" \
  -H "Accept: application/json" \
  -H "Content-type: application/json" \
  -X POST -d '{"inputs": "I love Beijing, because", "stream": false}' \
  http://127.0.0.1:1025/generate
```

日志仅记录：

```text
time_total=0.000985
```

**分析：**

- **`time_total ≈ 1ms` 过短**，手册成功样例约 **1.2s** 且带 `"generated_text":"..."`。本次日志**未出现 JSON 正文**，更像是连接未进到推理（服务未就绪、端口未监听、或请求在宿主机而服务在容器内等）。
- **建议在服务打印 `Daemon start success!` 之后**，于**同一网络命名空间**（容器内或已映射端口）再执行 curl，并检查完整响应体，例如：

```bash
curl -s -w "\ntime_total=%{time_total}\n" \
  -H "Content-Type: application/json" \
  -X POST -d '{"inputs":"I love Beijing, because","stream":false}' \
  http://127.0.0.1:1025/generate | head -c 500
```

期望含 `"generated_text"`；`time_total` 在百毫秒～秒级属正常。

### 可忽略的 WARN（不影响启动）

| 日志 | 说明 |
|------|------|
| `MINDIE_LLM_* will be deprecated` | 环境变量更名提示 |
| `libdcmi.so security check faild` | 容器内未完整挂载驱动 profiling 库，仅影响 NPU 占用统计 |
| `speculationGamma is not found, use default` | 未开并行解码时可忽略 |

### 问题与对策对照（本机路径）

| 阶段 | 现象 | 处理 |
|------|------|------|
| 配置 | `npuDeviceIds` 与 `worldSize` 不一致 | 单卡：`worldSize: 1`，`npuDeviceIds: [[0]]` |
| Warmup | `RuntimeError: Warmup failed` | `cpuMemSize: 5`，`npuMemSize: 8`；必要时缩小 `maxSeqLen` / `maxPrefillTokens` |
| 启动 | `Daemon start success!` | 服务可用，再测 curl / API |

### 与手册性能参考

手册 **4.1.3** 给出 Qwen3-4B-w8a8 参考：**TTFT ≈ 566 ms**，**≈ 23.2 token/s**。本日志未包含完整推理耗时；待 curl 返回 `generated_text` 后，可用同一条命令对比 `time_total` 与吞吐。

---

## 与其它章节的关系

| 主题 | 手册章节 | 本仓库 |
|------|----------|--------|
| 驱动 / CANN | 2.8 / 2.9 | [README.md](../README.md) 第 3 章 |
| π0 具身模型 | 4.2.1 | [pai0/README.md](../pai0/README.md) |
| GenPose OM 推理 | —（不走 MindIE） | [genpose2/README.md](../genpose2/README.md) |

---

## 附录：本机运维命令摘录

```bash
# 磁盘
df -h
docker system df
docker system prune -a -f

# Docker 数据目录迁移（可选）
systemctl stop docker
mkdir -p /data/docker
# 编辑 /etc/docker/daemon.json → "data-root": "/data/docker"
systemctl start docker
```

---

## 参考

- 香橙派：`OrangePi_AI_Station_昇腾_用户手册_v0.2.pdf` **§4.1**
- 资料下载：http://www.orangepi.cn/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-Pi-AI-Station.html
- 魔乐社区模型：https://modelers.cn/user/XLRJ
- [MindIE 是什么（昇腾社区）](https://www.hiascend.com/document/detail/zh/mindie/10RC3/whatismindie/mindie_what_0001.html)
- [Git LFS releases（arm64）](https://github.com/git-lfs/git-lfs/releases)

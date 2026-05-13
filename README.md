# Orange

本文档按章节整理项目说明与环境配置，后续内容将在此陆续补充。

## 目录

1. [第 1 章 开发环境](#第-1-章-开发环境)
2. [第 2 章 NVMe 系统盘分区：查看与操作](#第-2-章-nvme-系统盘分区查看与操作)
3. [第 3 章 昇腾 CANN 安装与环境验证](#第-3-章-昇腾-cann-安装与环境验证)

---

## 第 1 章 开发环境

本章记录工具链与运行环境的搭建步骤（含香橙派 / Linux aarch64 等场景）。

### 1.1 Miniconda 安装（香橙派 / Linux aarch64）

以下流程在 openEuler、香橙派（ARM64）上验证通过；若官方源较慢，可使用清华镜像下载安装脚本。

#### 步骤 1：下载安装脚本

```bash
curl -fsSL -o Miniconda3.sh \
  https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-aarch64.sh
```

x86_64 机器请改用：`Miniconda3-latest-Linux-x86_64.sh`（路径同上，仅文件名不同）。

#### 步骤 2：校验文件（可选）

```bash
ls -lh Miniconda3.sh
head -c 200 Miniconda3.sh | xxd | head
```

安装包体积约一百多 MB，文件开头应为 `#!/bin/sh`，且头部注释中包含 `Miniconda3`、`linux-aarch64` 等信息。

#### 步骤 3：静默安装到用户目录

```bash
bash Miniconda3.sh -b -p "$HOME/miniconda3"
```

安装结束时若提示已设置 `PYTHONPATH`，可能与 conda 自带 Python 冲突；需要时请检查 `~/.bashrc` 等处的 `PYTHONPATH`，仅保留与当前解释器兼容的路径。

#### 步骤 4：初始化 Bash 并生效

```bash
"$HOME/miniconda3/bin/conda" init bash
source ~/.bashrc
```

使用 zsh 时请将 `init bash` 改为 `init zsh`，并执行 `source ~/.zshrc`。

#### 步骤 5：验证

```bash
conda --version
conda env list
```

命令提示符前出现 `(base)` 即表示默认环境已激活。

#### 步骤 6：清理（可选）

```bash
rm Miniconda3.sh
```

---

## 第 2 章 NVMe 系统盘分区：查看与操作

本章说明在新系统、新 NVMe SSD 作为系统盘时，如何**查看**磁盘与分区，以及在常见场景下如何**扩充分区 / 文件系统**（以 openEuler aarch64 为例，其它发行版命令基本一致）。

### 2.1 常用查看命令

先分清三件事：**物理盘**、**分区**、**文件系统与挂载点**。

| 目的 | 命令 | 说明 |
|------|------|------|
| 树状列出块设备及挂载关系 | `lsblk -f` | 可看 `FSTYPE`、`LABEL`、`UUID`、`MOUNTPOINT`、`FSAVAIL` |
| 只看磁盘与分区容量 | `lsblk` | 不配 `-f` 时更简洁 |
| 分区表与扇区范围 | `sudo fdisk -l /dev/nvme0n1` | GPT/MBR、每个分区的 `Start`/`End`、`Size` |
| 图形化分区信息（含空闲） | `sudo parted /dev/nvme0n1 print free` | 单位可用 `unit MiB` 等 |
| 各挂载点空间使用 | `df -h` | 重点看 `/` 的 `Size`/`Avail`/`Use%` |
| Inode 是否耗尽（少见） | `df -i /` | `IUse%` 100% 时也会“空间不足” |

设备名一般为 **`/dev/nvme0n1`**（第一块 NVMe），其第一个分区多为 **`/dev/nvme0n1p1`**。若系统把根设备显示为 `/dev/root`，多为内核符号链接或 initramfs 中的命名，与真实分区对应关系仍以 `lsblk` 为准。

### 2.2 “新盘新系统”分区一般是谁做的

- **使用官方镜像安装系统时**：分区通常在安装向导或 kickseed/autoinstall 里已经建好，装完开机即可用。
- **若镜像只划了很小的根分区**（例如整盘 128G，但 `/` 只有十几 G）：表现为 `fdisk -l` 里磁盘总容量很大，而第一个分区只占前面一小段，后面是 **未分配空间**。这时不必重装，可按 **2.4 节** 扩充分区与文件系统。

### 2.3 如何理解分区是否“占满整盘”

同时看两条输出即可：

```bash
lsblk -f
sudo fdisk -l /dev/nvme0n1
```

- 若 **`nvme0n1` 总容量** 远大于 **`nvme0n1p1` 分区 Size**，说明还有空间未划入分区。
- 若 **`df -h /`** 里 **`Avail` 为 0 且 `Use%` 100%**，可能是分区已满；也可能是分区很小——需结合上面两点判断。

### 2.4 扩充分区与 ext4 根分区在线扩容

适用：**根分区在 `/dev/nvme0n1p1`、文件系统为 ext4、磁盘末尾存在未分配空间**。

1. **尽量先腾出少量根分区空间**（删缓存、半截安装目录等），避免在 100% 满盘时操作极端边界情况。
2. **安装 growpart**（openEuler）：

   ```bash
   sudo dnf install -y cloud-utils-growpart
   ```

3. **扩展第 1 个分区到磁盘末尾**，然后 **在线扩容文件系统**：

   ```bash
   sudo growpart /dev/nvme0n1 1
   sudo resize2fs /dev/nvme0n1p1
   ```

4. **验证**：

   ```bash
   df -h /
   lsblk
   ```

若未安装 `growpart`，可用 `parted` 调整分区大小后再 `resize2fs`，具体以当前系统自带工具为准；扩分区前建议查阅发行版文档并做好备份。

### 2.5 GPT 相关告警（可选了解）

若 `fdisk` 提示 **GPT PMBR size mismatch**、**backup GPT table is not on the end of the device**，多与“分区表记录的大小与实际磁盘不一致”有关；在正确扩充分区并同步分区表后，部分机器上告警会消失。涉及 **`sgdisk -e`** 等修复命令前务必阅读厂商文档并备份，避免误操作损坏分区表。

### 2.6 新建数据分区并挂载（可选）

若你希望单独划分 **`/dev/nvme0n1p2`** 存放数据（与缩小/移动根分区相比，通常更简单的是：**根分区扩满整盘**，再在盘上新建分区——取决于你是否愿意保留未分配空间）：

1. 使用 `fdisk` / `parted` / `gdisk` 在未分配空间上新建分区。
2. `sudo mkfs.ext4 -L 卷标 /dev/nvme0n1p2`
3. `sudo mkdir -p /data`，用 **`blkid`** 取 UUID，在 **`/etc/fstab`** 增加一行挂载（建议 `nofail`），再 `sudo mount -a`。

本章仅作常用操作备忘；生产环境重大变更前请优先遵循 **OrangePi / openEuler / 昇腾** 官方手册。

---

## 第 3 章 昇腾 CANN 安装与环境验证

本章记录本机一次**安装成功**后的安装器提示、环境变量配置与 **`ascend-dmi`** 压测输出，便于对照《OrangePi_AI_Station_昇腾_用户手册》2.9 等章节排查问题。版本以当时安装包为准：**CANN / 310p ops 8.5.0**。

### 3.1 安装器结束时的提示（摘录）

```
Driver:    Installed in /usr/local/Ascend/driver.
ops_310p:  Ascend-cann-310p-ops_8.5.0_linux-aarch64 install success, installed in /usr/local/Ascend.

Please make sure that the environment variables have been configured.
-  To take effect for current user, you can exec command below: source /usr/local/Ascend/cann-8.5.0/set_env.sh or add "source /usr/local/Ascend/cann-8.5.0/set_env.sh" to ~/.bashrc.
```

### 3.2 环境变量（当前会话 / 持久化）

当前 shell 立即生效：

```bash
source /usr/local/Ascend/cann-8.5.0/set_env.sh
```

持久化：将同一行 `source /usr/local/Ascend/cann-8.5.0/set_env.sh` 追加到 `~/.bashrc`（或其它登录 shell 的配置文件），重新登录或 `source ~/.bashrc` 后生效。

### 3.3 ascend-dmi 验证（命令与输出记录）

说明：`ascend-dmi` 会提示是否继续（Y/N）。非交互场景可用管道自动确认；**不要在测试已结束后再单独输入 `Y`**，否则 shell 会把 `Y` 当作命令执行，出现 `bash: Y: command not found`。

非交互示例：

```bash
echo Y | ascend-dmi -f -d 0 --et 60 -t int8
echo Y | ascend-dmi -f -d 0 --et 60 -t fp16
```

本机一次成功运行的终端输出（摘录）：

```
This test will affect the business on this server. To ensure the correctness and accuracy of the test, perform the operation separately.Do you want to continue?(Y/N)Y
Y
-----------------------------------------------------------------------------------------
    Device        Execute Times        Duration(ms)        TOPS@INT8          Power(W)
-----------------------------------------------------------------------------------------
    0             600,000,000          7111                176.950            85.4000015
-----------------------------------------------------------------------------------------

This test will affect the business on this server. To ensure the correctness and accuracy of the test, perform the operation separately.Do you want to continue?(Y/N)Y
-----------------------------------------------------------------------------------------
    Device        Execute Times        Duration(ms)        TFLOPS@FP16          Power(W)
-----------------------------------------------------------------------------------------
    0             600,000,000          14222               88.475             85.1999969
-----------------------------------------------------------------------------------------
```

其中在 `(Y/N)Y` 之后若再出现单独一行 `Y`，多为工具从标准输入再读一次时的显示，一般可忽略。**测试已返回 shell 提示符后，不要再单独输入 `Y`**，否则会被当作命令名，出现 `bash: Y: command not found`。

### 3.4 与本仓库其它章节的关联

- **根分区过小**会导致 CANN 安装阶段出现 `No space left on device` 或解压失败；可先完成 [第 2 章](#第-2-章-nvme-系统盘分区查看与操作) 的扩容，再安装 toolkit / ops。
- 安装大体积 `.run` 包时，若临时目录落在空间紧张的分区，可按安装器日志提示设置 `export TMPDIR=...`（例如指向空间充足的 tmpfs 或数据目录）。

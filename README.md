# Orange

本文档按章节整理项目说明与环境配置，后续内容将在此陆续补充。

## 目录

1. [第 1 章 开发环境](#第-1-章-开发环境)
2. （待补充）

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

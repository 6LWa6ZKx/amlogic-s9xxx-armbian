# Armbian 构建流程详解

## 📋 完整执行链路

```
用户触发 Workflow
    ↓
build-armbian-arm64-server-image.yml (GitHub Actions Workflow)
    ↓
┌─────────────────────────────────────────────────────────────────┐
│ 步骤 1: Compile Armbian (第182-194行)                            │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ 执行: ./compile.sh (Armbian 官方编译脚本)                    │ │
│ │ 输入: RELEASE=jammy, BOARD=odroidn2                          │ │
│ │ 输出: Armbian_*-trunk_*_arm64_*.img.gz (通用 ARM64 镜像)    │ │
│ └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────────┐
│ 步骤 2: Rebuild Armbian (第223-235行)                            │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ 调用: uses: 6LWa6ZKx/amlogic-s9xxx-armbian@main            │ │
│ │       ↓                                                       │ │
│ │ 执行: action.yml (第121-233行)                               │ │
│ │       ↓                                                       │ │
│ │ 核心命令 (action.yml 第217行):                               │ │
│ │ sudo ./rebuild -b s905x3 -r 6LWa6ZKx/... -k 6.6.y ...      │ │
│ │       ↓                                                       │ │
│ │ 调用: rebuild 脚本 (仓库根目录)                              │ │
│ └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔍 Rebuild 步骤详细解析

### 1️⃣ **action.yml 做了什么**

当你在 workflow 中写：
```yaml
- name: Rebuild Armbian
  uses: 6LWa6ZKx/amlogic-s9xxx-armbian@main
  with:
    build_target: armbian
    kernel_repo: 6LWa6ZKx/amlogic-s9xxx-armbian
```

GitHub Actions 会：
1. **Clone 你的 Action 仓库** 到 `/home/runner/work/_actions/6LWa6ZKx/amlogic-s9xxx-armbian/main/`
2. **执行 action.yml** 中定义的 `runs.steps`（第 124-233 行）

### 2️⃣ **action.yml 的执行流程**

#### 阶段 1: 准备镜像文件 (第136-172行)
```bash
# 1. 定位镜像文件
armbian_file="build/output/images/*.img.gz"

# 2. 复制到工作目录
cp -vf ${armbian_file} build/output/images/

# 3. 解压镜像
gzip -df ${down_file}  # 解压 .img.gz
xz -d ${down_file}     # 或解压 .img.xz

# 4. 检查并重命名文件（必须包含 -trunk_）
# 例如: Armbian_24.11.0-trunk_jammy_arm64_6.6.12.img
```

#### 阶段 2: 构建命令参数 (第207-215行)
```bash
# 将你的输入参数转换为命令行参数
make_command=""
make_command="${make_command} -b s905x3"              # armbian_board
make_command="${make_command} -r 6LWa6ZKx/..."        # kernel_repo ⭐
make_command="${make_command} -u stable"              # kernel_usage
make_command="${make_command} -k 6.6.y"               # armbian_kernel
make_command="${make_command} -a true"                # auto_kernel
make_command="${make_command} -t ext4"                # armbian_fstype
make_command="${make_command} -n 6LWa6ZKx"            # builder_name

# 最终执行的命令 (第217行):
sudo ./rebuild -b s905x3 -r 6LWa6ZKx/amlogic-s9xxx-armbian -u stable \
               -k 6.6.y -a true -t ext4 -n 6LWa6ZKx
```

#### 阶段 3: 执行 rebuild 脚本 (第217行)
```bash
sudo ./rebuild "${make_args[@]}"
```
这会调用仓库根目录的 `rebuild` 脚本。

---

## 🛠️ rebuild 脚本详解

### rebuild 脚本的核心功能

#### 1. 参数解析 (rebuild 脚本 第240-250行左右)
```bash
-b <board>        # 设备型号
-r <repo>         # 内核仓库 ⭐
-u <usage>        # 内核标签
-k <version>      # 内核版本
-a <auto>         # 自动内核
-t <fstype>       # 文件系统
-n <builder>      # 构建者
```

#### 2. 下载内核 (rebuild 脚本中的 download_kernel 函数)
```bash
# 从你指定的 kernel_repo 下载内核
# 例如: https://github.com/6LWa6ZKx/amlogic-s9xxx-armbian/releases

# 下载 URL 格式:
# https://github.com/${kernel_repo}/releases/download/kernel_${kernel_usage}/${kernel_file}
# 实际例子:
# https://github.com/6LWa6ZKx/amlogic-s9xxx-armbian/releases/download/kernel_stable/boot-6.6.12-xxx.tar.gz
# https://github.com/6LWa6ZKx/amlogic-s9xxx-armbian/releases/download/kernel_stable/dtb-amlogic-6.6.12-xxx.tar.gz
# https://github.com/6LWa6ZKx/amlogic-s9xxx-armbian/releases/download/kernel_stable/modules-6.6.12-xxx.tar.gz
```

#### 3. 提取和替换 (rebuild 脚本的 extract_armbian 和 replace_kernel 函数)
```bash
# 1. 挂载原始镜像
losetup -f -P ${armbian_image}
mount /dev/loop0p1 /tmp/boot
mount /dev/loop0p2 /tmp/root

# 2. 删除旧内核
rm -rf /tmp/boot/*
rm -rf /tmp/root/lib/modules/*

# 3. 解压新内核文件
tar -xzf boot-6.6.12.tar.gz -C /tmp/boot/
tar -xzf dtb-amlogic-6.6.12.tar.gz -C /tmp/boot/dtb/amlogic/
tar -xzf modules-6.6.12.tar.gz -C /tmp/root/lib/modules/

# 4. 添加设备特定文件
cp build-armbian/u-boot/s905x3-*.bin /tmp/boot/
cp build-armbian/armbian-files/platform-files/amlogic/* /tmp/root/
```

#### 4. 设备适配 (rebuild 脚本的 refactor_bootfs 和 refactor_rootfs 函数)
```bash
# 根据 armbian_board 参数，为每个设备：
# 1. 复制对应的 DTB 文件
# 2. 配置 U-Boot 参数
# 3. 设置 extlinux.conf
# 4. 添加设备专用脚本

# 例如 s905x3 设备需要：
FDTFILE='amlogic/meson-sm1-x96-max-plus.dtb'
UBOOT_OVERLOAD='u-boot-s905x3-x96max.bin'
```

#### 5. 写入系统信息 (rebuild 脚本第1185-1189行)
```bash
# 写入 /etc/ophub-release
echo "BUILD_REPOSITORY='github.com/armbian/build'" >> /etc/ophub-release
echo "REBUILD_REPOSITORY='github.com/6LWa6ZKx/amlogic-s9xxx-armbian'" >> /etc/ophub-release
echo "KERNEL_VERSION='6.6.12'" >> /etc/ophub-release
echo "KERNEL_BRANCH='stable'" >> /etc/ophub-release
echo "BUILDER_NAME='6LWa6ZKx'" >> /etc/ophub-release
echo "PACKAGED_DATE='2026-01-28'" >> /etc/ophub-release
```

#### 6. 生成多个镜像
```bash
# 对于 -b s905x3_s922x，会生成：
Armbian_24.11.0_amlogic_s905x3_jammy_6.6.12.img
Armbian_24.11.0_amlogic_s922x_jammy_6.6.12.img
```

#### 7. 压缩输出 (action.yml 第221行)
```bash
pigz -qf *.img  # 并行压缩
# 或
gzip -qf *.img  # 普通压缩
```

---

## 🎯 关键参数的使用位置

| 参数 | workflow 输入 | action.yml | rebuild 脚本 | 最终体现 |
|------|--------------|------------|-------------|---------|
| **kernel_repo** | 第232行 | 第209行 `-r` | download_kernel 函数 | 从此仓库下载内核 ⭐ |
| **kernel_usage** | 第233行 | 第210行 `-u` | 构建下载 URL | releases/download/kernel_**stable** |
| **armbian_kernel** | 第230行 | 第211行 `-k` | 选择内核版本 | /boot/vmlinuz-6.6.12 |
| **armbian_board** | 第229行 | 第208行 `-b` | 选择设备配置 | 生成对应设备镜像 |
| **builder_name** | 第235行 | 第215行 `-n` | 写入 ophub-release | /etc/ophub-release |
| **armbian_fstype** | 第234行 | 第213行 `-t` | 格式化分区 | ROOTFS 文件系统类型 |

---

## 📂 输出文件结构

```
build/output/images/
├── Armbian_24.11.0_amlogic_s905x3_jammy_6.6.12.img.gz
│   ↓ 内核来自: https://github.com/6LWa6ZKx/.../releases/download/kernel_stable/
│   ↓ 包含: boot-6.6.12.tar.gz, dtb-amlogic-6.6.12.tar.gz, modules-6.6.12.tar.gz
│
├── Armbian_24.11.0_amlogic_s922x_jammy_6.6.12.img.gz
└── ...
```

---

## ✅ 验证内核来源

在构建好的系统中：

```bash
# 1. 查看内核版本
uname -r
# 输出: 6.6.12-ophub (或 6.6.12-6LWa6ZKx)

# 2. 查看构建信息
cat /etc/ophub-release
# 输出包含:
# KERNEL_VERSION='6.6.12'
# KERNEL_BRANCH='stable'
# REBUILD_REPOSITORY='github.com/6LWa6ZKx/amlogic-s9xxx-armbian'
# BUILDER_NAME='6LWa6ZKx'

# 3. 查看内核模块
ls /lib/modules/
# 输出: 6.6.12-ophub (或你的自定义后缀)
```

---

## 🔧 调试技巧

### 查看 action.yml 日志
在 GitHub Actions 运行日志中，搜索：
- `Start to rebuild armbian...` - 找到 rebuild 命令
- `sudo ./rebuild` - 查看实际执行的命令参数

### 查看内核下载日志
在 rebuild 执行过程中，会输出：
```
Download kernel from: https://github.com/6LWa6ZKx/amlogic-s9xxx-armbian/releases/download/kernel_stable/...
```

### 验证内核文件
```bash
# 在构建系统中
ls -lh /boot/
ls -lh /lib/modules/
cat /etc/ophub-release
```

---

## 📊 参数传递流程图

```
workflow 输入
    ↓
    kernel_repo: "6LWa6ZKx/amlogic-s9xxx-armbian"
    ↓
action.yml (第209行)
    ↓
    make_command="${make_command} -r ${{ inputs.kernel_repo }}"
    ↓
rebuild 脚本 (参数解析)
    ↓
    kernel_repo="6LWa6ZKx/amlogic-s9xxx-armbian"
    ↓
download_kernel 函数
    ↓
    下载 URL: https://github.com/6LWa6ZKx/amlogic-s9xxx-armbian/releases/download/kernel_stable/boot-6.6.12.tar.gz
    ↓
replace_kernel 函数
    ↓
    解压到镜像: /boot/, /lib/modules/
    ↓
最终镜像
    ↓
    使用你仓库的内核 ✅
```

---

## 🎓 总结

**关键点：**
1. ✅ `kernel_repo` 参数决定从哪里下载内核
2. ✅ 必须在你的仓库创建 release tag: `kernel_stable`
3. ✅ release 中必须包含内核文件: boot-*.tar.gz, dtb-*.tar.gz, modules-*.tar.gz
4. ✅ `builder_name` 只是签名，不影响功能
5. ✅ rebuild 脚本是核心，负责整个改造过程

**执行顺序：**
1. Compile Armbian → 生成基础镜像（使用官方内核）
2. Rebuild Armbian → 替换为你的内核 + 适配设备

**最重要的命令：**
```bash
sudo ./rebuild -b <devices> -r <your-repo> -k <kernel-version>
```
这个命令会从 `<your-repo>` 下载内核并打包到镜像中！🎯

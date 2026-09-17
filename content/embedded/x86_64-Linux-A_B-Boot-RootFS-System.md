---
title: 'x86_64 Linux A/B 系统设计：从 GRUB 选槽到 OverlayFS 根文件系统'
date: 2026-09-14T15:11:50+08:00
lastmod: 2026-09-17
description: '从 BIOS、GRUB 选槽到 initramfs 切根与 systemd 健康确认，设计基于 SquashFS 和 OverlayFS 的 Linux A/B 升级系统，并明确断电回退与共享数据的边界。'
draft: false
tags: ["linux", "grub", "A/B slot", "initramfs", "OverlayFS", "x86"]
author: ["zhumouren"]
---

## 1. 设计目标与启动全貌

设备在线升级时，写入镜像只是其中一步。更新可能在断电时中止，新内核可能无法挂载根目录，业务也可能在系统启动后才暴露问题。A/B 设计的目标，是在候选版本尚未被确认时保留一套可启动的旧系统，并为失败后的重启指定明确的去向。

本文面向熟悉 Linux 分区、挂载与 systemd 的读者，讨论 **Legacy BIOS + GPT** 的 x86_64 设备：GRUB 2 选择启动槽位，自定义 BusyBox initramfs 组装根文件系统，systemd 启动主系统并组织健康检查。RootFS 使用只读 SquashFS，运行时写入由持久化的 OverlayFS 可写层（upper）承接。

这是一篇**架构设计与关键机制说明**：给出分区约定、状态转换、挂载顺序和验收条件。GRUB 选槽使用伪代码，`/init` 展示核心片段；完整部署还需实现错误分支、升级程序、超时监督和恢复环境。文中没有提供目标硬件实测记录，因此不据此宣称已经达到生产级可靠性。

设计遵循三个约定：

- **成套更新**：一个槽位包含匹配的 kernel、initramfs 和 RootFS，始终一起选择、一起升级。
- **保留回退对象**：当前运行槽与已确认的稳定槽一致时，才允许更新另一槽；候选版本经过健康检查后才成为默认版本。
- **分离系统与数据**：系统基线保存在 SquashFS；系统写入进入槽位独立的可写层；需要跨版本保留的业务数据单独存放。

选择 SquashFS，是为了把系统基线做成可整体校验、替换的只读镜像；选择 OverlayFS，是为了兼容需要可写根目录的软件。代价是 upper 会保存偏离基线的内容，必须与镜像版本一起管理。普通重启不会恢复出厂状态；本文也不覆盖纯 UEFI 启动、磁盘冗余或完整的可信启动链。

```text
设备上电
  → BIOS 自检、初始化启动所需硬件，选择磁盘
  → 执行磁盘 LBA 0（保护性 MBR）中的 GRUB 引导代码
  → 加载 p1 内嵌的 GRUB core.img，访问 p2 的模块与 grub.cfg
  → grub.cfg 读取启动状态，选择 A/B，将该槽 kernel 和 initramfs 装入内存
  → GRUB 按 Linux 启动协议传参并跳转到内核入口
  → kernel 解压、初始化内存和驱动，将 initramfs 解包到初始根文件系统
  → 执行 initramfs 中的 /init（PID 1，本文为 BusyBox shell 脚本）
  → /init 挂载 SquashFS 和 DATA，再挂载 OverlayFS 到 /newroot
  → exec switch_root /newroot /sbin/init
  → systemd 接管 PID 1，按依赖启动服务，开放登录或业务接口
  → 独立健康检查通过后，才确认试启动版本
```

**initramfs 是文件系统归档，不是独立执行的程序；OverlayFS 是内核文件系统，由早期用户态请求挂载。** `/init` 与 systemd 属于同一个内核下的前后两个用户态阶段，切根不会重新启动内核。若改用 systemd 构建 initramfs，它会更早成为 PID 1，并通过 `systemctl switch-root` 交接，不能原样套用本文的 BusyBox 脚本。两种方式见 [systemd initrd 接口](https://systemd.io/INITRD_INTERFACE/)。

下文会分别标明构建机命令、GRUB 伪代码和早期用户态片段。`/dev/sdX`、UUID、PARTUUID 及 `<...>` 都是占位值。分区、格式化和首次写入示例只用于已经确认身份的部署空盘；在线升级的写入范围见第 7 节。

## 2. 磁盘分区与目录职责

| 分区 | 名称 | 大小 | 格式 | 作用 |
| --- | --- | --- | --- | --- |
| p1 | BIOS_GRUB | 2 MiB | 无文件系统 | GPT 下为 GRUB 的 `core.img` 提供嵌入空间 |
| p2 | GRUB | 128 MiB | ext4 | 保存共享的 GRUB 模块、`grub.cfg`、`grubenv` |
| p3 | BOOT_A | 512 MiB | ext4 | 保存 A 槽 kernel、initramfs 和版本清单 |
| p4 | ROOTFS_A | 3 GiB | raw SquashFS | A 槽只读系统镜像 |
| p5 | BOOT_B | 512 MiB | ext4 | 保存 B 槽 kernel、initramfs 和版本清单 |
| p6 | ROOTFS_B | 3 GiB | raw SquashFS | B 槽只读系统镜像 |
| p7 | DATA | 14 GiB | ext4 | 保存 OverlayFS 可写层、业务数据及升级记录；选槽状态保存在 p2 |

分区容量合计约 **21.127 GiB**，还需预留 GPT 和对齐空间，实际可选 32 GB 或更大的磁盘。p1 的 GPT 类型为 `EF02`，不能格式化；这里没有 ESP 分区，因此不适用于纯 UEFI 启动。BIOS/GPT 的嵌入方式参见 [GRUB BIOS 安装说明](https://www.gnu.org/software/grub/manual/grub/html_node/BIOS-installation.html)。

各分区内部约定如下，路径均相对于该分区根目录：

```text
p2 GRUB                 p3 BOOT_A / p5 BOOT_B
└── grub/               ├── vmlinuz
    ├── grub.cfg        ├── initramfs.img
    ├── grubenv         └── manifest.txt
    └── i386-pc/

p7 DATA
├── overlay/
│   ├── A/{upper,work}/
│   └── B/{upper,work}/
├── shared/             # 跨版本保留的配置、业务数据
└── update/             # 升级清单、进度和诊断记录
```

`raw SquashFS` 表示镜像直接写入 p4/p6，分区开头就是 SquashFS 超级块，外面没有 ext4，也不是把一个 `.squashfs` 文件放进分区。Linux 可以直接挂载该块设备，无需 loop 设备。SquashFS 的只读与压缩特性参见 [内核文档](https://docs.kernel.org/filesystems/squashfs.html)。

## 3. 在构建机上准备磁盘和镜像

### 3.1 创建 GPT 分区

构建机需要 GPT 分区工具、ext4 工具、SquashFS 工具、GRUB BIOS 工具，以及用于制作 initramfs 的 BusyBox、cpio、gzip。不同发行版的软件包名称可能不同。

以下 Bash 脚本在**空盘**上创建固定布局，使用 `sgdisk` 的默认分区对齐设置：

```bash
#!/usr/bin/env bash
set -euo pipefail

DISK=/dev/sdX
sudo sgdisk --clear \
  --new=1:0:+2M       --typecode=1:ef02 --change-name=1:BIOS_GRUB \
  --new=2:0:+128M     --typecode=2:8300 --change-name=2:GRUB \
  --new=3:0:+512M     --typecode=3:8300 --change-name=3:BOOT_A \
  --new=4:0:+3G       --typecode=4:8300 --change-name=4:ROOTFS_A \
  --new=5:0:+512M     --typecode=5:8300 --change-name=5:BOOT_B \
  --new=6:0:+3G       --typecode=6:8300 --change-name=6:ROOTFS_B \
  --new=7:0:+14G      --typecode=7:8300 --change-name=7:DATA \
  "$DISK"
sudo partprobe "$DISK"
sudo udevadm settle

sudo mkfs.ext4 -L GRUB /dev/sdX2
sudo mkfs.ext4 -L BOOT_A /dev/sdX3
sudo mkfs.ext4 -L BOOT_B /dev/sdX5
sudo mkfs.ext4 -L DATA /dev/sdX7
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,PARTUUID "$DISK"
```

NVMe 分区名形如 `/dev/nvme0n1p3`，不能直接套用 `sdX3`。记录 p2/p3/p5 的文件系统 UUID，以及 p4/p6/p7 的 PARTUUID，后续用于定位分区；其中 p2 的 UUID 用于主系统按需挂载 GRUB 状态分区。PARTUUID 属于 GPT 分区，重写 SquashFS 不会改变它；重新分区则可能改变。同一设备上连接的磁盘不能存在重复的这些标识，整盘克隆后尤其需要检查。

### 3.2 构建只读 RootFS

准备一个完整的 x86_64 根目录 `rootfs/`，其中包含 systemd、共享库、业务程序、配置和与 kernel 匹配的 `/lib/modules/<kernel-release>/`。同时预建 `/dev`、`/proc`、`/sys`、`/run`、`/data` 等挂载点，并确认 `/sbin/init` 最终指向可执行的 systemd，其动态加载器及依赖库均在镜像内。本文将 `/usr` 一并打入 RootFS，不使用独立 `/usr` 分区。

打包前，在镜像内写入版本标识 `/usr/lib/ab-release`，包含唯一的 `release_id` 和预期的 `kernel_release`；initramfs 中的 `/etc/ab-initramfs-release` 使用相同字段。两份文件都由同一次构建生成，供 `/init` 在切根前比对。最终产物的摘要放在外部清单中，避免把 SquashFS 自身的摘要写回镜像造成循环依赖。

```sh
sudo mksquashfs rootfs/ rootfs.squashfs -comp xz -noappend
stat -c '%s' rootfs.squashfs
sha256sum rootfs.squashfs
```

镜像大小必须小于等于目标分区容量；文件属主、权限和必要的扩展属性应在打包前准备好。下面的 **Linux 构建机 Bash 脚本**演示首次部署：给 A/B 写入同一份已验证镜像，并按镜像实际长度读回校验。两处目标分区都必须确认未被挂载或作为 OverlayFS 下层使用。

```bash
#!/usr/bin/env bash
set -euo pipefail

IMAGE=rootfs.squashfs
IMAGE_BYTES=$(stat -c '%s' "$IMAGE")
IMAGE_HASH=$(sha256sum "$IMAGE" | awk '{print $1}')

write_rootfs() {
    local target=$1 capacity readback_hash
    if [[ ! -b "$target" ]]; then
        printf '目标不是块设备：%s\n' "$target" >&2
        return 1
    fi
    capacity=$(sudo blockdev --getsize64 "$target")
    if (( IMAGE_BYTES == 0 || IMAGE_BYTES > capacity )); then
        printf '镜像为空或超过分区容量：%s\n' "$target" >&2
        return 1
    fi

    sudo dd if="$IMAGE" of="$target" bs=4M conv=fsync status=progress
    readback_hash=$(sudo head -c "$IMAGE_BYTES" "$target" | sha256sum | awk '{print $1}')
    if [[ "$readback_hash" != "$IMAGE_HASH" ]]; then
        printf '镜像读回校验失败：%s\n' "$target" >&2
        return 1
    fi
}

write_rootfs /dev/sdX4
write_rootfs /dev/sdX6
```

不能对整个 3 GiB 分区求哈希后直接与较小的镜像比较。上述读回能检查读到的内容是否匹配，但可能命中缓存，不能代替断电重启后的介质验证。在线升级时只能对非活动槽执行对应写入，不能原样执行最后两行。

### 3.3 准备 kernel 与 initramfs

早期启动至少需要 GPT、实际磁盘控制器、ext4、SquashFS、对应解压算法、OverlayFS、devtmpfs 和 tmpfs 支持。磁盘及文件系统驱动可编进内核；可模块化的驱动若采用模块方式，必须把模块、依赖及所需固件放进 initramfs，在首次使用前加载。内核解包 initramfs 所需的功能则必须内建。

例如采用 XZ 压缩的 SquashFS 时，需要 `CONFIG_SQUASHFS_XZ`；采用 gzip initramfs 时，需要 `CONFIG_BLK_DEV_INITRD` 和 `CONFIG_RD_GZIP`。AHCI、NVMe 等控制器驱动按目标硬件选择。

initramfs 镜像通常是压缩的 cpio 归档，解包后形成早期用户态环境。本文使用静态链接的 BusyBox，并按需加入其他工具，至少包含：

- 带 `#!/bin/sh` 的可执行 `/init`、`/bin/sh` 以及对应的 BusyBox applet 链接。
- `mount`、`mkdir`、`chmod`、`cp`、`switch_root`、`sync`、`reboot` 等工具。
- 分区识别工具和必要的 ext4 检查工具；动态链接程序还要携带加载器与依赖库。
- `/dev/console` 等必要节点，以及 `/proc`、`/sys`、`/dev`、`/run`、`/newroot` 目录。

准备好 `initramfs/` 目录后，可在构建流水线中打包：

```bash
#!/usr/bin/env bash
set -euo pipefail

chmod +x initramfs/init
(cd initramfs && find . -print0 | cpio --null -o --format=newc | gzip -9) > initramfs.img
```

这里的 `pipefail` 用于让 `find` 或 `cpio` 失败传递到流水线；失败后必须丢弃输出文件，不能继续发布。它是构建机 Bash 的设置，不应未经检查就复制到 BusyBox `/init`。内核解包外部 initramfs 后执行 `/init`，由它负责寻找和挂载真正的根目录，见 [initramfs 文档](https://docs.kernel.org/filesystems/ramfs-rootfs-initramfs.html)。

### 3.4 用版本清单绑定整套产物

每个 BOOT 分区中的 `manifest.txt` 应描述同一次构建的全部产物。以下只约定字段，`<...>` 均需由构建流水线生成实际值：

```text
release_id=<唯一构建标识>
architecture=x86_64
kernel_release=<与目标内核 uname -r 一致的值>
vmlinuz_bytes=<vmlinuz 的字节数>
vmlinuz_sha256=<vmlinuz 的 SHA-256>
initramfs_bytes=<initramfs.img 的字节数>
initramfs_sha256=<initramfs.img 的 SHA-256>
rootfs_bytes=<rootfs.squashfs 的字节数>
rootfs_sha256=<rootfs.squashfs 的 SHA-256>
```

首次部署时，将同一构建的 `vmlinuz`、`initramfs.img` 和清单分别复制到 BOOT_A、BOOT_B 的根目录，核对大小与摘要，刷盘并卸载；p4/p6 则写入对应 SquashFS。GRUB 只按约定路径加载文件，不会自动理解这份自定义清单。

升级程序为每次操作生成唯一的 `update_id`，将源槽、目标槽、源版本、目标 `release_id`、清单摘要和处理阶段保存在 `/data/update`。`release_id` 标识产物，`update_id` 区分对同一产物的多次升级尝试。健康确认必须匹配本次操作，不能只检查槽位字母。`uname -r` 也可能在不同构建间重复，不能代替产物摘要校验。

**摘要用于校验内容一致，签名用于验证来源。** 若要校验升级包签名，应让签名覆盖清单，并由清单绑定各产物的大小和摘要。解析清单时按数据读取，不要直接 `source` 或 `eval`。

## 4. BIOS 与 GRUB：选择成套的启动槽位

### 4.1 安装共享 GRUB

```sh
sudo mkdir -p /mnt/grub
sudo mount /dev/sdX2 /mnt/grub
sudo grub-install --target=i386-pc --boot-directory=/mnt/grub /dev/sdX
sudo grub-editenv /mnt/grub/grub/grubenv create
sudo grub-editenv /mnt/grub/grub/grubenv set stable_slot=A trial_slot=
```

这里安装目标是**整块磁盘**。`i386-pc` 是 GRUB 的 BIOS 平台名称，即使目标 Linux 为 x86_64，仍使用这个平台。

`grub-editenv ... create` 和初始 `stable_slot=A` 只在首次部署时执行，示例假定 A 的产物已通过出厂基线验证。在线升级不能重新创建环境块，否则会覆盖已有启动状态。

上电后，BIOS 读取并执行 LBA 0 中的引导代码；该代码载入 `core.img` 的首扇区，再由其中的代码载入剩余部分。`core.img` 携带访问启动分区所需的功能，再从 p2 加载正常模式所需模块并执行 `grub.cfg`。读取 `grubenv` 是配置脚本通过 `load_env` 发起的动作，不是 BIOS 的能力。各镜像职责见 [GRUB image files](https://www.gnu.org/software/grub/manual/grub/html_node/Images.html)。

p1 不挂载；p2 可以在正常系统中按需挂到 `/boot/grub-store`，此时环境块路径是 `/boot/grub-store/grub/grubenv`。`grub-install` 安装引导程序后，仍需把实现第 4.2～4.3 节约定的配置写入 `/mnt/grub/grub/grub.cfg`。日常升级只更新非活动槽的 BOOT 与 RootFS，不必重新安装共享 GRUB。

### 4.2 把槽位信息交给内核

下面仅展示 A 槽的**GRUB 加载命令及参数约定**，不构成可直接部署的 `menuentry`。`boot_mode` 由下一节的选槽逻辑设置，为 `stable` 或 `trial`；只有查找、内核加载和 initramfs 加载都成功后，控制流程才允许执行 `boot`：

```text
insmod part_gpt
insmod ext2
search --no-floppy --fs-uuid --set=root <BOOT_A_UUID>
linux /vmlinuz ab.slot=A ab.mode=${boot_mode} root=PARTUUID=<ROOTFS_A_PARTUUID> ab.data=PARTUUID=<DATA_PARTUUID> panic=10
initrd /initramfs.img
```

B 槽使用相同结构，替换 `ab.slot=B`、`BOOT_B_UUID` 和 `ROOTFS_B_PARTUUID`。实际配置可将两个自动启动项命名为 `slotA`、`slotB`。GRUB 的 `ext2` 模块也用于读取受支持的 ext3/ext4；文件系统特性必须与实际 GRUB 版本兼容。

`linux` 装载内核并设置命令行，`initrd` 装载这里的 initramfs 归档。命令名 `initrd` 不意味着文件必须是旧式块设备 initrd。加载失败必须转入回退或恢复分支，不能继续执行正常引导。GRUB 的菜单项在满足条件时会自动调用 `boot`，因此错误分支也必须正确终止该启动项，不能用一个成功返回的日志命令掩盖失败。参见 [menuentry 的执行语义](https://www.gnu.org/software/grub/manual/grub/html_node/menuentry.html)。

这里有两个不同的“root”：

- GRUB 的 `root` 指向 BOOT 分区，决定从哪里读取 `/vmlinuz` 和 `/initramfs.img`。
- 内核命令行的 `root=PARTUUID=...` 指向 SquashFS 分区，由本文的 `/init` 解析，用作 OverlayFS 下层。

`ab.slot`、`ab.mode` 和 `ab.data` 是本方案自定义参数，Linux 不会自动完成对应操作。GRUB 只负责把参数传给内核；内核再通过 `/proc/cmdline` 将它们提供给 `/init`。在这个自定义 initramfs 中，`root=` 也是交给 `/init` 使用的输入，不能仅靠它让内核自动构建 OverlayFS。启动命令参见 [GRUB Linux 启动说明](https://www.gnu.org/software/grub/manual/grub/html_node/GNU_002fLinux.html)和 [search 命令](https://www.gnu.org/software/grub/manual/grub/html_node/search.html)。

### 4.3 用一次试启动实现回退

为便于说明，采用“一次试启动”策略。`grubenv` 保存两个自定义变量：

| 变量 | 含义 | 示例 |
| --- | --- | --- |
| `stable_slot` | 最近确认健康、默认启动的槽位 | `A` |
| `trial_slot` | 尚未消耗的单次试启动请求；空值表示没有待执行请求 | `B` 或空 |

在两个启动项之前执行的选择逻辑如下。这是**状态机伪代码**，需转成实际 GRUB 脚本并验证：

```text
清除内存中的旧变量，从固定的 p2 环境块读取 stable_slot、trial_slot
校验 stable_slot 为 A/B，trial_slot 为空或另一个槽位
如果读取失败、stable_slot 缺失或状态不合法：进入恢复流程

本次槽位 = stable_slot
boot_mode = stable
如果 trial_slot 非空：
    候选 = trial_slot
    清空 trial_slot，并用 save_env 持久化
    若成功：本次槽位 = 候选，boot_mode = trial
    若失败：禁止启动候选；重新读取并校验状态
        状态可用：保持启动稳定槽
        状态不可用：进入恢复流程

导出 boot_mode，供启动项构造内核命令行
设置 default 为本次槽位对应的 slotA 或 slotB
检查查找与加载结果，成功后才交给内核
候选在 GRUB 阶段加载失败：设置 boot_mode = stable，重新加载稳定槽整套产物
稳定槽也加载失败：进入恢复流程
```

自动启动项必须与选槽结果一致；人工维护入口应使用单独的 `recovery` 模式并禁止自动确认。`ab.mode` 用于传递本次决策，不是安全认证机制。

关键顺序是：**先消耗试启动机会，再把控制权交给新内核**。假设 A 稳定、B 待试启动，GRUB 在启动 B 前就把持久状态变成 `stable_slot=A, trial_slot=空`。B 如果没有完成健康确认，下次重启自然选择 A。

GRUB 的 `load_env`/`save_env` 可以读写环境块，但可写支持受磁盘访问方式和文件系统限制，必须在目标设备上验证，见 [GRUB 环境块说明](https://www.gnu.org/software/grub/manual/grub/html_node/Environment-block.html)。命令返回成功是进入候选版本的必要条件；掉电后的完整性仍取决于存储实现，第 8 节讨论这一边界。

环境块必须始终定位到 p2 上同一个文件。启动项里的 `search --set=root` 会把 GRUB 的 `root` 改到 p3/p5，因此不要依赖会变化的 `root` 拼接环境块路径；应固定 p2 的设备引用或使用正确的 `prefix`。Linux 侧 `grub-editenv` 写入成功，也不能证明 GRUB 启动阶段的 `save_env` 一定可写，两条路径都要实测。

GRUB 无法感知交出控制权后的内核卡死。`panic=10` 只处理 panic；死锁、业务无响应等情况需要硬件看门狗或独立超时机制触发重启，回退才有机会发生。

此策略解决的是**尚未确认的新版本启动失败**。一个已经确认的稳定版本日后出现故障，并不会自动切到另一槽；另一槽可能正在更新。运行期故障恢复需要额外的槽位有效性记录和故障计数策略。

还要区分**正在运行的槽位**与**持久化的稳定槽位**：试启动 B 时，前者是 B，后者仍然是 A。以 A 升级到 B 为例，状态变化如下：

| 时刻 | 当前运行槽位 | `stable_slot` | `trial_slot` | 此时重启的默认选择 |
| --- | --- | --- | --- | --- |
| 正常运行 A | A | A | 空 | A |
| B 已写完并发布候选 | A | A | B | 尝试 B，并先清空 trial |
| GRUB 已消耗试启动机会，B 尚未确认 | B（或仍在加载） | A | 空 | A |
| B 确认成功 | B | B | 空 | B |
| B 未确认，重启后回退 | A | A | 空 | A |

表中的结果以状态完整、读写成功为前提。`trial_slot` 在启动 B **之前**已经清空，因而不能用它判断试启动是否结束。`/run/ab/mode` 记录本次启动方式，在 B 确认成功后也不会自动变成 `stable`；是否允许下一轮升级，应结合当前槽位、持久状态和本次升级的最终处理结果判断。

## 5. initramfs：把只读镜像变成可写根目录

### 5.1 解析参数并找到块设备

内核执行 `/init` 时，它就是 PID 1；此时磁盘上的 systemd 尚未启动。`/init` 应依次执行：

1. 挂载 `/proc`、`/sys`、`/dev` 和 `/run`，获得命令行、设备信息、设备节点及运行时目录。
2. 从 `/proc/cmdline` 提取 `ab.slot`、`ab.mode`、`root=PARTUUID=...`、`ab.data=PARTUUID=...`；校验模式、槽位以及它与 RootFS 分区的对应关系。
3. 加载所需驱动，并限时等待两个块设备出现；超时就进入失败处理，不能无限等待。
4. 根据 PARTUUID 定位设备。仅挂载 devtmpfs 并不保证存在 `/dev/disk/by-partuuid/` 链接；使用 udev 建立链接，或携带支持 PARTUUID 查询的工具，例如 util-linux `blkid`。
5. 在 DATA 尚未挂载时执行必要的 ext4 检查。`e2fsck` 返回 0/1 可继续；要求重启时按受控流程重启，存在未修复错误时进入失败处理。

第 5 步应显式处理退出码：`0` 表示无错误，`1` 表示已修复，`2` 表示需要重启，其他错误位也可能组合出现。若 `/init` 使用 `set -e`，直接运行 `e2fsck` 会把“修复成功”的 `1` 也视为失败；应先捕获返回值，再分支处理。无人值守设备应使用经过验证的非交互检查策略，不能停在等待人工回答的提示上。参见 [e2fsck 手册](https://man7.org/linux/man-pages/man8/e2fsck.8.html)。

设备解析必须得到唯一结果；零个或多个匹配都应视为错误。下面片段假定 `/init` 已设置可用的 `PATH`、连接控制台输入输出，并准备好挂载点；BusyBox 的构建选项需包含所用 applet 及 bind/move 挂载支持。

接下来用 `ROOT_DEV`、`DATA_DEV`、`SLOT`、`BOOT_MODE` 表示已经解析和校验的结果。以下是 `/init` 中的**挂载核心片段**，不是完整的错误处理脚本；每条命令失败都应进入统一失败分支。`/init` 不应正常返回：PID 1 意外退出会使系统无法继续启动。

### 5.2 挂载下层、可写层与合并目录

```sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
mount -t tmpfs -o mode=0755,nodev,nosuid,strictatime tmpfs /run

# 此处完成参数解析、设备等待与 DATA 检查，得到下面四个变量。
# ROOT_DEV=/dev/sda4  DATA_DEV=/dev/sda7  SLOT=A  BOOT_MODE=stable

mkdir -p /run/ab/lower /run/ab/data /newroot
chmod 0700 /run/ab
mount -t squashfs -o ro "$ROOT_DEV" /run/ab/lower
mount -t ext4 -o rw "$DATA_DEV" /run/ab/data

# 在此核对 initramfs、运行内核与 lower 的版本；试启动还需匹配待确认记录。
UPPER=/run/ab/data/overlay/$SLOT/upper
WORK=/run/ab/data/overlay/$SLOT/work
mkdir -p "$UPPER" "$WORK"

mount -t overlay overlay \
  -o "lowerdir=/run/ab/lower,upperdir=$UPPER,workdir=$WORK" \
  /newroot
```

三个目录的关系如下：

```text
ROOTFS_A/B：只读 SquashFS ────────────── lowerdir
DATA：overlay/A 或 B/upper ─────────── upperdir
DATA：overlay/A 或 B/work ──────────── workdir
                                           │
                                           ▼
                                  /newroot（合并视图）
                                           │ switch_root
                                           ▼
                                      用户看到的 /
```

对普通文件，上层同名文件优先可见；目录则可能合并上下层内容。修改下层文件通常先复制到上层，再修改；删除下层文件通过上层的 whiteout 隐藏它。SquashFS 基线保持不变。`upper` 与 `work` 必须位于同一文件系统，初次准备的 `work` 应为空，不能与其他活动 OverlayFS 共用。机制和约束见 [OverlayFS 文档](https://docs.kernel.org/filesystems/overlayfs.html)。

**本方案中 A/B 必须使用各自的 upper/work，且 upper 要随镜像版本更新。** 例如，旧 B 曾删除 `/etc/myapp.conf`，upper 中就可能留下 whiteout；即使新 B 的 SquashFS 包含新的默认配置，合并后仍看不到它。因此重写 B 时，应在 B 未被使用的前提下重建其 upper/work；普通重启则保留它们。需要跨版本保留的配置从 `/data/shared` 按明确规则迁移。

OverlayFS 挂载期间，不能绕过合并视图直接修改对应的 lower、upper 或 work；这类修改的行为未定义。正常文件操作应通过合并后的 `/` 完成。可写层的维护入口只向系统管理程序开放，普通业务账户只获得所需的共享数据目录权限。

只读 lower 只保证镜像基线不被运行时写入改变，合并后的 `/` 仍可被 upper 覆盖。持久 upper 里的错误也会跨重启保留；恢复某槽出厂系统视图需要离线重建该槽的 upper/work。日志应限制容量，避免耗尽共享 DATA 后同时影响两个槽位。

### 5.3 保留挂载树，再交给 systemd

下层和可写层不仅要“挂载成功”，还必须在清理旧 initramfs 时保留下来。因此把它们放在 `/run` 这个独立 tmpfs 的子挂载中，再把整棵 `/run` 挂载树移到新根：

```sh
mkdir -p /newroot/dev /newroot/proc /newroot/sys /newroot/run /newroot/data
mount --bind /run/ab/data /newroot/data
printf '%s\n' "$SLOT" > /run/ab/slot
printf '%s\n' "$BOOT_MODE" > /run/ab/mode
# 版本比对通过后，保留本次 initramfs 的构建身份供主系统复核。
cp /etc/ab-initramfs-release /run/ab/initramfs-release

mount --move /dev /newroot/dev
mount --move /proc /newroot/proc
mount --move /sys /newroot/sys
mount --move /run /newroot/run

exec switch_root /newroot /sbin/init
```

移动 `/run` 会同时保留下层 SquashFS 与 DATA 的子挂载，使切根后仍可通过 `/run/ab/lower`、`/run/ab/data` 访问它们。`/data` 是 DATA 的另一个访问入口，业务数据放在 `/data/shared`；应用不应直接修改 `overlay` 内部目录。

这里明确使用 **BusyBox `switch_root`**：调用者必须为 PID 1，`/newroot` 必须是挂载点。它清理旧 initramfs 的文件，把新根挂载移到 `/`，再执行 `/sbin/init`；`exec` 使整个交接保持 PID 1。应在执行前检查新 init 及其依赖，并停止不需保留的早期用户态进程。普通 `chroot` 不能替代这一过程；实现细节见 [BusyBox switch_root 源码](https://github.com/mirror/busybox/blob/master/util-linux/switch_root.c)。

任意挂载、版本检查或切根准备失败，都不能继续启动业务。试启动时应记录原因并限时重启；正常稳定槽也失败时，应进入恢复环境，避免无休止重启。该恢复环境必须单独准备，双槽布局不会自动提供它。早期用户态尚无 systemd，失败处理不能依赖 `systemctl reboot`；应使用经过验证的 BusyBox 重启路径或看门狗，且 PID 1 不能直接退出。

## 6. systemd：启动业务并确认版本健康

接管后，systemd 读取新根目录中的 unit，建立设备与挂载依赖，启动基础服务、网络和业务，最终到达默认 target，例如 `multi-user.target`。这些服务按依赖关系并行启动，并非严格逐个串行。

根目录已经由 initramfs 挂好，因此 `/etc/fstab` 不应再把 p4/p6 当成普通 ext4 根分区挂载，也不要重复挂载 DATA。`/run`、`/dev` 等挂载需要与目标发行版的 systemd 初始化规则配合。

### 6.1 定义健康，而不是只判断启动完成

到达 `multi-user.target` 不等于业务健康。健康确认程序至少应检查：

- 当前槽位、`/run/ab/initramfs-release`、运行内核和只读 lower 的版本与本次升级记录一致，根目录确实是 OverlayFS。
- DATA 可写且剩余空间充足，关键配置迁移成功。
- 必要设备就绪，核心服务通过真实的本地接口或功能探测。
- 系统连续运行一段观察时间，无关键故障。

可将如下 unit 保存为 `ab-confirm.service`，并在制作 RootFS 时启用。`myapp.service` 是需替换的业务服务名；`ab-confirm` 是本方案需要实现的健康检查与状态提交程序：

```ini
[Unit]
Description=Confirm healthy A/B boot
Requires=myapp.service
After=myapp.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/ab-confirm
TimeoutStartSec=120

[Install]
WantedBy=multi-user.target
```

`After=` 保证启动顺序，业务是否就绪仍取决于服务类型与实际探测。配合 `Requires=` 时，前置服务启动失败可能使确认程序根本不执行。`TimeoutStartSec=120` 约束确认服务自身的启动过程，不覆盖排队等待依赖的时间，因此不能替代整机试启动监督。参见 [systemd.service](https://github.com/systemd/systemd/blob/main/man/systemd.service.xml) 与 [systemd.unit 的作业超时说明](https://github.com/systemd/systemd/blob/main/man/systemd.unit.xml)。

### 6.2 将候选版本确认为稳定版本

程序先检查 `/run/ab/mode`：只有本次是 `trial`、槽位和待确认升级记录匹配，且健康检查通过，才把当前槽位写入 `stable_slot`。普通 `stable` 启动和人工恢复启动不执行晋升。例如确认 B 时，在按需挂载的 p2 上更新：

```sh
grub-editenv /boot/grub-store/grub/grubenv set stable_slot=B trial_slot=
sync
```

实际程序应持有与升级程序共用的状态锁，重新核对 `update_id`、槽位和版本，再从已校验的运行状态获取写入值。持久化后重新读取验证，最后卸载或恢复 p2 的只读挂载，并将本次升级记录标记为已确认。上面的两条命令只展示环境块写入，不代表完整提交事务。

在候选版本确认前禁止开始下一轮升级；对已完成的 `update_id` 重复执行确认应直接结束。这样既防止旧任务覆盖新状态，也允许确认程序在重试时保持结果不变。若写入 `stable_slot` 后、更新升级记录前掉电，应按第 7.2 节核对并补记，不能反向把稳定槽改回旧值。

### 6.3 让失败真正触发回退

健康检查失败时不提交状态，并主动请求重启或交由看门狗处理。仅让 oneshot 服务返回失败，不会自动完成回退；看门狗的启动、喂狗条件及各阶段交接也需要实现。

应分别给设备发现与挂载、主系统启动、业务观察设置时间预算，并让独立监督覆盖整个试启动过程。监督不能依赖 `ab-confirm` 成功启动；也不能只因 PID 1 或喂狗进程还活着就无限续期。只有状态提交并复核成功，才结束试启动监督、转入正常运行期的健康策略。若硬件看门狗在 initramfs 中启用，还必须验证切根时的交接不会意外关闭它或停止喂狗。

### 6.4 关机也要遵守挂载依赖

正常关机先停止业务并刷盘，再由内存中的关机环境接管，解除绑定挂载和 OverlayFS，最后卸载下层与 DATA。自定义 initramfs 需要预先准备可执行的 `/run/initramfs/shutdown` 及其运行环境，才能使用 systemd 的关机回跳接口；仅把挂载保留在 `/run/ab` 不会自动获得该能力。不能在仍使用 OverlayFS 时强行卸载 DATA。接口约定见 [systemd initrd 文档](https://systemd.io/INITRD_INTERFACE/)。

## 7. 在线升级的完整流转

### 7.1 写入、发布与确认的顺序

假设当前稳定运行 A，目标是升级到 B：

| 步骤 | 操作 | 为什么这样做 |
| --- | --- | --- |
| 1. 检查状态 | 锁定升级流程，核对当前槽与稳定槽均为 A，结束旧记录，确认 B 未被使用 | 若撤销旧候选，必须先持久化清除 trial，才能重写 B |
| 2. 验证升级包 | 检查架构、版本、容量、签名及各文件摘要 | 在写盘前排除错误或不可信产物 |
| 3. 写 B 镜像 | 写 p6 的 SquashFS，再写 p5 的 kernel、initramfs 和清单 | 始终只修改非活动槽，状态仍指向 A |
| 4. 准备 B 数据 | 重建 B 的 upper/work，仅迁移允许保留的配置 | 防止旧上层遮挡新系统 |
| 5. 落盘与复核 | 刷盘，按实际长度读回校验，验证 BOOT 与 RootFS 版本配套 | 写入完成后才允许发布候选版本 |
| 6. 发布候选 | 先持久化含 `update_id` 的待确认记录，再设置 `trial_slot=B` 并复核；稳定槽仍为 A | 发布前再次核对稳定槽，避免把陈旧状态写回 |
| 7. 重启试运行 | GRUB 先清空 `trial_slot`，再启动 B 的整套产物 | 即使 B 启动失败，下次仍默认 A |
| 8. 确认或回退 | B 健康则提交 `stable_slot=B`；否则重启回到 A，随后结束本次记录 | 只有验证过的版本才成为默认版本 |

```text
稳定 A
  └─ 写入并验证 B ── 设置 trial=B ── 重启
                                    │
                              GRUB 先清空 trial
                                    │
                                 启动 B
                              ┌─────┴─────┐
                            健康         失败
                              │           │ 重启
                       提交 stable=B      └── 启动 A
```

这套顺序有三个明确的持久化时刻：**发布候选、消耗试启动请求、确认稳定版本**。候选发布前中断，默认仍是 A；请求消耗后、B 确认前中断，下次也是 A，即使 B 本来能够成功。只有确认完成后，重启才默认 B。这些结论均以选槽状态完整可读为前提。

升级记录应先写临时文件、同步文件内容，再在同一文件系统内替换正式记录并同步目录；产物和记录落盘后才发布 `trial_slot`。这种顺序用于缩小未完成写入的影响，并不把 p2 与 DATA 上的更新变成一个原子事务。

### 7.2 在重启后结束上一次升级

GRUB 决定启动哪个槽，升级记录解释一次更新进行到了哪里。两者位于不同分区，可能在一次掉电后暂时不同步。启动后的状态核对程序应在同一状态锁下，结合持久环境块、当前启动方式和源／目标版本处理残留记录。以下规则只用于自动启动路径；人工 `recovery` 模式只做诊断：

| 重启后观察到的状态 | 对旧记录的处理 |
| --- | --- |
| 正在以 `trial` 运行目标 B，稳定槽仍为 A，trial 为空且身份匹配 | 继续健康检查，禁止开始新升级 |
| 当前槽与稳定槽都是 B，trial 为空、目标版本匹配，但记录仍待确认 | 补记“已确认”，不再改写选槽决定 |
| 以 `stable` 启动源槽 A，trial 为空，源版本与记录匹配 | 记为“未采纳／已回退”，归档记录，不自动重试 B |
| 仍在 A，trial 指向 B，记录与候选一致 | 保留待试启动状态；取消时先持久化清除 trial |
| 槽位、版本、记录不一致，或任一状态无法读取 | 停止自动升级和确认，进入诊断或恢复流程 |

“未采纳”可能表示发布前掉电、GRUB 加载失败或 B 启动后失败，仅凭最后的槽位值不能确定故障阶段，应结合日志判断。记录结束后，还需确认当前系统健康、无存活的旧升级任务，才能发起下一轮操作。

### 7.3 系统回退与业务数据回退

业务数据也必须支持回退：B 若直接破坏性迁移 `/data/shared` 中的数据库，退回 A 后可能无法读取。应采用向后兼容的数据格式，或在确认前使用独立副本并准备可恢复的迁移方案。**系统槽位回退不会自动回滚共享数据。**

例如，B 可以先新增一个 A 会忽略的字段，待兼容窗口结束后再移除旧字段；如果必须更换不兼容格式，应在试运行前决定旧版本读哪个副本，以及试运行期间的新写入如何保留。仅备份数据库文件，并不足以定义完整的回退行为。

## 8. 可靠性边界与验证

### 8.1 回退成立的前提

本文状态机推导出的自动回退依赖三个前提：旧槽的产物与可写层仍可用，选槽状态仍完整可读，以及故障后确实能触发重启。A/B 分区本身不能保证这三个条件始终成立。

p1、p2、DATA 和物理磁盘仍是共享故障点。DATA 损坏可能使两槽都无法组成可写根目录；旧槽也可能受共享数据的不兼容迁移影响。单份 `grubenv` 不是具有断电原子性的事务存储，`sync` 不能保证它在任意掉电时都完整。

若产品要求在状态写入中断时仍能自动恢复，需要补充带序列号、校验和与双副本的状态协议，并在引导端实现读取与选择规则；不能只多备份一个文件就假定 GRUB 会自动使用。共享 GRUB 的更新应单独设计恢复路径，同时保留外部救援介质或维护入口。

只读 SquashFS 也不等于内容已经通过真实性验证。本文的校验用于确认产物一致性，默认升级程序及其信任配置可信；若还要检测启动后的块内容损坏或建立可信启动链，可进一步设计 [dm-verity](https://docs.kernel.org/admin-guide/device-mapper/verity.html) 与引导认证。即便验证了 lower，可写 upper 仍能覆盖合并根目录中的文件，必须一并纳入安全设计。

### 8.2 故障注入与验收

复现实验应固定硬件／虚拟机配置、磁盘型号与缓存设置、Linux／GRUB／BusyBox／systemd 版本，以及内核和 BusyBox 的构建配置。完整 `/init`、GRUB 错误分支、升级与确认程序、超时监督、关机与恢复环境都应纳入同一版本管理。

下表是**待执行的验收用例及预期结果，不是实测报告**。先在 BIOS 模式虚拟机验证控制流程，再在目标硬件测试实际断电、刷盘语义与看门狗覆盖范围；虚拟机重启不能替代真实掉电测试。

| 场景 | 预期结果 |
| --- | --- |
| 分别选择 A、B 正常启动 | kernel、initramfs、RootFS 和 upper 属于同一槽位；systemd 与业务就绪 |
| B 写镜像或写 BOOT 时断电 | 未发布候选，仍从 A 启动 |
| B 的 kernel、initramfs 或 RootFS 与清单不匹配 | 写入复核失败时不发布候选；启动后发现版本不符时不确认健康 |
| GRUB 无法读取 B 的 BOOT，或无法保存 trial 清除结果 | 不启动 B；状态仍有效时启动 A，否则进入恢复流程 |
| B 内核 panic、initramfs 挂载失败 | 超时重启后回到 A，不提交 B |
| B 业务无法通过健康检查 | 在限定时间内重启并回到 A |
| 业务依赖一直未就绪，确认服务未执行 | 独立监督仍能触发重启，不能无限等待 |
| 手动启动旧槽、普通稳定启动 | 不自动晋升槽位，不覆盖待处理的升级状态 |
| B 确认已落盘，但升级记录尚未更新时断电 | 下次启动 B 后补记已确认，不误撤销稳定槽 |
| 清除 trial 或提交 stable 时断电 | 验证状态损坏处理；有冗余协议时恢复有效记录，否则进入恢复入口 |
| DATA 满、损坏或无法修复 | 不误报健康，不自动格式化用户数据；进入受控恢复流程 |
| A → B → A 多轮升级 | 新 lower 不被旧 upper 遮挡，共享数据保持兼容 |
| 正常关机及反复掉电重启 | 挂载依赖处理正确，DATA 可恢复，业务数据符合预期 |

每次测试至少保存 `update_id`、产物摘要、故障注入点、重启前后状态、实际启动槽、恢复耗时与关键日志。缺少这些记录，就无法区分“设计上应当回退”和“设备上已经验证回退”。

### 8.3 按启动阶段定位问题

| 现象 | 优先检查 | 判断重点 |
| --- | --- | --- |
| 无法进入 GRUB 菜单 | BIOS 启动模式、p1 类型、整盘 GRUB 安装结果 | 先确认 Legacy BIOS + GPT 路径成立 |
| GRUB 找不到 kernel 或 initramfs | BOOT 的 UUID、`search` 结果及文件路径 | GRUB 的 `root` 应指向对应 BOOT 分区 |
| 停在 initramfs，找不到块设备 | `/proc/cmdline`、`dmesg`、驱动与 PARTUUID 查询结果 | 区分参数错误、驱动缺失和设备发现超时 |
| SquashFS 或 OverlayFS 挂载失败 | 内核日志、镜像摘要、upper/work 所在文件系统 | 核对解压支持、目录权限、空间及是否已有挂载引用 |
| 新版本文件仍显示旧内容 | 比较 `/run/ab/lower` 与合并后的 `/` | 排查旧 upper 文件、whiteout 和错误的槽位目录 |
| systemd 已启动但没有确认版本 | 确认服务日志、`/run/ab/mode`、版本记录和健康探测结果 | 区分服务未执行、探测失败和状态写入失败 |

进入主系统后，可使用以下只读命令收集信息；`findmnt`、`lsblk` 属于 util-linux，需要预先放入主系统，精简 initramfs 不一定包含它们：

```sh
cat /proc/cmdline
cat /run/ab/slot /run/ab/mode
cat /run/ab/initramfs-release
uname -r
cat /run/ab/lower/usr/lib/ab-release
findmnt -no SOURCE,FSTYPE,OPTIONS /
findmnt -R /run/ab
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,PARTUUID
df -h /data
df -i /data
journalctl -b -u ab-confirm.service -u myapp.service
```

分析上一次失败的启动，需要预先配置容量受限的持久日志，或将关键错误写入 `/data/update`；DATA 不可用时则依靠串口等独立诊断通道。保留现场与自动回退同样必要，否则设备虽然恢复运行，故障原因却可能随重启消失。

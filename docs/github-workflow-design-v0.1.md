# github工作流设计

版本：v0.1  
日期：2026-10-19  
对象：PogoPlug V4 / OpenWrt 25.12.0 / LIO iSCSI target  
状态：构建输入与流程设计已确定；最小片段尚待干净源码构建回归，GitHub Actions 尚未落地运行。

## 1．目标和本轮边界

将本轮已经完成手工编译、并在 PogoV4 上成功自动启动的系统，转化为可重复执行的 GitHub Actions 构建流程。构建机使用 amd64 Linux，为 ARMv5TE 的 PogoV4 交叉编译完整 OpenWrt。

本轮继续使用包含完整 rootfs 的 initramfs-uImage。精简早期启动环境的概念设计另见《uImage最小早期启动环境优化设计》，不纳入本轮实现。

工作流的核心业务输入只有三项：

1. OpenWrt 配置片段 `configs/openwrt.config`。
2. Linux 内核配置片段 `configs/kernel.config`。
3. 启动入口 `files/sbin/sd-init`。

源码和 feeds 的修订锁定、构建环境、导入命令、产物处理属于工作流本身的编排责任。三项业务输入足以表达本次定制，不代表可以省略这些构建条件。

本工作流负责构建、核对、导出和保存构建产物。SD 卡分区格式化、文件写入实体卡、修改机器内的 UBOOT 环境、NAND 刷写和实机重启均不在本轮工作流内。LVM 激活和 LIO 的 LUN、网络服务编排也不在本轮实现范围。

## 2．已经具备的事实与待验证项目

| 项目 | 当前证据与状态 |
|---|---|
| 完整源码编译 | 在 Sandbox00 的 Debian 构建环境中成功完成；提示符名称虽然为 ImageBuilder，实际已使用完整 OpenWrt 源码构建 |
| 本地工作目录 | `~/workspace/OpenWrt/PogoPlug_V4/Buildroot/openwrt` |
| 内核 | 本轮实际生成 Linux 6.12.71 |
| SD 自动启动 | 已完成 UBOOT → sd-init → FAT /boot/init → 正式 OpenWrt 的自动交接 |
| 根文件系统 | FAT 上的 SquashFS 文件经只读 loop 挂载，F2FS 提供可写 overlay |
| mount_root 衔接 | FAT /boot/init v0.2 已实机验证，正确拦截 `mount_root start ""` |
| 源码改动取证 | Git 跟踪的修改只有 `target/linux/kirkwood/config-6.12`；文件注入目录只有 `files/sbin/sd-init` |
| feeds | 五个 feed 的实际 HEAD 均与固定定义一致，未报告工作区修改 |
| 最小片段 | 已整理完成；尚未用这两个片段从干净源码完整重建 |
| GitHub 工作流 | 尚未创建或运行；托管 runner 的实际耗时、磁盘峰值待首次构建测量 |
| LIO 服务 | 本文只固定内核能力，不将启动成功解释为 LUN 服务已经验证通过 |

历史证据的详细过程归入《SD卡启动实验成果》。本文件保留足够的输入和流程信息，供下一页直接开展工作流实现。

## 3．固定构建基线

### 3.1 OpenWrt 源码

| 项目 | 固定值 |
|---|---|
| 仓库 | `https://git.openwrt.org/openwrt/openwrt.git` |
| 发布标签 | `v25.12.0` |
| Git 提交 | `4da53efc2fe744601e6189492dbd06b8930d72b8` |
| 目标 | `kirkwood/generic` |
| 设备 | `cloudengines_pogoplugv4` |
| 本轮实际工具链 | GCC 14.3.0 / musl / ARM EABI |

工作流检出该固定提交，并核对实际 HEAD。标签便于识别，提交值用于核对。不得在本轮自动跟随 master、main 或最新发布版本。

这次菜单保存形成的 `target/linux/kirkwood/config-6.12` 整份 diff 不作为工作流输入。工作流从干净源码开始，通过内核片段表达新增功能，不携带 CPU、DMA、CRC 等菜单归一化差异。

### 3.2 feeds

工作流在源码根目录生成 `feeds.conf`，采用以下完整定义：

```text
src-git packages https://git.openwrt.org/feed/packages.git^506c37591d7cd9b3cfd812e69e708349268f9865
src-git luci https://git.openwrt.org/project/luci.git^27a9c0c94264063894d40420956696d19b175744
src-git routing https://git.openwrt.org/feed/routing.git^662b57dff7d20634f2b8676400cf4299611cfe4b
src-git telephony https://git.openwrt.org/feed/telephony.git^2618106d5846a4a542fdf5809f0d3ed228ce439b
src-git video https://github.com/openwrt/video.git^094bf58da6682f895255a35a84349a79dab4bf95
```

这五条定义可以直接保存在工作流脚本中，不要求新增第三份配置片段。随后执行 feeds 更新和安装，并将各 feed 的实际 HEAD 记录到构建信息中。

```sh
./scripts/feeds update -a
./scripts/feeds install -a
```

首版保留本轮已经使用的五个 feed，不同时开展 feeds 裁剪实验。

## 4．两个配置片段的完整内容

原则：片段表达必要修订；已有默认项不重复声明；不手工展开软件包、加密库、CRC 和字符集等依赖。OpenWrt 层和内核层分别交给各自的配置系统解析。

### 4.1 configs/openwrt.config

```text
# OpenWrt v25.12.0 / Kirkwood / PogoPlug V4
CONFIG_TARGET_kirkwood=y
CONFIG_TARGET_kirkwood_generic=y
CONFIG_TARGET_kirkwood_generic_DEVICE_cloudengines_pogoplugv4=y

CONFIG_BUSYBOX_CUSTOM=y
CONFIG_BUSYBOX_CONFIG_STAT=y
CONFIG_BUSYBOX_CONFIG_FEATURE_STAT_FORMAT=y

CONFIG_PACKAGE_blkid=y
CONFIG_PACKAGE_losetup=y
CONFIG_PACKAGE_kmod-loop=y
CONFIG_PACKAGE_kmod-fs-vfat=y
CONFIG_PACKAGE_kmod-fs-f2fs=y
CONFIG_PACKAGE_kmod-usb-storage=y
CONFIG_PACKAGE_lvm2=y
```

共 13 项：3 项目标选择、3 项 BusyBox 定制、7 项软件包选择。

USB 控制器和硬盘盒能够枚举，不代表 USB 存储驱动已经具备。实机补入 `kmod-usb-storage` 后已成功识别外置 LVM 硬盘，因此显式选择该包；SCSI 等依赖由构建系统展开，不另向内核片段手工追加 `CONFIG_USB_STORAGE`。

本次固定基线的 PogoV4 默认配置已经提供 MMC 支持、initramfs/SquashFS 镜像类型，以及 BusyBox 的 cmp、chroot、switch_root，因此片段不重复列入这些默认项。默认项虽不写入片段，仍属于构建验收要求。

不从历史 `.config` 照抄 libdevmapper、kmod-dm、libblkid、libsmartcols、字符集、CRC、TPM、加密模块等展开结果，也不携带关闭五个 `CONFIG_FEED_*` 的历史选择。

### 4.2 configs/kernel.config

```text
# LIO: core, block-device backend, iSCSI target
CONFIG_TARGET_CORE=y
CONFIG_TCM_IBLOCK=y
CONFIG_ISCSI_TARGET=y
```

共 3 项，分别要求 LIO 核心、块设备后端和 iSCSI target 内建。

`CONFIG_CONFIGFS_FS` 由 TARGET_CORE 自动选中，不在片段中重复声明。本轮不改成自定义 kmod 包，不增加文件后端或其他 target fabric。

## 5．仓库内的文件责任

以下为待实现的仓库布局约定，不表示这些文件已提交到远程仓库。

| 仓库相对路径 | 责任 | 本轮是否必需 |
|---|---|---|
| `.github/workflows/build.yml` | 调度构建步骤、固定版本、保存产物 | 必需，待编写 |
| `configs/openwrt.config` | OpenWrt 最小配置输入 | 必需，已整理 |
| `configs/kernel.config` | 内核最小配置输入 | 必需，已整理 |
| `files/sbin/sd-init` | 注入内嵌 rootfs 的启动入口 | 必需，已有实机版本 |
| `deploy/boot/init` | FAT 分区内的根目录组装脚本 | 交付完整启动包时需要 |
| `deploy/boot/uEnv.txt` | FAT 分区内的 UBOOT 配置 | 交付完整启动包时需要 |

构建仓库与 OpenWrt 源码 checkout 分开，建议将后者放在 `work/openwrt`。构建路径允许变化，不能把 Sandbox00 的绝对目录写入片段。

`sd-init` 采用本轮实机使用的文件，权限 0755、LF 换行。本轮取证的 SHA-256 是：

```text
992bd6c1689fe644c6620fe21e778237ae9161f4943b978f84cb88c350af53db
```

这个摘要用于识别本轮版本；未来修订 sd-init 时，应在同一仓库提交中更新相应记录。

FAT 上的 init 和 uEnv.txt 不需要编进镜像。它们的内容和运行责任由《SD卡启动实验成果》记录；完整交付时应采用已经验证成功的 init v0.2，不能退回漏接空第二参数的 v0.1。

## 6．配置导入与构建顺序

### 6.1 准备环境和源码

1. 检出本项目仓库。
2. 准备 amd64 Linux 构建依赖，以普通用户执行 OpenWrt 的 make。
3. 在独立目录检出固定的 OpenWrt 提交。
4. 写入固定 feeds.conf，更新并安装 feeds。

只安装构建所需的宿主工具，不把交叉编译目标误设为 runner 的 amd64。本项目的产物架构仍然是 ARMv5TE。

### 6.2 导入三个业务输入

下面的命令说明设计要求，后续转换为工作流步骤；其中项目输入路径由工作流定位到实际 checkout 目录。

```sh
# 在 OpenWrt 源码根目录执行。
cp <项目仓库>/configs/openwrt.config .config

mkdir -p env files/sbin
cp <项目仓库>/configs/kernel.config env/kernel-config
install -m 0755 <项目仓库>/files/sbin/sd-init files/sbin/sd-init

make defconfig
```

这里两份片段的导入处理不同：

- `.config` 是 OpenWrt 配置入口。`make defconfig` 解析目标、软件包、BusyBox 等选择，补齐该层默认项和依赖。
- `env/kernel-config` 是本次版本官方 `include/target.mk` 纳入内核配置列表的固定入口。文件复制到位后，后续内核构建将它与 generic、target、subtarget 配置以及包产生的内核配置共同处理。

`make defconfig` 不替代 Linux 内核自己的配置解析。不要把内核片段追加到顶层 `.config`，也不要用复制片段覆盖构建目录中的 Linux 完整 `.config`。

首版工作流不调用交互式 `make menuconfig` 或 `make kernel_menuconfig`。构建中的内核配置归一化由官方构建流程负责；如果出现未处理的交互请求，应定位遗漏的输入或默认处理步骤，不使用人工按键作为 CI 的常规步骤。

### 6.3 下载与完整编译

配置导入并核对后，先下载源码包，再执行完整构建。首轮建议使用 `make -j2 V=s`，不只执行 `make target/linux/compile`，因为根文件系统、软件包安装和最终镜像同样是交付目标。

完整构建包含宿主工具、交叉工具链、内核、用户空间软件包、根目录拼装和镜像生成。遇到失败保留原始日志，再按需要以单任务详细输出定位，不将一次失败后恰好成功的重试解释为已经消除原因。

## 7．构建验收

### 7.1 OpenWrt 配置解析后

核对片段要求的目标、BusyBox 定制和七个软件包仍被选中。同时核对以下默认能力：

- `CONFIG_TARGET_ROOTFS_INITRAMFS=y`，使用内嵌 rootfs，而非 separate initramfs。
- `CONFIG_TARGET_ROOTFS_SQUASHFS=y`。
- 没有外部 CPIO 替代本轮根目录来源。

检查“要求是否满足”，不要求展开后的 `.config` 与最小片段相同。依赖增加是正常结果。

### 7.2 完整构建后

| 核对对象 | 必须满足的结果 |
|---|---|
| BusyBox 实际构建配置 | CMP、STAT、FEATURE_STAT_FORMAT、SWITCH_ROOT、CHROOT 已启用 |
| Linux 实际配置 | 三项 LIO 为 y；CONFIGFS 可用；SquashFS/XZ、overlay 可用 |
| SD 启动支持 | MMC、FAT、字符集、loop、F2FS 等启动需要的模块或内建功能可用 |
| USB 存储支持 | 实际内核配置中 USB_STORAGE 为 m；根目录中包含 usb-storage.ko 及其依赖，并具备 usb-storage 自动加载配置 |
| 根目录中的工具 | `/usr/sbin/blkid`、`/usr/sbin/losetup` 及其运行依赖已经安装 |
| sd-init | 构建入口与根目录中的内容一致，权限可执行 |
| initramfs 来源 | 官方构建确实使用注入后的根目录生成内嵌 initramfs |
| 最终镜像 | PogoV4 的 initramfs-uImage 和 root.squashfs 均存在且非空 |

核对模块时允许依赖解析结果提供需要的文件，不能为了凑齐清单而反向把所有依赖写进输入片段。

amd64 runner 不直接执行这些 ARM 程序。第一版 CI 检查配置、文件、镜像结构和构建结果；实体 PogoV4 的启动交接及服务测试仍是独立验收。

### 7.3 本轮已知日志的处理

此前未选中软件包的 feeds 缺失依赖提示不等于构建失败；仍须保留日志，根据实际选包和最终 make 结果判断。

实机上的 UBOOT 外层扫描提示、NAND UBI 探测、环境工具 CRC 警告等属于运行环境的独立事项，不通过修改最小构建片段来掩盖。它们的分析归入实验成果文档。

## 8．产物导出契约

所有路径以 OpenWrt 源码根目录为起点：

| 构建原文件 | 导出文件 |
|---|---|
| `bin/targets/kirkwood/generic/openwrt-kirkwood-generic-cloudengines_pogoplugv4-initramfs-uImage` | `openwrt/lio-uImage` |
| `build_dir/target-arm_xscale_musl_eabi/linux-kirkwood_generic/root.squashfs` | `openwrt/lio-root-aligned.squashfs` |

第二个构建路径是本次固定基线的实际路径。将来升级目标或工具链时，应重新确认，不能不加区分地选取任意 root.squashfs。

必须选择 initramfs-uImage。普通 `cloudengines_pogoplugv4-uImage` 是官方 NAND 方案放入 UBI kernel 卷的内核，不能替代本项目需要早期用户空间的启动镜像。

### 8.1 SquashFS 对齐

本轮官方打包命令使用 `-nopad`。实机已验证，未对齐镜像可能在 loop 容量截断后挂载失败；对副本补零到 4096 字节边界后成功。

导出过程固定为：

1. 将原始 root.squashfs 复制到导出路径。
2. 读取副本原长度 N。
3. 计算 `A = ((N + 4095) / 4096) * 4096`，采用整数运算。
4. 使用 truncate 将副本扩展到 A；扩展部分为零。
5. 比较前 N 字节与构建原文件一致，核对副本实际长度为 A。

不修改构建目录中的原始 root.squashfs。每次构建重新计算长度，不能使用上一次镜像的字节数。

### 8.2 交付包

核心交付包含两个镜像。若仓库同时纳入部署脚本，可以交付可直接对应 FAT 根目录的启动文件包：

| 包内相对路径 | 复制到 FAT 后的路径 |
|---|---|
| `openwrt/lio-uImage` | `/openwrt/lio-uImage` |
| `openwrt/lio-root-aligned.squashfs` | `/openwrt/lio-root-aligned.squashfs` |
| `boot/init` | `/boot/init` |
| `boot/uEnv.txt` | `/boot/uEnv.txt` |

这里的路径是 FAT 内部路径。Linux 将该分区挂到 `/boot` 后，FAT 脚本的运行路径是 `/boot/boot/init`。

首版保存为 GitHub Actions artifact，不自动建立 Release 或操作设备。若需要归档成 tar 包，应保留文件相对路径及适用的文件权限；配置和脚本统一使用 LF 换行。

## 9．构建证据和失败现场

每次构建应保存：

- 项目仓库提交、OpenWrt 标签及实际提交、五个 feeds 的实际提交。
- 两份输入片段、解析后的 OpenWrt .config、实际内核 .config 和 BusyBox .config。
- sd-init 的文件摘要、软件包 manifest、镜像摘要。
- SquashFS 原长度和对齐后长度。
- runner 操作系统、CPU/内存、构建前后的磁盘空间、构建用时及完整日志。

构建证据与可启动文件分别放入 artifact 的不同目录。失败时上传已经存在的日志和配置，不将不完整镜像包装成成功启动包。

这些证据用于对比本地成功版本与 CI 版本，不在启动脚本中增加额外的版本选择或运行时升级业务。

## 10．runner、资源和缓存的首版选择

首版建议使用 amd64 的 `ubuntu-24.04` 托管 runner，并固定这一系统标签，避免使用随时间切换发行版的 ubuntu-latest。具体宿主构建依赖在编写 workflow 时按该系统版本落实。

GitHub 托管 runner 的资源与仓库类型有关。当前官方文档对常规 x64 Ubuntu runner 列出的资源为：公开仓库 4 CPU、16 GB RAM、14 GB SSD；私有仓库 2 CPU、8 GB RAM、14 GB SSD。磁盘峰值和构建时长仍需以本项目首轮实际运行为准，不能由本地构建成功推定托管环境必然足够。

首版采用单设备、单任务、`-j2`，先完成无历史工具链/构建目录缓存的完整回归。记录资源后，再决定是否需要明确的空间清理、更大的 runner 或自托管 runner。

后续可以先缓存下载目录 dl，利用官方源码包摘要校验。是否进一步引入 ccache 或工具链缓存另行决定；缓存不能成为片段能够正确重建的前提。

首版建议由 workflow_dispatch 手工触发。自动触发范围在首次回归通过后再确定；不增加定时构建或多版本矩阵。

## 11．实施与验收顺序

1. 将本文件中的两个最小片段和实机 sd-init 放入项目仓库。
2. 编写 build.yml，落实固定源码、feeds 和输入导入顺序。
3. 用干净源码完成首次构建，确认没有依赖历史 target 配置修改或旧构建目录。
4. 核对最终配置、根目录工具、镜像以及 SquashFS 导出对齐。
5. 保存 artifact 与构建证据。
6. 将 CI 产物放到实验 SD 卡，完成一次 PogoV4 自动启动回归，核对挂载状态及 mount_root 拦截。
7. 通过后再考虑缓存、资源优化和触发方式扩展。

验收分为两个结果分别记录：GitHub 工作流完整编译及交付通过；对应产物的 PogoV4 实机启动回归通过。LUN 服务验收继续作为后续项目工作。

## 12．下一页开展工作所需的结论

- 最小业务输入为 13 项 OpenWrt 选择、3 项内核选择及 sd-init 文件。
- 不需要把本次 kernel_menuconfig 保存的整份 target 配置 diff 搬进仓库。
- env/kernel-config 是本次版本官方构建读取的内核配置入口。
- 顶层 make defconfig 与内核配置归一化是两个不同层次，按官方流程分别完成。
- 核心产物是内嵌 rootfs 的 initramfs-uImage 和补零对齐后的 SquashFS 副本。
- FAT init v0.2 和 uEnv.txt 是可选纳入交付包的部署输入，不影响两个镜像的编译。
- 当前尚未完成最小片段的干净构建验证，也没有已运行成功的 GitHub 工作流。
- 本文提出的 runner、并行度、手动触发和缓存策略是首版实施建议，不是已经取得的测试结论。

## 参考与依据

项目依据：本页完整构建日志、配置取证输出、SD 自动启动日志和最终挂载核对；源码与 feeds 提交以第 3 节固定值为准。

- [OpenWrt v25.12.0：include/target.mk](https://github.com/openwrt/openwrt/blob/v25.12.0/include/target.mk)：内核配置文件列表及 env/kernel-config 入口。
- [OpenWrt v25.12.0：include/kernel-defaults.mk](https://github.com/openwrt/openwrt/blob/v25.12.0/include/kernel-defaults.mk)：内核配置处理及 initramfs 构建。
- [OpenWrt v25.12.0：Kirkwood 镜像规则](https://github.com/openwrt/openwrt/blob/v25.12.0/target/linux/kirkwood/image/Makefile)：PogoV4 内核封装和镜像类型。
- [OpenWrt v25.12.0：diffconfig.sh](https://github.com/openwrt/openwrt/blob/v25.12.0/scripts/diffconfig.sh)：顶层配置差异导出工具；本项目片段进一步按业务需求精简。
- [GitHub 托管 runner 说明](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)：资源与系统标签；资源信息核对日期为 2026-10-19。
- [GitHub Actions 使用限制](https://docs.github.com/en/actions/reference/limits)：实现时核对适用的作业和资源限制。
- [GitHub Actions artifact 下载](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/download-workflow-artifacts)：首版产物交付入口。

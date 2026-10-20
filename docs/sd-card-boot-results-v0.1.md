# SD卡启动实验成果

版本：v0.1（阶段性技术成果记录）  
记录日期：2026-10-19  
对象：PogoPlug V4 / OpenWrt 25.12.0 / SD 文件镜像根目录 / LIO 内核支持  
用途：实验归档、设计追溯和换页继续工作的技术依据。

## 1．已经取得的阶段成果

本轮已完成完整 OpenWrt 源码编译，并在 PogoV4 上打通以下自动启动路径：UBOOT 从 SD 的 FAT 分区读取 initramfs-uImage；内核以 sd-init 为 PID 1；sd-init 按 LABEL 找到 BOOT 分区并只读挂载；执行 FAT 中的 /boot/init；从 FAT 上的 SquashFS 镜像文件组装只读根层，从 F2FS 数据分区组装可写层；交接给正式 OpenWrt init，最后由 procd 管理系统。

最重要的已验证结论是：正式只读 rootfs 可以作为普通文件存放在 FAT 分区，不必独占一个原始 SquashFS 分区；所需 loop、FAT 和 SD 驱动能够在内嵌 initramfs 中提前获得，解决挂载正式 rootfs 之前的依赖问题。

| 成果 | 当前证据 |
|---|---|
| 完整源码和交叉工具链构建 | 完整 make 完成镜像、软件包索引、profiles.json 和校验文件处理 |
| LIO 编译能力 | 实际内核配置中 TARGET_CORE、TCM_IBLOCK、ISCSI_TARGET、CONFIGFS_FS 均为 y |
| 从 SD 加载内核 | UBOOT fatload、iminfo 和 bootm 成功，内核识别 PogoV4 DTB |
| FAT 文件镜像根目录 | 只读 FAT、只读 loop、只读 SquashFS 链路成功，chroot 执行镜像内程序成功 |
| F2FS 可写层 | 旧 p3 的只读检查与后续可写挂载、新 p2 的格式化及自动挂载均成功 |
| overlay 根目录 | 手工组装、chroot 执行、PID 1 switch_root 均成功 |
| LABEL 识别 | 自动启动中 BOOT 解析为 mmcblk0p1，OVERLAY 解析为 mmcblk0p2 |
| 自动启动 | UBOOT → sd-init → FAT init → 正式 OpenWrt 全流程成功 |
| 虚拟文件系统交接 | 自动启动后曾核对 /proc、/sys、/dev、/tmp 各一层，无重复叠加 |
| 包装脚本修复 | FAT init v0.2 实机正确拦截 mount_root start 空第二参数；旧 loop 错误消失 |
| 可写层跨重启使用 | 后续启动读取到 /etc/urandom.seed；本证据不替代任意业务数据的持久化测试 |
| 最小构建输入整理 | 已形成 12 项 OpenWrt 片段、3 项内核片段及 sd-init 文件；尚待干净构建回归 |

这里的“启动成功”不等于 LIO 已提供可用 LUN，也不等于长期稳定性、掉电恢复、SD 写入寿命或全部异常分支已经完成测试。

## 2．项目目标和已经确定的取舍

### 2.1 目标

目标是在方便维护和减少 SD 写入之间取得实用平衡。启动镜像和只读系统放在 FAT 分区中，正常运行时保持该分区只读；可写配置及其他变更放在独立 F2FS 分区。镜像升级由 amd64 构建机重新构建、导出后部署，不在启动脚本中实现版本选择或在线升级系统。

已经决定：

- 保留现有可工作的 bodhi UBOOT，本轮不进行 NAND 或引导器刷写。
- UBOOT 配置由 FAT 分区的 /boot/uEnv.txt 承担。
- UBOOT 之后、正式 OpenWrt 之前的衔接分为镜像内 sd-init 和 FAT 上的 /boot/init 两层。
- BOOT 和 OVERLAY 均使用文件系统 LABEL 识别；早期扫描限定在 MMC 分区，不扩展为任意 USB/SATA 发现器。
- 始终以全新空 upper 能启动为设计基准；正常重启保留持久化 upper，不在每次启动清空它。
- mount_root 包装脚本在组装阶段生成，不修改官方镜像构建规则，也不依赖旧 upper 中预先留下的脚本。
- 先使用完整 initramfs 打通功能，精简早期环境列为低优先级的独立实验项目。

### 2.2 减少写入的实际边界

BOOT、loop 后端和 SquashFS 均以只读方式使用，这已经有挂载输出证据。F2FS 数据分区仍为可写，系统配置、随机种子以及后续业务数据会写入该分区；本轮未制定完整写回、日志或业务数据减写策略，也没有量化寿命收益。

F2FS 实际挂载输出包含 relatime/lazytime、background_gc、discard 等选项。它们是当前运行状态记录，不是另外经过调优对比后得出的寿命最优参数。

## 3．硬件、机器和构建基线

| 项目 | 本轮实际值 |
|---|---|
| 设备 | Cloud Engines PogoPlug Series 4 |
| CPU | Feroceon 88FR131，ARMv5TE |
| 内存 | 128 MiB |
| NAND | 128 MiB，擦除块 128 KiB，页 2048 字节 |
| UBOOT | U-Boot 2017.07-tld-1，Pogoplug V4 |
| 串口 | ttyS0，115200 |
| OpenWrt 标签 | v25.12.0 |
| 源码提交 | 4da53efc2fe744601e6189492dbd06b8930d72b8 |
| 系统显示版本 | OpenWrt 25.12.0，r32713-f919e7899d |
| Linux | 6.12.71 |
| 工具链 | GCC 14.3.0，musl，ARM EABI |
| target/subtarget | kirkwood/generic |
| profile | cloudengines_pogoplugv4 |
| 构建机器 | Sandbox00，amd64 Debian，用户 silicat |
| 构建目录 | ~/workspace/OpenWrt/PogoPlug_V4/Buildroot/openwrt |
| 导出目录 | ~/workspace/OpenWrt/PogoPlug_V4/Buildroot/sdcard |
| SD 分区准备机器 | SandPlug |

提示符中的 `(ImageBuilder)` 是构建环境名称。本轮在其中 clone 了完整源码、编译工具链和内核，实际使用的是完整 Buildroot，不是只能组装现有软件包的独立 ImageBuilder 产品。

源码使用浅克隆的发布标签，处于 detached HEAD 是正常现象，不表示下载或编译失败。make prereq 已通过；make menuconfig 是打开 OpenWrt 顶层配置菜单的命令；make kernel_menuconfig 是内核配置菜单。此前 Windows CMD/SSH 下菜单无响应的具体现象未形成可确认的根因记录，不应编造终端修复结论。

### 3.1 feeds 固定修订

```text
src-git packages https://git.openwrt.org/feed/packages.git^506c37591d7cd9b3cfd812e69e708349268f9865
src-git luci https://git.openwrt.org/project/luci.git^27a9c0c94264063894d40420956696d19b175744
src-git routing https://git.openwrt.org/feed/routing.git^662b57dff7d20634f2b8676400cf4299611cfe4b
src-git telephony https://git.openwrt.org/feed/telephony.git^2618106d5846a4a542fdf5809f0d3ed228ce439b
src-git video https://github.com/openwrt/video.git^094bf58da6682f895255a35a84349a79dab4bf95
```

最后一次取证中，五个 feed 的实际 HEAD 与上述定义一致，未报告 feed 工作区修改。

## 4．本轮配置修订项目清单

配置必须分为“已经编译和实机测试过的展开结果”与“为以后自动构建整理的最小输入”两类。前者已有实机证据；后者尚未经过一次干净源码完整重建。

### 4.1 必要业务修订

| 业务 | 配置责任及本轮结果 |
|---|---|
| 选择设备 | Kirkwood / generic / cloudengines_pogoplugv4 |
| 正式根目录和过渡环境 | 生成 SquashFS，同时生成内嵌完整 rootfs 的 initramfs-uImage |
| FAT32 | kmod-fs-vfat 及其字符集依赖进入镜像 |
| SD | 使用该设备默认的 kmod-mvsdio、MMC 核心与块设备支持 |
| loop 文件镜像 | kmod-loop 和独立 losetup 软件包 |
| F2FS 可写层 | kmod-fs-f2fs 及相关 CRC 支持 |
| LABEL 识别 | 独立 blkid 软件包及其动态库 |
| LVM | lvm2，构建系统带入 libdevmapper、kmod-dm 等依赖 |
| BusyBox | 开启自定义配置，补入 STAT 和 FEATURE_STAT_FORMAT；CMP、CHROOT、SWITCH_ROOT 在实际构建中可用 |
| LIO | TARGET_CORE、TCM_IBLOCK、ISCSI_TARGET 内建；CONFIGFS_FS 随核心启用 |
| 自定义文件 | files/sbin/sd-init，0755、LF 换行 |

### 4.2 实际内核配置核对

最后一次构建取证得到：

```text
CONFIG_BLK_DEV_LOOP=m
CONFIG_BLK_DEV_DM=m
CONFIG_TARGET_CORE=y
CONFIG_TCM_IBLOCK=y
# CONFIG_TCM_FILEIO is not set
# CONFIG_TCM_PSCSI is not set
# CONFIG_LOOPBACK_TARGET is not set
CONFIG_ISCSI_TARGET=y
# CONFIG_REMOTE_TARGET is not set
CONFIG_MMC=m
CONFIG_MMC_BLOCK=m
CONFIG_MMC_MVSDIO=m
# CONFIG_VHOST_SCSI is not set
CONFIG_F2FS_FS=m
CONFIG_OVERLAY_FS=y
CONFIG_VFAT_FS=m
CONFIG_CONFIGFS_FS=y
CONFIG_SQUASHFS=y
CONFIG_SQUASHFS_XZ=y
CONFIG_NLS_CODEPAGE_437=m
CONFIG_NLS_ISO8859_1=m
```

曾在完整软件包构建前看到 MMC、F2FS 未启用。随后完整构建与软件包配置处理后，这些功能最终为模块，并在实机成功加载。应以最终配置和镜像内模块为准，不能用过早阶段的配置误判最终镜像。

### 4.3 实际 BusyBox 与工具核对

实际 BusyBox 配置文件：

```text
build_dir/target-arm_xscale_musl_eabi/busybox-default/busybox-1.37.0/.config
```

最终确认：

```text
CONFIG_CHROOT=y
CONFIG_STAT=y
CONFIG_FEATURE_STAT_FORMAT=y
CONFIG_CMP=y
CONFIG_SWITCH_ROOT=y
```

曾因 STAT 缺失而在构建前核对中停止；补入 BusyBox 自定义选项后完成增量编译。stat 的命令入口为 /bin/stat → busybox；cmp 为 /usr/bin/cmp → ../../bin/busybox。

独立工具安装到 /usr/sbin/blkid 和 /usr/sbin/losetup。manifest 中两者均为 2.41.3-r1。file 检查显示它们是 ARM EABI5 动态链接程序，解释器为 /lib/ld-musl-arm.so.1；后来自动启动中实际运行成功，说明早期环境具备其运行所需依赖。

早期曾使用错误的 BusyBox 目录 glob，未找到配置；后来改为真实的两级 busybox-default/busybox-1.37.0 路径。不要把旧检查命令的空输出解释为没有 BusyBox。

### 4.4 原始 diff 与伴随变化

最后一次 Git 取证：被跟踪的改动仅有 target/linux/kirkwood/config-6.12，统计为 21 行新增、7 行删除；另有两个未跟踪的本地 .config 备份。文件注入目录只有 sd-init。

内核菜单保存除 LIO 之外，还带出 integrity、CRC、加密、CPU、DMA 等变化，并删除部分旧配置行。实际完整内核中 PREEMPT_NONE_BUILD、SquashFS 解压策略等仍可由默认配置恢复，不能只从 target 文件中的行消失推断功能消失。

这些原始差异作为历史证据保留在附录 F，不整体迁入未来工作流。

### 4.5 LIO 模块化问题的结论

本轮在源码树的 package、target/linux、feeds 中搜索 TARGET_CORE、TCM_IBLOCK、ISCSI_TARGET 以及对应 ko 文件的软件包定义，没有找到覆盖所需三项能力的现成 kmod 定义，因此当前采用内建 y。

这个搜索结果限定于当次源码和已安装 feeds，不表示 Linux 的 LIO 本身不能编成模块。若以后改成 m，还需要提供正确的 OpenWrt KernelPackage、依赖、模块文件安装和加载规则；仅在临时内核配置里改成 m，不能保证模块进入可安装镜像。该工作未在本轮实现。

## 5．整理好的两个最小配置片段

这两个片段以第 3 节固定源码和 feeds 为基线，只声明必要修订，不预先展开依赖。

### 5.1 configs/openwrt.config：12 项

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
CONFIG_PACKAGE_lvm2=y
```

### 5.2 configs/kernel.config：3 项

```text
# LIO: core, block-device backend, iSCSI target
CONFIG_TARGET_CORE=y
CONFIG_TCM_IBLOCK=y
CONFIG_ISCSI_TARGET=y
```

CONFIGFS 由 TARGET_CORE 选入；MMC、镜像类型及部分 BusyBox 功能在固定目标基线中已有默认值，不重复写入片段。依赖产生的 libdevmapper、kmod-dm、动态库、字符集、CRC 和加密模块由构建系统处理。

未来工作流从干净源码开始：顶层片段导入 .config，执行 make defconfig；内核片段复制为 env/kernel-config，由官方内核构建流程读取。两者不是把同一种完整 .config 复制到两个目录。env/kernel-config 是当前版本 include/target.mk 明确纳入配置列表的入口。

最后一次本地取证时 env/kernel-config 尚不存在；实机成功镜像仍来自此前 target/linux/kirkwood/config-6.12 的菜单修改。不能把已经整理好的最小片段写成已经重建验证过。

## 6．构建结果与镜像导出

### 6.1 区分内核编译完成与完整构建完成

早期日志的 target/linux/compile 正常结束，证明内核及模块编译通过。完整 make 结束时，又完成 rootfs 拼装、initramfs-uImage、factory/sysupgrade 镜像、manifest、软件包索引、profiles.json 和校验文件，才构成整套镜像的完整构建结果。

本轮完整构建成功，后续增加 FAT/loop 工具以及 sd-init、blkid、BusyBox stat 后，也分别完成了增量构建。没有必要为了这些修改执行 make clean 或重新交互配置内核。

### 6.2 构建路径

以下路径相对于 OpenWrt 源码根目录：

| 对象 | 路径 |
|---|---|
| 要导出的内核镜像 | bin/targets/kirkwood/generic/openwrt-kirkwood-generic-cloudengines_pogoplugv4-initramfs-uImage |
| 要导出的只读根镜像 | build_dir/target-arm_xscale_musl_eabi/linux-kirkwood_generic/root.squashfs |
| 普通 NAND 内核 | build_dir/target-arm_xscale_musl_eabi/linux-kirkwood_generic/cloudengines_pogoplugv4-uImage |
| 根目录拼装来源 | build_dir/target-arm_xscale_musl_eabi/root-kirkwood |
| 实际 Linux 配置 | build_dir/target-arm_xscale_musl_eabi/linux-kirkwood_generic/linux-6.12.71/.config |
| 设备 manifest | bin/targets/kirkwood/generic/openwrt-kirkwood-generic-cloudengines_pogoplugv4.manifest |

最终使用的 initramfs-uImage 在 UBOOT 中读取为 7,241,523 字节，uImage 数据长度为 7,241,459 字节。早期手工实验镜像为 7,221,683 字节。大小用于识别本轮历史产物，不作为未来构建的固定验收值。

镜像中的时间戳显示 2026-03-03，和实际编译日期不同；构建日志使用 SOURCE_DATE_EPOCH。不能仅凭 uImage 头部日期判断没有重新编译。

### 6.3 一个 uImage 的组成

官方规则将内核 zImage 追加 PogoV4 DTB，再封装 uImage。我们选用的是同时内嵌完整 rootfs 的 initramfs 版本，因此 UBOOT 只需要加载一个文件，不再分别加载 uInitrd 和外部 DTB。

uImage 头显示 uncompressed，是外层封装的压缩标志；不能据此推断内部 zImage 没有压缩。

普通 uImage 约 3.3 MiB，用于官方 NAND 的 kernel 卷；本项目的 initramfs-uImage 约 6.9 MiB，提供正式根目录挂载前的工具和模块。两者采用相同的 DTB/uImage 封装规则，但用户空间内容不同。

### 6.4 导出和对齐命令

以下为在 Sandbox00 构建目录使用的导出流程，只写 sdcard 目录中的副本：

```bash
bash -s -- <<'EOF'
set -euo pipefail
cd ~/workspace/OpenWrt/PogoPlug_V4/Buildroot/openwrt

lio_uimage=bin/targets/kirkwood/generic/openwrt-kirkwood-generic-cloudengines_pogoplugv4-initramfs-uImage
lio_squashfs=build_dir/target-arm_xscale_musl_eabi/linux-kirkwood_generic/root.squashfs
lio_output=../sdcard

printf '\n[01] 检查本轮镜像\n'
test -s "$lio_uimage"
test -s "$lio_squashfs"

printf '\n[02] 导出两个镜像副本\n'
mkdir -p "$lio_output"
cp "$lio_uimage" "$lio_output/lio-uImage"
cp "$lio_squashfs" "$lio_output/lio-root-aligned.squashfs"

printf '\n[03] 为 SquashFS 副本补零对齐\n'
lio_size=$(stat -c '%s' "$lio_squashfs")
lio_aligned=$(( (lio_size + 4095) / 4096 * 4096 ))
truncate -s "$lio_aligned" "$lio_output/lio-root-aligned.squashfs"
printf '原长度：%s；对齐后：%s\n' "$lio_size" "$lio_aligned"

printf '\n[04] 核对副本\n'
cmp "$lio_uimage" "$lio_output/lio-uImage"
cmp -n "$lio_size" "$lio_squashfs" "$lio_output/lio-root-aligned.squashfs"
test "$(stat -c '%s' "$lio_output/lio-root-aligned.squashfs")" -eq "$lio_aligned"
ls -lh "$lio_output/lio-uImage" "$lio_output/lio-root-aligned.squashfs"
EOF
```

本轮实际 SD 启动成功证明部署的镜像可以使用。以上流程记录了后续导出的统一方法；不把每个辅助命令都独立声称为已有一次新的实机回归。

## 7．SD 布局演进与当前布局

### 7.1 旧实验布局

此前使用 p1 FAT、p2 原始 SquashFS、p3 F2FS 的三分区布局。旧可写层中曾有 /upper/sbin/mount_root 包装脚本，硬编码检查 /dev/mmcblk0p2 的 /rom 和 /dev/mmcblk0p3 的 /overlay。

本页手工文件镜像实验改为使用 p1 中的 rootfs 文件，仍暂借旧 p3 作为数据分区，并使用 upper-lio-p1、work-lio-p1 避免受旧 upper/work 影响。该阶段已经成功进入正式 OpenWrt。

这些是历史路径，不作为当前自动启动的接口。当前自动脚本按 LABEL 发现设备，并使用 /p1-root/upper、/p1-root/work。

### 7.2 当前两分区布局

在 SandPlug 的 SD 读卡器 /dev/sda 上重建 MBR，逻辑扇区为 512 字节：

| 分区 | 起始扇区 | 扇区数 | 末扇区（包含） | 分区容量 | 文件系统 | LABEL |
|---|---:|---:|---:|---:|---|---|
| p1 | 2048 | 524288 | 526335 | 256 MiB | FAT32 | BOOT |
| p2 | 526336 | 16777216 | 17303551 | 8 GiB | F2FS | OVERLAY |

其余约 111 GiB 保留未分配。两个分区均按 MiB 边界对齐；第一分区的 MBR 类型为 0x0c，第二分区为 0x83。

本轮格式化后的身份记录：

| 对象 | 值 |
|---|---|
| MBR 标识 | f8d3becc |
| p1 PARTUUID | f8d3becc-01 |
| p1 文件系统 UUID | 7506-2347 |
| p2 PARTUUID | f8d3becc-02 |
| p2 文件系统 UUID | 0531059a-e65b-42c8-90c6-80cb4561d513 |

UUID 仅为本次实物记录；脚本契约使用 LABEL，不能把这些 UUID 当作重建新卡必须保持的值。

格式化使用 mkfs.fat -F 32 -n BOOT，以及 mkfs.f2fs -f -l OVERLAY。完整清盘命令不作为通用部署操作重放；下一次重新分区必须重新确认目标磁盘。

### 7.3 分区与格式化中的两个教训

第一次 fdisk 和对整盘执行 wipefs -n 没有发现分区表或签名，但创建分区后，在旧偏移上发现了 vfat 和 squashfs 残留。整盘检查无输出，不证明任意旧分区偏移都没有签名。

mkfs.f2fs 最初因旧 SquashFS 签名而拒绝格式化。确认目标后，仅对 p2 使用 -f 完成覆盖；不需要再次重建分区表。随后 blkid 和 lsblk 明确显示 BOOT/vfat 和 OVERLAY/f2fs。

读卡器链路提示不支持 BLKSECDISCARD、BLKDISCARD，但最终 format successful。该提示没有阻断格式化和后续实机挂载，也不能据此判定原生 MMC 插槽下是否支持 discard。

日志中的 format version with Linux 7.1.9 是执行格式化的 SandPlug 主机记录，不是要求以后使用该内核。

## 8．从手工验证到自动启动的实验链

### 8.1 内嵌 RAM shell 和单文件加载

最初在 UBOOT 使用以下手工命令进入 PID 1 shell：

```text
mmc rescan
fatload mmc 0:1 0x00800000 /openwrt/lio-uImage
iminfo 0x00800000
setenv bootargs 'console=ttyS0,115200 rdinit=/bin/sh'
bootm 0x00800000
```

iminfo 校验通过；内核识别 Cloud Engines PogoPlug Series 4；shell 中 echo $$ 为 1，uname -r 为 6.12.71。这验证了只加载一个 uImage 就能带起正确内核、DTB 和早期用户空间。

shell 提示不能访问 tty、没有 job control，是这个早期 PID 1 shell 的现象，不表示内核启动失败。

### 8.2 无早期设备管理器条件下访问 SD

在 shell 中挂载 proc、sysfs、临时 tmpfs /dev，创建 console/null，然后加载 mmc_core、mmc_block、mvsdio。通过 /sys/class/block 下的 dev 文件读取主次设备号并 mknod。

实机识别到 119 GiB 的 SDXC 卡，并列出当时的 p1、p2、p3。随后加载 NLS、FAT、vfat、loop 模块，完成 FAT 只读挂载。

这证明早期挂载链不依赖 udev 或 /dev/disk/by-label。当前自动脚本采用相同原则，以 blkid -p 直接探测超级块。正式 OpenWrt 的设备管理由其初始化流程处理；本轮没有单独完成整套 udev 软件包清单鉴定。

### 8.3 SquashFS 尾部截断问题及修复

原始镜像长度为 4,154,394 字节。losetup 提示未按 512 字节扇区结束，loop 容量变为 8,114 个扇区，即 4,154,368 字节。SquashFS 超级块 bytes_used 声明的有效长度仍为 4,154,394，实际 loop 暴露的数据比它少 26 字节。

此时 mount -t squashfs 报 Invalid argument。不是缺少 SquashFS 驱动，也不是 FAT 无法读取镜像。

保留原文件，另外生成补零副本，将长度补到 4,157,440 字节，也就是 4096 字节边界。重新只读绑定后，loop 容量为 8,120 个扇区，挂载成功；读取 OpenWrt 版本并执行其中 BusyBox 程序成功。

因此导出契约采用 4096 字节对齐。不能把某一轮固定长度复制到后续构建。后来的自动启动中 loop 容量为 8,192 个扇区，对应这一轮部署的 4 MiB 对齐镜像。

### 8.4 F2FS 和 overlay 分步验证

先加载 crc32_generic、f2fs，按 sysfs 创建设备节点，将旧 p3 以 ro,norecovery 挂载，查看旧 upper/work 和旧 mount_root 包装脚本。

结束只读检查后，重新可写挂载 F2FS，建立独立 upper-lio-p1、work-lio-p1，挂载 overlay 到 /newroot。在新根内 chroot 执行程序成功，并确认 /sbin/init 可执行、包装脚本可见。

### 8.5 PID 1 手工交接

最初用 busybox --list 检查 switch_root 时，BusyBox 返回 applet not found；该构建没有提供这一枚举调用方式。随后直接执行 busybox switch_root --help，得到了正确用法，证明 switch_root 已存在。

手工实验把存储层和 /dev、/sys、/proc 移入 /newroot，清除 INITRAMFS/PREINIT，使用 exec busybox switch_root -c /dev/console /newroot /sbin/init。后续正式 init、preinit、procd 均成功，PID 1 确认为 procd。

该次实验还成功执行了 mount_root done 的旧包装检查。不能把此证据扩展成新自动版每个调用分支都完成过实机测试。

### 8.6 两层脚本自动启动

随后将 sd-init 编入镜像，增加独立 blkid，并补齐 BusyBox stat -c。FAT 上放置 init，UBOOT 使用新的 uEnv.txt。自动启动完整完成脚本的 7 个和 9 个阶段，再进入 OpenWrt。

首个自动版本虽然成功启动，但在 preinit 中出现 mount_root: unable to create loop device。定位后只修订 FAT init，不重新编译两个镜像；第二次自动启动已正确拦截并消除了该错误。

## 9．当前启动文件和路径契约

| 文件 | 生产或保存位置 | 运行时责任 |
|---|---|---|
| sd-init | 构建树 files/sbin/sd-init | 内嵌 rootfs 中的 /sbin/sd-init，PID 1 入口 |
| lio-uImage | FAT /openwrt/lio-uImage | UBOOT 加载的 initramfs-uImage 副本 |
| lio-root-aligned.squashfs | FAT /openwrt/lio-root-aligned.squashfs | 正式只读 rootfs 文件 |
| FAT init | FAT /boot/init | 组装 loop、SquashFS、F2FS、overlay 并交接 |
| uEnv.txt | FAT /boot/uEnv.txt | 被现有 UBOOT 导入的启动命令 |
| mount_root 包装脚本 | 通过 overlay 合并视图写入 /sbin/mount_root | 阻止已组装根目录被 start/done 重复处理 |
| 官方原版 mount_root | SquashFS 中的 /rom/sbin/mount_root | 保持原程序可供其他操作转发 |

必须分清 FAT 内部路径和 Linux 挂载后的路径：

| FAT 内部路径 | FAT 挂到 Linux /boot 后 |
|---|---|
| /boot/init | /boot/boot/init |
| /boot/uEnv.txt | /boot/boot/uEnv.txt |
| /openwrt/lio-uImage | /boot/openwrt/lio-uImage |
| /openwrt/lio-root-aligned.squashfs | /boot/openwrt/lio-root-aligned.squashfs |

UBOOT 阶段仍固定从 mmc 0:1 加载镜像，不在这个旧版引导器中实现 LABEL 查找。LABEL=BOOT 和 LABEL=OVERLAY 是 Linux 早期脚本的身份契约；不能把这两层的识别方式混为一谈。

### 9.1 sd-init 的职责和接口

sd-init v0.1 共 7 个阶段：

1. 确认 PID 1 和所需工具，实际运行 blkid --version。
2. 准备 proc、sysfs、512 KiB 临时 tmpfs /dev 和 console/null，并接通串口输入输出。
3. 加载 MMC、NLS、FAT 驱动；已有模块不重复加载。
4. 在 15 秒发现窗口内扫描 MMC 分区，读取 LABEL/TYPE，找到唯一 BOOT/vfat；同一轮扫描遇到多个同名标签即拒绝选择。
5. 拒绝在已有 /boot 挂载之上叠加；以 ro,codepage=437,iocharset=iso8859-1 挂载 BOOT。
6. 检查 FAT /boot/init 为可读非空文件并通过 sh -n。
7. 导出 BOOT_DEV、BOOT_MNT=/boot，exec /bin/sh /boot/boot/init，保持 PID 1。

唯一性判断针对每次扫描当时可见的 MMC 分区；不是对未来热插入设备的持续监控。出现失败或意外退出时，trap 保留 PID 1 并打开维修子 shell，不让早期 init 简单退出导致失去控制台。

### 9.2 FAT init 的职责和接口

FAT init v0.2 的配置区：

```text
boot_init_image_rel=/openwrt/lio-root-aligned.squashfs
boot_init_upper_rel=/p1-root/upper
boot_init_work_rel=/p1-root/work
boot_init_wait=15
boot_init_rom=/mnt/root-ro
boot_init_data=/mnt/root-data
boot_init_root=/newroot
```

它继续作为 PID 1 执行 9 个阶段：检查 sd-init 接口和工具；加载 loop/CRC/F2FS；按 LABEL 找到唯一 OVERLAY/f2fs；只读绑定 loop0 并挂载 SquashFS；可写挂载 F2FS 并组装 overlay；生成、比较、安装 mount_root 包装脚本；检查新根、原版工具和控制台；移动存储挂载并正常卸载临时虚拟挂载；exec switch_root 进入正式 init。

loop0 固定使用，加载模块时 max_loop=1。若已经绑定后端则拒绝覆盖。镜像必须是可读非空普通文件，长度按 4096 字节对齐；绑定后核对 loop ro=1。

upper/work 在同一个 F2FS 上创建，不清空历史数据。脚本检查有关路径不是符号链接，且写入包装脚本始终经过 overlay 合并视图，不直接改动已经挂载的底层 upper。

失败路径保留存储挂载，尽量补回维修所需虚拟文件系统，并进入维修循环，不自动重复组装。本轮主要成功路径已有实机证据；所有注入故障和中断时机并未穷尽测试。

### 9.3 存储层移动与虚拟挂载清理

自动版把 BOOT、SquashFS 和 F2FS 分别移动到新根的 /boot、/rom、/overlay。随后将 PID 1 的标准输入输出接到新根的静态 /dev/console，再正常 umount 临时 /dev、/sys、/proc，最后 switch_root。

先切换控制台引用，是为了释放临时 /dev 上的文件描述符引用，避免正常卸载被占用。自动版没有使用强制卸载或懒卸载。

正式 OpenWrt init/procd 自行建立其需要的虚拟挂载。自动实验后曾核对每个关键虚拟挂载只有一层，证明这套清理衔接避免了把临时层与正式层重复叠加。

## 10．mount_root 包装机制及真实故障修复

### 10.1 为什么 /sbin 与 /rom 下不是同一份内容

SquashFS 原版 /sbin/mount_root 不修改。它作为只读层挂到 /rom 后，可通过 /rom/sbin/mount_root 访问。

组装阶段通过 /newroot/sbin/mount_root 写入包装脚本，修改落入 F2FS upper；正式系统的 /sbin/mount_root 因 overlay 规则显示这份包装脚本。因此两个路径可以对应不同内容：一个是 upper 中的 shell 包装，一个是只读原版程序。

包装脚本对已组装根的 start/done 只核对挂载并返回。其他操作使用 exec /rom/sbin/mount_root "$@"，直接指向只读原版，避开再次进入自身的递归路径。这不等于原版程序的所有其他操作都适合本 SD 布局；stop/ram 等实际业务仍需按用途单独评估。

### 10.2 如何摆脱旧 upper 的影响

每次启动都在 RAM 中生成本次预期包装文本，其中包含当次按 LABEL 解析出的 BOOT_DEV 和数据设备。再用 cmp -s 与合并视图中的目标比较：

- 返回 0：内容一致，保留现有文件。
- 返回 1：内容不同，在同目录写临时文件，设为 root:root、0755，再 mv 替换。
- 返回大于 1：比较出错，停止启动，不当作普通内容变化处理。

权限通过 stat -c '%a:%u:%g' 另行核对，必要时修正。因此空 upper 可以启动，旧版包装脚本也会自动更新；内容一致时无需每次重写。

### 10.3 v0.1 的漏接

旧模板采用 case "$#:${1-start}"，只接受 0:start、1:start、1:done。

OpenWrt 25.12.0 的实际 preinit 代码为：

```sh
mount_root start "$(compose_rootfs_mount_options)"
```

没有附加选项时相当于 mount_root start ""。空字符串仍是一个参数，所以实际匹配值为 2:start；旧模板漏接后转发原版，触发了额外的 rootdisk/loop 处理及 unable to create loop device。

此时我们组装的根目录仍然有效，随后 PID 1 和最终挂载核对均正常。错误来自不应再次运行的原版启动处理，不是第 04 步只读 loop 挂载失败。

### 10.4 v0.2 的处理

按第一个参数识别 start/done，再检查参数：无参数、start、start 加一个空参数均允许；start 的第二参数非空或参数过多则明确返回错误；done 不接受附加参数。挂载状态符合预期才返回 0，其他操作保留原样转发。

修改后的模板只需要更新 FAT /boot/init，两个镜像无需重编译。下一次启动会由 cmp 机制更新 upper 中的包装脚本。

隔离测试覆盖无参数、start、start 空参数、done、拒绝非空选项/多余参数、错误挂载条件以及参数原样转发；bash、dash 语法与测试均通过。这些是开发环境中的测试，不替代实机分支验证。

第二次自动启动日志明确出现：

```text
[p1-root] mount_root start: root already assembled
```

旧 loop 错误消失，OpenWrt 继续进入 procd 和 LAN 桥。这是 v0.2 修复的实机成功依据。

包装脚本返回非零本身不保证 OpenWrt 上层一定中止所有初始化；后续如果增加额外挂载选项要求，应同时设计组装阶段怎样兑现及怎样报告失败，不能只依赖 preinit 调用方传播错误。

## 11．最终运行证据与证据时点

### 11.1 首次自动启动后的直接核对

第一次自动启动后，虽然当时还存在旧模板漏接，直接核对已经确认：

```text
/proc/1/comm = procd
/dev/mmcblk0p1 /boot vfat ro,...
/dev/loop0 /rom squashfs ro,...
/dev/mmcblk0p2 /overlay f2fs rw,...
overlay / overlay rw,...
loop0 backing_file = /boot/openwrt/lio-root-aligned.squashfs
loop0 ro = 1
```

overlay 选项中的原始路径为：

```text
lowerdir=/mnt/root-ro
upperdir=/mnt/root-data/p1-root/upper
workdir=/mnt/root-data/p1-root/work
```

移动底层挂载后，这些创建时的路径字符串仍显示在挂载选项中，不表示运行时必须靠旧路径重新查找这些目录。

同次 /proc/self/mountinfo 中，/proc、/sys、/tmp、/dev 各出现一层。该结果证明自动版虚拟挂载清理方案在这次启动中成功。

### 11.2 v0.2 再次自动启动

修订后日志记录：约 3.14 秒执行 sd-init；约 7.48 秒进入正式 init；约 18.59 秒成功拦截 mount_root start；约 19.80 秒进入 procd init；约 36.14 秒 LAN 桥进入 forwarding。

这些时间从 Linux 启动计时，不包含前面 UBOOT 的网络 ping 和设备扫描耗时。它们是本次样本，不是性能保证。

v0.2 的 start 包装检查同时确认根 overlay、只读 /rom、可写 /overlay、只读 /boot 的组合符合预期。该次后续没有另行提交新的完整 mountinfo 或 done 调用输出；不要把上轮直接检查的时点写成第二次都重新测过。

### 11.3 内存与重复 rootfs 的认识

initramfs 与 SquashFS 主要从相同根目录拼装来源生成，所以用户空间内容大部分重复；这是当前复用官方构建产物的结果，并非启动原理要求必须携带两份完整用户空间。

内核日志中的 Freeing unused kernel image (initmem) memory: 13312K，发生在执行 sd-init 之前，不能等同于完整 initramfs 用户空间已释放。switch_root 清理旧 rootfs 文件后，可以释放不再被引用的早期文件占用；内核、已加载模块及其他运行对象仍然存在。

本轮没有进行交接前后内存占用的对照测量，也没有取得精简 initramfs 的稳态或峰值收益数字。优化另立专项。

## 12．日志提示、已解决问题与待处理事项

| 提示或现象 | 本轮判断与状态 |
|---|---|
| feeds 初期缺 ruby/host、libpam 等 | feeds 尚未完整安装时存在；安装后多数消失 |
| mjpg-streamer、python3-pysocks、python3-unidecode 缺失 | 最后仍见于未选中的 luci-app-mjpg-streamer/onionshare-cli 定义；未阻断本轮构建 |
| 检查这些未选包在 .config 中无输出 | 表示没有匹配到启用项，不能解释为固件构建失败 |
| early 配置中缺 loop/FAT | 补齐软件包并重建后，实际模块已进入根目录且实机可加载 |
| blkid not found | 旧镜像确实没有独立 blkid；新构建增加后自动 LABEL 识别成功 |
| BusyBox --list 不可用 | 改用具体 applet --help 确认；switch_root 已存在 |
| BusyBox STAT 缺失 | 增加 STAT、FEATURE_STAT_FORMAT 并重建，最终 stat -c 可用 |
| SquashFS Invalid argument | 原文件未对齐、loop 截断有效尾部；副本补零后成功 |
| mount_root unable to create loop device | v0.1 漏接 start 空第二参数；v0.2 实机已修复 |
| UBOOT ping 172.16.16.118 失败 | 外层旧环境行为，增加启动等待；不影响后来新入口成功 |
| USB/IDE/不存在的 MMC 扫描失败、缺 /boot/uImage | 外层 scan_disk 仍在执行；随后已加载 /openwrt/lio-uImage |
| UBOOT Caches not enabled | 出现在引导器阶段；Linux 随后明确启用 Feroceon L2，不能据此判定运行中的 Linux 缓存被关闭 |
| UBI no valid magic inside mtd2 | 当前 NAND 中没有成功连接所需 UBI，SD 启动仍成功；没有刷写 NAND 来消除提示 |
| 缺 fw_env.config、Failed to find NVMEM device | 初次自动启动出现；后续已有环境访问配置，具体调用链未完整定位 |
| Bad CRC, using default environment | 后续 Linux preinit 出现；需要确认当前 bodhi UBOOT 的实际环境位置，尚未处理 |
| LVM descriptor 3 /dev/watchdog leaked | 子进程继承文件描述符的警告；不是内存泄漏证据，本轮未修订 |
| 第一次缺 urandom.seed | 后续启动已能读取该文件；随机数初始化和 urngd 也有成功日志 |
| gpio_button_hotplug taints kernel | 外部模块加载提示，未阻断启动 |
| SATA link down | 在没有连接对应磁盘的实验条件下出现；不构成 SD 链路失败 |
| rtc internal RTC not ticking | RTC 提示仍存在，本轮没有修复硬件时钟或证明时间同步策略完备 |
| mmc non-supported SD ext reg layout | SD 随后完成识别、分区访问和启动；仍应保留原始日志，不直接推断所有扩展能力均可用 |
| db_root cannot open /etc/target | 与内建 LIO 的配置目录有关；尚未完成 target/LUN 配置与服务验证 |
| 首次无 root 密码提示 | 初始配置状态；本文不把密码、SSH 或 LAN 服务完整可用写成已验证成果 |

## 13．NAND 布局及环境 CRC 问题的当前结论

本轮研究用于解释警告，没有实施官方 NAND 安装。

### 13.1 OpenWrt 定义的 NAND 布局

| MTD | 名称 | 起始地址 | 结束地址（不含） | 大小 |
|---|---|---|---|---|
| mtd0 | uboot | 0x00000000 | 0x001c0000 | 1.75 MiB |
| mtd1 | uboot_env | 0x001c0000 | 0x00200000 | 256 KiB |
| mtd2 | ubi | 0x00200000 | 0x08000000 | 126 MiB |

配套 UBOOT 的环境偏移为 0x1c0000，环境长度 0x20000，擦除块 0x20000。OpenWrt 按设备型号生成的访问配置为：

```text
/dev/mtd1 0x0 0x20000 0x20000
```

这份配置当前确实存在；它是按官方布局预设，不是发现有效环境后才生成。对 /lib/preinit 直接 grep 环境工具没有输出，仅表示这些文件没有直接匹配文本，不能排除间接调用。

PogoV4 的其他 UBOOT 布局存在将环境放在 0x000c0000 的历史。当前保留的 2017.07-tld-1 是否确实使用这一位置，尚未只读确认。现有 CRC 警告可能来自读取位置或长度不匹配，不能认定真实环境已经损坏。

FAT uEnv.txt 已正确导入与 Linux 环境工具读 NAND CRC 失败是两个不同问题。不能为了消除提示而执行 saveenv、fw_setenv 或重写 NAND 环境。

### 13.2 官方 NAND 启动与当前 SD 的区别

官方常规方案在 mtd2 的 UBI 中存放 kernel、rootfs、rootfs_data：kernel 包含普通 uImage，rootfs 是 SquashFS，可写层通常为 UBIFS。UBOOT 从 UBI kernel 卷读到 RAM 后 bootm，Linux 挂载 UBI/ubiblock 上的根目录，preinit 再准备 overlay。

官方也提供 initramfs-uImage，供从 USB FAT 等介质临时启动 RAM 系统后安装。PogoV4 的官方镜像规则没有提供本项目这种现成 SD 分区镜像。

系统升级镜像与配套 UBOOT、环境分别管理，不能假定刷入系统固件会迁移旧环境。我们当前复用了官方 initramfs-uImage 的构建过程，并增加 SD 根目录衔接，没有在 NAND 上验证这套官方安装链。

## 14．已经验证与尚未验证的边界

### 14.1 可以作为后续基础的能力

已证明：完整 OpenWrt 自定义内核可在该设备启动；必要驱动和用户工具能够提前存在于 initramfs；SD FAT 文件镜像可作为正式只读 rootfs；F2FS 可作为 overlay 可写层；PID 1 能通过两层脚本交接给正式系统；正常自动启动不依赖旧 upper 预置业务；本次 mount_root 参数接口遗漏已定位并修复。

### 14.2 不能提前宣称完成的工作

- 两个最小片段从干净源码的完整重建与实机回归。
- GitHub Actions 的首次成功构建、资源峰值、缓存与产物交付。
- LVM 实际 PV/VG/LV 激活、设备节点和稳定身份验证。
- LIO 的 configfs 对象、iblock 后端、LUN、ACL、网络连接和读写服务测试。
- 无卡、镜像损坏、标签重复、模块缺失、挂载失败等全部故障注入及维修路径实机测试。
- 长时间运行、异常掉电、F2FS 恢复、SD 寿命与写入量测量。
- 对新内核/新版 OpenWrt 的兼容性或任意硬件差异的覆盖。
- NAND 环境 CRC 的根因与修复。
- 精简 initramfs 的实现和收益测量。

早期脚本能够创建设备节点，并不等于 LIO 后续一定能找到任意目标磁盘或逻辑卷。该部分需要正式系统服务顺序和稳定设备身份的另一次设计与验证。

## 15．当前文件版本、身份和换页接口

| 文件 | 当前版本或身份 |
|---|---|
| sd-init | v0.1，192 行，7564 字节 |
| FAT init | v0.2，331 行；归档文件名 pogo-p1-init，部署时命名 init |
| uEnv.txt | 本轮 SD FAT 单文件加载版本，711 字节 |
| openwrt.config | 12 项最小业务输入 |
| kernel.config | 3 项最小业务输入 |

文件摘要见附录 E，完整脚本和两个片段均在本文中原文展开。源码注释中的“BOOT 始终只读”指正常自动启动业务；维护时更新 FAT 文件需要由维护者另行安排受控写入，不由启动脚本自动执行。

优先下一步是按照《github工作流设计》实现干净构建和产物回归。环境 CRC 可作为独立事项继续只读确认；LIO/LVM 服务编排在启动底座稳定后开展。最小早期启动环境优化已明确降为低优先级专项。

## 16．证据索引与参考

### 16.1 本轮原始记录

| 证据 | 内容 |
|---|---|
| 聊天中的源码 clone、prereq、内核编译及模块核对输出 | 初始构建基线与 LIO/存储配置 |
| 粘贴的文本 (1)(20261018-053506).txt | feeds 安装及设备配置 |
| 粘贴的文本 (1)(20261018-193358).txt | 较早一轮完整镜像构建末尾 |
| 粘贴的文本 (1)(20261018-205724).txt | 增加 FAT/loop 工具后的构建记录 |
| 聊天中的手工挂载、hexdump、补零、chroot、switch_root 输出 | 单项链路验证、未对齐故障与修复、进入正式系统 |
| 聊天中的 SandPlug 分区、格式化和 blkid 输出 | 当前两分区布局及身份 |
| 粘贴的文本 (1)(20261019-041040).txt | 加入 sd-init/blkid/stat 后完整构建末尾 |
| 粘贴的文本 (1)(20261019-045632).txt | 第一次自动启动，含旧 mount_root 漏接 |
| 聊天中的 PID 1、挂载、loop、包装脚本和 preinit 核对 | 第一次自动启动后的直接运行状态 |
| 粘贴的文本 (1)(20261019-051508).txt | v0.2 修复后的再次自动启动 |
| 聊天中的 /proc/mtd、fw_env.config 输出 | 环境 CRC 访问布局取证 |
| 粘贴的文本 (1)(20261019-055731).txt | 最终源码改动、配置、feeds、sd-init 身份取证 |

以上文件名用于追溯原始记录。本文摘录了继续工作所需的主要事实，不假定下一页仍可访问原聊天附件。

### 16.2 官方代码依据

- [OpenWrt v25.12.0 的 preinit/80_mount_root](https://github.com/openwrt/openwrt/blob/v25.12.0/package/base-files/files/lib/preinit/80_mount_root)：start 的第二参数调用形式。
- [OpenWrt v25.12.0 的 include/target.mk](https://github.com/openwrt/openwrt/blob/v25.12.0/include/target.mk)：env/kernel-config 入口。
- [OpenWrt v25.12.0 的 include/kernel-defaults.mk](https://github.com/openwrt/openwrt/blob/v25.12.0/include/kernel-defaults.mk)：内核配置与 initramfs 构建。
- [OpenWrt v25.12.0 的 Kirkwood 镜像规则](https://github.com/openwrt/openwrt/blob/v25.12.0/target/linux/kirkwood/image/Makefile)：内核、DTB 和 uImage 封装。
- [Linux v6.12 的 target/Kconfig](https://github.com/torvalds/linux/blob/v6.12/drivers/target/Kconfig)：LIO 核心、iblock 与 CONFIGFS 选择关系。
- [OpenWrt PogoV4 设备说明](https://openwrt.org/toh/cloud_engines/pogo-v4)：官方安装链及 NAND 使用方式。
- [OpenWrt 最初 PogoV4 支持提交](https://lists.infradead.org/pipermail/lede-commits/2018-January/005615.html)：官方 NAND 分区定义的历史依据。
- [OpenWrt 配套 PogoV4 UBOOT 补丁](https://git.openwrt.org/openwrt/staging/adrian/tree/package/boot/uboot-kirkwood/patches/010-pogoplug_v4.patch?h=rft-devices)：环境偏移与 UBI kernel 启动方式的代码依据；该链接为历史源码参考，不替代第 3 节的本轮构建基线。

## 附录 A．sd-init v0.1 完整归档

安装到构建树 files/sbin/sd-init，权限 0755。以下内容由本轮归档文件直接附入。

```sh
#!/bin/sh
# sd-init v0.1：仅挂载 LABEL=BOOT，并交接给 FAT 分区内 /boot/init。
# 安装位置：OpenWrt 源码树 files/sbin/sd-init；文件权限 0755。
# 运行环境：BusyBox ash、util-linux blkid、配套 MMC/FAT 内核模块。
# 不使用函数；不写入 SD；不启动后台设备管理程序。

PATH=/usr/sbin:/usr/bin:/sbin:/bin
LC_ALL=C
export PATH LC_ALL
umask 022

if [ "$$" -ne 1 ]; then
    printf '[sd-init] 必须由内核作为 PID 1 调用，停止执行。\n' >&2
    exit 1
fi

set -u
sd_init_stage='入口检查'
sd_init_error='尚未完成向 FAT 脚本的交接'

# exec 成功时本 trap 不运行；失败或意外退出时保留 PID 1。
trap '
    sd_init_status=$?
    trap - 0 HUP INT TERM
    set +e
    set +u
    printf "\n[sd-init] 启动停止：阶段=%s；状态=%s\n" \
        "$sd_init_stage" "$sd_init_status" >&2
    printf "[sd-init] 原因：%s\n" "$sd_init_error" >&2
    printf "[sd-init] 进入串口维修循环；退出子 shell 后仍保留 PID 1。\n" >&2
    while :; do
        if [ -c /dev/console ]; then
            /bin/sh -i < /dev/console > /dev/console 2>&1
        else
            /bin/sh -i
        fi
        printf "\n[sd-init] 维修 shell 已退出，稍后重新打开。\n" >&2
        sleep 1
    done
' 0
trap 'sd_init_error="启动收到中断信号"; exit 1' HUP INT TERM

printf '\n[sd-init 01/07] 检查启动工具\n'
for sd_init_tool in mkdir mount mknod chmod awk insmod uname tr sleep; do
    if ! sd_init_tool_path=$(command -v "$sd_init_tool" 2>&1); then
        sd_init_error="缺少启动工具：$sd_init_tool"
        exit 1
    fi
done
if [ ! -x /usr/sbin/blkid ]; then
    sd_init_error='缺少 /usr/sbin/blkid，请在构建时选入 blkid 软件包及依赖'
    exit 1
fi
if ! sd_init_blkid_version=$(/usr/sbin/blkid --version 2>&1); then
    printf '%s\n' "$sd_init_blkid_version" >&2
    sd_init_error='blkid 无法运行，请检查其动态库和软件包完整性'
    exit 1
fi

sd_init_stage='准备临时虚拟文件系统'
sd_init_error='无法准备 proc、sysfs 或临时设备目录'
printf '\n[sd-init 02/07] 准备临时虚拟文件系统和控制台\n'
mkdir -p /proc /sys /dev /boot || exit 1
if [ ! -r /proc/self/mounts ]; then
    mount -t proc proc /proc || exit 1
fi
if ! awk '$2=="/sys" && $3=="sysfs" {ok=1} END {exit !ok}' \
    /proc/self/mounts; then
    if awk '$2=="/sys" {ok=1} END {exit !ok}' /proc/self/mounts; then
        sd_init_error='/sys 已被其他文件系统占用'
        exit 1
    fi
    mount -t sysfs sysfs /sys || exit 1
fi
if ! awk '$2=="/dev" && $3=="tmpfs" {ok=1} END {exit !ok}' \
    /proc/self/mounts; then
    if awk '$2=="/dev" {ok=1} END {exit !ok}' /proc/self/mounts; then
        sd_init_error='/dev 已被其他文件系统占用'
        exit 1
    fi
    mount -t tmpfs -o mode=0755,nosuid,size=512k tmpfs /dev || exit 1
fi
[ -c /dev/console ] || mknod /dev/console c 5 1 || exit 1
[ -c /dev/null ] || mknod /dev/null c 1 3 || exit 1
chmod 0600 /dev/console || exit 1
chmod 0666 /dev/null || exit 1
sd_init_error='无法连接 /dev/console'
exec < /dev/console > /dev/console 2>&1

sd_init_stage='加载 MMC 和 FAT 驱动'
printf '\n[sd-init 03/07] 加载 MMC、字符集和 FAT 驱动\n'
sd_init_modules="/lib/modules/$(uname -r)"
for sd_init_module in mmc_core mmc_block mvsdio nls_base \
    nls_cp437 nls_iso8859-1 fat vfat; do
    sd_init_sysname=$(printf '%s' "$sd_init_module" | tr '-' '_')
    [ -d "/sys/module/$sd_init_sysname" ] && continue
    if [ ! -f "$sd_init_modules/$sd_init_module.ko" ]; then
        # nls_base 可以内建且没有对应模块文件。
        [ "$sd_init_module" = nls_base ] && continue
        sd_init_error="缺少内核模块：$sd_init_modules/$sd_init_module.ko"
        exit 1
    fi
    sd_init_error="内核模块加载失败：$sd_init_module"
    insmod "$sd_init_modules/$sd_init_module.ko" || exit 1
done

sd_init_stage='发现 LABEL=BOOT'
printf '\n[sd-init 04/07] 等待唯一的 LABEL=BOOT，发现窗口为 15 秒\n'
sd_init_error='无法读取内核运行时间'
IFS=' ' read -r sd_init_uptime sd_init_unused < /proc/uptime || exit 1
sd_init_deadline=$(( ${sd_init_uptime%%.*} + 15 ))
BOOT_DEV=
while :; do
    sd_init_matches=0
    BOOT_DEV=
    for sd_init_partition in /sys/class/block/mmcblk*p*; do
        [ -f "$sd_init_partition/partition" ] || continue
        sd_init_name=${sd_init_partition##*/}
        sd_init_device="/dev/$sd_init_name"
        sd_init_error="无法准备设备节点：$sd_init_device"
        IFS=: read -r sd_init_major sd_init_minor \
            < "$sd_init_partition/dev" || exit 1
        if [ ! -b "$sd_init_device" ]; then
            if [ -e "$sd_init_device" ] || [ -L "$sd_init_device" ]; then
                sd_init_error="设备路径已被非块设备占用：$sd_init_device"
                exit 1
            fi
            mknod "$sd_init_device" b "$sd_init_major" "$sd_init_minor" || exit 1
        fi
        # 直接读取超级块；不使用缓存或 /dev/disk/by-label。
        sd_init_label=$(/usr/sbin/blkid -p -s LABEL -o value \
            "$sd_init_device" 2>/dev/null)
        [ "$?" -eq 0 ] || continue
        [ "$sd_init_label" = BOOT ] || continue
        sd_init_error="无法确定 BOOT 文件系统类型：$sd_init_device"
        sd_init_type=$(/usr/sbin/blkid -p -s TYPE -o value \
            "$sd_init_device" 2>/dev/null) || exit 1
        if [ "$sd_init_type" != vfat ]; then
            sd_init_error="LABEL=BOOT 的类型不是 vfat：$sd_init_device ($sd_init_type)"
            exit 1
        fi
        sd_init_matches=$((sd_init_matches + 1))
        BOOT_DEV=$sd_init_device
    done
    if [ "$sd_init_matches" -gt 1 ]; then
        sd_init_error='发现多个 LABEL=BOOT 的 MMC 分区，拒绝选择'
        exit 1
    fi
    [ "$sd_init_matches" -eq 1 ] && break
    sd_init_error='无法读取内核运行时间'
    IFS=' ' read -r sd_init_uptime sd_init_unused < /proc/uptime || exit 1
    if [ "${sd_init_uptime%%.*}" -ge "$sd_init_deadline" ]; then
        sd_init_error='等待结束，仍未发现 LABEL=BOOT 的 MMC 分区'
        exit 1
    fi
    sleep 1 || exit 1
done
printf '[sd-init] BOOT 设备：%s\n' "$BOOT_DEV"

sd_init_stage='只读挂载 BOOT'
printf '\n[sd-init 05/07] 只读挂载 BOOT 到 /boot\n'
BOOT_MNT=/boot
if awk '$2=="/boot" {ok=1} END {exit !ok}' /proc/self/mounts; then
    sd_init_error='/boot 已有挂载，拒绝重复叠加'
    exit 1
fi
sd_init_error='BOOT 的 FAT 文件系统挂载失败'
mount -t vfat -o ro,codepage=437,iocharset=iso8859-1 \
    "$BOOT_DEV" "$BOOT_MNT" || exit 1
sd_init_error='BOOT 挂载结果不符合 vfat、只读和设备来源约定'
awk -v expected="$BOOT_DEV" '
    $1==expected && $2=="/boot" && $3=="vfat" &&
        $4 ~ /(^|,)ro(,|$)/ {ok=1}
    END {exit !ok}
' /proc/self/mounts || exit 1

sd_init_stage='检查 FAT 启动脚本'
printf '\n[sd-init 06/07] 检查 FAT 分区内 /boot/init\n'
sd_init_script="$BOOT_MNT/boot/init"
if [ ! -f "$sd_init_script" ] || [ ! -r "$sd_init_script" ] || \
    [ ! -s "$sd_init_script" ]; then
    sd_init_error="启动脚本不是可读的非空普通文件：$sd_init_script"
    exit 1
fi
sd_init_error="FAT 启动脚本语法检查失败：$sd_init_script"
/bin/sh -n "$sd_init_script" || exit 1

sd_init_stage='交接给 FAT 启动脚本'
sd_init_error="无法执行 FAT 启动脚本：$sd_init_script"
printf '\n[sd-init 07/07] PID 1 交接给 %s\n' "$sd_init_script"
export BOOT_DEV BOOT_MNT
exec /bin/sh "$sd_init_script"

```

## 附录 B．FAT /boot/init v0.2 完整归档

部署为 FAT 分区内部的 /boot/init。正式系统中该文件可见于 /boot/boot/init；它由 sd-init 在早期执行，不作为普通运行时服务手工重跑。

```sh
#!/bin/sh
# FAT /boot/init v0.2；由 sd-init 以 /bin/sh 执行，继续作为 PID 1。
# FAT 根目录挂在 /boot，因此运行路径是 /boot/boot/init。
# 不含函数；不格式化、不清空 upper；BOOT 始终只读。

PATH=/usr/sbin:/usr/bin:/sbin:/bin
LC_ALL=C
export PATH LC_ALL
umask 022
if [ "$$" -ne 1 ]; then
    printf '[boot-init] 必须作为 PID 1 执行，停止操作。\n' >&2
    exit 1
fi
set -u
boot_init_stage='入口检查'
boot_init_error='尚未完成正式根目录交接'

# 只在此处调整镜像路径和可写层目录；不引用以前的实验 upper。
boot_init_image_rel=/openwrt/lio-root-aligned.squashfs
boot_init_upper_rel=/p1-root/upper
boot_init_work_rel=/p1-root/work
boot_init_wait=15
boot_init_rom=/mnt/root-ro
boot_init_data=/mnt/root-data
boot_init_root=/newroot

# 失败后保留存储挂载，仅补回维修需要的临时虚拟文件系统。
trap '
    boot_init_status=$?
    trap - 0 HUP INT TERM
    set +e
    set +u
    printf "\n[boot-init] 启动停止：阶段=%s；状态=%s\n" \
        "$boot_init_stage" "$boot_init_status" >&2
    printf "[boot-init] 原因：%s\n" "$boot_init_error" >&2
    cd /
    if [ ! -r /proc/self/mounts ]; then
        mount -t proc proc /proc
    fi
    if [ -r /proc/self/mounts ]; then
        if ! awk '\''$2=="/sys" {ok=1} END {exit !ok}'\'' /proc/self/mounts; then
            mount -t sysfs sysfs /sys
        fi
        if ! awk '\''$2=="/dev" {ok=1} END {exit !ok}'\'' /proc/self/mounts; then
            mount -t tmpfs -o mode=0755,nosuid,size=512k tmpfs /dev
        fi
    fi
    [ -c /dev/console ] || mknod /dev/console c 5 1
    [ -c /dev/null ] || mknod /dev/null c 1 3
    printf "[boot-init] 进入串口维修循环；存储挂载保留，不自动重试拼装。\n" >&2
    while :; do
        /bin/sh -i
        printf "\n[boot-init] 维修 shell 已退出，稍后重新打开。\n" >&2
        sleep 1
    done
' 0
trap 'boot_init_error="启动收到中断信号"; exit 1' HUP INT TERM

printf '\n[boot-init 01/09] 检查 sd-init 交接状态和工具\n'
if [ "${BOOT_MNT:-}" != /boot ] || [ -z "${BOOT_DEV:-}" ] || \
    [ ! -b "${BOOT_DEV:-}" ]; then
    boot_init_error='BOOT_DEV 或 BOOT_MNT 不符合交接约定'
    exit 1
fi
for boot_init_tool in awk mkdir mount umount insmod uname tr mknod \
    chmod chown stat wc cmp cat mv rm sleep sync; do
    if ! boot_init_tool_path=$(command -v "$boot_init_tool" 2>&1); then
        boot_init_error="缺少启动工具：$boot_init_tool"
        exit 1
    fi
done
if [ ! -x /usr/sbin/blkid ] || [ ! -x /usr/sbin/losetup ]; then
    boot_init_error='缺少 /usr/sbin/blkid 或 /usr/sbin/losetup'
    exit 1
fi
boot_init_error='BOOT 或临时虚拟文件系统不符合交接约定'
awk -v boot="$BOOT_DEV" '
    $1==boot && $2=="/boot" && $3=="vfat" && $4 ~ /(^|,)ro(,|$)/ {b=1}
    $2=="/proc" && $3=="proc" {p=1}
    $2=="/sys" && $3=="sysfs" {s=1}
    $2=="/dev" && $3=="tmpfs" {d=1}
    END {exit !(b && p && s && d)}
' /proc/self/mounts || exit 1
boot_init_help=$(/bin/busybox switch_root --help 2>&1)
case "$boot_init_help" in
    *'Usage: switch_root '*) ;;
    *) boot_init_error='BusyBox 缺少可用的 switch_root'; exit 1 ;;
esac
for boot_init_path in "$boot_init_rom" "$boot_init_data" "$boot_init_root"; do
    if awk -v p="$boot_init_path" '$2==p {ok=1} END {exit !ok}' /proc/self/mounts; then
        boot_init_error="目标已有挂载，拒绝重复叠加：$boot_init_path"
        exit 1
    fi
done
cd / || exit 1

boot_init_stage='加载 loop 和 F2FS 驱动'
printf '\n[boot-init 02/09] 加载 loop、CRC32 和 F2FS 驱动\n'
boot_init_modules="/lib/modules/$(uname -r)"
for boot_init_module in loop crc32_generic f2fs; do
    [ -d "/sys/module/$boot_init_module" ] && continue
    boot_init_error="无法加载模块：$boot_init_module"
    if [ "$boot_init_module" = loop ]; then
        insmod "$boot_init_modules/loop.ko" max_loop=1 || exit 1
    else
        insmod "$boot_init_modules/$boot_init_module.ko" || exit 1
    fi
done

boot_init_stage='发现 LABEL=OVERLAY'
printf '\n[boot-init 03/09] 等待唯一的 LABEL=OVERLAY\n'
boot_init_error='无法读取内核运行时间'
IFS=' ' read -r boot_init_uptime boot_init_unused < /proc/uptime || exit 1
boot_init_deadline=$(( ${boot_init_uptime%%.*} + boot_init_wait ))
boot_init_overlay_dev=
while :; do
    boot_init_matches=0
    for boot_init_partition in /sys/class/block/mmcblk*p*; do
        [ -f "$boot_init_partition/partition" ] || continue
        boot_init_name=${boot_init_partition##*/}
        boot_init_device="/dev/$boot_init_name"
        boot_init_error="无法准备设备节点：$boot_init_device"
        IFS=: read -r boot_init_major boot_init_minor \
            < "$boot_init_partition/dev" || exit 1
        if [ ! -b "$boot_init_device" ]; then
            if [ -e "$boot_init_device" ] || [ -L "$boot_init_device" ]; then
                boot_init_error="设备路径被非块设备占用：$boot_init_device"
                exit 1
            fi
            mknod "$boot_init_device" b "$boot_init_major" "$boot_init_minor" || exit 1
        fi
        boot_init_label=$(/usr/sbin/blkid -p -s LABEL -o value \
            "$boot_init_device" 2>/dev/null)
        [ "$?" -eq 0 ] || continue
        [ "$boot_init_label" = OVERLAY ] || continue
        boot_init_error="无法确定 OVERLAY 类型：$boot_init_device"
        boot_init_type=$(/usr/sbin/blkid -p -s TYPE -o value \
            "$boot_init_device" 2>/dev/null) || exit 1
        if [ "$boot_init_type" != f2fs ]; then
            boot_init_error="LABEL=OVERLAY 的类型不是 f2fs：$boot_init_device"
            exit 1
        fi
        boot_init_matches=$((boot_init_matches + 1))
        boot_init_overlay_dev=$boot_init_device
    done
    if [ "$boot_init_matches" -gt 1 ]; then
        boot_init_error='发现多个 LABEL=OVERLAY 的 MMC 分区，拒绝选择'
        exit 1
    fi
    [ "$boot_init_matches" -eq 1 ] && break
    boot_init_error='无法读取内核运行时间'
    IFS=' ' read -r boot_init_uptime boot_init_unused < /proc/uptime || exit 1
    if [ "${boot_init_uptime%%.*}" -ge "$boot_init_deadline" ]; then
        boot_init_error='等待结束，仍未发现 LABEL=OVERLAY'
        exit 1
    fi
    sleep 1 || exit 1
done
printf '[boot-init] OVERLAY 设备：%s\n' "$boot_init_overlay_dev"

boot_init_stage='只读挂载根镜像'
printf '\n[boot-init 04/09] 只读绑定 loop0 并挂载 SquashFS\n'
boot_init_image="$BOOT_MNT$boot_init_image_rel"
if [ ! -f "$boot_init_image" ] || [ ! -r "$boot_init_image" ] || [ ! -s "$boot_init_image" ]; then
    boot_init_error="根镜像不是可读的非空普通文件：$boot_init_image"
    exit 1
fi
boot_init_error='无法读取根镜像长度'
boot_init_image_size=$(wc -c < "$boot_init_image") || exit 1
if [ "$((boot_init_image_size % 4096))" -ne 0 ]; then
    boot_init_error="根镜像未按 4096 字节对齐：$boot_init_image_size"
    exit 1
fi
boot_init_error='loop0 设备不可用'
IFS=: read -r boot_init_major boot_init_minor < /sys/class/block/loop0/dev || exit 1
if [ ! -b /dev/loop0 ]; then
    [ ! -e /dev/loop0 ] && [ ! -L /dev/loop0 ] || exit 1
    mknod /dev/loop0 b "$boot_init_major" "$boot_init_minor" || exit 1
fi
if /usr/sbin/losetup /dev/loop0 > /dev/null 2>&1; then
    boot_init_error='loop0 已绑定后端，拒绝拆除或覆盖'
    exit 1
fi
boot_init_error='无法只读绑定或挂载根镜像'
/usr/sbin/losetup --read-only /dev/loop0 "$boot_init_image" || exit 1
IFS= read -r boot_init_loop_ro < /sys/class/block/loop0/ro || exit 1
[ "$boot_init_loop_ro" = 1 ] || exit 1
mkdir -p "$boot_init_rom" "$boot_init_data" "$boot_init_root" || exit 1
mount -t squashfs -o ro /dev/loop0 "$boot_init_rom" || exit 1

boot_init_stage='组装可写根目录'
printf '\n[boot-init 05/09] 挂载 F2FS 并组装 overlay\n'
boot_init_error='无法挂载 OVERLAY 分区'
mount -t f2fs -o rw "$boot_init_overlay_dev" "$boot_init_data" || exit 1
boot_init_upper="$boot_init_data$boot_init_upper_rel"
boot_init_work="$boot_init_data$boot_init_work_rel"
boot_init_error='可写层路径被符号链接占用'
for boot_init_path in "$boot_init_data${boot_init_upper_rel%/*}" \
    "$boot_init_data${boot_init_work_rel%/*}" "$boot_init_upper" "$boot_init_work"; do
    [ ! -L "$boot_init_path" ] || exit 1
done
boot_init_error='无法创建或挂载可写层目录'
mkdir -p "$boot_init_upper" "$boot_init_work" || exit 1
mount -t overlay -o "lowerdir=$boot_init_rom,upperdir=$boot_init_upper,workdir=$boot_init_work" \
    overlay "$boot_init_root" || exit 1

boot_init_stage='安装 mount_root 包装脚本'
printf '\n[boot-init 06/09] 核对并安装 mount_root 包装脚本\n'
boot_init_error='无法准备包装脚本模板'
mkdir -p /tmp/boot-init || exit 1
boot_init_expected=/tmp/boot-init/mount_root.expected
# WRAPPER BEGIN
{
    printf '#!/bin/sh\n'
    printf "boot_root_boot='%s'\n" "$BOOT_DEV"
    printf "boot_root_data='%s'\n" "$boot_init_overlay_dev"
    cat <<'BOOT_ROOT_WRAPPER'
case "${1-start}" in
    start|done)
        # OpenWrt preinit 会调用 mount_root start ""，空第二参数也计入 $#。
        if [ "${1-start}" = start ]; then
            if [ "$#" -gt 2 ] || [ -n "${2-}" ]; then
                printf '[p1-root] mount_root start: 参数过多或挂载选项非空；当前根已提前组装，拒绝忽略选项或再次组装。\n' >&2
                exit 2
            fi
        elif [ "$#" -ne 1 ]; then
            printf '[p1-root] mount_root done: 不接受额外参数。\n' >&2
            exit 2
        fi
        if awk -v boot="$boot_root_boot" -v data="$boot_root_data" '
            $2=="/" && $3=="overlay" && $4 ~ /(^|,)rw(,|$)/ {r=1}
            $1=="/dev/loop0" && $2=="/rom" && $3=="squashfs" &&
                $4 ~ /(^|,)ro(,|$)/ {l=1}
            $1==data && $2=="/overlay" && $3=="f2fs" &&
                $4 ~ /(^|,)rw(,|$)/ {d=1}
            $1==boot && $2=="/boot" && $3=="vfat" &&
                $4 ~ /(^|,)ro(,|$)/ {b=1}
            END {exit !(r && l && d && b)}
        ' /proc/self/mounts; then
            printf '[p1-root] mount_root %s: root already assembled\n' "${1-start}"
            exit 0
        fi
        printf '[p1-root] unexpected root mounts; mount_root stopped\n' >&2
        exit 1
        ;;
esac
exec /rom/sbin/mount_root "$@"
BOOT_ROOT_WRAPPER
} > "$boot_init_expected" || exit 1
/bin/sh -n "$boot_init_expected" || exit 1
# WRAPPER END

# INSTALL BEGIN：所有写入经过合并视图，绝不直接修改已挂载的 upper。
boot_init_target="$boot_init_root/sbin/mount_root"
boot_init_error='mount_root 目标不是普通文件'
if [ -L "$boot_init_target" ] || { [ -e "$boot_init_target" ] && [ ! -f "$boot_init_target" ]; }; then
    exit 1
fi
boot_init_cmp=1
if [ -f "$boot_init_target" ]; then
    cmp -s "$boot_init_expected" "$boot_init_target"
    boot_init_cmp=$?
fi
case "$boot_init_cmp" in
    0) printf '[boot-init] 包装脚本内容一致，保留现有文件。\n' ;;
    1)
        boot_init_error='无法安装新的 mount_root 包装脚本'
        boot_init_target_tmp="$boot_init_root/sbin/.mount_root.p1-new"
        rm -f "$boot_init_target_tmp" || exit 1
        cat "$boot_init_expected" > "$boot_init_target_tmp" || exit 1
        chown 0:0 "$boot_init_target_tmp" || exit 1
        chmod 0755 "$boot_init_target_tmp" || exit 1
        mv -f "$boot_init_target_tmp" "$boot_init_target" || exit 1
        ;;
    *) boot_init_error="cmp 比较出错，状态=$boot_init_cmp"; exit 1 ;;
esac
boot_init_error='无法核对或修正 mount_root 权限'
boot_init_mode=$(stat -c '%a:%u:%g' "$boot_init_target") || exit 1
if [ "$boot_init_mode" != 755:0:0 ]; then
    chown 0:0 "$boot_init_target" || exit 1
    chmod 0755 "$boot_init_target" || exit 1
fi
# INSTALL END

boot_init_stage='检查交接条件'
printf '\n[boot-init 07/09] 检查新根、原版程序和控制台\n'
boot_init_error='新根或下层的初始化程序不完整'
[ -x "$boot_init_root/sbin/init" ] && [ -x "$boot_init_rom/sbin/mount_root" ] || exit 1
/bin/sh -n "$boot_init_target" || exit 1
/bin/busybox chroot "$boot_init_root" /bin/busybox true || exit 1
mkdir -p "$boot_init_root/boot" "$boot_init_root/rom" "$boot_init_root/overlay" \
    "$boot_init_root/proc" "$boot_init_root/sys" "$boot_init_root/dev" || exit 1
boot_init_error='无法准备新根的静态控制台节点'
if [ ! -c "$boot_init_root/dev/console" ]; then
    [ ! -e "$boot_init_root/dev/console" ] && [ ! -L "$boot_init_root/dev/console" ] || exit 1
    mknod "$boot_init_root/dev/console" c 5 1 || exit 1
    chmod 0600 "$boot_init_root/dev/console" || exit 1
fi
boot_init_console_id=$(stat -c '%t:%T' "$boot_init_root/dev/console") || exit 1
[ "$boot_init_console_id" = 5:1 ] || exit 1
boot_init_error='交接前挂载状态不符合预期'
awk -v boot="$BOOT_DEV" -v data="$boot_init_overlay_dev" \
    -v rom="$boot_init_rom" -v overlay="$boot_init_data" -v root="$boot_init_root" '
    $1==boot && $2=="/boot" && $3=="vfat" && $4 ~ /(^|,)ro(,|$)/ {b=1}
    $1=="/dev/loop0" && $2==rom && $3=="squashfs" && $4 ~ /(^|,)ro(,|$)/ {l=1}
    $1==data && $2==overlay && $3=="f2fs" && $4 ~ /(^|,)rw(,|$)/ {d=1}
    $2==root && $3=="overlay" && $4 ~ /(^|,)rw(,|$)/ {r=1}
    END {exit !(b && l && d && r)}
' /proc/self/mounts || exit 1

boot_init_stage='移入存储挂载并清理临时环境'
printf '\n[boot-init 08/09] 移入存储挂载，清理临时虚拟挂载\n'
boot_init_error='无法移动根目录所依赖的存储挂载'
mount --move "$BOOT_MNT" "$boot_init_root/boot" || exit 1
mount --move "$boot_init_rom" "$boot_init_root/rom" || exit 1
mount --move "$boot_init_data" "$boot_init_root/overlay" || exit 1
sync || exit 1
cd / || exit 1
# 先释放标准输入输出对临时 /dev 的引用，再正常卸载它。
boot_init_error='无法将输入输出连接到新根控制台'
exec < "$boot_init_root/dev/console" > "$boot_init_root/dev/console" 2>&1
boot_init_error='临时虚拟文件系统卸载失败；未使用强制或懒卸载'
umount /dev || exit 1
umount /sys || exit 1
umount /proc || exit 1

boot_init_stage='交接给正式 OpenWrt'
boot_init_error='switch_root 交接失败'
printf '\n[boot-init 09/09] PID 1 交接给正式 OpenWrt /sbin/init\n'
unset INITRAMFS PREINIT BOOT_DEV BOOT_MNT
exec /bin/busybox switch_root -c /dev/console "$boot_init_root" /sbin/init

```

## 附录 C．本轮 uEnv.txt 完整归档

部署到 FAT 分区内部的 /boot/uEnv.txt。每个变量占一个物理行，使用 LF，行尾不加反斜杠。

```text
autoload=no

sd_uimage=/openwrt/lio-uImage
sd_loadaddr=0x00800000
custom_params=rdinit=/sbin/sd-init

bootcmd_exec=run bootcmd_sd
bootcmd_sd=echo "Booting OpenWrt from SD FAT..."; run sd_bootcmd

sd_bootcmd=run sd_rescan && run sd_load_uimage && run sd_check_uimage && run sd_set_bootargs && run sd_bootm || echo "ERROR: OpenWrt SD boot failed"

sd_rescan=echo "Initializing SD card..."; mmc rescan
sd_load_uimage=echo "Loading $sd_uimage..."; fatload mmc 0:1 $sd_loadaddr $sd_uimage && echo "uImage loaded: $filesize bytes"
sd_check_uimage=echo "Checking uImage..."; iminfo $sd_loadaddr
sd_set_bootargs=setenv bootargs console=ttyS0,115200 $custom_params
sd_bootm=echo "Starting kernel..."; bootm $sd_loadaddr

```

这份配置保留已验证的 bootcmd_exec 钩子，失败时打印错误并返回外层流程。外层 bootcmd 仍可能继续其他步骤，不能仅凭这一行 echo 承诺失败后一定停留在提示符。外层旧 ping 和 scan_disk 也未在本轮清理。

## 附录 D．当前运行状态的只读复核命令

以下用于下一页继续工作时采集现状；命令被记录在文档中，不表示已经在新一轮启动后执行。

```sh
sh -s -- <<'EOF'
printf '\n[01] PID 1 与启动参数\n'
cat /proc/1/comm
cat /proc/cmdline

printf '\n[02] 根目录与存储层\n'
awk '$2=="/" || $2=="/boot" || $2=="/rom" || $2=="/overlay"' /proc/mounts

printf '\n[03] loop 后端与只读状态\n'
cat /sys/class/block/loop0/loop/backing_file
cat /sys/class/block/loop0/ro

printf '\n[04] 关键虚拟挂载\n'
awk '$5=="/dev" || $5=="/sys" || $5=="/proc" || $5=="/tmp"' /proc/self/mountinfo

printf '\n[05] 标签与类型\n'
/usr/sbin/blkid -p /dev/mmcblk0p1 /dev/mmcblk0p2

printf '\n[06] mount_root 包装与 preinit 调用\n'
sed -n '1,100p' /sbin/mount_root
grep -n 'mount_root' /lib/preinit/80_mount_root
EOF
```

## 附录 E．归档文件 SHA-256

| 归档文件 | SHA-256 |
|---|---|
| `sd-init` | `992bd6c1689fe644c6620fe21e778237ae9161f4943b978f84cb88c350af53db` |
| `pogo-p1-init` | `35f9da289ced1202f0df82080b910a7b7f6ee222d380fc931bf81569b2bbb187` |
| `uEnv.txt` | `24811f9bc621676801e5f371f446124ee391069d764378f2347c3283f60f48e0` |
| `configs/openwrt.config` | `cddcda7e550498d2c8824445c2cd42a5f6c3e3bdbc76f549ecb43e896d71d429` |
| `configs/kernel.config` | `6cd4fa3cc9147b05356073147b296f50baff086c63866453472db0738d0f183f` |

FAT init v0.2 和 uEnv.txt 摘要取自此次提供并归档的源文件，用于识别文档内版本；没有声称对实机 FAT 文件另外执行过摘要核对。sd-init 摘要同时有构建机取证输出。

## 附录 F．最后取证的 target 内核配置原始 diff

历史证据，不能整体当作下一轮最小片段。

```diff
diff --git a/target/linux/kirkwood/config-6.12 b/target/linux/kirkwood/config-6.12
index 57e9bfd..45c9017 100644
--- a/target/linux/kirkwood/config-6.12
+++ b/target/linux/kirkwood/config-6.12
@@ -33,6 +33,7 @@ CONFIG_ATAGS=y
 CONFIG_ATA_LEDS=y
 CONFIG_AUTO_ZRELADDR=y
 CONFIG_BINFMT_FLAT_ARGVP_ENVP_ON_STACK=y
+CONFIG_BLK_DEV_INTEGRITY=y
 CONFIG_BLK_DEV_SD=y
 CONFIG_BLK_MQ_PCI=y
 CONFIG_CACHE_FEROCEON_L2=y
@@ -43,6 +44,7 @@ CONFIG_CLONE_BACKWARDS=y
 CONFIG_COMMON_CLK=y
 CONFIG_COMPACT_UNEVICTABLE_DEFAULT=1
 CONFIG_COMPAT_32BIT_TIME=y
+CONFIG_CONFIGFS_FS=y
 CONFIG_CPU_32v5=y
 CONFIG_CPU_ABRT_EV5T=y
 CONFIG_CPU_CACHE_VIVT=y
@@ -54,6 +56,7 @@ CONFIG_CPU_FEROCEON=y
 CONFIG_CPU_IDLE=y
 CONFIG_CPU_IDLE_GOV_LADDER=y
 CONFIG_CPU_LITTLE_ENDIAN=y
+CONFIG_CPU_MITIGATIONS=y
 CONFIG_CPU_PABRT_LEGACY=y
 CONFIG_CPU_PM=y
 CONFIG_CPU_THERMAL=y
@@ -61,13 +64,17 @@ CONFIG_CPU_THUMB_CAPABLE=y
 CONFIG_CPU_TLB_FEROCEON=y
 CONFIG_CPU_USE_DOMAINS=y
 CONFIG_CRC16=y
-# CONFIG_CRC32_SARWATE is not set
-CONFIG_CRC32_SLICEBY8=y
+CONFIG_CRC64=y
+CONFIG_CRC64_ROCKSOFT=y
+CONFIG_CRC_T10DIF=y
 CONFIG_CRYPTO_CRC32C=y
+CONFIG_CRYPTO_CRC64_ROCKSOFT=y
+CONFIG_CRYPTO_CRCT10DIF=y
 CONFIG_CRYPTO_DEFLATE=y
 CONFIG_CRYPTO_DES=y
 CONFIG_CRYPTO_DEV_MARVELL=y
 CONFIG_CRYPTO_DEV_MARVELL_CESA=y
+CONFIG_CRYPTO_ECB=y
 CONFIG_CRYPTO_HASH_INFO=y
 CONFIG_CRYPTO_HW=y
 CONFIG_CRYPTO_LIB_BLAKE2S_GENERIC=y
@@ -86,7 +93,8 @@ CONFIG_DEBUG_UART_8250=y
 CONFIG_DEBUG_UART_8250_SHIFT=2
 CONFIG_DEBUG_UART_PHYS=0xf1012000
 CONFIG_DEBUG_UART_VIRT=0xfed12000
-CONFIG_DMA_OPS=y
+CONFIG_DMA_NEED_SYNC=y
+CONFIG_DMA_OPS_HELPERS=y
 CONFIG_DNOTIFY=y
 CONFIG_DTC=y
 # CONFIG_EARLY_PRINTK is not set
@@ -95,6 +103,7 @@ CONFIG_EDAC_SUPPORT=y
 CONFIG_EXCLUSIVE_SYSTEM_RAM=y
 CONFIG_FIXED_PHY=y
 CONFIG_FIX_EARLYCON_MEM=y
+CONFIG_FORCE_NR_CPUS=y
 CONFIG_FORCE_PCI=y
 # CONFIG_FRAMER is not set
 CONFIG_FS_IOMAP=y
@@ -102,12 +111,12 @@ CONFIG_FUNCTION_ALIGNMENT=0
 CONFIG_FWNODE_MDIO=y
 CONFIG_FW_LOADER_PAGED_BUF=y
 CONFIG_FW_LOADER_SYSFS=y
-CONFIG_GCC_ASM_GOTO_OUTPUT_WORKAROUND=y
 CONFIG_GENERIC_ALLOCATOR=y
 CONFIG_GENERIC_ATOMIC64=y
 CONFIG_GENERIC_BUG=y
 CONFIG_GENERIC_CLOCKEVENTS=y
 CONFIG_GENERIC_CPU_AUTOPROBE=y
+CONFIG_GENERIC_CPU_DEVICES=y
 CONFIG_GENERIC_EARLY_IOREMAP=y
 CONFIG_GENERIC_IDLE_POLL_SETUP=y
 CONFIG_GENERIC_IRQ_CHIP=y
@@ -148,6 +157,7 @@ CONFIG_IRQSTACKS=y
 CONFIG_IRQ_DOMAIN=y
 CONFIG_IRQ_FORCED_THREADING=y
 CONFIG_IRQ_WORK=y
+CONFIG_ISCSI_TARGET=y
 CONFIG_KIRKWOOD_CLK=y
 CONFIG_KIRKWOOD_THERMAL=y
 CONFIG_LEDS_GPIO=y
@@ -155,6 +165,7 @@ CONFIG_LEDS_NETXBIG=y
 CONFIG_LEDS_NS2=y
 CONFIG_LIBFDT=y
 CONFIG_LOCK_DEBUGGING_SUPPORT=y
+# CONFIG_LOOPBACK_TARGET is not set
 CONFIG_LZO_COMPRESS=y
 CONFIG_LZO_DECOMPRESS=y
 CONFIG_MACH_KIRKWOOD=y
@@ -241,7 +252,6 @@ CONFIG_POWER_RESET=y
 CONFIG_POWER_RESET_GPIO=y
 CONFIG_POWER_RESET_LINKSTATION=y
 CONFIG_POWER_SUPPLY=y
-CONFIG_PREEMPT_NONE_BUILD=y
 CONFIG_PTP_1588_CLOCK_OPTIONAL=y
 CONFIG_RANDSTRUCT_NONE=y
 CONFIG_RATIONAL=y
@@ -249,6 +259,7 @@ CONFIG_REGMAP=y
 CONFIG_REGMAP_MMIO=y
 CONFIG_REGULATOR=y
 CONFIG_REGULATOR_FIXED_VOLTAGE=y
+# CONFIG_REMOTE_TARGET is not set
 CONFIG_RTC_CLASS=y
 CONFIG_RTC_DRV_MV=y
 CONFIG_RTC_I2C_AND_SPI=y
@@ -270,12 +281,14 @@ CONFIG_SPI=y
 CONFIG_SPI_MASTER=y
 CONFIG_SPI_MEM=y
 CONFIG_SPI_ORION=y
-CONFIG_SPLIT_PTLOCK_CPUS=999999
-CONFIG_SQUASHFS_DECOMP_MULTI_PERCPU=y
 CONFIG_SRAM=y
 CONFIG_SRAM_EXEC=y
 CONFIG_SWPHY=y
 CONFIG_SYS_SUPPORTS_APM_EMULATION=y
+CONFIG_TARGET_CORE=y
+# CONFIG_TCM_FILEIO is not set
+CONFIG_TCM_IBLOCK=y
+# CONFIG_TCM_PSCSI is not set
 CONFIG_THERMAL=y
 CONFIG_THERMAL_DEFAULT_GOV_STEP_WISE=y
 CONFIG_THERMAL_EMERGENCY_POWEROFF_DELAY_MS=0
@@ -296,6 +309,7 @@ CONFIG_USB_LED_TRIG=y
 CONFIG_USB_SUPPORT=y
 CONFIG_USE_OF=y
 # CONFIG_VFP is not set
+# CONFIG_VHOST_SCSI is not set
 CONFIG_VM_EVENT_COUNTERS=y
 CONFIG_WAN=y
 CONFIG_WATCHDOG_CORE=y

```

## 附录 G．最后取证的顶层 diffconfig 原始输出

这是已成功构建配置的差异导出，包含后续最小片段不再手工列入的依赖和历史 FEED 选择；不等于第 5 节的最小输入。

```text
CONFIG_TARGET_kirkwood=y
CONFIG_TARGET_kirkwood_generic=y
CONFIG_TARGET_kirkwood_generic_DEVICE_cloudengines_pogoplugv4=y
CONFIG_BUSYBOX_CUSTOM=y
CONFIG_BUSYBOX_CONFIG_FEATURE_STAT_FORMAT=y
CONFIG_BUSYBOX_CONFIG_STAT=y
# CONFIG_FEED_luci is not set
# CONFIG_FEED_packages is not set
# CONFIG_FEED_routing is not set
# CONFIG_FEED_telephony is not set
# CONFIG_FEED_video is not set
CONFIG_PACKAGE_blkid=y
CONFIG_PACKAGE_kmod-crypto-aead=y
CONFIG_PACKAGE_kmod-crypto-cbc=y
CONFIG_PACKAGE_kmod-crypto-crc32=y
CONFIG_PACKAGE_kmod-crypto-hmac=y
CONFIG_PACKAGE_kmod-crypto-manager=y
CONFIG_PACKAGE_kmod-crypto-null=y
CONFIG_PACKAGE_kmod-crypto-rng=y
CONFIG_PACKAGE_kmod-crypto-sha1=y
CONFIG_PACKAGE_kmod-crypto-sha256=y
CONFIG_PACKAGE_kmod-crypto-sha3=y
CONFIG_PACKAGE_kmod-crypto-sha512=y
CONFIG_PACKAGE_kmod-dax=y
CONFIG_PACKAGE_kmod-dm=y
CONFIG_PACKAGE_kmod-fs-f2fs=y
CONFIG_PACKAGE_kmod-fs-vfat=y
CONFIG_PACKAGE_kmod-keys-encrypted=y
CONFIG_PACKAGE_kmod-keys-trusted=y
CONFIG_PACKAGE_kmod-loop=y
CONFIG_PACKAGE_kmod-nls-cp437=y
CONFIG_PACKAGE_kmod-nls-iso8859-1=y
CONFIG_PACKAGE_kmod-nls-utf8=y
CONFIG_PACKAGE_kmod-random-core=y
CONFIG_PACKAGE_kmod-tpm=y
CONFIG_PACKAGE_libaio=y
CONFIG_PACKAGE_libblkid=y
CONFIG_PACKAGE_libdevmapper=y
CONFIG_PACKAGE_libpthread=y
CONFIG_PACKAGE_librt=y
CONFIG_PACKAGE_libsmartcols=y
CONFIG_PACKAGE_libuuid=y
CONFIG_PACKAGE_losetup=y
CONFIG_PACKAGE_lvm2=y

```

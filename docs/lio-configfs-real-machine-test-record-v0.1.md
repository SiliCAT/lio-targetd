# lio configfs 实机测试实验记录

版本：v0.1  
整理日期：2026-10-20  
项目：PogoPlug V4 / OpenWrt / lio-targetd  
性质：实机实验技术记录、能力边界与换页依据。

本文记录本讨论页中与 LIO configfs 有关的实验，分别说明已验证能力、明确失败的候选方案、尚未验证的业务，以及实验对象的最后已知状态。以用户贴出的实机输出为证据，不把设计决定、属性回读或源码推测写成已经通过的业务能力。

本文只交付 configfs 实验记录。前三阶段执行器代码、队列协议、配置解析及第四阶段接口设计，另文整理。本文涉及这些业务时，只记录实验对设计产生的约束。

## 1. 阅读基线与最终结论

### 1.1 能力判定总表

| 项目 | 判定 | 实验已经证明的范围 |
| --- | --- | --- |
| configfs 内核支持与手工挂载 | 已验证 | 内核列出 configfs；挂载后出现 target 配置入口 |
| 创建 iSCSI fabric、target、TPG、空 LUN | 已验证 | 本机能够创建相应 configfs 对象 |
| iblock 后端绑定真实 LV | 已验证 | 配置绑定成功，info 报告对应设备名称、主次设备号及 CLAIMED: IBLOCK |
| LUN 到后端的引用核对 | 已验证 | 解析符号链接后的规范路径与指定后端一致 |
| 两个 LUN 共享同一后端 | 已验证 | 两条引用指向同一后端，后端文件标识保持不变 |
| 逐个删除共享引用 | 已验证 | 删除一个 LUN 后另一个仍有效；删除最后一个后后端仍存在 |
| 无剩余引用时删除准确后端 | 已验证 | 在该实验中扫描未发现剩余引用，后端删除成功 |
| 无 LUN 的后端是否仍占用 LV | 已验证：仍占用 | LV 打开数从 0 变 1；普通停用被拒绝；删除后端后打开数归零 |
| 静态 ACL 映射的 write_protect 设置与独立性 | 已验证配置层 | 两个映射可以分别设置并回读，互不改变；未验证这种配置的客户端拒写效果 |
| 无认证、未预登记客户端的 iSCSI 接入 | 已验证 | 指定实验 target 登录成功，两个 LUN 可通过 SCSI generic 读取 |
| 可写后端的客户端 WRITE(10) 及回读 | 已验证 | 客户端写入成功，回读与发送载荷完全一致 |
| 仅用 iblock readonly=1 实现可靠服务端只读 | 本次方案失败 | 客户端显示 ro=1，但原始 WRITE(10) 返回 Good；底层 LV 中也读到写入载荷 |
| 按 target IQN＋LUN 对所有客户端统一实现混合 ro/rw | 尚未找到并验证实现路径 | 静态映射属性实验不能替代这一需求的完整验收 |
| 完整实例绑定的 configfs 独立证明 | 尚未解决 | 已读取的后端 info 没有 DISKSEQ、DM_UUID，不能单凭主次设备号证明完整实例 |
| 自动执行器完成发布、登记、释放的端到端业务 | 尚未实现 | 当前正式执行器前三阶段未操作 LIO；本文 configfs 对象由实验命令建立 |

### 1.2 本页末尾生效的需求收缩

用户最后明确决定：初版只可靠实现最小能力集合，暂不实现 ro；等未来找到实现路径并通过实验后再升级。

初版基线为：

1. 客户端无认证接入。
2. 发布位置按 target IQN＋TPG＋LUN_ID 区分；用户表达的策略粒度为 target IQN＋LUN。
3. 仅支持 ACCESS=rw，后端按 readonly=0 创建并回读核对。
4. ACCESS 字段保留；遇到 ro 明确报告当前版本不支持，停止相应配置组的发布动作，不自动改成 rw。
5. 发布状态中的冻结预期仍保留 ACCESS，初版取值为 rw。
6. 当前只读失败结果作为以后研究的依据保留。

这是需求和后续设计基线，不表示现有前三阶段脚本已完成上述所有修订，也不表示第四阶段已经落地。

## 2. 实验环境、路径与证据边界

### 2.1 服务端

| 项目 | 本轮实机信息 |
| --- | --- |
| 设备平台 | PogoPlug V4，OpenWrt kirkwood/generic |
| OpenWrt | 25.12.0，r32713-f919e7899d |
| 内核 | Linux 6.12.71，armv5tel |
| PID 1 | /sbin/procd |
| shell | /bin/sh → BusyBox ash，BusyBox 1.37.0 |
| configfs 挂载点 | /sys/kernel/config |
| target 根 | /sys/kernel/config/target |
| 后端根 | /sys/kernel/config/target/core |
| iSCSI fabric 根 | /sys/kernel/config/target/iscsi |
| 本轮独立实验 HBA | /sys/kernel/config/target/core/iblock_99 |
| 网络实验服务端地址 | 172.16.16.220 |
| 网络实验 portal | 172.16.16.220:32690 |

iblock_99 是本轮实验的对象命名，不是已经冻结的正式生产 HBA 名称。早期文档或命令中出现的 172.16.16.215 不作为本轮网络实验地址；实际成功使用的是 172.16.16.220。

### 2.2 客户端

客户端主机为 SandPlug，用户终端显示 silicat@SandPlug；测试命令以 root 权限执行。使用 open-iscsi、iscsiadm 与 sg3_utils 的 sg_raw 等工具。

本次会话输出为：

```text
tcp: [1] 172.16.16.220:32690,1 iqn.2026-10.local.pogov4:lio-ro-write-test (non-flash)
```

实验期间 LUN 0 对应 /dev/sg0、客户端块设备 sda；LUN 1 对应 /dev/sg1、块设备 sdb。这些都是当次枚举结果，换页或重连后不能直接当作稳定路径。

### 2.3 本文对“已验证”的使用方式

- mkdir、参数写入、链接创建、参数回读，只证明对应配置接口能够工作。
- info 中的路径字符串、对象文件标识、块设备号分别说明不同事实，不能互相替代。
- 客户端登录成功不等于数据读写已经验收。
- 客户端 ro 标志不等于服务端必然拒绝原始 SCSI 写命令。
- 本轮直接挂载、swap、holders 检查仅覆盖当时能读取的范围，不证明所有命名空间或所有进程均未使用该设备。
- 读取并比较首扇区不证明断电持久性、完整文件系统语义或生产负载稳定性。
- 每个资源的“最后已知状态”来自最后一次对应实机输出，不是整理文档时重新探测的结果。

## 3. 实验 CF-01：内核能力、configfs 挂载和 fabric 初始化

### 3.1 初次停止的原因

最初的盘点输出是：

```text
未确认 /sys/kernel/config 为 configfs，停止盘点。
```

随后检查 /proc/filesystems，发现：

```text
nodev   configfs
```

当时没有任何 configfs 挂载。问题是文件系统尚未挂载，不能据此判定内核没有编译 configfs 或 LIO。

### 3.2 手工挂载结果

挂载到 /sys/kernel/config 后，/proc/mounts 输出：

```text
none /sys/kernel/config configfs rw,relatime 0 0
```

目录内出现 target；target 下出现 core、dbroot、version。随后创建 target/iscsi，创建成功，目录中出现 discovery_auth、cpus_allowed_list、lio_version。

读取版本得到：

```text
Target Engine Core ConfigFS Infrastructure v5.0 on Linux/armv5tel on 6.12.71
```

### 3.3 结论与未覆盖范围

本机具备本次实验需要的 target 核心、iblock 与 iSCSI fabric 能力。内建功能不能以“找不到单独模块文件”为依据判定不可用。

本轮证明了手工挂载和手工初始化入口；没有据此证明 configfs 开机自动挂载、重启后对象恢复或 init 脚本准备流程已经实现。

## 4. 实验 CF-02：真实 LV 绑定与 LUN 引用核对

### 4.1 实验资源和前置检查

使用现有测试 LV vg2/BULKTEST00，当时身份为：

```text
DM_NAME=vg2-BULKTEST00
KERNEL=dm-2
实例=253:2/29
DM_UUID=LVM-X2w9s3qeto0ppFI6DZbt6iM3M6VoV3DKDLhBgYmbebbwq8cGV1zTeAXGxqp5mnRa
```

实验前未发现该设备在当前挂载命名空间的直接挂载；swap 列表为空；holders 目录为空。这些检查是本次实验的局部前提，不是全系统无人使用的证明。

创建的对象为：

| 对象 | 路径 |
| --- | --- |
| target | /sys/kernel/config/target/iscsi/iqn.2026-10.local.pogov4:lio-relation-probe |
| TPG | 上述 target 下的 tpgt_1 |
| 后端 | /sys/kernel/config/target/core/iblock_99/lio_relation_probe |
| 发布位置 | tpgt_1/lun/lun_0 |

TPG 保持禁用，portal 为空。本项没有开放网络入口。

### 4.2 操作次序

1. 创建独立 target、禁用 TPG 和空 LUN。
2. 创建 iblock 后端，经 control 提交设备路径和 readonly 参数，启用后端。
3. 核对设备绑定前后仍为原实例。
4. 在 LUN 内建立指向指定后端的 data 链接。
5. 比较规范化后的链接目标与预期后端路径。
6. 原文读取 info、udev_path、TPG 参数，保留实验对象。

本轮使用的关键接口语义如下；这里只记录接口，不是可以无条件重复执行的安装脚本：

```text
core/iblock_99/<后端>/control   ← udev_path=<设备路径>,readonly=<0或1>
core/iblock_99/<后端>/enable    ← 1
iscsi/<IQN>/tpgt_1/lun/lun_0/data → 指定后端
```

### 4.3 引用关系的实际表现

建立链接后，readlink 原文为：

```text
../../../../../../target/core/iblock_99/lio_relation_probe
```

预期和实际规范化目标均为：

```text
/sys/kernel/config/target/core/iblock_99/lio_relation_probe
```

因此核对应该比较解析后的准确对象路径，不能要求链接原文必须等于最初传入的绝对路径。

### 4.4 后端提供的可读证据

```text
Status: ACTIVATED  Max Queue Depth: 0  SectorSize: 512  HwMaxSectors: 240
        iBlock device: dm-2  UDEV PATH: /dev/mapper/vg2-BULKTEST00  readonly: 0
        Major: 253 Minor: 2  CLAIMED: IBLOCK
```

后端目录中观察到 action、alias、alua、attrib、control、enable、info、udev_path、wwn、statistics 等入口。其中 control 为写入入口，不能假设能从 control 直接回读原始配置。

独立的后端 udev_path 文件读取为空，而 info 中有 UDEV PATH。本轮仅通过 control 配置路径，没有为了制造核对证据再写独立 udev_path。

### 4.5 身份证明的实际边界

| 证据 | 本轮用途 | 不能替代的事实 |
| --- | --- | --- |
| data 链接规范目标 | 核对 LUN 引用指定后端 | 不能单独证明后端属于本执行器的哪次生命周期 |
| info 的 Major / Minor | 核对后端报告的块设备号 | 不含 DISKSEQ，不能单独区分设备号复用后的实例 |
| info 的 UDEV PATH | 查看后端保存的路径参数 | 路径以后可能仍存在但指向新实例 |
| sysfs 的 dev、DISKSEQ、dm/name、dm/uuid | 核对当时现场与冻结预期是否一致 | 不能把读到的新实例替换成历史任务的预期 |
| stat 得到的文件系统号与 inode | 本轮比较对象是否保持一致的辅助证据 | 不能单独证明设备实例或跨重启的对象归属 |

后端 info 没有给出 DISKSEQ 或 DM_UUID。因此本轮只宣称“LUN 到指定后端的引用核对通过”和“绑定前后设备身份核对一致”，没有宣称 configfs 自身提供了完整实例绑定证明。

## 5. 实验 CF-03：共享后端、逐个释放和外壳保留

### 5.1 两个发布位置共享同一后端

在同一 TPG 创建 LUN 1，使 LUN 0、LUN 1 的 data 均指向 lio_relation_probe。两次解析结果一致，后端文件标识保持不变；本次采集的后端标识为 13:84298。

这证明一个后端可被两个 TPG LUN 引用。它不表示已经验证两个不同 target 的共享、多个客户端并发写入或文件系统并发使用。

### 5.2 仅释放 LUN 0

解除 LUN 0 的后端链接并删除该 LUN 后，LUN 1 保持存在，仍引用原后端；后端 info 继续报告 253:2。

因此单个发布位置的释放不能顺带回收仍被其他位置引用的共享后端。

### 5.3 释放最后一个 LUN

LUN 1 也释放后，两个实验 LUN 均不存在，后端仍存在；info 变为：

```text
Status: DEACTIVATED  Max Queue Depth: 0  SectorSize: 512  HwMaxSectors: 240
        iBlock device: dm-2  UDEV PATH: /dev/mapper/vg2-BULKTEST00  readonly: 0
        Major: 253 Minor: 2  CLAIMED: IBLOCK
```

“最后一个 LUN 消失”与“后端对象消失”是两个不同事实。

### 5.4 无引用时回收后端

扫描当时的 target 树，没有发现仍指向该准确后端的符号链接。随后删除准确后端成功，其原路径不存在。

该结果是本次现场下的引用检查与删除结果，不证明未来所有 fabric、ACL、并发外部管理者或内核暂存引用都已经纳入正式回收算法。

### 5.5 保留外壳的实验结果与设计决定

删除 LUN 和后端后，空 target、禁用 TPG、空 HBA 保留；portal 从未创建。LV 仍为实例 253:2/29。

用户随后确认的管理策略是：空闲 target、TPG、portal 保留，不自动收缩外壳。这里“保留 portal”是设计规则；本项实际验证的是空 target、禁用 TPG、空 HBA 可以保留，没有测试活跃客户端存在时的外壳管理。

## 6. 实验 CF-04：无 LUN 后端的设备占用和 LV 下线顺序

本项原始实机输出另存于本页附件《粘贴的文本 (1)(20261020-075020).txt》。该实验是区分“发布释放”与“设备占用释放”的关键依据。

### 6.1 仅创建并启用后端，不创建 LUN

实验前 vg2/BULKTEST00 为实例 253:2/29，lvm lvdisplay 的 # open 为 0。只创建并启用后端后，info 显示 DEACTIVATED，但 # open 变为 1。

```text
绑定前：# open 0
绑定后：# open 1
后端：Status: DEACTIVATED
后端：Major: 253 Minor: 2  CLAIMED: IBLOCK
```

### 6.2 后端持有期间普通停用 LV

执行普通 lvchange -an，原始诊断为：

```text
Logical volume vg2/BULKTEST00 in use.
持有后端期间，停用返回值=5
```

随后核对：原实例仍存在，打开数仍为 1，后端仍报告同一设备。

这里是普通停用；没有强制移除 DM、没有模拟物理掉盘，也没有验证带客户端会话的停用行为。

### 6.3 删除后端后再次停用

检查无后端引用并删除准确实验后端后：

```text
# open 0
删除后端后，停用返回值=0
已确认没有该名称的活动 DM 实例。
```

随后重新激活：

```text
旧实例=253:2/29
新实例=253:2/30
```

### 6.4 对业务顺序的直接影响

正常计划下线必须把以下工作区分开：

1. 释放相应发布位置及其引用。
2. 在确认没有其他发布引用后，回收后端并确认本次设备占用已经释放。
3. 管理流程再停用 LV。
4. 后续设备 remove 事件用于观察或异常核查。

不能设计成“先正常停用 LV，等待 remove，再靠 remove 驱动后端释放”，因为本机普通停用会先被后端占用阻止。

ACTIVATED / DEACTIVATED 的实验变化与 LUN 引用变化对应，但 DEACTIVATED 不能作为“设备未被占用”的判据。这一点已有直接打开数和停用结果证明，不需要依赖状态名称的字面含义。

### 6.5 与已安装执行器的关系

该次下线和重新激活同时被前三阶段执行器观察到：旧实例 29 的 remove 为 SEQNUM 639、641；新实例 30 的 add/change 为 643、644。执行器 PID 4599 在两轮中处理 4 条记录，异常 0、rc=0，日志仍明确写着 LIO 未执行。

这说明事件观察链在运行，不能据此把手工 configfs 实验归功于自动执行器已经完成发布或释放。

## 7. 实验 CF-05：静态 ACL 映射的独立写保护属性

### 7.1 目的与实验对象

本项研究“ro/rw 可以作用在哪一层”，不是最终无认证任意客户端方案的验收。

| 项目 | 实际实验值 |
| --- | --- |
| target | iqn.2026-10.local.pogov4:lio-access-probe |
| TPG | tpgt_1，保持禁用，无 portal |
| 后端 | /sys/kernel/config/target/core/iblock_99/lio_access_probe |
| 后端设备 | vg2/BULKTEST00，实例 253:2/30 |
| 后端 readonly | 0 |
| TPG LUN | LUN 0、LUN 1，共享同一后端 |
| 显式 initiator ACL | iqn.2026-10.local.pogov4:lio-access-client |
| ACL 内映射 | lun_0、lun_1，分别指向两个 TPG LUN |

TPG 初始回读：enable=0、generate_node_acls=0、authentication=1、demo_mode_write_protect=1、prod_mode_write_protect=0。

### 7.2 普通 TPG LUN 与 ACL 映射的差别

普通 TPG LUN 目录内观察到 data 链接、ALUA 参数和 statistics；没有出现可直接设置的同名 write_protect 入口。

显式 ACL 的 mapped LUN 目录提供 write_protect，实际路径形态为：

```text
iscsi/<IQN>/tpgt_1/acls/<initiator IQN>/lun_<映射编号>/write_protect
```

 mapped LUN 内的 target_lun 链接指向 TPG LUN，而 TPG LUN 内的 data 链接再指向后端。这是两层不同的引用。

### 7.3 实际设置轨迹

| 步骤 | 映射 0 write_protect | 映射 1 write_protect | 观察 |
| --- | --- | --- | --- |
| 创建后的初始值 | 0 | 0 | 两者均可读取 |
| 设置 ro / rw | 1 | 0 | 分别回读正确 |
| 仅把映射 0 改为 rw | 0 | 0 | 映射 1 未改变 |
| 仅把映射 1 改为 ro | 0 | 1 | 映射 0 未改变 |
| 恢复实验排列 | 1 | 0 | 最后保留该排列 |

共享 TPG 参数与后端对象未因这些映射属性修改而改变；后端继续报告 readonly: 0。

### 7.4 判定范围

已验证：静态 initiator ACL 下的两个映射能够分别设置 write_protect，配置回读和独立性成立。

未验证：客户端通过这两个静态映射实际写入时是否得到预期拒绝；未验证任意自动接入客户端始终服从同一份 target IQN＋LUN 策略；未验证映射增删和客户端重连时的策略保持。

用户明确要求所有客户端统一接受服务端按 target IQN＋LUN 制定的策略，不引入客户端分别配置权限的业务。不能把该项静态属性实验直接写成最终需求已解决。

## 8. 实验 CF-06：无认证自动接入与专用 RO/RW 对照环境

### 8.1 创建专用临时设备的原因

真实写入实验使用新建的专用临时 LV，不在现有 BULKTEST00 上发送测试写入。它们分别命名为 vg2/LIO_RO_PROBE、vg2/LIO_RW_PROBE，创建后进行初始化，再绑定为实验后端。

这些临时资源不在正式 resources.names 名称表中；其存在不表示正式执行器已经接管。

### 8.2 网络对象与参数

```text
IQN=iqn.2026-10.local.pogov4:lio-ro-write-test
TPG=1
PORTAL=172.16.16.220:32690
enable=1
authentication=0
generate_node_acls=1
cache_dynamic_acls=1
demo_mode_write_protect=0
```

没有预先登记客户端静态 ACL。后来补证时又回读到 prod_mode_write_protect=0、demo_mode_discovery=1。

这里的自动 ACL 生成是实验实现未知客户端接入的内部机制，不表示本项目已经实现远程客户端权限管理。

### 8.3 两个测试对象

| 项目 | LUN 0：RO 候选 | LUN 1：RW 对照 |
| --- | --- | --- |
| LV | vg2/LIO_RO_PROBE | vg2/LIO_RW_PROBE |
| mapper | /dev/mapper/vg2-LIO_RO_PROBE | /dev/mapper/vg2-LIO_RW_PROBE |
| 内核设备 | dm-20 | dm-21 |
| 完整实例 | 253:20/31 | 253:21/32 |
| 后端 | core/iblock_99/lio_ro_write_ro | core/iblock_99/lio_ro_write_rw |
| 后端 readonly | 1 | 0 |
| VPD 序列号 | 202610200001 | 202610200002 |
| 当次客户端 sg | /dev/sg0 | /dev/sg1 |
| 当次客户端块设备 | sda | sdb |
| 客户端 ro 标志 | 1 | 0 |

完整 DM_UUID：

```text
RO:
LVM-X2w9s3qeto0ppFI6DZbt6iM3M6VoV3DKdmfBSUSCd8DAhmIWS5jryTp0g3MGlSMD

RW:
LVM-X2w9s3qeto0ppFI6DZbt6iM3M6VoV3DKFbmAhHdkh7682piRBrPfO1CazaSgqOnF
```

服务端原始身份保存在 /tmp/lio-ro-write-test/ro.identity 和 rw.identity。

### 8.4 客户端接入结果

客户端创建了手工启动、无认证的专用节点，登录成功。实验 target 的两个 LUN 可定位，后续 VPD 与 SCSI READ(10) 成功。

已证明这一配置可以完成本次客户端的无认证连接和 LUN 访问；没有验证多个客户端、重连后的自动映射变化、跨重启恢复或生产网络策略。

## 9. 实验中途停止与工具兼容性问题

### 9.1 第一次服务端准备停止

第一次输出显示实际 LAN 地址为 172.16.16.220，在环境检查阶段 rc=1 停止。随后使用该实际地址准备成功。不能把早先命令中的其他地址继续当作已经通过验证的 portal。

### 9.2 第一次客户端在序列号文本解析处停止

客户端首次登录成功后，序列号核对步骤输出的是 VPD 十六进制数据：

```text
00 80 00 0d 32 30 32 36 31 30 32 30 30 30 30 31 00
```

脚本在此停止。问题是所使用命令的文本输出形式不符合原脚本的提取预期，不能解释为 VPD 序列号错误或登录失败。当次尚未进入后续写入步骤，会话保留。

后续改为通过原始 INQUIRY CDB 获取 VPD 页，并对完整 17 字节逐字节比较，避免依赖某种可读文本标签。

### 9.3 服务端补证因 od 缺失停止

后续补证脚本在 RO 数据及 SHA256 已采集后，尝试输出前 64 字节时遇到：

```text
sh: od: not found
[结束] rc=127
```

这是展示工具缺失。前面的 RO 身份、参数、512 字节数据和哈希均有效；脚本未继续执行到 RW 的服务端采集。不能把整个补证过程写成“所有步骤完成”，也不能因此丢弃已经完成的证据。

## 10. 实验 CF-07：原始 SCSI 写入对照与只读失败

### 10.1 对象确认：完整 VPD 响应

对两个对象发送 VPD 页 0x80 的 INQUIRY，CDB 为：

```text
12 01 80 00 11 00
```

两个完整响应为：

```text
LUN 0:
00 80 00 0d 32 30 32 36 31 30 32 30 30 30 30 31 00

LUN 1:
00 80 00 0d 32 30 32 36 31 30 32 30 30 30 30 32 00
```

这 17 字节包含页头、长度为 13 的载荷、12 个序列号字符及末尾 NUL。后续比较已把末尾 NUL 纳入预期，并非只搜索某一串文本。

完整响应均一致；客户端分别报告 sda ro=1、sdb ro=0。

### 10.2 初始数据与测试载荷

读取两个对象的首个 512 字节扇区，保存 ro.before、rw.before；准备同样长度的 payload.bin。两个初始扇区哈希相同，载荷与初始数据不同。

本次实际 CDB：

```text
READ(10)，LBA 0，1 个逻辑块：
28 00 00 00 00 00 00 00 01 00

WRITE(10)，LBA 0，1 个逻辑块：
2a 00 00 00 00 00 00 00 01 00
```

后端报告逻辑扇区大小为 512 字节。sg_raw 通过 SCSI generic 发送命令，测试没有依赖对 /dev/sda 的普通块设备写入能否被客户端 ro 标志拦截。

### 10.3 RW 对照成功

向 /dev/sg1 发送 WRITE(10)：

```text
open /dev/sg1 with flags=0x802
cdb to send: Write(10) [2a 00 00 00 00 00 00 00 01 00]
SCSI Status: Good
可写对照返回值=0
```

随后 READ(10) 得到 rw.after，与 payload.bin 完全一致。

该对照证明命令、长度、载荷和读回方式能够正常完成本轮数据写入。

### 10.4 RO 候选实际失败

向 /dev/sg0 发送同一条 WRITE(10)：

```text
open /dev/sg0 with flags=0x802
cdb to send: Write(10) [2a 00 00 00 00 00 00 00 01 00]
SCSI Status: Good
只读对象返回值=0（预期7，Data Protect）
```

随后 READ(10) 得到 ro.after，和 ro.before 不同。脚本准确报告失败并停止，没有把非预期结果改写成成功，也没有自动断开会话或清理对象。

### 10.5 客户端补证：写后内容就是本次载荷

客户端证据目录：

```text
/tmp/lio-ro-client.AWZyoE/continue.cBx2ID
```

| 文件 | 字节数 | SHA256 |
| --- | --- | --- |
| payload.bin | 512 | 81ba4aca90b1bc03819b12005043509255d9dd3a232d0ec29ea2b3937ab60558 |
| ro.before | 512 | 076a27c79e5ace2a3d47f9dd2e83e4ff6ea8872b3c2218f66c92b89b55f36560 |
| ro.after | 512 | 81ba4aca90b1bc03819b12005043509255d9dd3a232d0ec29ea2b3937ab60558 |
| rw.before | 512 | 076a27c79e5ace2a3d47f9dd2e83e4ff6ea8872b3c2218f66c92b89b55f36560 |
| rw.after | 512 | 81ba4aca90b1bc03819b12005043509255d9dd3a232d0ec29ea2b3937ab60558 |

cmp 的结果另外确认：RO 写前不等于载荷，RO 写后等于载荷，RO 写前写后不同，RW 写后等于载荷。本轮补证没有再次发送写命令。

## 11. 实验 CF-08：服务端直接补证只读失败

### 11.1 配置及引用仍然吻合

补证发生在 2026-10-20 09:40:08 GMT 左右，服务端 uptime 33864.96 秒。证据目录为：

```text
/tmp/lio-ro-server-evidence.naLgdF
```

当时回读 TPG：enable=1、authentication=0、generate_node_acls=1、cache_dynamic_acls=1、demo_mode_write_protect=0、prod_mode_write_protect=0、demo_mode_discovery=1。

LUN 0 的 data 规范目标仍为：

```text
/sys/kernel/config/target/core/iblock_99/lio_ro_write_ro
```

后端 info 原文：

```text
Status: ACTIVATED  Max Queue Depth: 0  SectorSize: 512  HwMaxSectors: 240
        iBlock device: dm-20  UDEV PATH: /dev/mapper/vg2-LIO_RO_PROBE  readonly: 1
        Major: 253 Minor: 20  CLAIMED: IBLOCK
```

后端 enable=1，序列号仍为 202610200001；后端文件标识 13:102580。独立 udev_path 文件仍为空。

### 11.2 底层实例没有被替换

sysfs 的 dev、dm/name、dm/uuid、uevent 与原实验保存的身份一致，仍为 253:20/31、vg2-LIO_RO_PROBE。

mapper 节点的十六进制主次设备号为 fd:14，对应十进制 253:20。首扇区采集前后两份完整现场身份比较一致。

因此没有把当前某个同名但不同实例的 LV 当作原测试对象。

### 11.3 直接读取 LV 首扇区

通过 /dev/mapper/vg2-LIO_RO_PROBE 只读采集 512 字节，保存为：

```text
/tmp/lio-ro-server-evidence.naLgdF/ro.server-sector0.bin
```

SHA256 为：

```text
81ba4aca90b1bc03819b12005043509255d9dd3a232d0ec29ea2b3937ab60558
```

它与客户端 payload.bin、ro.after 完全相同。至此，配置、引用、设备实例、客户端响应、客户端回读和服务端底层数据组成了对应的失败证据。

这里使用普通块设备读取，没有主动刷新缓存，不把结果解释为断电后的介质持久性证明。

### 11.4 失败结论及根因边界

可以确定：在本机 Linux 6.12.71 和本次自动接入参数下，单独设置 iblock readonly=1 不能可靠阻止客户端的原始 WRITE(10)。数据实际改变，不是只发生了退出码或文本解析异常。

不能据此断言：所有 Linux 版本、所有 fabric、所有静态 ACL 或全部只读机制都失效；也不能仅凭这一实验给出最终内核缺陷归因。

讨论中查到的上游代码线索涉及自动映射的 lun_access_ro 与 demo_mode_write_protect，以及写命令对映射权限的检查。这与现象吻合，但没有逐项确认本机 6.12.71 构建树及 OpenWrt 补丁，因此保留为待核查线索，不作为已验证根因。

## 12. 从实验得到的第四阶段业务约束

### 12.1 现场核对权限

执行器可以为已确定的预期读取现场并比较；不一致时记录预期、实际值和失败位置，停止依赖该事实的动作。

读取现场得到的新实例，不能替换旧任务的冻结实例；例如 253:2/29 重新激活为 253:2/30 后，不得仅因 DM_NAME、DM_UUID 相同，就认为旧发布已经自行完成退出。

后端 info、sysfs 和对象引用提供的证明范围不同。尚无能力确认完整事实时返回错误，不能通过扩大读取或自动接管把故障掩盖成成功。

### 12.2 发布位置释放与后端释放分开

单个发布位置可以先解除引用。后端是否回收，要由剩余发布引用和准确对象归属决定。最后一个发布位置消失后，后端仍可能占用设备。

即使发布状态已经清除，未成功回收的后端仍要由后端状态承接责任，不能失去记录后被当作无事发生。

这是一项设计约束；当前发布状态和后端状态的正式接口仍需后续文档及代码落地。

### 12.3 remove 重新定位

设备 remove 是设备生命周期事件，不天然授予“正常释放 LIO”的业务授权。

正常释放后到达的 remove、没有发布过的设备 remove，都可能是正常观察；已知受管后端仍应持有设备时出现 remove，则需要按冻结身份核查并报告异常。不能统一把所有 remove 当成正常清理命令。

### 12.4 历史身份缓存的重大调整方向

原第三阶段独立历史缓存用于在 remove 时找回已经消失的设备身份。引入必须独立维护的后端记录后，后端本身也必须保存完整冻结身份，出现重复维护和一致性负担。

本页已记录方向：评估取消独立历史缓存，由后端记录承接受管设备身份；无准确后端关联的 remove 可过滤，读取失败或记录损坏仍报错。

这项调整尚未实施。当前第三阶段代码仍有 /tmp/lio-targetd/identities，不能把本文的新方向当作现有代码行为。

### 12.5 不扩大当前实现承诺

本页实验没有实现 UCI 状态提交、五项逻辑接口、正式共享对象归属、发布事务的失败处理或自动化 configfs 写入。它们不能因为底层 mkdir、ln、rmdir 成功就宣称闭环。

## 13. 未解决、未测试与后续可选实验

| 事项 | 当前状态 | 后续必须回答的问题 |
| --- | --- | --- |
| ro 的可靠实现 | 初版明确延后 | 如何让所有接入客户端的原始写请求被服务端拒绝，并在回读中证明数据未变 |
| demo_mode_write_protect=1 对照 | 只提出，未执行 | 新建或重新生成映射后，WRITE(10) 是否得到 Data Protect；不能只看属性回读 |
| 同一 TPG 不同 LUN 的混合 ro/rw | 未找到通过实测的路径 | 不能用整个 TPG 的共享开关替代逐 LUN 策略 |
| 静态 mapped LUN 的实际拒写 | 仅参数独立性验证 | 原始 SCSI 请求是否被拒绝；该方案如何适配不预登记客户端的需求 |
| 后端完整实例证明 | info 不够完整 | 冻结身份、受限现场核对、后端生命周期记录怎样组合成明确证据 |
| 设备异常消失 | 未测试物理掉盘或强制 DM 移除 | 已持有后端会怎样报错；哪些读回仍可信；如何保存故障归属 |
| 客户端仍在线时撤销发布 | 未验证 | 会话、在途 I/O、引用释放和后端回收分别怎样确认 |
| 多 target、跨 TPG 的后端共享 | 尚未完整测试 | 引用枚举及回收范围能否覆盖所有受管位置 |
| 多客户端并发读写 | 未测试 | 本项目的配置策略与使用方责任如何落实；不能推导文件系统多写安全 |
| 外部管理者并发修改 configfs | 未验证 | A 只能协调本方案进程；如何识别外部变化并明确报错 |
| 写入后状态登记失败、进程崩溃 | 未实现 | 怎样保留现场、阻止重复接管、交代部分成功 |
| 自动启动与重启恢复 | 未实现完整流程 | configfs 准备、状态重建、遗留对象识别各由谁负责 |
| 读写性能及长期稳定性 | 未测试 | 本轮单扇区实验不能代替压力、断网、断电和故障注入测试 |
| RW 服务端补证后半段 | 因 od 缺失未执行 | 对照端已有客户端写入回读证明；若继续取证，应单独读 RW，不能冒充原脚本已经完成 |

以后研究 ro 时，应使用专用临时资源，重新冻结对象身份、读取写前数据、发送可区分的新载荷，再核对响应与写后内容。不能直接复用“原先应该全零”的前提，因为本轮两个临时 LV 的首扇区已被写入。

初版实现不必等待这些 ro 研究；现阶段按仅 rw 的已确认需求继续。

## 14. 换页时的实验资源交接

### 14.1 最后已知对象状态

| 对象组 | 最后已知状态 | 后续操作前的注意点 |
| --- | --- | --- |
| configfs | 已挂载 /sys/kernel/config，iSCSI fabric 已存在 | 重启后需重新核对，不能照搬本次挂载结论 |
| iblock_99 | 保留，容纳本轮其他后端 | 不能把它当作空目录整体删除 |
| lio-relation-probe target / tpgt_1 | 空外壳保留，TPG 禁用，无 portal | lio_relation_probe 后端已删除，LUN 已删除 |
| CF-04 设备占用实验后端 | 已删除 | 原始输出明确确认回收；不要凭未知名字再重复删除 |
| vg2/BULKTEST00 | 重新激活为 253:2/30 | 后来的静态 ACL 实验再次绑定并占用它 |
| lio-access-probe target / tpgt_1 | 保留，TPG 禁用，无 portal | 包含 LUN 0、LUN 1 和显式静态 ACL 映射 |
| lio_access_probe 后端 | 保留，readonly=0，绑定 BULKTEST00 实例 30 | 不要直接普通停用该 LV 而期待 remove 自动清理 |
| lio-access-client 的两个映射 | 最后为 write_protect=1 / 0 | 这是静态属性实验，不是最终正式权限配置 |
| lio-ro-write-test target / tpgt_1 | 保留并启用 | portal 为 172.16.16.220:32690；客户端会话未注销 |
| lio_ro_write_ro | 保留，readonly=1，实例 253:20/31 | 原始 SCSI 写入已成功，不能视为可靠只读 |
| lio_ro_write_rw | 创建后保留，readonly=0，实例 253:21/32 | 客户端 RW 对照成功；最后服务端补证未继续采集到它 |
| 两个临时 LV | 未收到清理完成输出 | 首扇区已写入载荷，不是初始全零状态 |
| SandPlug 专用 node / session | 保留，node 为手工启动 | 下次先检查会话；不要重复运行要求 node 不存在的创建脚本 |

这里不下发清理命令。后续清理必须先确认准确对象、会话及引用，按实际依赖逆序处理；保留外壳的策略与删除专用临时 LV 的授权也应分开。

### 14.2 证据文件索引

| 所在主机 | 路径 | 内容 |
| --- | --- | --- |
| OpenWrt | /tmp/lio-ro-write-test/ro.identity | RO 临时 LV 原始冻结身份 |
| OpenWrt | /tmp/lio-ro-write-test/rw.identity | RW 临时 LV 原始冻结身份 |
| OpenWrt | /tmp/lio-ro-server-evidence.naLgdF/report.txt | 服务端补证日志，止于 od 缺失 |
| OpenWrt | 同目录 ro.identity.before / ro.identity.after | 首扇区采集前后身份，比较一致 |
| OpenWrt | 同目录 ro.server-sector0.bin | 从 RO 临时 LV 直接读取的 512 字节 |
| SandPlug | /tmp/lio-ro-client.AWZyoE | 最初客户端实验目录，会话创建及后续证据根 |
| SandPlug | 上述目录下 continue.cBx2ID | 完整 VPD、载荷、写前写后数据及 WRITE 日志 |

客户端 continue.cBx2ID 中的关键文件包括：vpd0.expected、vpd0.actual、vpd1.expected、vpd1.actual、payload.bin、ro.before、ro.after、rw.before、rw.after、ro.write.log、rw.write.log。

以上远端 /tmp 文件可能因重启消失。本文已固化关键身份、参数、响应和哈希；本文没有把远端二进制文件下载或归档，不能声称它们已随文档永久保存。

## 15. 后续文档与旧草案的适用关系

旧《openwrt lio-targetd 完整设计草案 v0.1》中“尚未验证真实 LV 绑定和客户端数据读写”的描述，应按本文更新：绑定、共享引用、无认证登录及单扇区 RW 已通过；可靠 ro 没有通过，初版已明确延后。

旧草案把 remove 与撤销发布紧密关联的设想，也必须结合本文 CF-04 的设备占用结果重新划界：计划释放接口与设备 remove 观察接口分开。

本文件是独立新增实验记录，没有原位修改旧草案、既有脚本或运行配置。之后的前三阶段实现记录、第四阶段业务编排稿，应引用这里的实机证据，并明确哪些旧接口仍在代码中、哪些调整尚待实施。

换页后最重要的已知事实是：引用关系与 RW 数据链路已经验证；后端回收与发布位置释放必须分别确认；仅 readonly=1 的服务端拒写方案已被实机否定；初版仅支持 rw。

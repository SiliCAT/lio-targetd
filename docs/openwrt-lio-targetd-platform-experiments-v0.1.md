# OpenWrt lio-targetd 平台适配与事件交接实验成果 v0.1

日期：2026-10-19  
对象：PogoPlug V4／128 MiB RAM／OpenWrt 25.12.0／Linux 6.12.71  
配套设计：《openwrt lio-targetd 完整设计草案 v0.1》  
文档性质：实机实验记录、结论边界与后续设计依据。  
状态：平台基础能力已取得实机证据；身份缓存规则形成讨论口径；正式业务脚本、完整调度协议和真实 LIO 数据访问尚未在本轮实现或验证。

## 1．用途与阅读约定

本文件整理本轮从只读盘点到真实 block 事件分发复验的全部关键成果。换页后，携带本文和完整设计草案，即可继续接口与自然语言程序编排，无需重新从零确认平台能力。

本轮依次完成：

1. 正式 OpenWrt 环境、shell、工具、hotplug 分发入口与 DM 身份只读盘点。
2. 文件描述符锁的竞争、释放、阻塞等待和跨 `exec` 继承实验。
3. 测试 LV 使用状态只读检查。
4. 单个 LV 停用、重新激活时的真实 remove／add／change 采样。
5. 直接调用 `hotplug-call` 的后台脱离与锁描述符隔离实验。
6. 通过 sysfs 提交两条带标识的人工 change，复验实际 block 事件分发链。

全文区分三种状态：

| 状态 | 含义 |
|---|---|
| 实机已验证 | 用户贴回的实机输出直接支持 |
| 设计讨论成果 | 根据证据形成的责任、接口或生命周期口径，尚未实现 |
| 待定／未验证 | 下一阶段仍需解决，不能按已实现能力引用 |

本文代码块分为原始证据摘录和关键实现摘录。后者用于解释实验方法，不是带完整初始化、保护和清理逻辑的生产脚本，也不应把各个片段拼接后直接执行。

## 2．原设计背景与本轮验证目标

lio-targetd 的目标是把配置选定的本地 LVM／DM 块设备，通过内核 LIO iblock 后端发布成 iSCSI LUN。用户空间编排有事运行、收敛完成后退出，内核发布对象继续工作。

既有调度设计只使用两把锁：

| 名称 | 建议生产路径 | 职责 |
|---|---|---|
| G：共享交接锁 | `/tmp/lio-targetd/handoff.lock` | 入队、请求信号、批次移交、执行权取得与退出交接 |
| A：执行权锁 | `/tmp/lio-targetd/active.lock` | 唯一执行器，从取得执行权持续覆盖到正常退出交接 |

正常生产者先等 G，在 G 内入队并设置请求信号，再非阻塞竞争 A。获胜者保持 A，通过 `exec` 接续执行器；未获 A 者在提交成功后退出。执行器完成业务后，在 G 内确认没有剩余工作，先释放 A，再释放 G。

关键顺序约束仍是：生产者持有 G 时不得阻塞等待 A；执行器不能先释放 G 再释放 A。本轮没有改变这套协议，也没有增加第三把锁。

本轮要验证的是上述设计依赖的目标环境接口，包括事件身份、shell 描述符能力、锁工具行为，以及后台请求是否影响 hotplug 分发。它不是完整协议的并发正确性测试。

## 3．正式系统与工具盘点

### 3.1 系统实测值

| 项目 | 实机结果 |
|---|---|
| 硬件项目对象 | PogoPlug V4 |
| 内核 | `6.12.71` |
| 架构输出 | `armv5tel` |
| OpenWrt | `25.12.0` |
| 发布修订 | `r32713-f919e7899d` |
| target | `kirkwood/generic` |
| 软件架构 | `arm_xscale` |
| 执行身份 | root，UID／GID 均为 0 |
| PID 1 | `/sbin/procd` |
| `/bin/sh` | 符号链接到 `busybox` |
| `/bin/ash` | 符号链接到 `busybox` |
| 检测 shell 的 `/proc/$$/exe` | `/bin/busybox` |
| BusyBox 版本 | applet 帮助显示 `v1.37.0 (2026-03-03 00:14:15 UTC)` |

由此确定正式脚本可围绕目标 BusyBox ash 编排，不能照搬构建主机 Bash 或 util-linux 特有接口。

### 3.2 实际工具入口

| 命令 | 实机入口／状态 |
|---|---|
| flock | `/usr/bin/flock`；帮助表明当前为 BusyBox 实现 |
| lock | `/bin/lock`；帮助显示 `No help available`，本轮未采用 |
| sleep | `/bin/sleep` |
| usleep | 未找到 |
| timeout | 未找到 |
| date | `/bin/date` |
| readlink | `/usr/bin/readlink` |
| stat | `/bin/stat` |
| awk／sed／grep | 分别为 `/usr/bin/awk`、`/bin/sed`、`/bin/grep` |
| env／logger | 分别为 `/usr/bin/env`、`/usr/bin/logger` |

直接运行 `busybox` 时返回 `busybox: applet not found`；`busybox --list` 没有取得有效 applet 清单。多个实际 applet 正常工作，版本也从帮助取得，因此本轮不把该自检结果解释为 BusyBox 整体损坏，其具体构建原因未研究。

### 3.3 计时能力边界

`sleep --help` 表示可以接受带 s／m／h／d 单位的参数，但本轮没有验证小数秒。

`date '+%s %N'` 仅取得秒数 `1792427667`，未取得可用的纳秒字段。该命令在此镜像上没有提供本轮所需的亚秒时间接口。

`/proc/uptime` 有小数输出，例如 `27877.17 27815.42`，后续实验使用它记录系统运行时间。

草案参考滤波值为 2000 ms，即 2 秒；现有整数秒 `sleep` 足以开展这类等待实验。任意毫秒滤波、计时误差和高负载下的调度延迟均未验证。

## 4．hotplug 分发与 LVM 既有入口

### 4.1 实际入口与配置

`/proc/sys/kernel/hotplug` 内容为 `/sbin/hotplug`。PID 1 为 procd，系统存在 `/etc/hotplug.json` 与 `/sbin/hotplug-call`。本轮按本机 OpenWrt 的实际事件分发链进行观察；没有单独追踪每个事件在内核 helper 与 netlink 接收侧的全部内部路径。

`hotplug.json` 中，普通 add 的设备节点创建动作位于后续类别脚本分发之前；remove 的 `/dev/%DEVNAME%` 删除动作也位于类别分发之前。最终按 SUBSYSTEM 是否存在对应目录，调用 `hotplug-call`。

`/sbin/hotplug-call` 的关键原文：

```sh
export HOTPLUG_TYPE="$1"
. /lib/functions.sh
PATH="/usr/sbin:/usr/bin:/sbin:/bin"
LOGNAME=root
USER=root
export PATH LOGNAME USER
export DEVICENAME="${DEVPATH##*/}"

if [ \! -z "$1" -a -d /etc/hotplug.d/$1 ]; then
        for script in $(ls /etc/hotplug.d/$1/* 2>&-); do (
                [ -f $script ] && . $script
        ); done
fi
```

可直接采用的结论：

- `DEVICENAME` 是分发脚本由 DEVPATH 尾部导出的变量，可用于识别 `dm-*`。
- 类别文件在各自子 shell 中被 source；可以 `return`，不依赖各文件自行 exec。
- 本机原有 `20-lvm2` 权限为 0644，符合 source 调用方式。
- 计划中的 `80-lio-targetd` 会排在 `20-lvm2` 之后；本轮事件探针使用 79，同样位于它之后。
- 类别脚本仍需要自行筛选，本机没有替本方案预先按 DM_NAME 精确分发。

### 4.2 LVM 现有职责保持不变

原有 block 脚本 `/etc/hotplug.d/block/20-lvm2` 原文：

```sh
[ "$ACTION" = "add" ] || return 0
[ -e "/dev/$DEVNAME" ] || return 0

/sbin/lvm vgscan --mknodes --devices /dev/$DEVNAME || :
/sbin/lvm vgchange -aly --devices /dev/$DEVNAME || :
```

它只检查 add 和设备节点存在，没有排除 `dm-*`。从脚本条件看，DM add 也可能进入这两条 LVM 命令；本轮没有因此修改它，也没有据此推断存在无限递归。

另外还有 `/lib/preinit/80_lvm2` 与 `/etc/init.d/lvm2`：preinit 和正常启动入口负责扫描、激活 LVM；init stop 会执行停用。lio-targetd 消费 DM 事件，不承担或替换这些激活职责。

## 5．当前 DM 设备与身份只读盘点

当时共观察到 20 个 DM 映射：

| 内核名称 | DM_NAME | 主次设备号 |
|---|---|---|
| dm-0 | vg2-SWAP | 253:0 |
| dm-1 | vg2-USBROOT | 253:1 |
| dm-2～dm-19 | vg2-BULKTEST00～vg2-BULKTEST17，依次对应 | 253:2～253:19 |

遍历输出中 `dm-10` 排在 `dm-2` 前，是文件名展开的字典序，不能当作设备创建顺序。

设备存在时，可通过本次 `dm-N` 定点读取：

```text
/sys/class/block/dm-N/dev
/sys/class/block/dm-N/dm/name
/sys/class/block/dm-N/dm/uuid
/sys/class/block/dm-N/uevent
```

例如实验前的 dm-2：

```text
sysfs 实际路径：/sys/devices/virtual/block/dm-2
dev：253:2
dm/name：vg2-BULKTEST00
dm/uuid：LVM-X2w9s3qeto0ppFI6DZbt6iM3M6VoV3DKDLhBgYmbebbwq8cGV1zTeAXGxqp5mnRa

uevent：
MAJOR=253
MINOR=2
DEVNAME=dm-2
DEVTYPE=disk
DISKSEQ=10
```

这些是当前 sysfs 快照，不是完整历史事件环境。尤其不能由快照里没有 DM_NAME，直接推导所有真实事件都没有该字段。

`/dev/mapper/vg2-BULKTEST00` 等本机入口是实际块设备节点，类型为 `b`，不是符号链接。例如该节点与 `/dev/dm-2` 都是 253:2。以后核对两条路径是否指向同一设备，应比较设备号，不能只靠解析符号链接。

这轮全量遍历只用于盘点。生产 hotplug 已知本次 dm-N 时，取名称不需要全盘扫描，也不需要调用 `lvs`。

## 6．锁与描述符实验

### 6.1 当前 BusyBox flock 接口

实机帮助：

```text
Usage: flock [-sxun] FD | { FILE [-c] PROG ARGS }

-s  Shared lock
-x  Exclusive lock (default)
-u  Unlock FD
-n  Fail rather than wait
```

本轮全部使用 FD 形式。实验中的数字描述符 8／9 是测试选择，不代表生产接口编号已经冻结。

### 6.2 基础继承与锁操作结果

最早的只读检查中，shell 以 `exec 8</dev/null` 打开 fd 8，再 `exec` 另一个 shell。前后均能读取 `/proc/$$/fd/8` 指向 `/dev/null`，返回值 0。该阶段只证明描述符继承；后续锁实验补上了锁所有权验证。

专门锁实验使用独立目录 `/tmp/lio-lock-test.fJJiPo`，取得结果：

| 操作 | 原始结果 | 可引用结论 |
|---|---|---|
| fd 8 取得独占锁 | 成功 | 目标工具可对已打开描述符加锁 |
| 竞争者先关闭继承的 fd 8，再独立打开同一文件竞争 | 返回 1 | 独立竞争受到独占锁约束 |
| 对已关闭 fd 7 执行 flock | 返回 1；`flock: : Bad file descriptor` | 错误与锁忙的退出码相同 |
| 显式 `flock -u 8` 后竞争 | 返回 0 | 显式释放有效 |
| 重新取得锁，关闭最后持有描述符后竞争 | 返回 0 | 关闭最后持有描述符可释放锁 |
| 持锁 shell exec 新 shell | 前后 PID 均为 3370；fd 8 保留 | 程序接续不需要释放锁重新竞争 |
| exec 后独立竞争 | 返回 1 | 接续后锁仍排斥竞争者 |
| 接收者关闭 fd 8 后竞争 | 返回 0 | 接收者可以结束该执行权持有 |
| 独立等待者阻塞等待，持有者解锁 | 等待者取得锁并返回 0 | G 的阻塞等待／释放唤醒路径可用 |

原始关键输出：

```text
比较：锁忙=1；操作错误=1
exec 前 PID=3370；已取得锁。
exec 后 PID=3370；原 PID=3370
继承的 fd 8：/tmp/lio-lock-test.fJJiPo/active.lock
exec 后独立竞争返回值=1
接收者关闭 fd 8 后竞争返回值=0
exec 实验进程返回值=0
释放前：尚未观察到等待者取得锁。
持有者：现在显式释放 fd 8 的锁。
等待者：已取得锁并退出。
解锁返回值=0
等待者返回值=0
```

### 6.3 实验方法中的关键细节

竞争者必须独立打开文件，不能拿继承的同一锁描述符冒充独立竞争：

```sh
# 原持有者：fd 8 已锁定。
(
    exec 8>&-
    exec 9>>"$lio_test_dir/active.lock" || exit 90
    flock -xn 9
)
```

`exec` 接续时保留原持锁描述符，接收者再用独立打开的描述符测试是否仍有排他性。这正是正式请求进程直接成为执行器的能力依据。

### 6.4 工具适配仍未完成

不能采用“flock 返回非零，就当作已有执行器接单”的写法。实机已经证明坏描述符和锁忙都返回 1。

本轮后续实验继续使用现有 BusyBox flock，没有安装新工具。正式实现的建议是选择 util-linux 版 flock：其 `-E` 可指定锁冲突专用退出码，其他失败有独立错误分类接口。选定退出码、目标 OpenWrt 构建包、实际安装路径、与现有 applet 的关系及实机验收尚待处理。

这是用户空间工具适配问题，不要求改变 G／A 协议。本轮没有把 stderr 文本匹配冻结为可靠生产接口，也没有验证当前 BusyBox 支持 util-linux 特有选项。

## 7．测试 LV 前置检查与操作边界

选定对象只有 `vg2/BULKTEST00`，当时为 dm-2／253:2。

停用前的只读检查取得：

| 检查 | 输出 |
|---|---|
| 当前挂载命名空间的 mountinfo 中匹配 253:2 | 无记录 |
| `/proc/swaps` | 只有表头，未列出启用的 swap |
| `/sys/class/block/dm-2/holders/` | 空目录 |
| configfs 已有 backstore 的 `udev_path` | 本次扫描没有输出 |

这些检查没有发现用途冲突，但不能单独证明任意进程都没有打开设备，也不是所有挂载命名空间的普查。

实际操作仅为：

```sh
/sbin/lvm lvchange -an vg2/BULKTEST00
/sbin/lvm lvchange -aly vg2/BULKTEST00
```

二者均返回 0；停用未使用强制选项。没有操作 SWAP、USBROOT 或其他 BULKTEST 卷，没有创建、删除、格式化 LV，也没有执行真实 LIO 发布。

## 8．真实 LV 停用／激活事件采样

### 8.1 采样位置与方法

临时探针路径为 `/etc/hotplug.d/block/79-lio-event-probe`，通过符号链接指向 `/tmp/lio-event-probe.XXXXXX/probe`。探针位于既有 `20-lvm2` 之后。

探针仅记录 `DEVICENAME=dm-*` 的事件，保存指定环境变量、当时 sysfs 属性和 `/dev/$DEVNAME` 状态。每条事件使用独立临时文件，记录结束后 rename 成 `.done`，主进程只读取已完成的记录。

主要环境字段采集表达式：

```sh
env | grep -E '^(ACTION|DEVPATH|DEVNAME|DEVICENAME|SUBSYSTEM|SEQNUM|MAJOR|MINOR|DEVTYPE|DISKSEQ|DM_[A-Z0-9_]+|HOTPLUG_TYPE)='
```

停用成功后等待 3 秒，重新激活成功后等待 5 秒，再打印记录。文件名随机，打印顺序不能当成事件顺序，需看 SEQNUM。有限观察窗口不构成任意负载下的事件完整性保证。

### 8.2 按事件序号还原的结果

全部四条事件共同具有：

```text
DEVNAME=dm-2
MAJOR=253
MINOR=2
HOTPLUG_TYPE=block
DEVPATH=/devices/virtual/block/dm-2
DEVICENAME=dm-2
SUBSYSTEM=block
DEVTYPE=disk
```

变化字段与现场读取结果：

| SEQNUM | ACTION | DISKSEQ | DM_COOKIE | UPTIME 第一项 | sysfs | `/dev/dm-2` |
|---|---|---|---|---|---|---|
| 623 | remove | 10 | 4194304 | 35863.36 | 不存在 | 不存在 |
| 625 | remove | 10 | 无该字段 | 35863.44 | 不存在 | 不存在 |
| 627 | add | 28 | 无该字段 | 35866.64 | 存在；名称、UUID 可读 | 存在，253:2 |
| 628 | change | 28 | 4194304 | 35866.75 | 存在；名称、UUID 可读 | 存在，253:2 |

四条事件环境均没有采到 DM_NAME 或 DM_UUID。add／change 的名称和 UUID 来自当时的 sysfs 定点读取，必须与事件本身携带字段区分。

两条 remove 的关键原文：

```text
ACTION=remove
SEQNUM=623
DM_COOKIE=4194304
DISKSEQ=10
SYSFS_EXISTS=0
ls: /dev/dm-2: No such file or directory

ACTION=remove
SEQNUM=625
DISKSEQ=10
SYSFS_EXISTS=0
ls: /dev/dm-2: No such file or directory
```

add／change 均可读到：

```text
dev：253:2
dm/name：vg2-BULKTEST00
dm/uuid：LVM-X2w9s3qeto0ppFI6DZbt6iM3M6VoV3DKDLhBgYmbebbwq8cGV1zTeAXGxqp5mnRa

uevent：
MAJOR=253
MINOR=2
DEVNAME=dm-2
DEVTYPE=disk
DISKSEQ=28
```

### 8.3 设备身份与实例身份的证据

| 属性 | 停用前 | 重新激活后 |
|---|---|---|
| dm-N | dm-2 | dm-2 |
| 主次设备号 | 253:2 | 253:2 |
| DM_NAME | vg2-BULKTEST00 | vg2-BULKTEST00 |
| DM_UUID | 同上完整 LVM UUID | 相同 |
| DISKSEQ | 10 | 28 |

据此可直接确定：

1. dm-N 与主次设备号会被复用，不能单独当作持久业务名称或激活实例身份。
2. 同一个 LV 再次激活，DM_UUID 可以不变。本例证明它不能单独区分这两次激活。
3. 本例 DISKSEQ 区分了旧、新设备实例；SEQNUM 则区分各条事件先后，两者职责不同。
4. remove 到达本轮探针时，原 sysfs 与设备节点已经消失；不能把“收到 remove 后现查名称”当作可靠入口。
5. 一次停用观察到同一 DISKSEQ 的两条 remove，重复事件必须能合并或幂等处理。

SEQNUM 的空缺不能证明漏采：该探针没有收录其他类别和不匹配设备的事件。两条 remove 的具体产生原因尚未确定；不能用是否带 DM_COOKIE 作为接收条件或业务身份。

### 8.4 add 可读性的适用范围

本次在 79 位置读取 add 时，DM_NAME、DM_UUID 均可用，支持在前端定点取身份后再筛选。该位置在 20-lvm2 之后，不能外推为设备刚创建的任意时刻都已就绪。

正式实现仍需处理早到 add、读取失败、读取期间设备切换，以及后续 change。延迟事件读到当前 sysfs 时，也必须核对实例信息，防止把复用编号的新设备身份贴到旧事件上。

## 9．身份缓存的设计讨论成果

本节是由实验证据推动的设计口径，尚无正式缓存文件、生产实现或生命周期联合测试。

### 9.1 写入权与现场采集分工

用户明确提出缓存由执行器维护、其他进程只读，并认可“现场身份随事件传递”。本轮据此采用以下责任方向：

| 角色 | 职责 |
|---|---|
| hotplug | 采集事件字段；可读时取得现场 DM_NAME／DM_UUID；必要时只读缓存辅助筛选 |
| 请求进程 | 携带现场信息入队，保留设备实例信息；不修改身份缓存 |
| 持有 A 的执行器 | 唯一缓存写入者；核对、建立、更新与删除记录，并与 LIO 操作协调 |
| 开机入口 | 提交已有设备检查，由执行器补齐身份记录 |

不能把身份采集全部延迟到执行器：设备可能在排队期间消失。hotplug 已成功取得的名称、UUID 应随请求保存。

建议请求携带 ACTION、SEQNUM、主次设备号、DISKSEQ，以及现场成功采集的 DM_NAME／DM_UUID。缺失值、校验规则和最终队列格式尚未冻结。

### 9.2 实例对应关系

建议缓存表达：

```text
主次设备号＋DISKSEQ → DM_NAME、DM_UUID
```

这不等于已经确定文件名或索引格式。DISKSEQ 缺失时如何处理、快照核对方法、启动补齐时序及异常恢复仍需编排。

缓存不能只把 dm-2 作为一个可覆盖槽位。例如旧 remove 尚未处理时，新 add 可能已经先入队或先被处理；只按 dm-2 查名称会混淆 DISKSEQ=10 与 28。

### 9.3 缓存未命中与只读发布

hotplug 中缓存未命中，不等于该设备不受管。此前 add 可能尚未被执行器消费，因此不能仅凭 miss 就丢弃 remove；应让请求携带可取得的实例信息，由执行器裁决。

只读进程不必为了读缓存进入 G。建议执行器先完整写临时文件，再在同一文件系统内原位 rename，供读者取得完整旧记录或新记录；读取失败按身份待定处理。该写入方案目前是建议，尚未实现验证。

### 9.4 删除时点

用户提出“关闭对应 LIO 发布时删除”。讨论将其细化为：执行器确认该设备实例的全部受管发布已经撤销、相关清理完成后，删除该实例记录。

| 情况 | 处理口径 |
|---|---|
| 一个 LUN 撤销，其他引用仍存在 | 保留 |
| 撤销或相关清理失败 | 保留 |
| 所有对应发布和相关清理完成 | 删除该实例记录 |
| 旧实例清理完成，新实例已出现 | 只删旧实例，不影响新实例 |

在受管发布之前应确保对应身份记录已经存在。开机补齐也要覆盖设备已出现但不会再来 add 的情形。

删除后迟到的事件仍按实例核对：已有 DM_NAME 的请求可以重查；只有旧实例编号且缓存已不存在的请求，不能套用当前同编号设备的名称。能证明该旧实例已无受管对象时，重复 remove 可结束处理；不能证明时如何待查／重查尚待定义。

本轮不据此宣称已经解决任意乱序、崩溃或缓存丢失情况下的身份恢复，也没有要求为重复 remove 永久保留全部历史。缓存缺失分支和清理后的可判定依据必须在正式编排中补齐。

## 10．直接调用 hotplug-call：后台脱离与描述符隔离

### 10.1 实验结构

建立独立的 `/etc/hotplug.d/lio-probe-<PID>/` 测试类别，以符号链接加载临时脚本，不改正式 block 文件。主进程持有临时 G，直接调用 `/sbin/hotplug-call <测试类别>`。

类别脚本启动请求进程后返回；请求进程独立打开 G 并阻塞等待。关键启动形式为：

```sh
/bin/sh "$lio_dir/request" "$lio_dir" \
    </dev/null >"$lio_dir/request.log" 2>&1 8>&- 9>&- &
return 0
```

该形式使用脚本文件，标准输入为 `/dev/null`，输出进入独立临时日志，并在启动侧关闭测试用无关锁描述符。后台启动后的父进程返回值不代表请求已经取得锁或完成业务，实验另用阶段标记观察。

防卡住保护在 8 秒仍未观察到分发返回时释放测试锁；实机没有触发该保护分支。

### 10.2 分发返回与后台继续运行的原始证据

```text
hotplug 脚本：已启动后台请求，即将返回。
hotplug-call 返回值=0
调用前 uptime=37689.49；返回后 uptime=37689.54
分发已经返回；请求仍在等待 G。

释放 G 后：
请求进程：准备阻塞等待 G。
请求进程：已经取得 G，继续运行 2 秒。
请求进程：完成。
```

本次调用约 0.05 秒返回。这是单次观测，不是性能上界。它直接验证 shell 分发这一段没有等待后台请求完成；当时尚未单独覆盖 procd 接收真实事件的完整分发路径，下一节补上该证据。

### 10.3 长业务子进程的锁隔离

主进程模拟执行器持有 G／A，启动一个持续 5 秒的业务子进程。启动侧关闭 fd 8、9，子进程检查自身 `/proc/$$/fd/8` 和 `/proc/$$/fd/9`。随后主进程先关闭 G，再关闭 A，独立进程分别竞争。

结果：

```text
业务子进程的 fd 8、9：
fd 8 已关闭。
fd 9 已关闭。

独立竞争 G 返回值=0
独立竞争 A 返回值=1
关闭 A 后独立竞争返回值=0
此时业务子进程尚未结束。
业务子进程返回值=0
```

由此确认：关闭无关描述符后，仍运行的业务子进程不会延长这些锁的持有；主执行器保持 A 时，A 仍具有排他性。

本段业务子进程采用 `sh -s` 加 heredoc 提供程序，stdin 最终由 heredoc 占用，因此本段只作为锁描述符隔离证据；请求进程使用脚本文件加 `/dev/null` 的形式才是上述后台启动方式的依据。

### 10.4 实验动作与生产退出规则的区别

为了证明子进程不拖住锁，实验刻意在业务子进程仍运行时关闭 A。这不是正式执行器的退出示例。

正式执行器必须等待本轮业务子进程完成，再做 clean 判断和 G 内退出交接。否则即使子进程已正确关闭 A，也可能让下一位执行器与旧业务同时操作 LIO。

本轮验证了正常执行下描述符隔离的效果；执行器异常死亡时如何收束仍在运行的业务子进程，尚未设计或验证。

## 11．真实 block 事件分发链复验

### 11.1 要验证的问题

直接调用 hotplug-call 能返回，仍不足以证明实际系统事件分发会继续推进。本轮因此验证：第一条真实 block 类别事件启动的请求等待 G 时，第二条事件能否继续进入同一类别脚本并启动自己的请求。

此处“真实分发链”指事件经内核 uevent 入口送入本机实际分发机制；事件来源是人工 change，不是新的硬件插拔或实际 LV 状态变化。

### 11.2 实验标识与事件提交

对象为当前 `vg2-BULKTEST00` 对应的 dm-2。临时探针为 `/etc/hotplug.d/block/79-lio-chain-probe`，只接收同时匹配以下条件的事件：

```text
ACTION=change
DEVICENAME=本次选定的 dm-N
SYNTH_UUID=本次实验 UUID
SYNTH_ARG_STEP=1 或 2
```

本次实验 UUID：

```text
603c4368-5e46-4ecf-8608-8aab34e580cf
```

内核 sysfs uevent 扩展格式为 `ACTION [UUID [KEY=VALUE ...]]`。UUID 形成环境变量 SYNTH_UUID；STEP=1／2 形成 SYNTH_ARG_STEP。关键提交形式：

```sh
printf 'change %s STEP=%s\n' "$lio_uuid" "$lio_step" \
    >"/sys/class/block/$lio_dm/uevent"
```

主进程先持有 G，提交第一条后等待请求 1 的 waiting 标记；保持 G 不释放，再提交第二条并等待请求 2 的 waiting 标记。每阶段有有限观察时间。两个请求均通过脚本文件后台启动，关闭继承的 8／9，独立打开同一临时 G。

实验只提交人工 change，没有停用、激活、格式化 LV，也没有绑定或撤销 LIO 对象。已盘点的原有 20-lvm2 只响应 add，因此这两条 change 不进入其 LVM 激活命令。

### 11.3 原始关键证据

```text
提交第 1 条 change。
已观察到请求 1 进入等待。
提交第 2 条 change。
已观察到请求 2 进入等待。

STEP=1 ACTION=change DEVICENAME=dm-2 SEQNUM=629
SYNTH_UUID=603c4368-5e46-4ecf-8608-8aab34e580cf
UPTIME=37925.03 37852.73

STEP=2 ACTION=change DEVICENAME=dm-2 SEQNUM=630
SYNTH_UUID=603c4368-5e46-4ecf-8608-8aab34e580cf
UPTIME=37926.03 37853.68

两个请求均已进入等待，尚未取得 G。
第二条事件已越过真实分发链，第一条后台等待未堵住它。

释放 G 后：
请求 1：开始等待 G。
请求 1：取得 G。
请求 1：完成。
完成标记存在。

请求 2：开始等待 G。
请求 2：取得 G。
请求 2：完成。
完成标记存在。

实验返回值=0
```

### 11.4 可引用结论与边界

保持 G 被占用时，两条事件都能经过实际 block 分发并建立独立请求进程；第一条请求的阻塞等待没有拖住后续分发。G 释放后，两个请求都取得过锁并完成。

两条记录约相差一秒，实验主进程使用一秒轮询等待标记，不能由此推断 procd 每条事件需要一秒。

分开打印的请求日志不能证明两者实际取得 G 的先后顺序。本轮没有把先来先得作为条件，也没有验证锁等待公平性。

本实验不是持续高频事件、进程数、内存峰值、OOM、队列容量或性能上界测试。它只证明所采用的后台启动方式在这次实际链路与锁等待条件下有效。

## 12．实验清理与副作用记录

各实验使用独立临时锁、阶段标记和日志；真实事件采样及分发复验只临时增加指定 probe 符号链接。脚本包含退出清理路径，用于撤下探针、释放测试锁、等待或结束测试请求并删除临时文件。

用户贴回的输出均到达对应完成位置，未贴出清理错误。本轮没有再独立列目录复核，因此不把“脚本配置了清理且未报告错误”扩大为对整个系统无残留的全面审计。

使用到的临时入口包括：

```text
/etc/hotplug.d/block/79-lio-event-probe
/etc/hotplug.d/block/79-lio-chain-probe
/etc/hotplug.d/lio-probe-<实验 PID>/
/tmp/lio-lock-test.XXXXXX/
/tmp/lio-event-probe.XXXXXX/
/tmp/lio-background-test.XXXXXX/
/tmp/lio-chain-test.XXXXXX/
```

实际状态变化只有单卷 `vg2/BULKTEST00` 的一次停用／重新激活，二者成功；以及其 sysfs uevent 入口的两次人工 change 提交。测试目录不属于正式 `/tmp/lio-targetd/` 运行状态。

## 13．对原完整设计草案的增量结论

本文作为配套技术依据，不直接修改或取代完整设计草案。

| 原待定事项 | 本轮推进结果 | 剩余边界 |
|---|---|---|
| 目标 shell | BusyBox ash；描述符可跨 exec 保留 | 生产编号、调用接口与所有分支清理待编排 |
| 锁语义 | 排他竞争、解锁、最后关闭释放、阻塞唤醒、exec 连续性均验证 | 完整 G／Q／R／A 协议联合实验未做 |
| flock 错误分类 | 锁忙与坏 FD 均为 1，确认存在接口缺口 | 正式建议 util-linux；包、路径、返回值适配未落实 |
| hotplug 后台启动 | 文件脚本＋独立 stdio＋关闭无关 FD 的方式可用 | 不能据此承诺无限事件下资源成本 |
| 实际分发是否被 G 等待拖住 | 两条带标识 change 的真实 block 分发通过 | 高负载和其他事件类型不在本轮覆盖范围 |
| remove 身份 | 本例无 DM_NAME／UUID，sysfs 与节点已消失 | 需实现实例缓存及缺失分支 |
| 设备编号复用 | 同一 dm-2／253:2，DISKSEQ 从 10 变 28 | DISKSEQ 缺失、身份快照核对策略待定 |
| 早到 add | 本次 79 位置可读名称、UUID，随后有 change | 不能删除未就绪分支 |
| 身份缓存 | 执行器单写；现场身份随事件；全部发布清理完成后删该实例 | 最终格式、初始化、乱序及失败恢复待编排 |

特别需要避免把本文读成以下已经完成：LIO 后端绑定、客户端发现登录、数据读写、ro／rw 实现、配置变更撤销、调度最后一条事件交接、崩溃恢复。这些均未由本轮实验验证。

## 14．换页后的直接起点

基础平台实验可以在这里告一段落。下一阶段先做接口与自然语言编排，不立即交付正式业务脚本。

建议顺序：

1. 确定轻量筛选与配置名单接口：受管名称如何从 `.luns` 派生，避免 hotplug 另维护一套业务名单；与身份缓存配合。
2. 明确身份请求字段及校验：原始动作、事件序号、设备号、DISKSEQ、可选现场名称与 UUID，及身份缺失时如何提交。
3. 编排缓存建立、只读查询、实例核对、删除与迟到事件分支；明确 G 外读写快照的边界。
4. 落实 flock 工具适配，确认目标构建输入与实际路径；复验专用锁忙返回值。
5. 编排请求脚本 G／Q／R／A 与执行器 exec 接续接口，保持已确定双锁协议。
6. 编排执行器批次、固定滤波、业务子进程等待、正常退出交接及失败分支。
7. 接口明确后做联合调度实验，重点验证 clean 附近最后一条事件始终有人接手。
8. 经明确的“落地完整代码”指令后实现正式文件，再进入真实 LIO 发布及客户端验收。

本轮最应保留的设计依据是：便宜筛选放 hotplug；现场身份随事件传递；remove 依赖提前取得的实例对应；独立请求负责等 G；获胜请求持 A 直接 exec；无关子进程关闭锁描述符；正常执行器等待业务完成后才交接退出。

## 15．证据来源与技术参考

### 15.1 主要证据

本轮用户在聊天中贴回的完整实机输出是本文件结论的主要依据，按第 1 节六个阶段组织。事件序号、DISKSEQ、PID、uptime、工具帮助和返回值保留于对应章节，属于本轮现场值，不是未来部署必须沿用的常量。

基础业务与 G／A 协议来自附件《openwrt lio-targetd 完整设计草案 v0.1》（`openwrt-lio-targetd-design-v0.1.md`）。此前 SD 启动与内核 LIO 能力成果不在本轮重复验证。

### 15.2 外部参考及其用途

- [Linux sysfs uevent ABI](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-uevent)：人工事件 `ACTION [UUID [KEY=VALUE ...]]` 格式及 SYNTH_UUID／SYNTH_ARG_* 命名。本轮两条带标识 change 已在目标机实际使用成功。
- [util-linux flock 官方手册源码](https://kernel.googlesource.com/pub/scm/utils/util-linux/util-linux/+/master/sys-utils/flock.1.adoc)：`-E` 冲突退出码及工具错误分类，用于后续工具选型建议，不代表已经在目标机安装或验证。

外部资料不替代实机证据。尤其不能用 util-linux 文档为当前 BusyBox flock 补出未列出的选项。

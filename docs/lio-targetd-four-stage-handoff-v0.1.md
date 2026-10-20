# lio-targetd 四阶段设计与实现换页文档

版本：v0.1  
整理日期：2026-10-20  
项目：PogoPlug V4 / OpenWrt / LVM / Linux LIO  
截止点：configfs 实机实验已单独成文；初版需求收缩为只支持 rw；开始整理前三阶段实现和第四阶段设计。

本文供下一页直接接续开发。正文记录业务目标、脚本边界、接口、记录格式、逻辑分支、通过的实验和未实施调整；附录完整保存三个当前源码文件，避免只留下概念而丢失可继续修改的代码基线。

## 1. 先读：当前做到哪里，下一步从哪里接

### 1.1 进度总表

| 层次 | 已有成果 | 当前完成边界 |
| --- | --- | --- |
| 周边入口 | 80-lio-targetd、iscsi-luns-queue-update 已交付 | hotplug 轻量采集、后台提交、Q/R 入队、竞争并交接 A |
| 第一阶段 | 调度骨架已实现，真实启动及事件链已验证 | A/G、Q/R/B、完整性校验、固定滤波、退出交接和失败日志 |
| 第二阶段 | 配置解析与业务判断已实现并联调 | 每活动周期一次配置视图，名单与策略独立，逐事件及逐 LUN 判断 |
| 第三阶段 | 事件历史身份、受限现场核对已实现并实测 | I1 缓存建立/复用/remove 关联、实例隔离、现场不符失败 |
| 第三阶段延期机制 | 入队扩展、队尾计数、成功移交和终止已通过独立副本实验 | 真实业务授权钩子仍为 0；不等于正式发布依赖已接通 |
| 第四阶段 | 当前状态、对象归属、发布/后端释放分离等已讨论大部分 | 正式状态表、五项接口实现和 configfs 编排尚未落地 |
| LIO 底层能力 | 手工 configfs、共享引用、后端占用、无认证接入及 RW 已实测 | 这些操作不是前三阶段执行器自动完成的 |
| ro 能力 | readonly=1 候选方案被实机否定 | 初版明确不实现 ro，仅 rw |

### 1.2 不能丢失的四个“设计已变、代码未变”

1. **独立历史身份缓存拟取消。**后续希望由后端记录保存完整冻结身份，减少两处维护。当前代码仍维护 identities 下的 I1，尚未删改。
2. **设备 remove 改为观察/异常核查入口。**正常发布释放和后端回收应在计划停用 LV 前完成。当前第三阶段 remove 只关联历史，并没有 LIO 撤销。
3. **初版仅支持 rw。**当前第二阶段解析器仍接受 ro/rw；拒绝 ro 的新规则尚未补入代码。
4. **真实延期依据需重接。**机制已验证，但不能因存在旧记录就假定 remove 将负责清理；release/remove 重新划界后，需重新确定“明确旧实例收尾依赖”的授权事实。

下一页应先对齐上述差异，完成第四阶段状态与接口的最后冻结，再给出自然语言编排稿，得到落地指令后修订代码。不要重做已经验证的 A/G 和队列机制，也不要把未落地设计说成已有实现。

## 2. 项目目标、最小范围与工作原则

目标是在小内存 OpenWrt 上，按设备事件增量处理选定 LVM LV 的 iSCSI 发布。周边脚本轻量，持有 A 的本机执行器是业务调度和操作中心，有事运行，处理完并交接后退出。

当前优先只打通“设备事件 → 本机执行器 → 受管 LIO”的线路。配置修改协调、手工统一重查、开机全量补齐、远程 Web 调度、远程状态切换属于后续版本，当前不借助隐藏扫描或文件监视器提前实现。

### 2.1 用户已经确认的原则

- 不靠“尽量恢复成功”掩盖原始错误，不吞错误换表面成功；升级依据来自真实失败日志。
- 除非恢复本身已经是明确授权业务，初版不自动恢复、补写状态、接管未知对象或循环等现场变好。
- 不相符时记录实际值并停止依赖动作；无法确认事实或一致性，也返回错误。
- 现场读取要区分“核对已有预期”与“获取新事实”；前者不自动授予发现、替换和扩大对象集合的权限。
- 名称表由用户独立维护；策略是理论期望，名称表表达现实资源准入，两者不要求完全一致。
- 未来远程调度只提交请求；本机执行权始终由 A 决定，远程不得直接绕过执行器修改 LIO。
- 当前状态文件、配置意图、历史日志分开；不把状态表做成永久历史账本。
- 空闲 target、TPG、portal 保留，不自动收缩外壳。
- 初版客户端无认证接入，仅支持 ACCESS=rw；ro 以后通过实验再升级。

### 2.2 编码和交付约定

目标 shell 为 BusyBox ash，脚本用 /bin/sh；命令清单式、无 shell 函数、中文提示、root 运行，单脚本上限已放宽到 600 行。未获明确指示不使用 Python。

先细化自然语言编排稿，再落地完整代码。远程执行使用 heredoc，不使用 ssh "cmd"。本文件没有修改已安装脚本和配置。

## 3. 平台与实机证据基线

| 项目 | 实机基线 |
| --- | --- |
| OpenWrt | 25.12.0，r32713-f919e7899d，kirkwood/generic |
| Linux | 6.12.71，armv5tel |
| PID 1 | /sbin/procd |
| shell | /bin/sh、/bin/ash 指向 BusyBox；帮助输出确认 1.37.0 |
| 锁工具 | /usr/bin/util-linux-flock，util-linux 2.41.5 |
| flock 通用入口 | 安装后 /usr/bin/flock 指向 /usr/bin/util-linux-flock |
| 计时 | /proc/uptime 可读；sleep 整数秒可用；当前不依赖 date 纳秒格式 |
| 日志 | /tmp/lio-targetd.log，root:600，追加写 |
| 运行目录 | /tmp/lio-targetd，root:700 |
| 配置目录 | /etc/lio-targetd |

原 BusyBox flock 的“锁忙”和“操作错误”均返回 1，不能满足请求脚本的分支区分要求。无需立刻重新编译：本机已通过 apk 安装独立 flock 包，安装输出为 flock 2.41.5-r1，并形成 util-linux 专用入口。

实机验证结果：竞争忙用 -E 200 返回 200；无效 fd 返回 65；关闭最后一个持锁描述符后可重新取得锁；持锁 shell 经 exec 进入另一个 shell，PID 和 fd 不变，锁继续存在，接收者关闭 fd 后释放。

代码固定调用 util-linux 专用路径，避免依赖 alternatives 恰好指向哪一个实现。不要混入 --fcntl；本项目使用现已验证的 flock 锁语义。

## 4. 文件清单与责任边界

| 文件 | 当前责任 | 禁止顺手加入的责任 |
| --- | --- | --- |
| /etc/hotplug.d/block/80-lio-targetd | DM 筛选、事件现场快照、异步启动请求、准备日志出口 | 不读名称表/策略/身份缓存，不碰 Q/R，不等锁，不操作 LVM/LIO |
| /usr/libexec/iscsi-luns-queue-update | 校验 11 个参数，G 内提交 Q/R，竞争 A，exec 交接 | 不解析业务配置、不采集身份、不写身份/发布记录、不操作 LVM/LIO |
| /usr/libexec/lio-targetd | 当前完整前三阶段；未来第四阶段唯一受管写入者 | 不由普通现场读回自动修正任务身份，不偷偷实现远程配置协调 |
| /etc/lio-targetd/settings.conf | 固定滤波参数 | 不存运行时队列/锁 |
| /etc/lio-targetd/resources.names | 用户独立维护的准确 DM_NAME 名单 | 不由事件脚本自动生成或同步策略 |
| /etc/lio-targetd/<NAME>.target | target/TPG/portal 参数和属性意图 | 不是实际发布状态 |
| /etc/lio-targetd/<NAME>.luns | LUN_ID、DM_NAME、ACCESS、ADMIN_STATE | 不是设备发现结果或所有权凭证 |
| /tmp/lio-targetd/identities/* | 当前第三阶段 I1 历史身份 | 尚不代表已发布或归属，拟由后端记录替代 |
| 未来当前发布/共享对象状态文件 | 冻结已经实际执行并确认的当前状态 | 尚未创建；不是已有正式文件契约 |

旧草案中的 /etc/init.d/lio-targetd 统一重查/启动补齐入口没有在本轮实现。名称表删项后的存量发布处理确实需要未来协调流程；普通编辑文件本身不触发当前执行器。

## 5. 事件链与 hotplug 入口

### 5.1 实际分发环境

/proc/sys/kernel/hotplug 为 /sbin/hotplug；PID 1 为 procd。/etc/hotplug.json 按 SUBSYSTEM 分发到 /sbin/hotplug-call。

hotplug-call 设置 HOTPLUG_TYPE、PATH、USER、LOGNAME，设置 DEVICENAME=${DEVPATH##*/}，按脚本名顺序遍历 /etc/hotplug.d/<类别>。每个脚本在独立子 shell 中被 source，因此 80-lio-targetd 使用 return，不是独立执行入口。

已有 block/20-lvm2 仅对 add 执行 vgscan/vgchange，属于系统既有 LVM 激活链。80-lio-targetd 不替代它，也不把热插拔脚本改成长时间运行的业务执行器。

### 5.2 筛选顺序

1. HOTPLUG_TYPE=block。
2. SUBSYSTEM=block。
3. ACTION 只接受 add、change、remove。
4. DEVICENAME（或 DEVPATH 尾部）必须是 dm- 后接十进制数字。
5. 仅相关 DM 事件准备日志出口，检查 DEVPATH 与数字字段。
6. add/change 实例字段齐全时，按 /sys${DEVPATH} 定点采集；remove 不读 sysfs。
7. 启动独立后台请求，立即返回。

没有名称表筛选。因此“未知身份也提交”的范围是已经通过上述 DM 类别和动作筛选的事件，不是把所有系统 hotplug 事件都提交。用户确认把进一步判断下沉到执行器，避免给 hotplug 增加业务负担。

### 5.3 现场快照的可信条件

对 add/change：先读 uevent 的 MAJOR/MINOR/DISKSEQ，与事件值比较；读取 dm/name 和可读 UUID；再读一遍实例字段比较。采样两端仍吻合才提交 source=sysfs，携带名称和可取得的 UUID。

实例不吻合、设备已消失或不能得到可信快照时，不改成当前新实例；提交 source=none、名称/UUID 为 -。非法数字、协议控制字符、DEVPATH 矛盾等按入口错误处理。UUID 读取问题会留日志；UUID 未取得时可为 -，不能伪装为已核对。

数字字段缺失用 - 传递并记录缺失，不在 hotplug 中扫描补齐。remove 即使没有 DM_NAME、DM_UUID，也携带事件自身的旧实例字段提交。

### 5.4 后台编排与日志出口

hotplug 打开 /tmp/lio-targetd.log 为追加 fd 7；已有路径必须是非符号链接普通文件且 root:600。子请求 stderr 接 fd 7，随后关闭 7、8、9，stdin/stdout 接 /dev/null。

后台启动返回 0 只表示请求进程已启动，不表示已入队，也不表示 LIO 已完成。请求和执行器的错误由继承的 stderr 进入日志。日志准备失败才用 logger 作诊断出口，logger 也失败时写原 stderr，入口仍返回错误。

## 6. hotplug → 请求脚本：参数及队列协议

### 6.1 恰好 11 个参数

```text
V1 ACTION SEQNUM KERNEL MAJOR MINOR DISKSEQ DM_NAME DM_UUID SOURCE DEVPATH
```

| 序号 | 字段 | 当前约束 |
| --- | --- | --- |
| 1 | VERSION | V1 |
| 2 | ACTION | add / change / remove |
| 3 | SEQNUM | 十进制数字或 -；不用于重排或业务去重 |
| 4 | KERNEL | dm-数字 |
| 5 | MAJOR | 十进制数字或 - |
| 6 | MINOR | 十进制数字或 - |
| 7 | DISKSEQ | 十进制数字或 - |
| 8 | DM_NAME | 缺失为 - |
| 9 | DM_UUID | 缺失为 - |
| 10 | SOURCE | sysfs / none |
| 11 | DEVPATH | /devices/ 下路径，尾部与 KERNEL 一致，无 .、.. 分量 |

所有参数非空且无控制字符。none 要求名称与 UUID 均为 -；sysfs 不得用于 remove，且三项实例字段必须齐全。不同入口还会继续做自己承担的合法性检查，不把调用方输入当 shell 代码执行。

### 6.2 Q 初次记录：12 字段

记录以 TAB 分隔，最后增加 END，并以换行结束：

```text
V1<TAB>ACTION<TAB>SEQNUM<TAB>KERNEL<TAB>MAJOR<TAB>MINOR<TAB>DISKSEQ<TAB>DM_NAME<TAB>DM_UUID<TAB>SOURCE<TAB>DEVPATH<TAB>END<LF>
```

请求脚本每次先追加一个换行，再写记录，以隔开之前可能留下的残片。所以真实日志中“总行 4、空行 2、有效事件 2”是这种追加方式的预期表现，不是丢掉了一半事件。

END 和换行用于发现截断；不是事务提交证明，也不能使失败写入自动变完整。前置换行不会把旧坏记录变成合法记录，执行器仍会保留 B 报错。

### 6.3 执行器重入记录：13 字段

仅执行器成功延期时，在 END 前增加 TAIL_COUNT：

```text
V1 … DEVPATH<TAB>TAIL_COUNT<TAB>END<LF>
```

12 字段输入计数默认为 0；13 字段计数仅接受 0、1、2。达到 3 是终止结果，不重新写入队列。

前 11 项原事件字段保持不变，不能把缓存补充后的名称或当前现场值写回去。请求脚本仍只生产 12 字段；hotplug 和请求参数无需为内部重入同步增加字段。

这不是定长对齐文本。视觉列宽不重要，准确 TAB 边界重要。执行器手动拆分字段，不用会折叠连续空白的通用 IFS 拆分掩盖空字段。

## 7. A/G、Q/R/B 与描述符契约

| 符号 | 路径 | 意义 |
| --- | --- | --- |
| G | /tmp/lio-targetd/handoff.lock | 串行化队列提交、批次移交、取得执行权与退出交接 |
| A | /tmp/lio-targetd/active.lock | 本机唯一执行权，覆盖整个执行器活动周期 |
| Q | /tmp/lio-targetd/queue | 后续待处理事件，可不存在 |
| R | /tmp/lio-targetd/request | 零字节普通文件，存在表示需要入口评估 |
| B | /tmp/lio-targetd/batch | 执行器独占的当前批次快照 |

G 用 fd 8，A 用 fd 9。锁文件长期保留，不删除重建；换 inode 会破坏同一把锁的含义。Q/R/B 不是锁文件。

### 7.1 请求进程完整编排

1. 设置 PATH、LC_ALL=C、umask 077，安装失败上下文和信号处理。
2. 关闭外来 fd 8/9，校验 11 参数、root 和 util-linux-flock。
3. 创建/核对 root:700 运行目录，拒绝异常路径类型。
4. 打开 G 为 fd 8，阻塞取得 G。
5. G 内检查 Q/R 类型，追加完整事件到 Q，再创建/清空零字节 R。
6. 打开 A 为 fd 9，以 -x -n -E 200 竞争。
7. rc=200：已有执行器，本请求 Q/R 已提交，关闭描述符后返回 0。
8. rc 为其他非零：工具/操作错误；保留已形成的 Q/R，报错退出，不冒充锁忙。
9. rc=0：已取得 A，先关闭 fd 8 释放 G，再检查执行器文件，exec /bin/sh 执行器 V1 9。
10. exec 保留同 PID、fd 9 和锁；不再次用参数传事件，事件已在 Q。

执行器不存在或 exec 失败也可能发生在 Q/R 已提交之后；失败日志记录提交状态，不能宣称请求已由另一进程接管。

### 7.2 描述符继承纪律

等待 G 的 flock 子命令需要 fd 8，但关闭 fd 9；普通外部业务命令关闭 fd 8/9，防止子进程继续持锁。调用者关闭自己最后一个持锁描述符后，应能释放锁，不被仍运行的业务子进程拖住。

独立竞争实验必须先关闭继承的持锁描述符，再独立打开同一个锁文件，否则不能证明不同打开文件描述之间的真实竞争。

## 8. 第一阶段：调度骨架的完整分支

### 8.1 入口

唯一内部调用形式为：exec /bin/sh /usr/libexec/lio-targetd V1 9。执行器要求 root、运行目录 root:700、既有 A/G、继承 fd 9 对应 A 的同一设备号/inode。

该 fd 身份检查只能排除显然传错文件，不能自行证明已经持锁；持锁事实来自请求脚本先取得 A 再 exec 的内部契约。不能直接运行执行器然后把打开了 A 当成已持有 A。

settings.conf 在接管周期入口读取，失败时 Q/R/B 不动。固定 PATH、LC_ALL=C、umask 077；描述符 6 用于批次/全局配置读取，业务配置读取使用 fd 5。

### 8.2 G 内的入口裁决

先检查遗留 B：任何已经存在的 B，包括空文件或异常对象，均停止，不覆盖、不重放。然后检查 Q/R 为非链接普通文件，R 必须零字节。

| Q | R | 处理 |
| --- | --- | --- |
| 字节非空 | 存在 | mv Q→B，成功后删除本轮 R，释放 G 后处理 B |
| 字节非空 | 不存在 | mv Q→B，不补写 R；记录异常但仍处理本批，周期最终非零 |
| 不存在或零字节 | 存在 | 删除 R，不创建 B；记录空轮，固定滤波 |
| 不存在或零字节 | 不存在 | 在 G 内作唯一正常退出决定 |

G 内不逐行计数、不解析配置、不读 sysfs、不做 LIO 业务。只含空行的 Q 仍字节非空，会交给 G 外校验。

mv 成功前不能删除 R；mv 后 R 删除失败就保留实际 B/R，不把 B 移回 Q。新的请求重新打开当前 Q，不会继续写旧 B。

### 8.3 B 两遍处理

第一遍 validate：完整检查所有非空行，字段数、END、换行、字段控制字符、动作、数字、设备名、来源一致性和 DEVPATH。读取字节数与 stat 大小核对，用于发现 NUL、读错误或外部变化。

任何坏记录使整批不进入逐条业务处理；保留 B，报错退出。可继续检查的坏行尽量一次记录多处问题。真正空行忽略，仅含空格/TAB 的行不自动当空行。

第二遍 log：重新读 B，按文件实际行序处理并记录事件。没有按 SEQNUM 排序、去重或动作折叠。第二、三阶段实际挂在这一遍中，名字 log 不代表只打印一句文本。

第一遍结束后，若尚未加载本周期配置，就加载一次，然后才进入第二遍。

### 8.4 固定滤波

完成 B 的独立处理、摘要和删除后，或消费一次只有 R 的空轮后，G 已释放，A 仍持有。打印开始时间，sleep 整数秒，打印结束时间，再回 G 内裁决。

新事件可在此期间提交 Q/R，不重置计时；当前不是“最后一次事件之后再等待”的滑动防抖。sleep 失败保留错误退出，不自动重新计时。

### 8.5 唯一正常退出交接

确认 B 不存在、Q 不存在或空、R 不存在后，仍持 G，先关闭 A，再关闭 G；之后不再读写 Q/R/B 或继续做业务。最后输出退出摘要。

新请求若先拿到 G，旧执行器随后会看到 Q/R；旧执行器若先决定退出，新请求等到 G 释放后可以竞争已经释放的 A。这个顺序消除正常退出窗口中的责任遗漏。

日志在锁释放后写出，故新旧周期的日志行可能交错，不能以打印先后替代锁协议或事件历史先后。

## 9. 第二阶段：配置格式、周期视图和判断

### 9.1 配置读取频率

settings.conf：每个执行器活动周期入口读取一次。

resources.names、使用到的 .target/.luns：第一个校验通过的 B 后读取一次；本进程后续所有批次复用，包括无效配置状态。失败也不自动重读。下一位执行器是新进程，重新读取当时文件。

活动周期不是单个 B：一个 PID 可持 A 连续处理多个 B 和多个固定滤波。

代码细节：当前读取条件是“校验完成且 lio_cfg_loaded=0”，没有再要求有效事件数大于 0；因此只有空行但字节非空的 B 也会触发配置加载。仅有 R、没有 B 的空轮不触发业务配置加载。这里以实际代码为准，不悄悄改写实现。

周期内存视图不等于多文件原子快照。G 不负责配置编辑协调；当前实验应先完成编辑，再触发事件。持续有事件而执行器不退出，就持续复用原视图。

### 9.2 settings.conf

```text
DEBOUNCE_MS=2000
```

当前仅接受这一键：无引号、无前导零的正十进制整数，必须是 1000 的整数倍，2000 对应 sleep 2。1500 明确报不支持，不取整、不回退默认值。重复、缺失、未知键、读失败都报错。

### 9.3 名称表

每有效行一个准确 DM_NAME；支持空白行和整行注释，不支持通配、行尾注释或 shell 展开。拒绝重复、空白/控制字符、斜杠和引号。空名单合法，文件缺失/不可读/非法不是空名单。

名单和配置不是相互生成关系：删除名单项目是取消现实资源准入的表达，但当前事件线路没有自动提交统一重查，也不能仅凭这次编辑就宣称旧发布已撤销。

### 9.4 配置组发现

由 .luns 枚举配置组 NAME，必须有同名 .target。NAME 接受字母、数字、下划线、点、横线，拒绝空、单独 . 或 ..。只有 .target 没有 .luns 时，记录未使用且不解析它。

没有 .luns 可以是合法空策略集合；枚举失败或已有文件损坏不能解释为正常空集合。

### 9.5 .target 当前九项必填键

| 键 | 规则 |
| --- | --- |
| IQN_PREFIX | 非空，iqn. 开头，无空白、控制字符、斜杠和引号 |
| TPG | 当前只支持 1 |
| PORTAL | 一个 IPv4:端口；各段 0～255，端口 1～65535 |
| ATTR_generate_node_acls | 0/1 |
| ATTR_cache_dynamic_acls | 0/1 |
| ATTR_authentication | 0/1 |
| ATTR_demo_mode_discovery | 0/1 |
| ATTR_demo_mode_write_protect | 0/1 |
| ATTR_tpg_enabled_sendtargets | 0/1 |

完整 IQN=IQN_PREFIX:NAME。有效行严格无引号 KEY=value；不 source/eval。未知键、重复键、缺键和非法值使整组无效。这里只是当前格式校验，不宣称完整 IQN 标准、内核属性存在和认证部署已经验证。

初版后续需求固定无认证和 rw；上述通用字段的当前解析接受范围不等于初版已经承诺实现全部组合。prod_mode_write_protect 是实机读过的属性，不在当前九键解析器内，不能直接加入配置而不修订解析器。

### 9.6 .luns 四列与当前内存表

```text
LUN_ID DM_NAME ACCESS ADMIN_STATE
```

恰好四列；LUN_ID 为 0 或无前导零正整数，同组唯一；ACCESS 当前代码接受 ro/rw；ADMIN_STATE 为 on/off。坏行使整组无效，不接纳剩余部分。可一设备多条策略，跨 target 的相同 LUN_ID 合法。

周期内有效策略行是 TAB 分隔的 11 项内部表：

```text
DM_NAME GROUP IQN TPG LUN_ID ACCESS ADMIN_STATE PORTAL ATTRS CONFIG_FILE CONFIG_LINE
```

这是执行器内存结构，不是新的外部队列协议。名称表用带换行边界的精确名称集合保存，避免子串误命中。

### 9.7 判断顺序与配置不完整

原始事件先记日志；第三阶段接入后先按事件/精确缓存取得关联身份，再作配置判断：

1. 名单无效：配置失败。
2. 身份不是 known：身份未确认。
3. 准确名称不在有效名单：未准入，不新建。
4. 已准入：遍历所有有效组的准确命中条目，逐 LUN 输出。
5. 整体配置不完整：总体结果为配置判断不完整；有效组结果仍可列出，不能把零命中说成没有策略。
6. 整体完整且零命中：无策略。
7. 整体完整且有命中：已匹配策略；on 表示待核对候选，off 只是期望未发布。

无效组不妨碍记录其他组已知结果，但整个周期保持失败标记。有效名单已经足以排除未准入名称；无效配置组的错误另行保留，不会因此被清掉。

remove 命中 on 策略时记录“历史策略匹配，remove 仅关联、不发布”，不增加 on 候选数。未准入、无策略、off 均不自动授权删除旧对象。

## 10. 当前四份最小实验配置

### 10.1 settings.conf

```text
DEBOUNCE_MS=2000
```

### 10.2 resources.names

```text
# 用户独立维护的资源名称表，每行一个准确 DM_NAME。
vg2-BULKTEST00
vg2-BULKTEST01
vg2-BULKTEST03
vg2-LIODEBUG_MISSING
```

### 10.3 experiment.runtime.target

```text
IQN_PREFIX=iqn.2026-10.local.pogov4
TPG=1
PORTAL=127.0.0.1:3260
ATTR_generate_node_acls=1
ATTR_cache_dynamic_acls=1
ATTR_authentication=0
ATTR_demo_mode_discovery=1
ATTR_demo_mode_write_protect=0
ATTR_tpg_enabled_sendtargets=1
```

### 10.4 experiment.runtime.luns

```text
# LUN_ID  DM_NAME               ACCESS  ADMIN_STATE
0         vg2-BULKTEST00        rw      on
1         vg2-BULKTEST01        rw      off
2         vg2-BULKTEST02        rw      on
3         vg2-LIODEBUG_MISSING  rw      on
```

00：名单准入、on 候选；01：名单准入、off；02：有策略但未准入；03：准入但无策略；MISSING：有配置但无真实事件，当前不主动发现设备。SWAP、USBROOT 等 DM 不准入。

正常解析摘要应为名称 4、有效组 1、策略 4、完整视图。实验 target 的 loopback portal 与手工网络实验的 172.16.16.220:32690 是两个不同配置，不能混淆。

## 11. 第三阶段：当前身份缓存与受限现场核对

### 11.1 实例键与身份来源

实例键为 MAJOR:MINOR/DISKSEQ，例如 253:2/28 与 253:2/29。dm-N、DEVPATH 用于交叉核对；DM_NAME 和 DM_UUID 不代替实例键。

索引数字按十进制字符串去除前导零，DISKSEQ 不转为目标 shell 整数，不按大小比较替换旧身份。缺一项就不能构成精确键，不用现场补值。

授权来源：可信事件快照、同完整实例键的合法 I1 缓存。现场采集仅用于比较，不向身份或业务预期提供新值。

### 11.2 I1 缓存精确格式

目录 /tmp/lio-targetd/identities，root:700；文件 root:600，例如 253_2_28。单行、TAB 分隔、12 字段、末尾换行：

| 序号 | 字段 | 值或约束 |
| --- | --- | --- |
| 1 | VERSION | I1 |
| 2 | KERNEL | dm-数字，交叉核对事件 |
| 3～5 | MAJOR、MINOR、DISKSEQ | 规范化实例键 |
| 6 | DM_NAME | 历史名称 |
| 7 | DM_UUID | 历史 UUID 或 - |
| 8 | DEVPATH | 与事件一致 |
| 9 | PROOF_SOURCE | event_sysfs |
| 10 | PROOF_SEQNUM | 首次建立时的序号，可为 - |
| 11 | PURPOSE | identity |
| 12 | END | END |

终端列宽可能把 631 与 identity 看似粘连；已有实测字段数确认是 12，不能只凭视觉判断缺 TAB。

读取检查文件/目录类型、属主权限、字节数、单行换行、字段数、版本、来源、用途、键、路径及已知身份。坏记录、权限错或链接不是 miss，不自动改权限、删除或重建。

### 11.3 身份关联分支

| 事件/缓存 | 当前结果 |
| --- | --- |
| sysfs 且完整键和有效名称 | 以事件快照为主身份，关联来源 event_sysfs |
| 同键 hit，名称/双方有效 UUID 一致 | 复用；事件 UUID 缺失可由精确缓存提供已有预期 |
| 同键名称不同或双方有效 UUID 冲突 | error，保留旧缓存，停止依赖动作 |
| none、完整键、精确缓存 hit | known，关联来源 cache，原始事件来源仍 none |
| none 且 miss，或键不完整 | unknown，不补采，不建立新缓存，周期失败 |
| 缓存读取/结构/权限失败 | error，不转 miss，不另找现场身份 |

原始事件日志不被关联结果覆盖。后面业务判断可能携带关联后的名称但保留原始 SOURCE；应联合“身份关联”日志理解来源。

### 11.4 建立、复用、保留和跳过

仅当身份 known、名单准入、配置完整、至少一个 on 命中、非 remove，才进入历史保存与后续现场核对。

新建必须由完整 sysfs 事件快照提供数据。先写同目录唯一临时文件，检查内容、大小、权限，再 rename；不覆盖已有记录。失败保留实际临时文件及旧状态。

已有有效同键记录复用，不改证明序号、内容、inode 或修改时间。remove 仅保留。未准入、无策略、off、配置不完整，不单独建新缓存，也不删除旧缓存。

历史保存先于当前现场核对：排队期间设备已离场时，可以保存原事件历史，然后明确报现场不符。后续核对失败不回滚历史快照，缓存建立也不意味着可发布。

### 11.5 受限现场核对顺序

只对通过上述条件且缓存建立/复用成功的 on 候选执行：

1. 冻结实例、名称、已知 UUID、DEVPATH 及 /dev/mapper/<预期名称>。
2. before：定点读取 uevent，核对 MAJOR/MINOR/DISKSEQ；重复、缺失或非法字段为无法确认。
3. 核对名称；有预期 UUID 才核对 UUID，没有则明确跳过。
4. 核对 mapper 路径为块设备，stat 的十六进制 rdev 转为设备号后比较。
5. after：再次核对实例、名称和已有 UUID，检测采样期间变化。
6. 全部必选项通过才记现场通过；任何不符或无法确认，记录检查项、预期、实际、rc，停止本条后续依赖。

这是前后采样，不是原子设备绑定，不证明无人使用、本地挂载完备性或 iblock 全部条件。一事件多 on 条目共用这次核对，失败阻断所有依赖它的条目。

### 11.6 当前 remove 的精确行为

按旧完整实例键读缓存，在名称表/配置判断之前完成历史关联；不读取当前 sysfs 找旧名称。命中后现场核对跳过、LIO 未执行、缓存删除未执行；重复 remove 独立关联同一记录。

当前 miss/键缺失仍累计失败。以后“无受管后端关联可正常过滤”的新口径尚未实现，不得误把现有日志中的 unknown 当成新版行为。

当前代码没有缓存正式删除接口调用；早期“所有发布收尾后删除缓存”只是旧设计，后续拟取消独立缓存，不能直接沿用为必须实现的新增业务。

## 12. 第三阶段延期：机制、字段和精确轨迹

### 12.1 允许延期的业务含义

原讨论例子：新实例 add/change 已到，但对应发布位置仍由旧实例占用，旧实例释放尚未确认。不能把旧实例替换成新实例而跳过退出交代。

延期是明确依赖未满足时的成功移交，不是核对失败后随便重试。现场不符、记录损坏、工具失败均不能转成延期。

当前源码固定：lio_defer=0、lio_wait_old=-、lio_wait_slot=-；没有生产模拟开关。只有独立执行器副本通过源码注入实验判据验证分支。

### 12.2 队尾计数的最终约定

- “队尾”是本 B 最后一个有效记录，基于 validate 得到的有效总数判定，忽略空行。
- 不检查 Q 是否为空；Q 中新任务不阻止本 B 队尾计数。
- 非队尾：保留本行原计数，不加、不清零。
- 队尾：计数加 1；小于 3 才允许重入；第 3 次记延期失败，不再提交。
- 每行独立计数，不把 add/change 合并成一个业务计数器。

如果某行总有其他记录在它后面，允许持续不计数；机制不承诺持续无限输入下的时间上界。已经确认的有限批次模式中，最后一条先终止，前面的随后成为队尾并最终退出。

### 12.3 成功移交的落点

持 A，短暂取得 G；核对 Q/R；把原前 11 项加计数，完整追加到 Q 尾部，再设置零字节 R；完成后释放 G，才记成功移交。

不在 B 中原位删行。该行处理结果记为成功移交，当前 B 正常收尾后删除；下一轮消费 Q 中新记录。即“逻辑上消掉本行并移到队尾”，不是处理过程中修改正在遍历的 B。

追加或标记失败时保留 B 与实际 Q/R，故障退出；可能留下不完整提交，不自动去重、补齐或重放。

### 12.4 实机独立副本实验

实验目录 /tmp/lio-defer-test.BOCkIG。副本使用独立配置和运行目录，真实 sysfs 保持实例 29；没有生产 LIO 写入。

| 用例 | 处理次数 | 成功移交 | 延期失败 | 最终 rc |
| --- | --- | --- | --- | --- |
| single-block | 3 | 2 | 1 | 1 |
| two-block | 9 | 7 | 2 | 1 |
| release | 2 | 1 | 0 | 0 |

single-block，SEQNUM 638：

```text
638:1:成功移交
638:2:成功移交
638:3:延期失败
```

two-block，原始 add 637、change 638：

```text
637:0:成功移交  638:1:成功移交
637:0:成功移交  638:2:成功移交
637:0:成功移交  638:3:延期失败
637:1:成功移交
637:2:成功移交
637:3:延期失败
```

release：第一轮 638:1 成功移交；第二轮模拟依赖解除，638:1 移交=无，正常结束。计数保留为 1，不影响已经解除依赖的成功结果。

每次落点核对还检查原字段一致=1、字段数=13、R 零字节=1；正式身份缓存和真实实例未改。

## 13. 故障分类、返回状态与现场保留

| 类别 | 当前处理 | B 是否正常消费 |
| --- | --- | --- |
| 未准入、完整配置无策略、off | 正常资格/期望判断 | 完成本阶段日志后消费 |
| 配置无效/不完整 | 保留错误，周期非零，不自动重读 | 完成独立尝试后消费 |
| 身份未知、缓存失败/冲突、现场不符/无法确认 | 停止本条依赖动作，周期非零，其他独立事件可继续 | 正常尝试结束后消费 |
| 第三次队尾延期 | 记失败，不再重入，周期非零 | 完成本条和摘要后消费 |
| B 格式、读取、日志或内部一致性失败 | 立即停止后续依赖，故障退出 | 保留 B |
| 遗留 B | 不覆盖、不重放 | 原样保留 |
| Q→B 失败 | 不消费 R，不反向恢复 | 保留实际现场 |
| R 删除失败 | 保留已形成 B 和实际 R | 不继续成功路径 |
| B 删除失败 | 报错，不假定已清理 | 保留实际状态 |
| sleep/锁/工具失败 | 原始 rc、stderr 和阶段上下文 | 不重试 |

业务失败后删除 B，只表示该阶段已经尝试并记录，不代表发布已经收敛。失败结果不会因后面的成功被清零。正常完成退出交接也可以返回 1。

正常退出依赖 G 内 A 后 G 的关闭顺序。故障 trap 只关闭本进程描述符，记录阶段/行/路径，不重新入锁修复 Q/R，不拉起替代执行器。掉电、SIGKILL 或日志出口坏掉时不承诺最后一条日志。

当前使用退出码 0、业务累计失败 1 以及原始工具非零；不要与第四阶段拟议的业务返回码表混用。

## 14. 日志、计数与已解决的观测问题

### 14.1 从 dmesg 改到独立日志

最初通过内核日志观察启动事件，执行器摘要显示 40 事件，但 dmesg 中部分轮次和事件缺失。检查 printk_devkmsg 得 ratelimit，并看到：

```text
printk: sh: 29 output lines suppressed due to ratelimiting
```

这证明当时日志展示不完整，不能把缺行直接判成队列丢失。之后 80-lio-targetd 修订为直接追加 /tmp/lio-targetd.log；当前主证据在该文件，不在 dmesg，也不应只看 logread。

文本中的 <14>、<12>、<11> 为当前日志优先级标识，不表示该文件已经由 syslog 接收。当前没有实现日志轮转或跨重启持久化，/tmp 可随重启清空。

### 14.2 必须联合阅读的日志层次

1. 开始、轮次、validate/log 批次摘要。
2. 周期名称、目标参数、策略、完整性摘要。
3. 原始事件记录。
4. 身份关联：原始来源与关联来源、hit/miss/error。
5. 逐 LUN 判断及事件总体业务判断。
6. 第三阶段结果：缓存动作、现场结果、检查项、预期/实际、rc、UUID、延期和 LIO 未执行。
7. 固定滤波起止及退出交接摘要。

现场通过日志常显示最后完成的 after:uuid，这不是只检查了 UUID；之前的核对顺序见源码。失败时记录首次停止的检查项。

### 14.3 计数边界

事件总体五类互斥：未准入＋身份未确认＋配置失败＋无策略＋已匹配=有效记录数。on/off 是命中条目数，一事件可多命中，不能与事件数混加。

第三阶段按身份 known/unknown/error、缓存动作五类、现场四类分别计数，各维度总数等于有效记录数；失败记录数另计，不能把三维计数相加当事件数。

退出“事件”包含重入处理次数；首次记录按 12 字段计，重入记录按 13 字段计。事件含重入=1 是统计口径标志，不表示本次必然发生重入；真正看重入记录和成功移交计数。

## 15. 前三阶段的实机成果与证据索引

### 15.1 平台和分发实验

- BusyBox 锁语义：独立竞争、显式解锁、最后 fd 关闭、exec 继承、阻塞等待释放唤醒通过；同时确认 BusyBox 忙/错误返回码无法区分。
- util-linux 实验：忙=200、无效 fd=65、关闭释放、exec 保持 PID 与锁通过。
- 主进程持 G 调用 hotplug-call：分发约 37689.49→37689.54 秒已返回，请求仍等待 G；释放后请求继续。
- 业务子进程 fd 8/9 已关闭；主进程释放 G/A 后独立竞争成功，仍运行的子进程未拖住锁。
- 真实 uevent：主进程持 G 连续提交 change，SEQNUM 629、630 均到达等待状态；第一条后台等待未堵塞第二条真实分发链。

### 15.2 真实 DM 事件身份

早期停用/激活实验：旧实例 253:2/10，remove 623、625；remove 中没有 DM_NAME/DM_UUID，sysfs 已消失。重新激活 add 627、change 628，实例变为 253:2/28；名称与 UUID 相同。

日志展示顺序曾 change 在 add 前，但 SEQNUM 和 uptime 表明应区分事件历史与采集/打印顺序。系统队列不能承诺业务按生成序执行；也不能靠排序省略旧生命周期。

### 15.3 配置故障与恢复

向 experiment.runtime.target 注入 LIO_TEST_UNKNOWN=1；执行器 PID 3093 报未知键、行 11、有效组 0/无效组 1、视图不完整，业务配置失败，rc=1。

从保留备份恢复原配置，再触发 change；新执行器 PID 3129 重新读到完整组，匹配 on 候选，异常 0、rc=0。这证明新周期读取恢复配置；不能据此说同一活动周期自动刷新。

备份目录 /tmp/lio-targetd-config-test 在当时恢复后保留；后续重启可能清除。

### 15.4 身份生命周期真实实验

| 步骤 | 关键实机结果 |
| --- | --- |
| change 631，实例 28 | PID 3345，miss→建立 253_2_28，现场通过，rc=0 |
| change 632，实例 28 | PID 3430，hit→复用，内容、inode、权限、大小、mtime 不变，证明序号仍 631 |
| 停用实例 28 | PID 3505，remove 633/635 均精确 hit，关联来源 cache，现场跳过，缓存原样保留，rc=0 |
| 重新激活实例 29 | PID 3594，add 637 建立 253_2_29，change 638 复用；旧 28 不变，UUID 相同，rc=0 |
| 设备占用实验后激活实例 30 | PID 4599，两轮处理 remove 639/641、add 643/change 644，缓存 30 的证明序号 643，异常 0、rc=0 |

### 15.5 独立执行器副本的边界实验

| 实验目录 | 注入及结果 |
| --- | --- |
| /tmp/lio-late-remove.CopDIE | 真实现场为 29，仅重放旧 remove 633/635；准确关联 28，不读当前现场替代，28/29 正式及副本缓存不变，rc=0 |
| /tmp/lio-boundary-fail.cmFaaB/missing-old | 仅保留 29 缓存，旧 28 remove 为 unknown；不从现场 29 补建 28，rc=1 |
| 同根 stale-change | 重放 change 632，冻结实例 28；before:实例 明确预期 28/实际 29，停止，不替换身份，rc=1 |
| /tmp/lio-defer-test.BOCkIG | 三项延期/解除实验，精确轨迹见第 12 节，均符合实验预期 |

副本只替换运行/配置路径及明确实验钩子。它们验证分支，不是生产状态文件已经存在，更不是实际 LIO 收尾成功。

### 15.6 本地辅助检查与未覆盖范围

已有 Node 辅助脚本覆盖请求接口、调度并发/故障注入、配置和可观测性、假 sysfs/节点的身份分支等；文件名包括 check-request-interface.js、check-executor-interface.js、check-policy-interface.js、check-identity-interface.js。

本地 dash/fixture 检查不能替代目标 ash 和真实设备；原第三阶段稿中提出的每项缓存破坏、所有 I/O 故障，并不都已有本机实测输出。本文只把上表明确实测项目记为实机成功。

本次文档整理只对当前三个源码执行 sh -n 和内容完整性核对，没有重跑整个远端实验，也未更改线上对象。

## 16. 第四阶段：当前发布状态表与历史身份的区别

### 16.1 为什么原先需要历史身份

remove 经常只剩旧实例键，sysfs 已消失，缺 DM_NAME/DM_UUID。I1 把此前可信事件的身份留到后续 remove，解决历史关联；它不证明曾发布，也不证明任何 configfs 对象归属。

发布状态则回答“当前哪个发布位置由哪次实例生命周期占用，预期和对象引用是什么”；后端状态回答“本程序哪个后端对象仍承担哪个实例的设备占用”。这两者是第四阶段必须有的现实责任记录。

### 16.2 从生命周期记录收缩为当前状态

最初考虑一条记录代表某实例在某位置的一次发布生命周期，讨论过 PREPARING、ACTIVE、CLOSED。最终改成当前发布状态表：

- 以发布位置作唯一索引，一位置至多一条当前记录。
- 不保留 CLOSED 历史条目，不为读当前状态遍历全部历史生命周期。
- 发布成功并确认后登记；释放确认完成后清除；更早生命周期由日志交代。
- 不要求 PREPARING，不能把有意图但未完成的记录当当前已发布事实。
- ACTIVE 若仅重复“当前条目存在”的含义，不必作为必须字段；具体编码尚未落地。

没有状态记录不等于 configfs 一定没有对象：操作部分成功、登记失败、外部对象均可能形成差异，必须报错保留，不自动补登记接管。

### 16.3 新方向：由后端记录承接身份

后端记录必须保存完整冻结身份，否则无法证明其设备归属。既然已有这一份受管身份，再保留“所有历史事件”的独立 I1 会增加一致性负担。

用户提出并已记录：拟取消独立历史身份存储，由后端记录承接受管身份，后端确认释放后清除，早期历史由日志交代。不是再增加一套 H/B 对账来维持重复存储。

在后端记录落地前，不应先删现有 I1 导致第三阶段失去依据；应先重新定义身份来源和 remove 查询接口，再修订实现和对应测试。

## 17. 第四阶段：发布位置唯一键、冻结预期与引用

### 17.1 唯一位置与生命周期区分

逻辑唯一键为完整 target IQN＋TPG＋LUN_ID；当前只支持 TPG=1，但不能让内核 DM 编号成为发布位置索引。

同一位置经历旧实例释放、新实例发布时，可以有生命周期标识 PUB_ID，区分旧清除请求和新当前记录。它不是把位置索引改成历史账本，也不能只靠路径字符串判断“还是同一个对象”。

### 17.2 UCI 命名 section 转义约定

已讨论采用命名 section；原始完整 IQN 保留为字段，索引用可逆的标点转义，避免整串十六进制不直观：

| 原字符 | 编码 |
| --- | --- |
| . | _d |
| : | _c |
| - | _h |
| _ | _u |

下划线自身也转义，是为了避免原文中的 _d 和由点编码得到的 _d 冲突。应逐字符编码或先处理原文下划线；不能把刚生成的转义前缀再次转义。

后接位置后缀 _t<TPG>_l<LUN_ID>。例如：

```text
原位置：iqn.2026-10.local.pogov4:experiment.runtime / TPG 1 / LUN 0
索引：iqn_d2026_h10_dlocal_dpogov4_cexperiment_druntime_t1_l0
```

读取后仍需将原始 IQN、TPG、LUN_ID 与 section 索引交叉核对，索引不是唯一可信内容。其他 IQN 字符的完整允许范围、解码器和错误分支尚待最终实现规则，不能任意丢字符或替换成同一符号。

### 17.3 最小信息要求（概念字段，不是已存在的 UCI schema）

| 信息组 | 必须能表达的事实 |
| --- | --- |
| 发布位置 | 原始 IQN、TPG、LUN_ID，交叉核对命名索引 |
| 发布生命周期 | PUB_ID 或等价生成标识，防止旧请求清除新条目 |
| 设备冻结身份 | MAJOR/MINOR/DISKSEQ、KERNEL、DEVPATH、DM_NAME、已知 DM_UUID |
| 访问冻结预期 | 初版 ACCESS=rw，不能从当前修改后的配置替换旧预期 |
| 后端引用 | 后端逻辑 ID/生命周期 ID 与准确 configfs 对象引用 |
| 上层共享引用 | 所属受管 target、TPG 及必要的生成/归属标识 |
| 发布对象引用 | 对应 LUN 和链接的准确受管引用；无法可靠推导者必须保存 |
| 日志关联 | 支持把建立、核对、释放与输入事件对应起来的标识 |

能够从已冻结字段无歧义推导的路径无需重复存储；重复字段必须定义一致性检查。仅保存 IQN 和 LUN 编号不足以证明当前对象属于哪次生命周期，inode 也不能单独承担跨重启归属。

## 18. 第四阶段：共享对象状态结构与管理边界

讨论中的共享对象至少包括 target、TPG 和后端；当前发布记录引用它们。四类逻辑条目与“物理上分成几份 UCI 文件”是不同问题，后者尚未冻结。

| 条目类型（逻辑） | 需要保存的责任 | 主要共享关系 |
| --- | --- | --- |
| target | 受管 IQN、准确对象归属与生命周期信息 | 多 TPG 或多发布位置可依赖 |
| TPG | target 引用、TPG 编号、冻结共享参数与 portal 范围 | 同 TPG 多 LUN 共用属性和网络入口 |
| backend | 后端逻辑/生成 ID、configfs 引用、完整冻结设备身份、绑定预期 | 一个后端可被多个发布位置引用 |
| publication | 第 17 节的唯一发布位置、生命周期、冻结预期和对象引用 | 引用共享 TPG 与 backend |

这是当前设计信息需求，不伪造已经完成的 section 类型名称、文件路径或字段拼写。

### 18.1 单写者与引用关系

状态修改只由持 A 的本机执行器完成；周边脚本不写，远程管理不直接写。未来管理层编辑配置/提交请求，不能绕过执行器接管事实。

后端先成功建立、核对并可信登记，才能作为后续发布的受管依赖。状态读写失败、对象归属冲突、现场不符、已知操作失败或无法确认均返回错误。

引用数优先由当前发布条目集合推导，不额外维护一份可失真的独立 refcount。集合枚举失败不是“零引用”。路径相同但归属/生成不一致是冲突，不复用。

### 18.2 单发布与共享对象的动作范围

释放单个发布位置，只处理其准确 LUN 引用和发布状态；不改变别的 LUN，不为一个条目任意修改共享 TPG 属性。

后端设备释放是独立动作：确认没有其他受管发布引用，核查实际引用及归属，成功删除准确后端并确认相应占用释放后，才能清除后端状态。失败时保留后端责任记录，即使单发布状态已经清除。

空 target/TPG/portal 保留，不自动递归收缩。共享参数存在冲突时不“顺便改到期望”，而是明确报错。

### 18.3 UCI 的位置

讨论倾向用 OpenWrt UCI 读写命名的当前状态条目，不把已有 .target/.luns/名称表自动改成 UCI 配置。

UCI 不是并发和跨 configfs 事务协议，A 仍决定执行权。状态文件放在哪个运行时目录、包名、私有 delta/staging、写后读回、commit 失败分支尚需冻结并实机验证。不能仅凭 uci batch 返回就承诺整组命令原子成功，不能假设 commit 把 configfs 和文件同时提交。

## 19. 第四阶段：五项逻辑接口及异常规则

以下按业务含义保留五项接口。接口名是本文便于接续讨论的逻辑称谓，不是已经存在的子命令、shell 函数或已发布调用 ABI。

| 逻辑接口 | 输入要点 | 正常输出/作用 |
| --- | --- | --- |
| 按发布位置查询 | IQN、TPG、LUN_ID | 一条完整当前记录，或经过成功查询确认不存在 |
| 按实例查询关联 | 完整实例键，必要的冻结交叉字段 | 当前该实例的受管关联集合；新方向以独立后端记录为身份入口 |
| 登记当前状态 | 准确位置/对象标识、冻结预期、已完成的实际核对结果 | 建立经确认的当前条目，不接管未知对象、不覆盖冲突条目 |
| 确认并清除当前状态 | 预期生命周期/归属、准确对象及已完成的释放事实 | 只清除目标条目，不删除后来占用同位置的新生命周期 |
| 记录故障 | 原始错误、阶段、事件、位置/对象/实例、预期/实际、停止动作 | 留诊断；不借记录故障取得恢复或状态修改权限 |

共享对象需要等价的读取、登记、清除能力；不是在现有三脚本中凭空出现了另外五个独立可执行文件。

### 19.1 缺失、重复、冲突和失败

| 情况 | 准确结果原则 |
| --- | --- |
| 查询确认无当前条目 | 单独的“确认不存在”；与读取失败区分 |
| 文件/目录无法枚举、读取失败、结构坏 | 错误；不能解释为空集合或不存在 |
| 同一逻辑唯一位置重复记录 | 结构/冲突错误；不取第一条、不自动去重 |
| 相同位置、不同实例或生命周期 | 冲突或明确依赖；只有另行满足延期授权才能移交 |
| 相同对象路径但归属标识不相符 | 对象归属冲突，不覆盖、不接管 |
| 重复登记完全相同预期 | 不能仅因字段相同就直接成功，还要有允许复用及现场一致的依据 |
| 清除时原记录不存在 | 不能无条件推断“已正常释放”；需看调用是否持有对应完成证据及实际对象核对规则 |
| 写入、提交或写后确认失败 | 错误；保留实际状态，不用重建记录掩盖 |
| configfs 部分操作成功后失败 | 记录已完成动作与遗留对象，不自动反向回滚 |

### 19.2 返回码讨论的保留状态

曾讨论用独立码区分：0 成功、10 已确认不存在、64 参数错误、65 结构错误、66 读取失败、67 写入/提交确认失败、68 冲突、69 现场核对不符或无法确认、70 日志失败。

这组数字属于第四阶段候选接口表，尚未写入现有代码，也不作为已冻结 ABI。换页后可以沿用它继续定稿，但必须先明确各接口输出格式和 shell 调用方式；不能为了凑表把当前所有 exit 1、工具 rc=65 或锁忙 200 重新解释成同一种业务码。

## 20. 第四阶段：计划释放、remove 与乱序的最终边界

### 20.1 计划释放

管理意图明确授权释放某发布位置时，执行器依据冻结状态核对准确对象，撤销该位置；按共享引用决定是否回收后端。只有后端确实释放了相应占用，管理流程才适合普通停用 LV。

配置变更怎样触发这个管理意图是未来专门流程，当前不要把未实现的统一重查混进事件入口。

### 20.2 设备 remove

本机实验已证明：无 LUN 后端仍占用 LV，普通 lvchange -an 报 in use、rc=5；删除后端后才能停用。因此设备 remove 不能作为正常释放后端的通用起点。

新设计按完整旧实例查询后端关联：

- 确认没有受管关联：可正常过滤，例如从未发布或已经正常释放后的 remove。
- 有受管后端关联：作为异常观察核查准确后端、记录事实；不自动撤销、重绑或吞故障。
- 查询失败、状态坏、无法确认：报错，不能假装无关联。

这里的过滤基于新后端接口，尚未替换当前 I1 miss 就失败的实现。

### 20.3 同名新实例先到

例如旧实例 10、新实例 28：新 add 先处理时，不能因现场是 28 就把旧状态中的 10 换掉。需要查对应位置/后端的当前冻结记录。

若有明确定义的旧实例收尾任务且尚未完成，可按第 12 节机制成功移交；若只有旧记录存在、收尾是否已经授权或进行未知，应报冲突或故障，不能假定旧 remove 到来会替它清理。

remove 一直不来并不是无限等候的理由。获准延期的队尾第三次仍不满足即报错退出；不使用“Q 还不为空”阻止计数。

## 21. 第四阶段尚待补齐的编排与验证

### 21.1 已对齐的粗编排（未落地，不替代最终自然语言程序稿）

在沿用前三阶段调度的前提下，每条获准候选的大体处理关系为：

1. 保存原始事件，使用授权身份来源建立冻结预期，读取本周期配置意图。
2. 按发布位置查询当前状态，分别处理“已确认不存在”“同一生命周期可核对复用”“不同实例/生命周期冲突”“读取失败”。
3. 对已有关联，只按冻结引用核对其实际对象；不能用当前配置重新计算出另一批对象后当作旧对象。
4. 对需要新建的 rw 发布，核对实例及准确节点；确认目标对象不是未知所有者留下的冲突对象。
5. 准备或核对受管 target/TPG/portal 共享依赖，以及准确后端。共享参数不符就报错，不为一个 LUN 擅自改变共享策略。
6. 后端成功建立、绑定、核对后，先完成其可信登记，再作为发布依赖使用；登记失败不继续把它当已受管后端。
7. 建立准确 LUN 和后端链接，逐项核对真实关系；完成事实成立后才登记当前发布，不提前用 PREPARING 冒充成功。
8. 已明确授权释放时，分别确认单发布引用解除和后端设备占用释放；按各自完成事实清除相应状态，不能一并提前删除。
9. 设备 remove 使用第 20 节的观察/核查分支，不能直接跳到计划释放步骤。
10. 每个已完成或未完成动作均留下结果；独立业务失败继续遵循原批次失败累计规则，调度/日志完整性故障仍保留 B 退出。

此顺序保留已讨论的责任关系，尚未承诺“TPG 在哪一步启用”“有其他发布时怎样准备 portal”“接口物理调用形式”等细节已经冻结。对象成功而状态登记失败，是必须保留并报错的现场，不靠自动回滚或自动补登记遮盖。

### 21.2 接续任务

1. 确认新身份来源：后端记录最小字段、何时登记、按实例查询；取消独立 I1 后哪些事件无需留身份。
2. 冻结当前状态 schema：发布位置 section、共享对象条目、逻辑 ID/生命周期 ID、状态文件位置、原文/索引交叉核对。
3. 把五项逻辑接口参数、输出和返回码定为可调用契约，特别是确认不存在、重复登记、重复清除的具体规则。
4. 在目标 OpenWrt 验证 UCI 的实际读写/提交/失败可观察性，不把通用介绍当作本机结果。
5. 明确完整实例绑定证明的组合：info 没有 DISKSEQ/DM_UUID，现场采样只能核对，不能直接宣布内核原子绑定证明已完成。
6. 编排只支持 rw 的 target/TPG/portal、后端、LUN 创建和逐项读回；同名未知对象、部分创建、登记失败的处理要先写明。
7. 分别定义单发布释放确认、最后引用后的后端回收确认；不要用 DEACTIVATED 当设备未占用。
8. 接入真实延期授权，保持已验证的重入与计数协议。
9. 修订现有代码中的 ro 接受范围、I1 分支、remove 筛选及日志。保持热路径入口和 A/G 不扩张业务责任。
10. 分步实验：状态纯读写 → 禁用 TPG/无 portal 的对象核对 → rw 单发布/共享引用 → 受控释放 → 无认证客户端 RW；每次区分配置层与数据层成功。

这些是接续开发任务，不是本文新批准的自动恢复、远程管理或 ro 研究工作。第四阶段尚不能只理解为“加几条写 configfs 命令”。

### 21.3 换页时的真实现场与证据位置

下面是最后一次对应实机输出，不是本文写作时重新扫描的结果：

| 对象 | 最后已知状态 |
| --- | --- |
| vg2/BULKTEST00 | dm-2，253:2/30；后续静态映射实验的 lio_access_probe 后端仍占用它 |
| identities | 正式测试已形成实例 28、29、30 的独立 I1；证明序号分别 631、637、643；后续 /tmp 仍可能随重启清空 |
| relation-probe | target、禁用 TPG、空 HBA 保留；实验 LUN 和 lio_relation_probe 后端已删除 |
| access-probe | target 为 iqn.2026-10.local.pogov4:lio-access-probe；TPG 禁用，无 portal；LUN 0/1 共用 lio_access_probe；静态映射最后为 write_protect 1/0 |
| ro-write-test | target 为 iqn.2026-10.local.pogov4:lio-ro-write-test；TPG 已启用，portal 172.16.16.220:32690；客户端 session/node 未注销 |
| RO 临时 LV | vg2/LIO_RO_PROBE，dm-20，253:20/31，后端 lio_ro_write_ro，readonly=1 但真实写入成功 |
| RW 临时 LV | vg2/LIO_RW_PROBE，dm-21，253:21/32，后端 lio_ro_write_rw，readonly=0，RW 对照成功 |

BULKTEST00 的完整 DM_UUID：

```text
LVM-X2w9s3qeto0ppFI6DZbt6iM3M6VoV3DKDLhBgYmbebbwq8cGV1zTeAXGxqp5mnRa
```

只读失败的最关键补证：客户端 payload.bin、ro.after、服务端 ro.server-sector0.bin 的 SHA256 均为：

```text
81ba4aca90b1bc03819b12005043509255d9dd3a232d0ec29ea2b3937ab60558
```

服务端证据为 /tmp/lio-ro-server-evidence.naLgdF；客户端为 /tmp/lio-ro-client.AWZyoE/continue.cBx2ID。最后服务端采集在 RO 取证完成后因 od 缺失返回 127，RW 的服务端后半段未执行。RO 的配置、实例、数据和哈希证据仍成立，不能把工具展示失败说成写保护失败原因。

本页最终不继续 ro 对照实验，按用户决定延后。两个临时 LV 已写入载荷，不是初始全零资源。下一页先只读确认现场；不能把上述设备路径当当前永久事实，也不能直接停用被后端占用的 BULKTEST00、重复创建已有 node 或整体删除 iblock_99。

## 22. 配套文档、源码基线和使用方式

### 22.1 配套记录

- 《lio configfs 实机测试实验记录 v0.1》：单独文件 lio-configfs-real-machine-test-record-v0.1.md。保存了完整能力表、八组实验、RO 失败哈希和远端资源最后状态。
- openwrt-lio-targetd-platform-experiments-v0.1.md：早期 shell、锁、真实 DM 和后台分发实验。
- lio-targetd-scheduler-natural-language-v0.1.md：第一阶段原编排稿。
- lio-targetd-policy-natural-language-v0.1.md：第二阶段原编排稿。
- lio-targetd-identity-natural-language-v0.2.md：第三阶段当时的编排稿；文首“尚未落地”是历史状态，现已有代码，但其独立缓存方向又待调整。
- lio-targetd-identity-design-decision-20261020.md：重大身份存储调整备忘；最后一节所说下一项 ro/rw 已被后续实验和仅 rw 决定更新。

旧完整设计草案提供项目背景，不覆盖本文后续决策。遇到冲突，以本文标注的当前代码行为和最新用户决定分别判断，不能笼统说“新设计已经覆盖代码”。

### 22.2 本文完整源码附录

下附三个文件的原样当前内容，不对开头注释或其他代码作整理性修改。源码是继续编辑的基线，不是安装命令；不要把实验副本路径替换、模拟依赖钩子带入正式文件。

这里冻结的是本页交付的本地源码副本；本次没有从 OpenWrt 重新下载脚本逐字节比对。若下一页现场文件已被手工修改，应先回读并比较，而不是覆盖它。

| 文件 | 行数 | SHA256 |
| --- | --- | --- |
| 80-lio-targetd | 246 | 1c7c03189afb57da70f5656c70a34a2ef456ebfeb85e6f5d71fbe8132a9e9a66 |
| iscsi-luns-queue-update | 221 | 037165ca7b0fd148d276d998b33766eb84627c230b81540021f7f1ce7f457df1 |
| lio-targetd | 591 | e1ce3b1d7e04ea3c4b10033f65145e73182f9b6e871200287c8b9e154fa2ba48 |

lio-targetd 当前开头只有第三阶段注释，但实现包含前三阶段；本文保留其原样并在此明确，不能以注释判断缺失了第一、第二阶段。

附录内的源码区块具有明确 BEGIN/END 标记，可完整复制提取；提取后核对上表 SHA256。本文的原样代码仍接受 ro、仍保留 I1、仍关闭真实延期钩子，正是需要接续修订的版本。


## 附录 A：80-lio-targetd

原样基线 SHA256：1c7c03189afb57da70f5656c70a34a2ef456ebfeb85e6f5d71fbe8132a9e9a66

<!-- BEGIN SOURCE: 80-lio-targetd -->
```sh
#!/bin/sh
# 由 hotplug-call 在 block 类别的子 shell 中 source。
# 只筛选、采集和异步传递；不读取名单、策略、缓存、队列或锁。

[ "${HOTPLUG_TYPE-}" = block ] || return 0
[ "${SUBSYSTEM-}" = block ] || return 0
case "${ACTION-}" in
    add|change|remove) ;;
    *) return 0 ;;
esac

_lio_hp_action=$ACTION
_lio_hp_devpath=${DEVPATH-}
_lio_hp_kernel=${DEVICENAME:-${_lio_hp_devpath##*/}}
case "$_lio_hp_kernel" in
    dm-*)
        _lio_hp_number=${_lio_hp_kernel#dm-}
        case "$_lio_hp_number" in
            ''|*[!0-9]*) return 0 ;;
        esac
        ;;
    *) return 0 ;;
esac

# 只为相关 DM 事件准备日志；不创建队列目录、不等待任何锁。
# hotplug-call 为每份脚本提供独立子 shell，umask 不影响其他脚本。
umask 077
_lio_hp_log=/tmp/lio-targetd.log
_lio_hp_log_error=
_lio_hp_log_rc=0
if [ -L "$_lio_hp_log" ]; then
    _lio_hp_log_error='日志文件不能是符号链接'
    _lio_hp_log_rc=1
elif [ -e "$_lio_hp_log" ]; then
    if [ ! -f "$_lio_hp_log" ]; then
        _lio_hp_log_error='日志路径已存在但不是普通文件'
        _lio_hp_log_rc=1
    elif _lio_hp_log_identity=$(stat -c '%u:%a' "$_lio_hp_log" 8>&- 9>&-); then
        if [ "$_lio_hp_log_identity" != 0:600 ]; then
            _lio_hp_log_error='已有日志必须属于 root 且权限为 600'
            _lio_hp_log_rc=1
        fi
    else
        _lio_hp_log_rc=$?
        _lio_hp_log_error='读取已有日志的属主及权限失败'
    fi
fi

# 以追加方式打开 fd 7；新文件权限为 600，已有内容不清空。
if [ "$_lio_hp_log_rc" -eq 0 ] && {
    _lio_hp_request=/usr/libexec/iscsi-luns-queue-update
    _lio_hp_error=
    _lio_hp_missing=

    case "$_lio_hp_devpath" in
        /devices/*) ;;
        *) _lio_hp_error='DEVPATH 缺失或不是 /devices/ 下的绝对路径' ;;
    esac
    case "$_lio_hp_devpath" in
        *[[:cntrl:]]*|*/../*|*/./*)
            _lio_hp_error='DEVPATH 含控制字符或非规范路径分量'
            ;;
    esac
    if [ "${_lio_hp_devpath##*/}" != "$_lio_hp_kernel" ]; then
        _lio_hp_error='DEVPATH 尾部与 DEVICENAME 不一致'
    fi

    for _lio_hp_field in SEQNUM MAJOR MINOR DISKSEQ; do
        case "$_lio_hp_field" in
            SEQNUM)  _lio_hp_value=${SEQNUM-} ;;
            MAJOR)   _lio_hp_value=${MAJOR-} ;;
            MINOR)   _lio_hp_value=${MINOR-} ;;
            DISKSEQ) _lio_hp_value=${DISKSEQ-} ;;
        esac
        case "$_lio_hp_value" in
            '') _lio_hp_missing="$_lio_hp_missing $_lio_hp_field" ;;
            *[!0-9]*)
                printf '<11>lio-targetd.hotplug: 非法数字字段；字段=%s 值=%s ACTION=%s 设备=%s\n' \
                    "$_lio_hp_field" "$_lio_hp_value" "$_lio_hp_action" \
                    "$_lio_hp_kernel" >&7
                return 1
                ;;
        esac
    done

    if [ -n "$_lio_hp_error" ]; then
        printf '<11>lio-targetd.hotplug: %s；ACTION=%s DEVPATH=%s DEVICENAME=%s\n' \
            "$_lio_hp_error" "$_lio_hp_action" "$_lio_hp_devpath" \
            "${DEVICENAME-}" >&7
        return 1
    fi

    _lio_hp_seqnum=${SEQNUM:--}
    _lio_hp_major=${MAJOR:--}
    _lio_hp_minor=${MINOR:--}
    _lio_hp_diskseq=${DISKSEQ:--}
    _lio_hp_name=-
    _lio_hp_uuid=-
    _lio_hp_source=none

    if [ -n "$_lio_hp_missing" ]; then
        printf '<12>lio-targetd.hotplug: 事件字段缺失；字段=%s ACTION=%s 设备=%s SEQNUM=%s\n' \
            "$_lio_hp_missing" "$_lio_hp_action" "$_lio_hp_kernel" \
            "$_lio_hp_seqnum" >&7 || return 1
    fi

    # remove 不读 sysfs。add/change 仅在事件实例字段齐全时定点取证。
    if [ "$_lio_hp_action" != remove ] &&
       [ "$_lio_hp_major" != - ] &&
       [ "$_lio_hp_minor" != - ] &&
       [ "$_lio_hp_diskseq" != - ]; then
        _lio_hp_sys="/sys$_lio_hp_devpath"
        _lio_hp_live=1
        _lio_hp_read_name=
        _lio_hp_read_uuid=

        for _lio_hp_phase in before after; do
            _lio_hp_sys_major=
            _lio_hp_sys_minor=
            _lio_hp_sys_diskseq=

            [ -d "$_lio_hp_sys" ] || { _lio_hp_live=0; break; }
            if [ ! -r "$_lio_hp_sys/uevent" ]; then
                if [ -e "$_lio_hp_sys/uevent" ]; then
                    printf '<11>lio-targetd.hotplug: uevent 不可读；路径=%s SEQNUM=%s\n' \
                        "$_lio_hp_sys/uevent" "$_lio_hp_seqnum" >&7 || return 1
                fi
                _lio_hp_live=0
                break
            fi

            if while IFS='=' read -r _lio_hp_key _lio_hp_value; do
                case "$_lio_hp_key" in
                    MAJOR)   _lio_hp_sys_major=$_lio_hp_value ;;
                    MINOR)   _lio_hp_sys_minor=$_lio_hp_value ;;
                    DISKSEQ) _lio_hp_sys_diskseq=$_lio_hp_value ;;
                esac
            done <"$_lio_hp_sys/uevent" 2>&7; then
                :
            else
                _lio_hp_rc=$?
                printf '<11>lio-targetd.hotplug: 读取实例属性失败；rc=%s 阶段=%s 路径=%s SEQNUM=%s\n' \
                    "$_lio_hp_rc" "$_lio_hp_phase" "$_lio_hp_sys/uevent" \
                    "$_lio_hp_seqnum" >&7 || return 1
                _lio_hp_live=0
                break
            fi

            if [ "$_lio_hp_sys_major" != "$_lio_hp_major" ] ||
               [ "$_lio_hp_sys_minor" != "$_lio_hp_minor" ] ||
               [ "$_lio_hp_sys_diskseq" != "$_lio_hp_diskseq" ]; then
                _lio_hp_live=0
                break
            fi

            [ "$_lio_hp_phase" = before ] || continue
            if [ ! -r "$_lio_hp_sys/dm/name" ]; then
                if [ -e "$_lio_hp_sys/dm/name" ]; then
                    printf '<11>lio-targetd.hotplug: dm/name 不可读；设备=%s SEQNUM=%s\n' \
                        "$_lio_hp_kernel" "$_lio_hp_seqnum" >&7 || return 1
                fi
                _lio_hp_live=0
                break
            fi

            if IFS= read -r _lio_hp_read_name <"$_lio_hp_sys/dm/name" 2>&7; then
                :
            else
                _lio_hp_rc=$?
                printf '<11>lio-targetd.hotplug: 读取 DM_NAME 失败；rc=%s 设备=%s SEQNUM=%s\n' \
                    "$_lio_hp_rc" "$_lio_hp_kernel" "$_lio_hp_seqnum" >&7 || return 1
                _lio_hp_live=0
                break
            fi
            if [ -z "$_lio_hp_read_name" ]; then
                _lio_hp_live=0
                break
            fi

            if [ -r "$_lio_hp_sys/dm/uuid" ]; then
                if IFS= read -r _lio_hp_read_uuid <"$_lio_hp_sys/dm/uuid" 2>&7; then
                    :
                else
                    _lio_hp_rc=$?
                    printf '<11>lio-targetd.hotplug: 读取 DM_UUID 失败；rc=%s 设备=%s SEQNUM=%s\n' \
                        "$_lio_hp_rc" "$_lio_hp_kernel" "$_lio_hp_seqnum" >&7 || return 1
                    _lio_hp_read_uuid=
                fi
            elif [ -e "$_lio_hp_sys/dm/uuid" ]; then
                printf '<11>lio-targetd.hotplug: dm/uuid 不可读；设备=%s SEQNUM=%s\n' \
                    "$_lio_hp_kernel" "$_lio_hp_seqnum" >&7 || return 1
            fi
        done

        if [ "$_lio_hp_live" -eq 1 ]; then
            case "$_lio_hp_read_name$_lio_hp_read_uuid" in
                *[[:cntrl:]]*)
                    printf '<11>lio-targetd.hotplug: 现场名称或 UUID 含协议控制字符；设备=%s SEQNUM=%s\n' \
                        "$_lio_hp_kernel" "$_lio_hp_seqnum" >&7
                    return 1
                    ;;
            esac
            _lio_hp_name=$_lio_hp_read_name
            _lio_hp_uuid=${_lio_hp_read_uuid:--}
            _lio_hp_source=sysfs
        fi
    fi

    if [ ! -f "$_lio_hp_request" ] || [ ! -r "$_lio_hp_request" ]; then
        printf '<11>lio-targetd.hotplug: 请求脚本不存在或不可读；路径=%s ACTION=%s 设备=%s SEQNUM=%s\n' \
            "$_lio_hp_request" "$_lio_hp_action" "$_lio_hp_kernel" \
            "$_lio_hp_seqnum" >&7
        return 1
    fi

    # 先接独立 stderr，再关闭临时出口和约定的锁描述符。
    /bin/sh "$_lio_hp_request" \
        V1 "$_lio_hp_action" "$_lio_hp_seqnum" "$_lio_hp_kernel" \
        "$_lio_hp_major" "$_lio_hp_minor" "$_lio_hp_diskseq" \
        "$_lio_hp_name" "$_lio_hp_uuid" "$_lio_hp_source" "$_lio_hp_devpath" \
        2>&7 7>&- 8>&- 9>&- </dev/null >/dev/null &
    _lio_hp_launch_rc=$?
    _lio_hp_pid=$!

    if [ "$_lio_hp_launch_rc" -ne 0 ] || [ -z "$_lio_hp_pid" ]; then
        printf '<11>lio-targetd.hotplug: 后台启动失败；rc=%s ACTION=%s 设备=%s SEQNUM=%s\n' \
            "$_lio_hp_launch_rc" "$_lio_hp_action" "$_lio_hp_kernel" \
            "$_lio_hp_seqnum" >&7
        return 1
    fi

    # 这里只表示后台启动已提交，不表示请求已入队或业务已经完成。
    return 0
} 7>>"$_lio_hp_log"; then
    return 0
else
    _lio_hp_rc=$?
    if [ "$_lio_hp_log_rc" -ne 0 ]; then _lio_hp_rc=$_lio_hp_log_rc; fi
    _lio_hp_message="日志出口准备或打开失败；rc=$_lio_hp_rc 原因=${_lio_hp_log_error:-打开文件失败} 路径=$_lio_hp_log ACTION=$_lio_hp_action 设备=$_lio_hp_kernel"
    if logger -t lio-targetd.hotplug "$_lio_hp_message"; then
        :
    else
        printf 'lio-targetd.hotplug: %s\n' "$_lio_hp_message" >&2
    fi
    return 1
fi
```
<!-- END SOURCE: 80-lio-targetd -->

## 附录 B：iscsi-luns-queue-update

原样基线 SHA256：037165ca7b0fd148d276d998b33766eb84627c230b81540021f7f1ce7f457df1

<!-- BEGIN SOURCE: iscsi-luns-queue-update -->
```sh
#!/bin/sh
# hotplug 请求接收者：持 G 提交 Q/R，再竞争 A 并交接执行权。
# 不读取名称表、配置策略、身份缓存，不操作 LVM 或 LIO。

PATH=/usr/sbin:/usr/bin:/sbin:/bin
LC_ALL=C
export PATH LC_ALL
umask 077

lio_dir=/tmp/lio-targetd
lio_flock=/usr/bin/util-linux-flock
lio_worker=/usr/libexec/lio-targetd
lio_stage=init
lio_detail=初始化
lio_submission=none
lio_path=-
lio_action=-
lio_seqnum=-
lio_kernel=-
lio_major=-
lio_minor=-
lio_diskseq=-

# 仅失败时补充上下文；命令原始 stderr 始终保留。
# hotplug 已把 stderr 接到独立日志出口，手工调用则显示在终端。
# 成功 exec 后本 trap 不再运行，由执行器安装自己的退出处理。
trap '
    lio_exit_rc=$?
    trap - 0
    exec 9>&- 8>&-
    if [ "$lio_exit_rc" -ne 0 ]; then
        printf "<11>lio-targetd.request: 请求失败；rc=%s 阶段=%s 原因=%s 提交=%s PID=%s ACTION=%s SEQNUM=%s 设备=%s 实例=%s:%s/%s 路径=%s\n" \
            "$lio_exit_rc" "$lio_stage" "$lio_detail" "$lio_submission" \
            "$$" "$lio_action" "$lio_seqnum" "$lio_kernel" \
            "$lio_major" "$lio_minor" "$lio_diskseq" "$lio_path" >&2
    fi
    exit "$lio_exit_rc"
' 0
trap 'exit 129' HUP
trap 'exit 130' INT
trap 'exit 143' TERM

# 本入口不接受调用方传入的锁；只有向执行器交接时才保留 fd 9。
exec 8>&- 9>&-

lio_stage=validate
lio_detail='要求恰好 11 个参数'
[ "$#" -eq 11 ] || exit 1

lio_detail='参数为空或含控制字符'
for lio_value do
    case "$lio_value" in
        ''|*[[:cntrl:]]*) exit 1 ;;
    esac
done

lio_action=$2
lio_seqnum=$3
lio_kernel=$4
lio_major=$5
lio_minor=$6
lio_diskseq=$7
lio_name=$8
lio_uuid=$9
lio_source=${10}
lio_devpath=${11}

lio_detail='不支持的协议版本'
[ "$1" = V1 ] || exit 1
lio_detail='不支持的事件动作'
case "$lio_action" in
    add|change|remove) ;;
    *) exit 1 ;;
esac

lio_detail='内核设备名不是 dm-数字'
case "$lio_kernel" in
    dm-*)
        lio_number=${lio_kernel#dm-}
        case "$lio_number" in
            ''|*[!0-9]*) exit 1 ;;
        esac
        ;;
    *) exit 1 ;;
esac

lio_detail='事件数字字段必须是十进制数字或缺失标记 -'
for lio_value in "$lio_seqnum" "$lio_major" "$lio_minor" "$lio_diskseq"; do
    case "$lio_value" in
        -) ;;
        *[!0-9]*) exit 1 ;;
    esac
done

lio_detail='DEVPATH 非法或与内核设备名不一致'
case "$lio_devpath" in
    /devices/*) ;;
    *) exit 1 ;;
esac
case "$lio_devpath" in
    */../*|*/./*) exit 1 ;;
esac
[ "${lio_devpath##*/}" = "$lio_kernel" ] || exit 1

lio_detail='现场身份字段与身份来源不一致'
case "$lio_source" in
    none)
        [ "$lio_name" = - ] && [ "$lio_uuid" = - ] || exit 1
        ;;
    sysfs)
        [ "$lio_action" != remove ] || exit 1
        [ "$lio_major" != - ] && [ "$lio_minor" != - ] &&
            [ "$lio_diskseq" != - ] || exit 1
        ;;
    *) exit 1 ;;
esac

lio_stage=preflight
lio_detail='无法确认运行用户'
if lio_uid=$(id -u); then :; else exit $?; fi
lio_detail='请求脚本必须由 root 运行'
[ "$lio_uid" = 0 ] || exit 1
lio_detail='util-linux flock 不存在或不可执行'
lio_path=$lio_flock
[ -f "$lio_flock" ] && [ -x "$lio_flock" ] || exit 1

lio_stage=runtime-dir
lio_path=$lio_dir
lio_detail='运行目录不能是符号链接'
[ ! -L "$lio_dir" ] || exit 1
lio_detail='创建运行目录失败'
if mkdir -p -m 700 "$lio_dir"; then :; else exit $?; fi
lio_detail='读取运行目录属主及权限失败'
if lio_dir_identity=$(stat -c '%u:%a' "$lio_dir"); then :; else exit $?; fi
lio_detail='运行目录必须属于 root 且权限为 700'
[ -d "$lio_dir" ] && [ "$lio_dir_identity" = 0:700 ] || exit 1

# 锁文件始终保留；不能删除后重建，否则可能出现不同 inode 的锁。
lio_stage=lock-files
for lio_path in "$lio_dir/handoff.lock" "$lio_dir/active.lock"; do
    lio_detail='锁文件不能是符号链接'
    [ ! -L "$lio_path" ] || exit 1
    lio_detail='已有锁文件不是普通文件'
    if [ -e "$lio_path" ] && [ ! -f "$lio_path" ]; then exit 1; fi
done

lio_stage=open-G
lio_path=$lio_dir/handoff.lock
lio_detail='打开 G 锁文件失败'
exec 8>>"$lio_path"

lio_stage=wait-G
lio_detail='阻塞取得 G 失败'
if "$lio_flock" -x 8; then :; else exit $?; fi

# 以下 Q/R 检查、追加和标记均在 G 内完成。
lio_stage=queue-files
for lio_path in "$lio_dir/queue" "$lio_dir/request"; do
    lio_detail='Q/R 文件不能是符号链接'
    [ ! -L "$lio_path" ] || exit 1
    lio_detail='已有 Q/R 文件不是普通文件'
    if [ -e "$lio_path" ] && [ ! -f "$lio_path" ]; then exit 1; fi
done

lio_stage=append-Q
lio_path=$lio_dir/queue
lio_detail='追加事件记录失败；可能留有不完整记录'
# 前置换行隔开旧残片；执行器忽略空行，仅接受字段完整且以 END 结束的行。
if printf '\n%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\tEND\n' \
    "$@" >>"$lio_path"; then
    lio_submission=Q-written
else
    exit $?
fi

lio_stage=set-R
lio_path=$lio_dir/request
lio_detail='设置 R 失败；已写入的 Q 保留'
# 使用普通内建命令，使重定向失败可以显式处理；R 的存在即表示有请求。
if printf '%s' '' >"$lio_path"; then
    lio_submission=Q-R-committed
else
    exit $?
fi

lio_stage=open-A
lio_path=$lio_dir/active.lock
lio_detail='打开 A 锁文件失败；已提交的 Q/R 保留'
exec 9>>"$lio_path"

lio_stage=try-A
lio_detail='竞争 A 失败；已提交的 Q/R 保留'
if "$lio_flock" -x -n -E 200 9; then
    lio_a_rc=0
else
    lio_a_rc=$?
fi
case "$lio_a_rc" in
    0) ;;
    200)
        # 已有执行器。本请求已提交，退出处理关闭本进程的 A/G 描述符。
        exit 0
        ;;
    *) exit "$lio_a_rc" ;;
esac

# 已取得 A，先释放 G，让后续请求能够继续入队。
lio_stage=release-G
lio_path=$lio_dir/handoff.lock
lio_detail='关闭 G 描述符失败'
exec 8>&-

lio_stage=worker-check
lio_path=$lio_worker
lio_detail='执行器不存在或不可读；已提交的 Q/R 保留'
[ -f "$lio_worker" ] && [ -r "$lio_worker" ] || exit 1

lio_stage=handoff
lio_detail='exec 交接失败；已提交的 Q/R 保留'
# 同 PID、保留已持锁的 fd 9；事件已在 Q 中，不再次通过参数传递。
exec /bin/sh "$lio_worker" V1 9
```
<!-- END SOURCE: iscsi-luns-queue-update -->

## 附录 C：lio-targetd（前三阶段完整实现）

原样基线 SHA256：e1ce3b1d7e04ea3c4b10033f65145e73182f9b6e871200287c8b9e154fa2ba48

<!-- BEGIN SOURCE: lio-targetd -->
```sh
#!/bin/sh
# 第三阶段：保存事件历史身份并定点核对；不补采身份、不操作或删除 LIO。
PATH=/usr/sbin:/usr/bin:/sbin:/bin
LC_ALL=C
export PATH LC_ALL
umask 077
lio_dir=/tmp/lio-targetd; lio_settings=/etc/lio-targetd/settings.conf; lio_flock=/usr/bin/util-linux-flock; lio_config=${lio_settings%/*}; lio_cfg_loaded=0; lio_cache_dir=$lio_dir/identities; lio_sysroot=/sys; lio_devroot=/dev; lio_transfers=0; lio_first_records=0; lio_reentered=0; lio_clock=/proc/uptime
# UPTIME 为开机以来的秒数；关键阶段用内建 read 采集，不启动计时子进程。
lio_nl='
'
lio_g=$lio_dir/handoff.lock; lio_a=$lio_dir/active.lock; lio_q=$lio_dir/queue; lio_r=$lio_dir/request; lio_b=$lio_dir/batch; lio_stage=init; lio_detail=初始化; lio_path=-; lio_round=0; lio_line_no=0; lio_events=0; lio_anomalies=0; lio_result=0; lio_normal=0; lio_batch_state=none
trap '
    lio_exit_rc=$?
    trap - 0
    exec 6<&- 9>&- 8>&-
    if [ "$lio_exit_rc" -ne 0 ] && [ "$lio_normal" -eq 0 ]; then
        printf "<11>lio-targetd.executor: 执行失败；rc=%s PID=%s 阶段=%s 原因=%s 轮次=%s 行=%s 批次=%s 路径=%s\n"  "$lio_exit_rc" "$$" "$lio_stage" "$lio_detail"  "$lio_round" "$lio_line_no" "$lio_batch_state" "$lio_path" >&2
    fi
    exit "$lio_exit_rc"
' 0
trap 'exit 129' HUP
trap 'exit 130' INT
trap 'exit 143' TERM
exec 8>&-
lio_stage=entry; lio_detail='入口必须为 V1 9'; [ "$#" -eq 2 ] || exit 1
[ "$1" = V1 ] && [ "$2" = 9 ] || exit 1
lio_detail='读取运行用户失败'; if lio_uid=$(id -u 8>&- 9>&-); then :; else exit $?; fi
lio_detail='必须由 root 运行'; [ "$lio_uid" = 0 ] || exit 1
lio_detail='util-linux flock 不存在或不可执行'; lio_path=$lio_flock; [ -f "$lio_flock" ] && [ -x "$lio_flock" ] || exit 1
lio_stage=runtime-check; lio_path=$lio_dir; lio_detail='运行目录缺失或为符号链接'; [ -d "$lio_dir" ] && [ ! -L "$lio_dir" ] || exit 1
lio_detail='读取运行目录权限失败'; if lio_identity=$(stat -c '%u:%a' "$lio_dir" 8>&- 9>&-); then :; else exit $?; fi
lio_detail='运行目录必须属于 root 且权限为 700'; [ "$lio_identity" = 0:700 ] || exit 1
for lio_path in "$lio_g" "$lio_a"; do
    lio_detail='锁文件缺失、不是普通文件或为符号链接'; [ -f "$lio_path" ] && [ ! -L "$lio_path" ] || exit 1
done
lio_stage=A-check; lio_path=$lio_a; lio_detail='读取 A 文件身份失败'; if lio_a_identity=$(stat -c '%d:%i' "$lio_a" 8>&- 9>&-); then :; else exit $?; fi
lio_detail='读取继承的 fd 9 身份失败'; if lio_fd_identity=$(stat -L -c '%d:%i' /proc/self/fd/0 <&9 8>&- 9>&-); then :; else exit $?; fi
lio_detail='fd 9 与 A 文件不一致'; [ "$lio_fd_identity" = "$lio_a_identity" ] || exit 1
lio_stage=settings; lio_path=$lio_settings; lio_detail='全局配置不存在、不是普通文件或不可读'; [ -f "$lio_settings" ] && [ -r "$lio_settings" ] || exit 1
lio_detail='读取全局配置大小失败'; if lio_expected=$(stat -c '%s' "$lio_settings" 8>&- 9>&-); then :; else exit $?; fi
lio_seen=0; lio_bytes=0; lio_detail='打开全局配置失败'
exec 6<"$lio_settings"
while :; do
    lio_line=
    if IFS= read -r lio_line <&6; then lio_read_rc=0; else lio_read_rc=$?; fi
    case "$lio_read_rc" in
        0|1) ;;
        *) lio_detail='读取全局配置失败'; exit "$lio_read_rc" ;;
    esac
    [ "$lio_read_rc" -eq 0 ] || [ -n "$lio_line" ] || break
    lio_line_no=$((lio_line_no + 1)); lio_bytes=$((lio_bytes + ${#lio_line}))
    if [ "$lio_read_rc" -eq 0 ]; then lio_bytes=$((lio_bytes + 1)); fi
    case "$lio_line" in
        *[![:space:]]*) ;;
        *) continue ;;
    esac
    case "$lio_line" in
        \#*) continue ;;
        DEBOUNCE_MS=*)
            lio_detail='全局配置含重复的 DEBOUNCE_MS'; [ "$lio_seen" -eq 0 ] || exit 1
            lio_seen=1; lio_ms=${lio_line#DEBOUNCE_MS=}; lio_detail='滤波值必须为无前导零的正整数毫秒'
            case "$lio_ms" in
                ''|0*|*[!0-9]*) exit 1 ;;
            esac
            lio_detail='本阶段仅支持 1000 毫秒的正整数倍'; case "$lio_ms" in
                *000) lio_seconds=${lio_ms%000} ;;
                *) exit 1 ;;
            esac
            [ -n "$lio_seconds" ] || exit 1
            ;;
        *) lio_detail='全局配置含未知键或非法格式'; exit 1 ;;
    esac
done
exec 6<&-
lio_detail="全局配置读取字节数不符；读取=$lio_bytes 预期=$lio_expected，可能有读取错误或 NUL 字节"; [ "$lio_bytes" = "$lio_expected" ] || exit 1
lio_detail='全局配置缺少 DEBOUNCE_MS'; [ "$lio_seen" -eq 1 ] || exit 1
lio_detail='准备制表符失败'; if lio_tab=$(printf '\t' 8>&- 9>&-); then :; else exit $?; fi
lio_line_no=0; lio_stage=start-log; lio_detail='读取开始时间失败'
if IFS=' ' read -r lio_uptime lio_idle <"$lio_clock"; then :; else exit $?; fi
lio_detail='写入执行器开始日志失败'; if printf '<14>lio-targetd.executor: 开始；PID=%s 协议=V1 滤波毫秒=%s 模式=历史缓存及受限核对，未操作LIO UPTIME=%s\n'  "$$" "$lio_ms" "$lio_uptime" >&2; then :; else exit $?; fi
while :; do
    lio_stage=open-G; lio_path=$lio_g; lio_detail='G 文件缺失或类型异常'; [ -f "$lio_g" ] && [ ! -L "$lio_g" ] || exit 1
    lio_detail='打开 G 失败'; exec 8<"$lio_g"
    lio_stage=wait-G; lio_detail='阻塞取得 G 失败'; if "$lio_flock" -x 8 9>&-; then :; else exit $?; fi
    lio_stage=entry-check; lio_path=$lio_b; lio_detail='发现遗留 B；不覆盖、不重放'; if [ -e "$lio_b" ] || [ -L "$lio_b" ]; then
        lio_batch_state=stale
        exit 1
    fi
    for lio_path in "$lio_q" "$lio_r"; do
        lio_detail='Q/R 为符号链接'; [ ! -L "$lio_path" ] || exit 1
        lio_detail='已有 Q/R 不是普通文件'; if [ -e "$lio_path" ] && [ ! -f "$lio_path" ]; then exit 1; fi
    done
    lio_path=$lio_r; lio_detail='R 不是零字节标记'; [ ! -s "$lio_r" ] || exit 1
    lio_has_r=0
    if [ -f "$lio_r" ]; then lio_has_r=1; fi
    if [ ! -s "$lio_q" ] && [ "$lio_has_r" -eq 0 ]; then
        lio_stage=exit-A; lio_path=$lio_a; lio_detail='正常交接关闭 A 失败'; exec 9>&-
        lio_stage=exit-G; lio_path=$lio_g; lio_detail='正常交接关闭 G 失败'; exec 8>&-
        lio_stage=exit-log; lio_path=-; lio_detail='读取退出时间失败'; if IFS=' ' read -r lio_uptime lio_idle <"$lio_clock"; then :; else exit $?; fi
        lio_detail='写入退出摘要失败'; if printf '<14>lio-targetd.executor: 退出交接完成；PID=%s 轮次=%s 事件=%s 异常=%s rc=%s UPTIME=%s 首次记录=%s 重入记录=%s 成功移交=%s 事件含重入=1\n'  "$$" "$lio_round" "$lio_events" "$lio_anomalies" "$lio_result" "$lio_uptime" "$lio_first_records" "$lio_reentered" "$lio_transfers" >&2; then :; else exit $?; fi
        lio_normal=1
        exit "$lio_result"
    fi
    lio_round=$((lio_round + 1)); lio_line_no=0; lio_has_batch=0; lio_missing_r=0
    if [ -s "$lio_q" ]; then
        lio_stage=move-Q-B; lio_path=$lio_b; lio_detail='Q 移交为 B 失败'; if mv "$lio_q" "$lio_b" 8>&- 9>&-; then
            lio_has_batch=1; lio_batch_state=owned
        else
            exit $?
        fi
        if [ "$lio_has_r" -eq 0 ]; then lio_missing_r=1; fi
    fi
    if [ "$lio_has_r" -eq 1 ]; then
        lio_stage=consume-R; lio_path=$lio_r; lio_detail='消费 R 失败；保留实际 Q/R/B 状态'; if rm "$lio_r" 8>&- 9>&-; then :; else exit $?; fi
    fi
    lio_stage=release-G; lio_path=$lio_g; lio_detail='批次入口关闭 G 失败'; exec 8>&-
    lio_stage=round-log; lio_detail='读取轮次开始时间失败'; if IFS=' ' read -r lio_uptime lio_idle <"$lio_clock"; then :; else exit $?; fi
    lio_detail='写入轮次开始日志失败'; if printf '<14>lio-targetd.executor: 轮次开始；PID=%s 轮次=%s 有批次=%s UPTIME=%s\n' "$$" "$lio_round" "$lio_has_batch" "$lio_uptime" >&2; then :; else exit $?; fi
    if [ "$lio_missing_r" -eq 1 ]; then
        lio_anomalies=$((lio_anomalies + 1)); lio_result=1; lio_stage=missing-R-log; lio_detail='写入 R 缺失异常日志失败'
        if printf '<11>lio-targetd.executor: Q 非空但 R 缺失；仍处理已移交批次；PID=%s 轮次=%s B=%s\n'  "$$" "$lio_round" "$lio_b" >&2; then :; else exit $?; fi
    fi
    if [ "$lio_has_batch" -eq 1 ]; then
        lio_path=$lio_b; lio_stage=batch-size; lio_detail='读取 B 大小失败'; if lio_expected=$(stat -c '%s' "$lio_b" 8>&- 9>&-); then :; else exit $?; fi
        for lio_pass in validate log; do
            lio_stage=batch-$lio_pass; lio_i_known=0; lio_i_unknown=0; lio_i_error=0; lio_c_new=0; lio_c_reuse=0; lio_c_keep=0; lio_c_error=0; lio_c_skip=0; lio_l_ok=0; lio_l_mismatch=0; lio_l_error=0; lio_l_skip=0; lio_stage3_bad=0; lio_denied=0; lio_unknown=0; lio_config_failed=0; lio_no_policy=0; lio_matched=0; lio_on=0; lio_off=0; lio_line_no=0; lio_blank=0; lio_valid=0; lio_bad=0; lio_bytes=0; lio_detail='打开 B 失败'; exec 6<"$lio_b"
            while :; do
                lio_line=
                if IFS= read -r lio_line <&6; then lio_read_rc=0; else lio_read_rc=$?; fi
                case "$lio_read_rc" in
                    0|1) ;;
                    *) lio_detail='读取 B 失败'; exit "$lio_read_rc" ;;
                esac
                [ "$lio_read_rc" -eq 0 ] || [ -n "$lio_line" ] || break
                lio_line_no=$((lio_line_no + 1)); lio_bytes=$((lio_bytes + ${#lio_line}))
                if [ "$lio_read_rc" -eq 0 ]; then lio_bytes=$((lio_bytes + 1)); fi
                if [ -z "$lio_line" ] && [ "$lio_read_rc" -eq 0 ]; then
                    lio_blank=$((lio_blank + 1))
                    continue
                fi
                lio_reason=
                if [ "$lio_read_rc" -ne 0 ]; then lio_reason='记录末尾缺少换行'; fi
                lio_rest=$lio_line
                set --
                while :; do
                    case "$lio_rest" in
                        *"$lio_tab"*)
                            lio_field=${lio_rest%%"$lio_tab"*}
                            set -- "$@" "$lio_field"
                            lio_rest=${lio_rest#*"$lio_tab"}
                            ;;
                        *) set -- "$@" "$lio_rest"; break ;;
                    esac
                done
                if [ "$#" -ne 12 ] && [ "$#" -ne 13 ]; then
                    lio_reason='字段数不是 12 或 13'
                else
                    for lio_field do
                        case "$lio_field" in
                            ''|*[[:cntrl:]]*) lio_reason='字段为空或含控制字符' ;;
                        esac
                    done
                    [ "$1" = V1 ] || lio_reason='不支持的记录版本'
                    lio_tail=0; lio_extended=0
                    if [ "$#" -eq 12 ]; then [ "${12}" = END ] || lio_reason='记录缺少 END'; else
                        lio_extended=1; lio_tail=${12}
                        case "$lio_tail" in 0|1|2) ;; *) lio_reason='队尾计数必须为 0、1 或 2' ;; esac
                        [ "${13}" = END ] || lio_reason='记录缺少 END'
                    fi
                    case "$2" in add|change|remove) ;; *) lio_reason='非法事件动作' ;; esac
                    case "$4" in
                        dm-*)
                            lio_number=${4#dm-}
                            case "$lio_number" in ''|*[!0-9]*) lio_reason='非法 DM 设备名' ;; esac
                            ;;
                        *) lio_reason='非法 DM 设备名' ;;
                    esac
                    for lio_field in "$3" "$5" "$6" "$7"; do
                        case "$lio_field" in
                            -) ;;
                            ''|*[!0-9]*) lio_reason='非法数字字段' ;;
                        esac
                    done
                    case "${11}" in /devices/*) ;; *) lio_reason='非法 DEVPATH' ;; esac
                    case "${11}" in */../*|*/./*) lio_reason='DEVPATH 含异常路径分量' ;; esac
                    lio_devpath=${11}
                    [ "${lio_devpath##*/}" = "$4" ] || lio_reason='DEVPATH 与设备名不一致'
                    case "${10}" in
                        none)
                            if [ "$8" != - ] || [ "$9" != - ]; then lio_reason='none 却携带现场身份'; fi
                            ;;
                        sysfs)
                            if [ "$2" = remove ] || [ "$5" = - ] || [ "$6" = - ] || [ "$7" = - ]; then
                                lio_reason='sysfs 身份与事件实例不一致'
                            fi
                            ;;
                        *) lio_reason='非法身份来源' ;;
                    esac
                fi
                if [ -n "$lio_reason" ]; then
                    lio_bad=$((lio_bad + 1)); lio_detail='写入坏记录日志失败'
                    if printf '<11>lio-targetd.executor: 批次记录非法；PID=%s 轮次=%s 遍=%s 行=%s 原因=%s 原文保留于=%s\n'  "$$" "$lio_round" "$lio_pass" "$lio_line_no" "$lio_reason" "$lio_b" >&2; then :; else exit $?; fi
                    continue
                fi
                lio_valid=$((lio_valid + 1))
                if [ "$lio_pass" = log ]; then
                    lio_detail='写入事件记录日志失败'; if printf '<14>lio-targetd.executor: 事件记录；PID=%s 轮次=%s 行=%s ACTION=%s SEQNUM=%s 设备=%s 实例=%s:%s/%s DM_NAME=%s DM_UUID=%s 来源=%s DEVPATH=%s\n'  "$$" "$lio_round" "$lio_line_no" "$2" "$3" "$4" "$5" "$6" "$7"  "$8" "$9" "${10}" "${11}" >&2; then :; else exit $?; fi
                    if [ "$lio_extended" -eq 0 ]; then lio_first_records=$((lio_first_records + 1)); else lio_reentered=$((lio_reentered + 1)); fi
                    # 保存原始事件；缓存仅补充独立关联上下文，不覆盖原始日志或重排输入。
                    lio_ev_action=$2; lio_ev_seq=$3; lio_ev_kernel=$4; lio_ev_major=$5; lio_ev_minor=$6; lio_ev_disk=$7; lio_ev_name=$8; lio_ev_uuid=$9; lio_ev_source=${10}; lio_ev_path=${11}; lio_name=$8; lio_uuid=$9; lio_id_source=none; lio_id_state=unknown; lio_id_error=; lio_cache_read=未读; lio_cache_action=跳过; lio_live=跳过; lio_live_item=-; lio_live_expected=-; lio_live_actual=-; lio_live_rc=0; lio_sys=-; lio_node=-; lio_uuid_check=跳过; lio_uuid_reads=0; lio_event_on=0; lio_this_fail=0; lio_key_ok=1; lio_cache_file=-
                    for lio_key_part in major minor disk; do
                        case "$lio_key_part" in major) lio_number=$lio_ev_major ;; minor) lio_number=$lio_ev_minor ;; disk) lio_number=$lio_ev_disk ;; esac
                        if [ "$lio_number" = - ]; then lio_key_ok=0; else lio_number=${lio_number#"${lio_number%%[!0]*}"}; lio_number=${lio_number:-0}; fi
                        case "$lio_key_part" in major) lio_key_major=$lio_number ;; minor) lio_key_minor=$lio_number ;; disk) lio_key_disk=$lio_number ;; esac
                    done
                    if [ "$lio_ev_source" = sysfs ] && [ "$lio_ev_name" != - ] && [ "$lio_key_ok" -eq 1 ]; then lio_id_state=known; lio_id_source=event_sysfs; fi
                    lio_stage=identity-read; lio_detail='读取实例缓存失败'; if [ "$lio_key_ok" -eq 1 ]; then
                        lio_cache_file=$lio_cache_dir/${lio_key_major}_${lio_key_minor}_${lio_key_disk}; lio_cache_read=miss
                        if [ -e "$lio_cache_dir" ] || [ -L "$lio_cache_dir" ]; then
                            if [ -d "$lio_cache_dir" ] && [ ! -L "$lio_cache_dir" ]; then
                                if lio_ci=$(stat -c '%u:%a' "$lio_cache_dir" 8>&- 9>&-); then [ "$lio_ci" = 0:700 ] || lio_id_error='缓存目录属主或权限异常'; else lio_id_error="缓存目录 stat 失败，rc=$?"; fi
                            else lio_id_error='缓存目录类型异常'; fi
                            if [ -z "$lio_id_error" ] && { [ -e "$lio_cache_file" ] || [ -L "$lio_cache_file" ]; }; then
                                if [ -f "$lio_cache_file" ] && [ ! -L "$lio_cache_file" ]; then
                                    if lio_ci=$(stat -c '%u:%a:%s' "$lio_cache_file" 8>&- 9>&-); then
                                        lio_cs=${lio_ci##*:}
                                        case "$lio_ci" in 0:600:*) ;; *) lio_id_error='缓存记录属主或权限异常' ;; esac
                                        if [ -z "$lio_id_error" ]; then
                                            if IFS= read -r lio_cache_line <"$lio_cache_file"; then
                                                [ "$lio_cs" = "$((${#lio_cache_line} + 1))" ] || lio_id_error='缓存字节数、行数或 NUL 校验失败'
                                            else lio_id_error="缓存记录读取失败或缺换行，rc=$?"; fi
                                        fi
                                    else lio_id_error="缓存记录 stat 失败，rc=$?"; fi
                                else lio_id_error='缓存记录不是非链接普通文件'; fi
                                if [ -z "$lio_id_error" ]; then
                                    lio_rest=$lio_cache_line; set --
                                    while :; do case "$lio_rest" in *"$lio_tab"*) set -- "$@" "${lio_rest%%"$lio_tab"*}"; lio_rest=${lio_rest#*"$lio_tab"} ;; *) set -- "$@" "$lio_rest"; break ;; esac; done
                                    if [ "$#" -ne 12 ]; then lio_id_error='缓存字段数非法'; else
                                        for lio_field do case "$lio_field" in ''|*[[:cntrl:]]*) lio_id_error='缓存字段为空或含控制字符' ;; esac; done
                                        if [ "$1" != I1 ] || [ "$2" != "$lio_ev_kernel" ] || [ "$3:$4/$5" != "$lio_key_major:$lio_key_minor/$lio_key_disk" ] || [ "$8" != "$lio_ev_path" ] || [ "$9" != event_sysfs ] || [ "${11}" != identity ] || [ "${12}" != END ]; then lio_id_error='缓存格式、来源或实例交叉核对失败'; fi
                                        case "$6" in -|*[[:space:]/]*|*\"*|*\'*) lio_id_error='缓存名称非法' ;; esac
                                        case "${10}" in -) ;; *[!0-9]*) lio_id_error='缓存证明序号非法' ;; esac
                                        if [ -z "$lio_id_error" ]; then
                                            lio_cache_read=hit
                                            if [ "$lio_id_state" = known ]; then
                                                if [ "$lio_name" != "$6" ] || { [ "$lio_uuid" != - ] && [ "$7" != - ] && [ "$lio_uuid" != "$7" ]; }; then lio_id_error='同实例事件与缓存身份冲突'; fi
                                                if [ "$lio_uuid" = - ]; then lio_uuid=$7; fi
                                            else lio_name=$6; lio_uuid=$7; lio_id_state=known; lio_id_source=cache; fi
                                        fi
                                    fi
                                fi
                            fi
                        fi
                    fi
                    if [ -n "$lio_id_error" ]; then lio_cache_read=error; lio_id_state=error; lio_cache_action=失败; lio_this_fail=1; fi
                    if [ "$lio_id_state" = unknown ]; then lio_this_fail=1; fi
                    if printf '<14>lio-targetd.executor: 身份关联；PID=%s 轮次=%s 行=%s SEQNUM=%s 原始来源=%s 关联来源=%s 状态=%s DM_NAME=%s DM_UUID=%s 缓存读取=%s 文件=%s 原因=%s\n' "$$" "$lio_round" "$lio_line_no" "$lio_ev_seq" "$lio_ev_source" "$lio_id_source" "$lio_id_state" "$lio_name" "$lio_uuid" "$lio_cache_read" "$lio_cache_file" "${lio_id_error:--}" >&2; then :; else exit $?; fi
                    set -- V1 "$lio_ev_action" "$lio_ev_seq" "$lio_ev_kernel" "$lio_ev_major" "$lio_ev_minor" "$lio_ev_disk" "$lio_name" "$lio_uuid" "$lio_ev_source" "$lio_ev_path"
                    lio_stage=policy-judge; lio_detail='事件策略判断失败'; lio_hits=-; lio_query=跳过; lio_kind=denied; lio_label=名称未准入; lio_reason='名称表未准入，不新建发布'; lio_priority=14; if [ "$lio_cfg_names_ok" -eq 0 ]; then
                        lio_kind=config; lio_label=配置失败; lio_reason='周期名称表无效'; lio_priority=11
                    elif [ "$lio_id_state" != known ]; then
                        lio_kind=unknown; lio_label=身份未确认; lio_reason='事件及精确缓存无法确认身份，不现场补采'; lio_priority=12
                    else
                        lio_allowed=0
                        case "$lio_cfg_names" in *"$lio_nl$8$lio_nl"*) lio_allowed=1 ;; esac
                        if [ "$lio_allowed" -eq 1 ]; then
                            lio_hits=0; lio_query=完成
                            if [ "$lio_cfg_complete" -eq 0 ]; then lio_query=仅有效组; fi
                            lio_scanned=0
                            while IFS="$lio_tab" read -r lio_pd_dm lio_pd_group lio_pd_iqn lio_pd_tpg lio_pd_lun lio_pd_access lio_pd_state lio_pd_portal lio_pd_attrs lio_pd_file lio_pd_line; do
                                [ -n "$lio_pd_dm" ] || continue
                                lio_scanned=$((lio_scanned + 1))
                                [ "$lio_pd_dm" = "$8" ] || continue
                                lio_hits=$((lio_hits + 1))
                                if [ "$lio_pd_state" = on ]; then
                                    if [ "$lio_ev_action" = remove ]; then lio_pd_result='历史策略匹配，remove仅关联、不发布'; else lio_event_on=$((lio_event_on + 1)); lio_on=$((lio_on + 1)); lio_pd_result='配置允许，待现场复核'; fi
                                else
                                    lio_off=$((lio_off + 1)); lio_pd_result='条目期望未发布，是否需要撤销尚未检查'
                                fi
                                lio_pd_scope=本组配置有效
                                if [ "$lio_cfg_complete" -eq 0 ]; then lio_pd_scope='仅本组已知结果，整体配置判断未完成'; fi
                                lio_detail='写入 LUN 条目判断失败'; if printf '<14>lio-targetd.executor: LUN判断；PID=%s 轮次=%s 行=%s SEQNUM=%s 实例=%s:%s/%s DM_NAME=%s DM_UUID=%s 来源=%s NAME=%s IQN=%s TPG=%s LUN_ID=%s ACCESS=%s ADMIN_STATE=%s PORTAL=%s 属性=%s 视图完整=%s 范围=%s 结果=%s 文件=%s 配置行=%s\n'  "$$" "$lio_round" "$lio_line_no" "$3" "$5" "$6" "$7" "$8" "$9" "${10}" "$lio_pd_group" "$lio_pd_iqn" "$lio_pd_tpg" "$lio_pd_lun" "$lio_pd_access" "$lio_pd_state" "$lio_pd_portal" "$lio_pd_attrs" "$lio_cfg_complete" "$lio_pd_scope" "$lio_pd_result" "$lio_pd_file" "$lio_pd_line" >&2; then :; else exit $?; fi
                            done <<LIO_POLICY_ROWS
$lio_cfg_policies
LIO_POLICY_ROWS
                            lio_detail='周期策略内存表记录数不符'; [ "$lio_scanned" -eq "$lio_cfg_count" ] || exit 1
                            if [ "$lio_cfg_complete" -eq 0 ]; then
                                lio_kind=config; lio_label=配置判断不完整; lio_reason='存在无效配置组或目录枚举失败'; lio_priority=11
                            elif [ "$lio_hits" -eq 0 ]; then
                                lio_kind=no_policy; lio_label=无策略; lio_reason='名称已准入，完整配置中没有匹配条目'
                            else
                                lio_kind=matched; lio_label=已匹配策略; lio_reason='已记录配置意图，现场结果另见第三阶段日志，LIO未执行'
                            fi
                        fi
                    fi
                    case "$lio_kind" in
                        denied) lio_denied=$((lio_denied + 1)) ;;
                        unknown) lio_unknown=$((lio_unknown + 1)); lio_result=1; lio_anomalies=$((lio_anomalies + 1)) ;;
                        config) lio_config_failed=$((lio_config_failed + 1)); lio_result=1 ;;
                        no_policy) lio_no_policy=$((lio_no_policy + 1)) ;;
                        matched) lio_matched=$((lio_matched + 1)) ;;
                    esac
                    lio_detail='写入事件业务判断失败'; if printf '<%s>lio-targetd.executor: 业务判断；PID=%s 轮次=%s 行=%s ACTION=%s SEQNUM=%s 设备=%s 实例=%s:%s/%s DM_NAME=%s DM_UUID=%s 来源=%s 视图完整=%s 策略查询=%s 命中=%s 结果=%s 原因=%s\n'  "$lio_priority" "$$" "$lio_round" "$lio_line_no" "$2" "$3" "$4" "$5" "$6" "$7" "$8" "$9" "${10}" "$lio_cfg_complete" "$lio_query" "$lio_hits" "$lio_label" "$lio_reason" >&2; then :; else exit $?; fi
                    # 历史保存是独立动作，先完成；后面的现场失败不会回滚历史事实。
                    if [ "$lio_cache_read" = hit ]; then lio_cache_action=保留; fi
                    lio_stage=identity-save; lio_detail='保存事件历史身份失败'; if [ "$lio_kind" = matched ] && [ "$lio_cfg_complete" -eq 1 ] && [ "$lio_event_on" -gt 0 ] && [ "$lio_ev_action" != remove ] && [ "$lio_id_state" = known ]; then
                        if [ "$lio_cache_read" = hit ]; then lio_cache_action=复用; else
                            if [ "$lio_ev_source" != sysfs ] || [ "$lio_key_ok" -ne 1 ]; then lio_id_error='新缓存只能来自完整事件快照'; else
                                if [ ! -e "$lio_cache_dir" ] && [ ! -L "$lio_cache_dir" ]; then
                                    if mkdir -m 700 "$lio_cache_dir" 8>&- 9>&-; then :; else lio_id_error="建立缓存目录失败，rc=$?"; fi
                                fi
                                if [ -z "$lio_id_error" ]; then
                                    if lio_tmp=$(mktemp "$lio_cache_dir/.identity.XXXXXX" 8>&- 9>&-); then
                                        lio_save_line="I1$lio_tab$lio_ev_kernel$lio_tab$lio_key_major$lio_tab$lio_key_minor$lio_tab$lio_key_disk$lio_tab$lio_ev_name$lio_tab$lio_ev_uuid$lio_tab$lio_ev_path${lio_tab}event_sysfs$lio_tab$lio_ev_seq${lio_tab}identity${lio_tab}END"
                                        if printf '%s\n' "$lio_save_line" >"$lio_tmp"; then
                                            if lio_ci=$(stat -c '%u:%a:%s' "$lio_tmp" 8>&- 9>&-) && IFS= read -r lio_written <"$lio_tmp"; then
                                                if [ "$lio_ci" != "0:600:$((${#lio_save_line} + 1))" ] || [ "$lio_written" != "$lio_save_line" ]; then lio_id_error='临时缓存回读或权限校验失败'; fi
                                            else lio_id_error="临时缓存校验读取失败，rc=$?"; fi
                                            if [ -z "$lio_id_error" ]; then
                                                if [ -e "$lio_cache_file" ] || [ -L "$lio_cache_file" ]; then lio_id_error='新建前发现已有实例记录，不覆盖'; elif mv "$lio_tmp" "$lio_cache_file" 8>&- 9>&-; then lio_cache_action=建立; else lio_id_error="缓存 rename 失败，rc=$? 临时文件=$lio_tmp"; fi
                                            fi
                                        else lio_id_error="写入缓存失败，rc=$? 临时文件=$lio_tmp"; fi
                                    else lio_id_error="建立临时缓存失败，rc=$?"; fi
                                fi
                            fi
                            if [ -n "$lio_id_error" ]; then lio_cache_action=失败; lio_this_fail=1; fi
                        fi
                        if [ "$lio_cache_action" = 建立 ] || [ "$lio_cache_action" = 复用 ]; then
                            # 预期仅来自事件/精确缓存；现场值只用于比较及错误日志。
                            lio_stage=live-check; lio_detail='现场定点核对失败'; lio_live=通过; lio_sys=$lio_sysroot$lio_ev_path; lio_node=$lio_devroot/mapper/$lio_name; for lio_live_phase in before after; do
                                lio_live_item=$lio_live_phase:实例; lio_live_expected=$lio_key_major:$lio_key_minor/$lio_key_disk; lio_live_actual=-; lio_live_rc=0
                                if [ ! -d "$lio_sys" ]; then lio_live=不相符; break; fi
                                if lio_uevent=$(cat "$lio_sys/uevent" 8>&- 9>&-); then :; else lio_live_rc=$?; lio_live=无法确认; break; fi
                                lio_l_major=-; lio_l_minor=-; lio_l_disk=-; lio_l_seen=' '
                                while IFS='=' read -r lio_l_key lio_l_value; do
                                    case "$lio_l_key" in MAJOR|MINOR|DISKSEQ)
                                        case "$lio_l_seen" in *" $lio_l_key "*) lio_live=无法确认 ;; esac
                                        lio_l_seen="$lio_l_seen$lio_l_key "; case "$lio_l_value" in ''|*[!0-9]*) lio_live=无法确认 ;; esac
                                        case "$lio_l_key" in MAJOR) lio_l_major=$lio_l_value ;; MINOR) lio_l_minor=$lio_l_value ;; DISKSEQ) lio_l_disk=$lio_l_value ;; esac ;;
                                    esac
                                done <<LIO_LIVE_UEVENT
$lio_uevent
LIO_LIVE_UEVENT
                                lio_live_actual=$lio_l_major:$lio_l_minor/$lio_l_disk; if [ "$lio_l_major" = - ] || [ "$lio_l_minor" = - ] || [ "$lio_l_disk" = - ]; then lio_live=无法确认; fi
                                [ "$lio_live" = 通过 ] || break
                                if [ "$lio_live_actual" != "$lio_live_expected" ]; then lio_live=不相符; break; fi
                                for lio_live_attr in name uuid; do
                                    if [ "$lio_live_attr" = name ]; then lio_live_expected=$lio_name; else lio_live_expected=$lio_uuid; [ "$lio_uuid" != - ] || continue; fi
                                    lio_live_item=$lio_live_phase:$lio_live_attr; lio_live_actual=-
                                    if lio_live_actual=$(cat "$lio_sys/dm/$lio_live_attr" 8>&- 9>&-); then
                                        if [ "$lio_live_actual" != "$lio_live_expected" ]; then lio_live=不相符; break; fi
                                    else lio_live_rc=$?; lio_live=无法确认; break; fi
                                    if [ "$lio_live_attr" = uuid ]; then lio_uuid_reads=$((lio_uuid_reads + 1)); fi
                                done
                                [ "$lio_live" = 通过 ] || break
                                [ "$lio_live_phase" = before ] || continue
                                lio_live_item=后端节点; lio_live_expected=$lio_key_major:$lio_key_minor; lio_live_actual=-
                                if [ ! -b "$lio_node" ]; then if [ -e "$lio_node" ]; then lio_live_actual=非块设备; else lio_live_actual=节点不存在; fi; lio_live=不相符; break; fi
                                if lio_rdev=$(stat -L -c '%t:%T' "$lio_node" 8>&- 9>&-); then
                                    case "$lio_rdev" in *[!0-9a-fA-F:]*|:*|*:|*:*:*) lio_live=无法确认 ;; *:*) lio_live_actual=$((0x${lio_rdev%:*})):$((0x${lio_rdev#*:})) ;; *) lio_live=无法确认 ;; esac
                                    [ "$lio_live" = 通过 ] || break
                                    if [ "$lio_live_actual" != "$lio_live_expected" ]; then lio_live=不相符; break; fi
                                else lio_live_rc=$?; lio_live=无法确认; break; fi
                            done
                            if [ "$lio_uuid" != - ]; then if [ "$lio_uuid_reads" -eq 2 ]; then lio_uuid_check=通过; else lio_uuid_check=未完成; fi; fi
                            if [ "$lio_live" != 通过 ]; then lio_this_fail=1; fi
                        fi
                    fi
                    # 真实延期判据尚未接入。仅未来的已执行发布记录可授权此处置 1；无生产模拟开关。
                    lio_defer=0; lio_wait_old=-; lio_wait_slot=-
                    case "$lio_defer" in 0|1) ;; *) lio_detail='延期接入点状态非法'; exit 1 ;; esac
                    lio_transfer=无
                    if [ "$lio_defer" -eq 1 ]; then
                        lio_stage=defer; lio_detail='延期必须有通过的核对、明确旧实例和发布位置'; [ "$lio_live" = 通过 ] && [ "$lio_wait_old" != - ] && [ "$lio_wait_slot" != - ] && [ "$lio_this_fail" -eq 0 ] || exit 1
                        if [ "$lio_valid" -eq "$lio_batch_total" ]; then lio_tail=$((lio_tail + 1)); fi
                        if [ "$lio_tail" -ge 3 ]; then lio_transfer=延期失败; lio_this_fail=1; else
                            lio_path=$lio_g; lio_detail='重新提交取得 G 失败'; exec 8<"$lio_g"; if "$lio_flock" -x 8 9>&-; then :; else exit $?; fi
                            for lio_path in "$lio_q" "$lio_r"; do
                                lio_detail='重新提交 Q/R 类型异常'; [ ! -L "$lio_path" ] || exit 1; if [ -e "$lio_path" ] && [ ! -f "$lio_path" ]; then exit 1; fi
                            done
                            lio_detail='重新提交 R 非零'; [ ! -s "$lio_r" ] || exit 1; lio_path=$lio_q; lio_detail='重新追加任务失败，保留 B/Q/R'; if printf '\nV1\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\tEND\n' "$lio_ev_action" "$lio_ev_seq" "$lio_ev_kernel" "$lio_ev_major" "$lio_ev_minor" "$lio_ev_disk" "$lio_ev_name" "$lio_ev_uuid" "$lio_ev_source" "$lio_ev_path" "$lio_tail" >>"$lio_q"; then :; else exit $?; fi
                            lio_path=$lio_r; lio_detail='重新提交设置 R 失败，保留 B/Q/R'; : >"$lio_r" || exit $?; exec 8>&-; lio_transfer=成功移交; lio_transfers=$((lio_transfers + 1))
                        fi
                    fi
                    if [ "$lio_this_fail" -eq 1 ]; then
                        lio_result=1; lio_stage3_bad=$((lio_stage3_bad + 1)); if [ "$lio_kind" != unknown ]; then lio_anomalies=$((lio_anomalies + 1)); fi; lio_priority=11
                    else lio_priority=14; fi
                    case "$lio_id_state" in known) lio_i_known=$((lio_i_known + 1)) ;; unknown) lio_i_unknown=$((lio_i_unknown + 1)) ;; error) lio_i_error=$((lio_i_error + 1)) ;; esac
                    case "$lio_cache_action" in 建立) lio_c_new=$((lio_c_new + 1)) ;; 复用) lio_c_reuse=$((lio_c_reuse + 1)) ;; 保留) lio_c_keep=$((lio_c_keep + 1)) ;; 失败) lio_c_error=$((lio_c_error + 1)) ;; 跳过) lio_c_skip=$((lio_c_skip + 1)) ;; esac
                    case "$lio_live" in 通过) lio_l_ok=$((lio_l_ok + 1)) ;; 不相符) lio_l_mismatch=$((lio_l_mismatch + 1)) ;; 无法确认) lio_l_error=$((lio_l_error + 1)) ;; 跳过) lio_l_skip=$((lio_l_skip + 1)) ;; esac
                    lio_stage=identity-log; lio_detail='写入第三阶段结果失败'; lio_path=$lio_b; if printf '<%s>lio-targetd.executor: 第三阶段结果；PID=%s 轮次=%s 行=%s SEQNUM=%s 实例=%s:%s/%s 关联=%s 缓存动作=%s 缓存原因=%s 现场核对=%s 检查项=%s 预期=%s 实际=%s rc=%s sysfs=%s 节点=%s UUID核对=%s 队尾计数=%s 移交=%s 等待实例=%s 发布位置=%s LIO=未执行 缓存删除=未执行\n' "$lio_priority" "$$" "$lio_round" "$lio_line_no" "$lio_ev_seq" "$lio_ev_major" "$lio_ev_minor" "$lio_ev_disk" "$lio_id_state" "$lio_cache_action" "${lio_id_error:--}" "$lio_live" "$lio_live_item" "$lio_live_expected" "$lio_live_actual" "$lio_live_rc" "${lio_sys:--}" "${lio_node:--}" "$lio_uuid_check" "$lio_tail" "$lio_transfer" "$lio_wait_old" "$lio_wait_slot" >&2; then :; else exit $?; fi
                fi
            done
            exec 6<&-
            lio_detail="B 读取字节数不符；读取=$lio_bytes 预期=$lio_expected，可能有读取错误、NUL 字节或外部改动"; [ "$lio_bytes" = "$lio_expected" ] || exit 1
            lio_detail='批次含坏记录；保留 B，不执行后续步骤'; [ "$lio_bad" -eq 0 ] || exit 1
            lio_detail='读取批次阶段完成时间失败'; if IFS=' ' read -r lio_uptime lio_idle <"$lio_clock"; then :; else exit $?; fi
            lio_detail='写入批次摘要失败'; if printf '<14>lio-targetd.executor: 批次阶段完成；PID=%s 轮次=%s 阶段=%s 总行=%s 空行=%s 有效事件=%s UPTIME=%s\n'  "$$" "$lio_round" "$lio_pass" "$lio_line_no" "$lio_blank" "$lio_valid" "$lio_uptime" >&2; then :; else exit $?; fi
            if [ "$lio_pass" = validate ]; then lio_batch_total=$lio_valid; fi
            if [ "$lio_pass" = validate ] && [ "$lio_cfg_loaded" -eq 0 ]; then
                lio_stage=policy-load; lio_detail='读取配置开始时间失败'; if IFS=' ' read -r lio_uptime lio_idle <"$lio_clock"; then :; else exit $?; fi
                lio_detail='写入配置读取开始日志失败'; if printf '<14>lio-targetd.executor: 周期配置读取开始；PID=%s 目录=%s UPTIME=%s\n' "$$" "$lio_config" "$lio_uptime" >&2; then :; else exit $?; fi
                lio_cfg_loaded=1; lio_cfg_names_ok=1; lio_cfg_complete=1; lio_cfg_names=$lio_nl; lio_cfg_policies=; lio_cfg_name_count=0; lio_cfg_good=0; lio_cfg_bad=0; lio_cfg_count=0; lio_cfg_errors=0
                if ls -A "$lio_config" >/dev/null 8>&- 9>&-; then :; else
                    lio_cfg_rc=$?; lio_cfg_complete=0; lio_cfg_errors=$((lio_cfg_errors + 1))
                    if printf '<11>lio-targetd.executor: 配置目录枚举失败；rc=%s PID=%s 目录=%s\n' "$lio_cfg_rc" "$$" "$lio_config" >&2; then :; else exit $?; fi
                fi
                for lio_cf_unit in "$lio_config/resources.names" "$lio_config"/*.luns "$lio_config"/.*.luns "$lio_config/.luns"; do
                    lio_cf_group=-; lio_cf_unit_bad=0; lio_cf_rows=; lio_cf_name_rows=; lio_cf_row_count=0
                    if [ "$lio_cf_unit" = "$lio_config/resources.names" ]; then
                        lio_cf_kinds=names
                    else
                        [ -e "$lio_cf_unit" ] || [ -L "$lio_cf_unit" ] || continue
                        lio_cf_group=${lio_cf_unit##*/}; lio_cf_group=${lio_cf_group%.luns}; lio_cf_kinds='target luns'; lio_cf_keys=' '; lio_cf_ids=$lio_nl; lio_cf_attrs=; lio_cf_iqn=-; lio_cf_tpg=-; lio_cf_portal=-
                        case "$lio_cf_group" in
                            ''|.|..|*[!A-Za-z0-9_.-]*)
                                lio_cf_unit_bad=1; lio_cfg_errors=$((lio_cfg_errors + 1))
                                if printf '<11>lio-targetd.executor: 配置组名称非法；PID=%s 文件=%s\n' "$$" "$lio_cf_unit" >&2; then :; else exit $?; fi ;;
                        esac
                    fi
                    for lio_cf_kind in $lio_cf_kinds; do
                        case "$lio_cf_kind" in
                            names) lio_cf_file=$lio_cf_unit ;;
                            target) lio_cf_file=${lio_cf_unit%.luns}.target ;;
                            luns) lio_cf_file=$lio_cf_unit ;;
                        esac
                        lio_path=$lio_cf_file; lio_cf_bad=0; lio_cf_problem=; lio_cf_no=0; lio_cf_bytes=0; if [ ! -f "$lio_cf_file" ] || [ ! -r "$lio_cf_file" ]; then
                            lio_cf_problem='文件缺失、不是普通文件或不可读'
                        elif lio_cf_size=$(stat -c '%s' "$lio_cf_file" 8>&- 9>&-); then
                            if {
                                while :; do
                                    lio_cf_line=
                                    if IFS= read -r lio_cf_line <&5; then lio_cf_rc=0; else lio_cf_rc=$?; fi
                                    case "$lio_cf_rc" in 0|1) ;; *) lio_cf_problem="读取失败，rc=$lio_cf_rc"; break ;; esac
                                    [ "$lio_cf_rc" -eq 0 ] || [ -n "$lio_cf_line" ] || break
                                    lio_cf_no=$((lio_cf_no + 1)); lio_cf_bytes=$((lio_cf_bytes + ${#lio_cf_line}))
                                    if [ "$lio_cf_rc" -eq 0 ]; then lio_cf_bytes=$((lio_cf_bytes + 1)); fi
                                    lio_cf_text=$lio_cf_line
                                    while :; do
                                        case "$lio_cf_text" in ' '*|"$lio_tab"*) lio_cf_text=${lio_cf_text#?} ;; *) break ;; esac
                                    done
                                    case "$lio_cf_text" in ''|\#*) continue ;; esac
                                    lio_cf_reason=
                                    if [ "$lio_cf_kind" = target ]; then
                                        lio_cf_key=${lio_cf_line%%=*}; lio_cf_value=${lio_cf_line#*=}
                                        case "$lio_cf_key" in
                                            IQN_PREFIX|TPG|PORTAL|ATTR_generate_node_acls|ATTR_cache_dynamic_acls|ATTR_authentication|ATTR_demo_mode_discovery|ATTR_demo_mode_write_protect|ATTR_tpg_enabled_sendtargets) ;;
                                            *) lio_cf_reason='未知键或键值行格式非法' ;;
                                        esac
                                        case "$lio_cf_line" in *=*) ;; *) lio_cf_reason='键值行缺少等号' ;; esac
                                        case "$lio_cf_keys" in *" $lio_cf_key "*) lio_cf_reason='配置键重复' ;; esac
                                        if [ -z "$lio_cf_reason" ]; then
                                            lio_cf_keys="$lio_cf_keys$lio_cf_key "
                                            case "$lio_cf_key" in
                                                IQN_PREFIX)
                                                    case "$lio_cf_value" in iqn.?*) ;; *) lio_cf_reason='IQN_PREFIX 格式非法' ;; esac
                                                    case "$lio_cf_value" in *[[:space:][:cntrl:]/]*|*\"*|*\'*) lio_cf_reason='IQN_PREFIX 含非法字符' ;; esac
                                                    lio_cf_iqn=$lio_cf_value ;;
                                                TPG)
                                                    [ "$lio_cf_value" = 1 ] || lio_cf_reason='本阶段仅支持 TPG=1'
                                                    lio_cf_tpg=$lio_cf_value ;;
                                                PORTAL)
                                                    lio_cf_portal=$lio_cf_value; lio_cf_ip=${lio_cf_value%:*}; lio_cf_port=${lio_cf_value##*:}
                                                    case "$lio_cf_value" in *:*) ;; *) lio_cf_reason='PORTAL 缺少端口' ;; esac
                                                    case "$lio_cf_ip" in ''|.*|*.|*..*|*[!0-9.]*) lio_cf_reason='PORTAL IPv4 格式非法' ;; esac
                                                    case "$lio_cf_port" in ''|*[!0-9]*) lio_cf_reason='PORTAL 端口格式非法' ;; esac
                                                    if [ -z "$lio_cf_reason" ]; then
                                                        lio_cf_ifs=$IFS; IFS=.; set -- $lio_cf_ip; IFS=$lio_cf_ifs
                                                        [ "$#" -eq 4 ] || lio_cf_reason='PORTAL IPv4 必须有四段'
                                                        lio_cf_part=0
                                                        for lio_cf_num in "$@" "$lio_cf_port"; do
                                                            lio_cf_part=$((lio_cf_part + 1)); lio_cf_decimal=${lio_cf_num#"${lio_cf_num%%[!0]*}"}; lio_cf_decimal=${lio_cf_decimal:-0}
                                                            if [ "$lio_cf_part" -le 4 ]; then
                                                                if [ "${#lio_cf_decimal}" -gt 3 ] || [ "$lio_cf_decimal" -gt 255 ]; then lio_cf_reason='PORTAL IPv4 数值越界'; fi
                                                            elif [ "${#lio_cf_decimal}" -gt 5 ] || [ "$lio_cf_decimal" -eq 0 ] || [ "$lio_cf_decimal" -gt 65535 ]; then
                                                                lio_cf_reason='PORTAL 端口数值越界'
                                                            fi
                                                        done
                                                    fi ;;
                                                ATTR_*)
                                                    case "$lio_cf_value" in 0|1) ;; *) lio_cf_reason='TPG 属性必须为 0 或 1' ;; esac
                                                    lio_cf_attrs="$lio_cf_attrs $lio_cf_key=$lio_cf_value" ;;
                                            esac
                                        fi
                                    else
                                        lio_cf_ifs=$IFS; IFS=" $lio_tab"; set -f
                                        set -- $lio_cf_line
                                        set +f; IFS=$lio_cf_ifs
                                        if [ "$lio_cf_kind" = names ]; then
                                            if [ "$#" -ne 1 ]; then lio_cf_reason='名称表每行必须恰好一列'; fi
                                            lio_cf_dm=${1-}
                                        else
                                            if [ "$#" -ne 4 ]; then lio_cf_reason='LUN 策略每行必须恰好四列'; fi
                                            lio_cf_dm=${2-}
                                        fi
                                        case "$lio_cf_dm" in ''|*[[:space:][:cntrl:]/]*|*\"*|*\'*) lio_cf_reason='DM_NAME 为空或含非法字符' ;; esac
                                        if [ -z "$lio_cf_reason" ] && [ "$lio_cf_kind" = names ]; then
                                            case "$lio_cfg_names" in
                                                *"$lio_nl$lio_cf_dm$lio_nl"*) lio_cf_reason='名称表项目重复' ;;
                                                *) lio_cfg_names="$lio_cfg_names$lio_cf_dm$lio_nl"; lio_cfg_name_count=$((lio_cfg_name_count + 1))
                                                    lio_cf_name_rows="$lio_cf_name_rows$lio_cf_dm$lio_tab$lio_cf_no$lio_nl" ;;
                                            esac
                                        elif [ -z "$lio_cf_reason" ]; then
                                            case "$1" in 0) ;; ''|0*|*[!0-9]*) lio_cf_reason='LUN_ID 必须为无前导零的非负整数' ;; esac
                                            case "$3" in ro|rw) ;; *) lio_cf_reason='ACCESS 必须为 ro 或 rw' ;; esac
                                            case "$4" in on|off) ;; *) lio_cf_reason='ADMIN_STATE 必须为 on 或 off' ;; esac
                                            case "$lio_cf_ids" in *"$lio_nl$1$lio_nl"*) lio_cf_reason='同组 LUN_ID 重复' ;; esac
                                            if [ -z "$lio_cf_reason" ]; then
                                                lio_cf_ids="$lio_cf_ids$1$lio_nl"; lio_cf_rows="$lio_cf_rows$2$lio_tab$lio_cf_group$lio_tab$lio_cf_iqn:$lio_cf_group$lio_tab$lio_cf_tpg$lio_tab$1$lio_tab$3$lio_tab$4$lio_tab$lio_cf_portal$lio_tab$lio_cf_attrs$lio_tab$lio_cf_file$lio_tab$lio_cf_no$lio_nl"; lio_cf_row_count=$((lio_cf_row_count + 1))
                                            fi
                                        fi
                                    fi
                                    if [ -n "$lio_cf_reason" ]; then
                                        lio_cf_bad=1; lio_cfg_errors=$((lio_cfg_errors + 1)); lio_detail='写入配置错误日志失败'
                                        if printf '<11>lio-targetd.executor: 配置错误；PID=%s 组=%s 文件=%s 行=%s 原因=%s\n' "$$" "$lio_cf_group" "$lio_cf_file" "$lio_cf_no" "$lio_cf_reason" >&2; then :; else exit $?; fi
                                    fi
                                done
                            } 5<"$lio_cf_file"; then :; else
                                lio_cf_problem="打开或读取配置失败，rc=$?"
                            fi
                            if [ "$lio_cf_bytes" != "$lio_cf_size" ]; then lio_cf_problem="读取字节数不符，读取=$lio_cf_bytes 预期=$lio_cf_size"; fi
                        else
                            lio_cf_problem="读取配置大小失败，rc=$?"
                        fi
                        if [ "$lio_cf_kind" = target ]; then
                            for lio_cf_key in IQN_PREFIX TPG PORTAL ATTR_generate_node_acls ATTR_cache_dynamic_acls ATTR_authentication ATTR_demo_mode_discovery ATTR_demo_mode_write_protect ATTR_tpg_enabled_sendtargets; do
                                case "$lio_cf_keys" in *" $lio_cf_key "*) ;; *) lio_cf_problem="$lio_cf_problem 缺少键:$lio_cf_key" ;; esac
                            done
                        fi
                        if [ -n "$lio_cf_problem" ]; then
                            lio_cf_bad=1; lio_cfg_errors=$((lio_cfg_errors + 1)); lio_detail='写入配置读取错误日志失败'
                            if printf '<11>lio-targetd.executor: 配置读取失败；PID=%s 组=%s 文件=%s 行=%s 原因=%s\n' "$$" "$lio_cf_group" "$lio_cf_file" "$lio_cf_no" "$lio_cf_problem" >&2; then :; else exit $?; fi
                        fi
                        if [ "$lio_cf_bad" -ne 0 ]; then lio_cf_unit_bad=1; fi
                    done
                    if [ "$lio_cf_kinds" = names ]; then
                        if [ "$lio_cf_unit_bad" -ne 0 ]; then lio_cfg_names_ok=0; lio_cfg_complete=0; else
                            lio_detail='写入周期名称明细失败'
                            while IFS="$lio_tab" read -r lio_sn_dm lio_sn_line; do
                                [ -n "$lio_sn_dm" ] || continue
                                if printf '<14>lio-targetd.executor: 周期名称；PID=%s DM_NAME=%s 文件=%s 配置行=%s\n' "$$" "$lio_sn_dm" "$lio_cf_unit" "$lio_sn_line" >&2; then :; else exit $?; fi
                            done <<LIO_NAME_ROWS
$lio_cf_name_rows
LIO_NAME_ROWS
                        fi
                    elif [ "$lio_cf_unit_bad" -ne 0 ]; then
                        lio_cfg_bad=$((lio_cfg_bad + 1)); lio_cfg_complete=0
                    else
                        lio_cfg_good=$((lio_cfg_good + 1)); lio_cfg_count=$((lio_cfg_count + lio_cf_row_count)); lio_cfg_policies="$lio_cfg_policies$lio_cf_rows"; lio_detail='写入目标解析日志失败'
                        if printf '<14>lio-targetd.executor: 目标配置已解析，未应用；PID=%s NAME=%s IQN=%s:%s TPG=%s PORTAL=%s 属性=%s 文件=%s\n' "$$" "$lio_cf_group" "$lio_cf_iqn" "$lio_cf_group" "$lio_cf_tpg" "$lio_cf_portal" "$lio_cf_attrs" "${lio_cf_unit%.luns}.target" >&2; then :; else exit $?; fi
                        lio_detail='写入周期策略明细失败'
                        while IFS="$lio_tab" read -r lio_sn_dm lio_sn_group lio_sn_iqn lio_sn_tpg lio_sn_lun lio_sn_access lio_sn_state lio_sn_portal lio_sn_attrs lio_sn_file lio_sn_line; do
                            [ -n "$lio_sn_dm" ] || continue
                            if printf '<14>lio-targetd.executor: 周期策略；PID=%s NAME=%s DM_NAME=%s LUN_ID=%s ACCESS=%s ADMIN_STATE=%s 文件=%s 配置行=%s\n' "$$" "$lio_sn_group" "$lio_sn_dm" "$lio_sn_lun" "$lio_sn_access" "$lio_sn_state" "$lio_sn_file" "$lio_sn_line" >&2; then :; else exit $?; fi
                        done <<LIO_GROUP_ROWS
$lio_cf_rows
LIO_GROUP_ROWS
                    fi
                done
                for lio_cf_file in "$lio_config"/*.target "$lio_config"/.*.target "$lio_config/.target"; do
                    [ -e "$lio_cf_file" ] || [ -L "$lio_cf_file" ] || continue
                    if [ ! -e "${lio_cf_file%.target}.luns" ] && [ ! -L "${lio_cf_file%.target}.luns" ]; then
                        lio_detail='写入未使用 target 日志失败'; if printf '<14>lio-targetd.executor: target 无同名 luns，未使用且未解析；PID=%s 文件=%s\n' "$$" "$lio_cf_file" >&2; then :; else exit $?; fi
                    fi
                done
                if [ "$lio_cfg_errors" -ne 0 ]; then lio_result=1; lio_anomalies=$((lio_anomalies + lio_cfg_errors)); fi
                lio_detail='读取配置完成时间失败'; if IFS=' ' read -r lio_uptime lio_idle <"$lio_clock"; then :; else exit $?; fi
                lio_detail='写入周期配置摘要失败'; if printf '<14>lio-targetd.executor: 周期配置读取完成；PID=%s 名称=%s 名单有效=%s 有效组=%s 无效组=%s 策略=%s 视图完整=%s 错误=%s 后续批次复用=1 UPTIME=%s\n' "$$" "$lio_cfg_name_count" "$lio_cfg_names_ok" "$lio_cfg_good" "$lio_cfg_bad" "$lio_cfg_count" "$lio_cfg_complete" "$lio_cfg_errors" "$lio_uptime" >&2; then :; else exit $?; fi
                lio_path=$lio_b
            fi
            if [ "$lio_pass" = log ]; then
                lio_stage=policy-summary; lio_detail='业务分类计数不符'; [ "$((lio_denied + lio_unknown + lio_config_failed + lio_no_policy + lio_matched))" -eq "$lio_valid" ] || exit 1
                lio_detail='第三阶段计数不符'; [ "$((lio_i_known + lio_i_unknown + lio_i_error))" -eq "$lio_valid" ] && [ "$((lio_c_new + lio_c_reuse + lio_c_keep + lio_c_error + lio_c_skip))" -eq "$lio_valid" ] && [ "$((lio_l_ok + lio_l_mismatch + lio_l_error + lio_l_skip))" -eq "$lio_valid" ] || exit 1
                if printf '<14>lio-targetd.executor: 第三阶段摘要；PID=%s 轮次=%s 处理记录=%s 失败记录=%s 缓存=%s 身份已知=%s 身份未知=%s 身份错误=%s 缓存建立=%s 缓存复用=%s 缓存保留=%s 缓存失败=%s 缓存跳过=%s 现场通过=%s 现场不符=%s 现场未确认=%s 现场跳过=%s LIO=未执行\n' "$$" "$lio_round" "$lio_valid" "$lio_stage3_bad" "$lio_cache_dir" "$lio_i_known" "$lio_i_unknown" "$lio_i_error" "$lio_c_new" "$lio_c_reuse" "$lio_c_keep" "$lio_c_error" "$lio_c_skip" "$lio_l_ok" "$lio_l_mismatch" "$lio_l_error" "$lio_l_skip" >&2; then :; else exit $?; fi
                lio_detail='写入业务摘要失败'; if printf '<14>lio-targetd.executor: 业务摘要；PID=%s 轮次=%s 事件=%s 未准入=%s 身份未确认=%s 配置失败=%s 无策略=%s 已匹配=%s on候选=%s off条目=%s 复用周期视图=1 视图完整=%s\n'  "$$" "$lio_round" "$lio_valid" "$lio_denied" "$lio_unknown" "$lio_config_failed" "$lio_no_policy" "$lio_matched" "$lio_on" "$lio_off" "$lio_cfg_complete" >&2; then :; else exit $?; fi
            fi
        done
        lio_stage=remove-B; lio_detail='删除已完成的 B 失败'; if rm "$lio_b" 8>&- 9>&-; then
            lio_batch_state=none; lio_events=$((lio_events + lio_valid))
        else
            exit $?
        fi
    else
        lio_stage=empty-round; lio_path=-; lio_detail='写入空轮日志失败'; if printf '<14>lio-targetd.executor: 消费 R 空轮；PID=%s 轮次=%s\n'  "$$" "$lio_round" >&2; then :; else exit $?; fi
    fi
    lio_stage=debounce-log; lio_path=-; lio_detail='读取滤波开始时间失败'; if IFS=' ' read -r lio_uptime lio_idle <"$lio_clock"; then :; else exit $?; fi
    lio_detail='写入固定滤波日志失败'; if printf '<14>lio-targetd.executor: 固定滤波开始；PID=%s 轮次=%s 秒=%s UPTIME=%s\n'  "$$" "$lio_round" "$lio_seconds" "$lio_uptime" >&2; then :; else exit $?; fi
    lio_stage=debounce; lio_detail='固定滤波等待失败或被中断'; if sleep "$lio_seconds" 8>&- 9>&-; then :; else exit $?; fi
    lio_stage=debounce-end; lio_detail='读取滤波结束时间失败'; if IFS=' ' read -r lio_uptime lio_idle <"$lio_clock"; then :; else exit $?; fi
    lio_detail='写入滤波结束日志失败'; if printf '<14>lio-targetd.executor: 固定滤波结束；PID=%s 轮次=%s UPTIME=%s\n' "$$" "$lio_round" "$lio_uptime" >&2; then :; else exit $?; fi
done
```
<!-- END SOURCE: lio-targetd -->

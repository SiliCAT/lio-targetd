**文件写入也失败了**，仍报 `No usable execution environment is available`，未能生成可下载文件。以下为完整文档正文。

# uci lio测试

版本：v0.1  
日期：2026-10-20  
状态：配置设计及三轮 OpenWrt 实机 UCI 实验完成；尚未适配 lio-targetd 主线脚本。

## 1．目的、基准与边界

本次工作以《lio-targetd 四阶段设计与实现换页文档》为基准。第四阶段暂停，先调整用户配置的表达方式，独立验证 UCI 读写能力，之后再与主线汇合。

本页附件因执行环境不可用而未能读取。本文引用的旧配置约定，来自用户直接贴出的第 9.2～9.7 节、第 10 节；实验结果来自用户回传的 OpenWrt 实机输出。本文不能替代尚未读取的主线全文。

本次完成：

1. 设计 `/etc/config/lio`，承接原 `.target`、`.luns`、`settings.conf` 表达的用户意图。
2. 将资源开放过滤清单的位置改为 `/etc/config/resources.names`。
3. 将清单路径配置为 `dev.filter`，完整 UCI 路径为 `lio.dev.filter`。
4. 完成读取、暂存、撤销、提交、匿名节增删重排，以及解析边界实验。
5. 明确字段重复赋值采用 UCI 原生行为，应用继续检查业务唯一性。

本轮没有修改主线脚本，没有调用服务或 configfs，没有验证新配置驱动实际 LIO 发布的能力。

## 2．配置文件责任与结构

| 文件 | 责任 | 读取方式 |
|---|---|---|
| `/etc/config/lio` | 用户配置：防抖、清单位置、target 与 LUN 策略 | UCI |
| `/etc/config/resources.names` | 独立维护的资源开放过滤清单 | 普通文本清单解析 |

清单与 target/LUN 配置不相互生成。迁移清单位置不改变其文本格式。

`lio` 使用两个固定命名节，以及若干匿名业务节：

| 节类型 | 节名 | 用途 |
|---|---|---|
| `settings` | `settings` | 防抖参数 |
| `device` | `dev` | 资源过滤清单位置 |
| `target` | 匿名 | 一个 target 配置组 |
| `lun` | 匿名 | 一条 LUN 策略 |

例如：

```uci
config device 'dev'
        option filter '/etc/config/resources.names'
```

对应：

```sh
uci get lio.dev.filter
```

`lio` 为包名，`dev` 为节名，`filter` 为选项名。`dev.filter` 不作为包含点号的单个 option 名。

## 3．完整实验配置

### 3.1 `/etc/config/lio`

```uci
config settings 'settings'
        option debounce_ms '2000'

config device 'dev'
        option filter '/etc/config/resources.names'

config target
        option name 'experiment.runtime'
        option iqn_prefix 'iqn.2026-10.local.pogov4'
        option tpg '1'
        option portal '127.0.0.1:3260'
        option generate_node_acls '1'
        option cache_dynamic_acls '1'
        option authentication '0'
        option demo_mode_discovery '1'
        option demo_mode_write_protect '0'
        option tpg_enabled_sendtargets '1'

config lun
        option target 'experiment.runtime'
        option lun_id '0'
        option dm_name 'vg2-BULKTEST00'
        option access 'rw'
        option admin_state 'on'

config lun
        option target 'experiment.runtime'
        option lun_id '1'
        option dm_name 'vg2-BULKTEST01'
        option access 'rw'
        option admin_state 'off'

config lun
        option target 'experiment.runtime'
        option lun_id '2'
        option dm_name 'vg2-BULKTEST02'
        option access 'rw'
        option admin_state 'on'

config lun
        option target 'experiment.runtime'
        option lun_id '3'
        option dm_name 'vg2-LIODEBUG_MISSING'
        option access 'rw'
        option admin_state 'on'
```

### 3.2 `/etc/config/resources.names`

```text
# 用户独立维护的资源名称表，每行一个准确 DM_NAME。
vg2-BULKTEST00
vg2-BULKTEST01
vg2-BULKTEST03
vg2-LIODEBUG_MISSING
```

### 3.3 实验策略含义

| DM_NAME | 清单准入 | 策略 | 原有业务含义 |
|---|---|---|---|
| `vg2-BULKTEST00` | 是 | LUN 0，rw，on | 已准入，on 候选 |
| `vg2-BULKTEST01` | 是 | LUN 1，rw，off | 已准入，期望未发布 |
| `vg2-BULKTEST02` | 否 | LUN 2，rw，on | 有策略但未准入 |
| `vg2-BULKTEST03` | 是 | 无 | 已准入但无策略 |
| `vg2-LIODEBUG_MISSING` | 是 | LUN 3，rw，on | 有配置，但当前没有真实事件 |

当前不主动发现 MISSING 设备。SWAP、USBROOT 等未列入清单的 DM 继续不准入。

未来主线正确消费该配置时，预期业务摘要为：

```text
名称 4、有效组 1、策略 4、完整视图
```

本轮未运行主线业务解析器，因此该摘要是迁移后的验收预期，不是本轮已经取得的应用层结果。

实验 portal 为 `127.0.0.1:3260`，与此前手工网络实验使用的 `172.16.16.220:32690` 不同。

## 4．字段映射与业务身份

### 4.1 字段映射

| 原配置 | 新表达 |
|---|---|
| `settings.conf`：`DEBOUNCE_MS` | `lio.settings.debounce_ms` |
| 原清单固定位置 | `lio.dev.filter` 指定路径 |
| `NAME.target`、`NAME.luns` 中的 NAME | target 的 `name`；lun 的 `target` |
| `IQN_PREFIX` | `iqn_prefix` |
| `TPG` | `tpg` |
| `PORTAL` | `portal` |
| 六个 `ATTR_属性名` | 对应属性名，去掉 `ATTR_` 前缀 |
| `LUN_ID` | `lun_id` |
| `DM_NAME` | `dm_name` |
| `ACCESS` | `access` |
| `ADMIN_STATE` | `admin_state` |

### 4.2 采用匿名业务节的原因

原组名允许点号、横线，例如：

```text
experiment.runtime
wims.bulk-test
```

因此将业务组名保存在 option 的值中，避免将它直接作为 UCI 节名。

身份约定如下：

| 对象 | 业务身份 |
|---|---|
| target | `name` |
| LUN | `target`＋`lun_id` |
| lun 与 target 的关联 | lun 的 `target` 准确匹配 target 的 `name` |
| 完整 IQN | `iqn_prefix:NAME` |

本例完整 IQN：

```text
iqn.2026-10.local.pogov4:experiment.runtime
```

`@lun[n]` 表示当前按类型排列的访问位置，不是 LUN_ID。`cfg...` 是 UCI 节标识，不作为业务身份或外部持久关联键。

一设备多条策略合法；跨 target 使用相同 LUN 编号合法。

## 5．继续保留的业务校验

以下属于配置设计，尚待主线实现。UCI 能读取配置，不等于这些规则已经得到应用层验证。

### 5.1 固定节

UCI 解析后的配置应存在：

- 名为 `settings`、类型为 `settings` 的节。
- 名为 `dev`、类型为 `device` 的节。

`debounce_ms` 必填，保持原数值规则：

- 无前导零的正十进制整数。
- 必须为 1000 的整数倍。
- 2000 对应 `sleep 2`。
- 1500 明确报不支持，不取整、不回退默认值。

`filter` 必填，指定清单绝对路径。缺失或空值不隐式回退到固定路径。

迁移后采用 UCI 自身的引号语法，旧 `KEY=value` 格式的“无引号”限制取消；数值内容和值域约束保留。

### 5.2 target

`name` 沿用原 NAME 规则：允许字母、数字、下划线、点、横线；拒绝空值及单独的 `.`、`..`。

被使用的 target 除 `name` 外，九项必填：

| 字段 | 规则 |
|---|---|
| `iqn_prefix` | 非空，`iqn.` 开头，无空白、控制字符、斜杠和引号 |
| `tpg` | 当前只支持 1 |
| `portal` | 一个 IPv4:端口；地址各段 0～255，端口 1～65535 |
| `generate_node_acls` | 0/1 |
| `cache_dynamic_acls` | 0/1 |
| `authentication` | 0/1 |
| `demo_mode_discovery` | 0/1 |
| `demo_mode_write_protect` | 0/1 |
| `tpg_enabled_sendtargets` | 0/1 |

上述接受范围不代表全部组合已验证。初版后续需求仍固定无认证和 rw。

`prod_mode_write_protect` 不在本次配置字段中。

### 5.3 lun

以下五项必填：

```text
target lun_id dm_name access admin_state
```

规则：

- `lun_id` 为 0 或无前导零正整数，同组唯一。
- `dm_name` 为准确设备名称。
- `access` 沿用当前解析范围 `ro/rw`。
- `admin_state` 为 `on/off`。
- 同组任一策略损坏，使整组无效，不接纳剩余部分。

未知字段、缺失字段及非法业务值仍由应用校验。

旧内存表中的 `CONFIG_FILE`、`CONFIG_LINE` 如何映射到 UCI 来源诊断，本轮尚未完成设计，不能宣称已经对齐。

### 5.4 名称清单

清单格式保持原约定：

- 每有效行一个准确 DM_NAME。
- 支持空白行和整行注释。
- 不支持通配、行尾注释、shell 展开。
- 拒绝重复名称，以及名称中的空白、控制字符、斜杠和引号。
- 空清单合法。
- 文件缺失、不可读、内容非法，不得解释为空清单。

## 6．配置组发现与原有业务边界

为延续原来“由 `.luns` 发现配置组”的语义，新设计按 lun 的 `target` 值收集被使用的组，与 target 的 `name` 准确关联：

1. 每个被使用组必须准确对应一个 target。
2. 缺少对应 target，该组无效。
3. 同名 target 重复，该组无效，不选取其中一个。
4. 未被任何 lun 引用的 target 记录为未使用，不解析其余策略字段。
5. 成功读取且没有 lun 节，可以表示合法空策略集合。
6. 业务无效组不妨碍记录其他有效组结果，但整个周期保持配置不完整。
7. lun 损坏到无法确认所属组时，也保留总体失败标记。

文件读取或语法解析失败不能解释为空策略集合。整包语法错误的边界见第 9 节。

本次配置迁移不改变以下既有边界：

- 先关联事件身份，再做配置判断。
- 名单未准入不新建。
- off 表示期望未发布。
- remove 匹配 on 时仅关联历史策略，不发布、不增加 on 候选数。
- 未准入、无策略、off 均不自动授权删除旧对象。
- 编辑清单本身不等于提交统一重查，也不等于旧发布已经撤销。

本轮不新增 reload、重新扫描、主动设备发现或删除授权。

## 7．第一轮：注入与基础读写

实机记录目录：

```text
/tmp/lio-uci-test.ILCjaF
```

### 7.1 首次执行与修正

开始时 `/etc/config/lio` 尚不存在。

```sh
command -v uci
```

返回：

```text
/sbin/uci
```

说明 UCI 已安装，无需安装操作。

最初脚本直接执行：

```sh
lio_pending=$(uci -q changes lio)
```

因配置尚不存在，非零返回触发 `set -e`，在注入文件之前退出。

随后修订为：先判断 `/etc/config/lio` 是否存在，存在时才检查未提交修改。修订后完整实验成功。

### 7.2 配置读取

注入两份配置后，`uci show lio` 成功显示两个固定节、一个匿名 target、四个匿名 lun。

| 命令 | 返回值 |
|---|---|
| `uci get lio.settings.debounce_ms` | `2000` |
| `uci get lio.dev.filter` | `/etc/config/resources.names` |
| `uci get 'lio.@target[0].name'` | `experiment.runtime` |
| `uci get 'lio.@target[0].iqn_prefix'` | `iqn.2026-10.local.pogov4` |
| `uci get 'lio.@target[0].portal'` | `127.0.0.1:3260` |
| `uci get 'lio.@lun[3].dm_name'` | `vg2-LIODEBUG_MISSING` |

通过以下方式成功读取普通清单：

```sh
lio_filter=$(uci get lio.dev.filter)
cat "$lio_filter"
```

这验证了路径配置和文件读取，尚未验证清单业务解析器的迁移。

### 7.3 暂存与撤销

执行：

```sh
uci set lio.settings.debounce_ms=3000
uci set 'lio.@lun[1].admin_state=on'
uci changes lio
```

结果：

- 新 UCI 读取立即返回 `3000`、`on`。
- 磁盘文件仍为 `2000`、`off`。
- 暂存修改中的匿名节表示为：

```text
lio.cfg058dd4.admin_state='on'
```

执行：

```sh
uci revert lio
```

之后读取恢复为 `2000`、`off`，未提交修改为空。

这证明后续 UCI CLI 读取可以看到尚未 commit 的修改。未来应用如何处理暂存配置，以及如何固定周期内配置视图，本轮尚未决定。

### 7.4 提交与恢复

将防抖值改为 3000 并提交：

```sh
uci set lio.settings.debounce_ms=3000
uci commit lio
```

磁盘文件确实变为 3000。

随后恢复：

```sh
uci set lio.settings.debounce_ms=2000
uci commit lio
```

初始与最终 `uci export lio` 的结果经过 `cmp` 比较一致，最终未提交修改为空。

此处验证的是 **UCI 导出配置一致**，不承诺原文件字节、空行、缩进或注释完全不变。实机输出显示 commit 会重写排版。

## 8．第二轮：解析边界与清单共存

实机记录目录：

```text
/tmp/lio-uci-boundary.cGeAHf
```

样本均放在独立临时目录，通过：

```sh
uci -c "$lio_test_dir" show "$lio_case"
```

记录标准输出、错误输出和退出码。

### 8.1 重复 option

样本：

```uci
config settings 'settings'
        option debounce_ms '2000'
        option debounce_ms '3000'
```

结果：

```text
退出码：0
duplicate_option.settings=settings
duplicate_option.settings.debounce_ms='3000'
```

无错误输出。该样本中后值生效，原文重复信息没有保留在解析结果中。

### 8.2 重复命名节

样本：

```uci
config settings 'settings'
        option debounce_ms '2000'

config settings 'settings'
        option debounce_ms '3000'
```

结果：

```text
退出码：0
duplicate_section.settings=settings
duplicate_section.settings.debounce_ms='3000'
```

无错误输出。

该样本最终只呈现一个 settings 节，值为 3000。未据此推断不同节类型冲突或不同字段组合的全部行为。

### 8.3 重复业务组名

样本：

```uci
config target
        option name 'experiment.runtime'
        option portal '127.0.0.1:3260'

config target
        option name 'experiment.runtime'
        option portal '127.0.0.1:3261'
```

结果退出码 0，两份匿名 target 均保留。

因此应用能够枚举两个节，并检查 `name` 的业务唯一性。UCI 本身不替应用完成该检查。

### 8.4 非法业务值与未知选项

样本：

```uci
config settings 'settings'
        option debounce_ms '1500'
        option unknown_key 'example'
```

结果退出码 0，两个 option 均保留，无错误输出。

因此防抖值域、必填项、未知字段等规则仍需应用校验。

### 8.5 前半部合法、后半部语法损坏

样本先声明合法 settings，再写入未闭合的引号：

```uci
config settings 'settings'
        option debounce_ms '2000'

config target
        option name 'experiment.runtime
```

结果：

```text
退出码：1
标准输出为空
```

错误：

```text
uci: Parse error (EOF with unterminated ') at line 5, byte 40
```

本次 `uci show` 没有返回前半部有效配置。

### 8.6 普通清单与 UCI 包同目录

在独立 `coexist/` 目录放入：

- 合法 UCI 文件 `lio`。
- 普通文本清单 `resources.names`。

执行不指定包名的枚举：

```sh
uci -c "$lio_test_dir/coexist" show
```

结果退出码 0，仅输出：

```text
lio.dev=device
lio.dev.filter='/etc/config/resources.names'
```

错误输出为空。

本次实证：**该实机的不指定包名 show 枚举，没有被同目录的普通清单干扰。**

未测试显式将 `resources.names` 当作 UCI 包加载，也未测试其他系统组件的目录扫描行为。

## 9．实验后确定的规则

### 9.1 字段重复与业务重复分别处理

旧配置的重复拒绝包含两种性质：

| 对象 | 原规则 |
|---|---|
| `settings.conf` | 重复字段键报错 |
| 单份 `.target` | 重复字段键使整组无效 |
| 同组 `.luns` | LUN_ID 必须唯一 |
| 名称清单 | 准确 DM_NAME 必须唯一 |
| 多条策略中的 DM_NAME | 允许重复 |

经本轮讨论，确定采用：

> 字段重复赋值遵循 UCI 原生行为，由应用检查业务唯一键重复；不额外增加原文重复键检查来保留旧文本格式的限制。

具体处理：

| 情况 | 新原则 |
|---|---|
| 同一节重复 option | 使用 UCI 解析结果；本次实测后值生效 |
| 重复同名固定节 | 校验 UCI 解析后的对象，不追溯原文重复声明 |
| target 的 `name` 重复 | 应用拒绝该业务组 |
| 同组 `lun_id` 重复 | 应用拒绝该组 |
| 清单 DM_NAME 重复 | 清单解析器拒绝 |
| 多条策略使用同一 DM_NAME | 允许 |
| 不同 target 使用相同 LUN_ID | 允许 |

固定节仍需检查解析后是否存在、类型是否正确、字段是否完整和值是否合法。

因此，前面实验过程中记录的“是否需要增加原文重复键检查”，在本轮讨论结束时已经收敛，不再作为待定方案。

### 9.2 业务错误与语法错误

对于成功解析的 UCI 配置，应用可以继续按组检查业务规则；一个业务无效组不必阻止记录其他有效组结果。

对于文件语法错误，本轮样本使 `uci show` 退出 1，并且没有输出合法前半部。消费程序必须检查退出状态，不能将失败后的空输出解释为合法空策略集合。

原来的多文件配置中，“坏组不妨碍读取其他组”的能力，不能直接推广到单个 UCI 文件的语法损坏。

这一差异需要在主线汇合时承接；本轮没有设计容错恢复解析器或新的部分发布策略。

## 10．第三轮：匿名节新增、重排、提交与删除

实机记录目录：

```text
/tmp/lio-uci-edit.ajFHFN
```

实验包名：

```text
lio_probe_5547
```

从实际 `/etc/config/lio` 复制出独立实验包，后续操作针对副本。

### 10.1 新增匿名节

使用 `uci add` 新增 lun，返回：

```text
cfg088dd4
```

设置：

```text
target=experiment.runtime
lun_id=99
dm_name=vg2-UCI_PROBE
access=rw
admin_state=off
```

新增后，其位置为：

```text
@lun[4]
```

读取 `lun_id` 返回 99。

### 10.2 重排

将新增节移至整个配置包的位置 0。

重排后：

| 访问位置 | lun_id |
|---|---|
| `@lun[0]` | 99 |
| `@lun[1]` | 0 |
| `@lun[2]` | 1 |
| `@lun[3]` | 2 |
| `@lun[4]` | 3 |

实机输出同时显示，lun 可以排在 settings、dev、target 之前。

因此未来应用必须按字段建立关联，不能依赖 target 必须先于 lun 声明。

### 10.3 提交后重新读取

commit 后，由新的 UCI 进程读取：

```text
99
vg2-UCI_PROBE
```

说明重排和字段内容已成功保存。

### 10.4 删除与恢复

删除当前 `@lun[0]` 并 commit。

最终导出与初始导出经过 `cmp` 比较一致，未提交修改为空。

已验证匿名节新增、字段设置、重排、提交后重读、删除，以及恢复初始配置。

### 10.5 已验证的接口形态

以下为已执行脚本中的接口摘录，依赖脚本中的实验目录和包名变量：

```sh
lio_sid=$(uci -c "$lio_test_dir" add "$lio_package" lun)

uci -c "$lio_test_dir" set \
    "$lio_package.$lio_sid.lun_id=99"

uci -c "$lio_test_dir" reorder \
    "$lio_package.$lio_sid=0"

uci -c "$lio_test_dir" commit "$lio_package"

uci -c "$lio_test_dir" get \
    "$lio_package.@lun[0].lun_id"

uci -c "$lio_test_dir" delete \
    "$lio_package.@lun[0]"

uci -c "$lio_test_dir" commit "$lio_package"
```

这些不是脱离实验上下文的独立执行脚本。

## 11．实验结束状态与主线汇合清单

根据用户回传结果：

- 实际 `/etc/config/lio` 保持初始实验配置。
- `debounce_ms=2000`。
- LUN 1 的 `admin_state=off`。
- 第一轮结束时无未提交修改。
- 实际名称清单保持四个准确名称。
- 第二、三轮使用独立实验样本或副本。
- 旧 `/etc/lio-targetd/` 配置没有在本轮被迁移程序修改或停用。

| 汇合项目 | 状态 |
|---|---|
| 新配置布局、完整实验样本 | 已设计并注入 |
| 固定节与匿名节读取 | 已实测 |
| 点号、横线、IQN、portal、路径保存 | 已实测 |
| 暂存、撤销、提交、恢复 | 已实测 |
| 匿名节新增、重排、删除 | 已实测 |
| 提交后由新进程读取 | 已实测 |
| 重复项、业务非法值、引号损坏 | 已实测上述样本 |
| 普通清单与全局 show 枚举共存 | 已实测上述样本 |
| 原文重复键拒绝 | 已决定改为 UCI 原生处理 |
| 业务字段校验、业务唯一性 | 已有设计，尚未接入主线 |
| 原事件、缓存、准入、匹配链路消费新配置 | 尚未修改和验收 |
| CONFIG_FILE、CONFIG_LINE 来源诊断迁移 | 尚未完成设计 |
| 暂存配置消费策略、周期内一致视图 | 尚未决定 |
| 单文件语法错误的周期失败处理 | 已发现差异，待主线承接 |
| 编辑后自动重查、reload、停用旧文件 | 本轮未实现 |
| 第四阶段 LIO 执行 | 继续暂停 |

本轮三个现场记录目录均位于 `/tmp`，不能假定重启后仍存在：

```text
/tmp/lio-uci-test.ILCjaF
/tmp/lio-uci-boundary.cGeAHf
/tmp/lio-uci-edit.ajFHFN
```

后续汇合应以本文记录的设计决定和实验事实为依据，不将 UCI 基础读写成功等同于 lio-targetd 主线适配完成。

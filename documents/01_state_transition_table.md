# CiA402 状态转移表

**基准实现**：`cia402device/lib/cia402device/cia402device.c`

**ALstatus 状态码映射**

ALstatus 反映 EtherCAT AL 状态机的当前状态，值来自 ESC 的 AL Status 寄存器 0x0130，由协议栈写入后经 `axis->ALstatus` 指针读入。

| 值 | AL 状态 | 含义 |
|---|---|---|
| 0x01 | INIT | 从站已识别，邮箱未配置 |
| 0x02 | Pre-Operational | 邮箱（SDO/CoE）可用，过程数据未交换 |
| 0x03 | Bootstrap | 可选的固件更新状态 |
| 0x04 | Safe-Operational | 输入 PDO 正常上传，输出 PDO 仍被禁止 |
| 0x08 | Operational | 输入输出 PDO 全部周期交换，进入正常工作状态 |
| 0x10 | Error 指示位 | 与上述值按位组合，表示状态切换出错 |

> 本状态机仅对 ALstatus 做 `== 0x08` 与 `!= 0x08` 两种比较；
> Error 位置位时一律视为非 Operational，例如 0x18 即 Operational 叠加 Error。

## 1. 概述

状态机入口 `cia402_state_machine(axis, controlword)`，每个通信周期调用一次：

- **输入**
  - `controlword`：对象 0x6040，主站周期下发的控制字
  - `*axis->ALstatus`：EtherCAT AL 状态码，取值见文首映射表，状态机仅判定其是否等于 0x08
- **输出**（全部每周期刷新）
  - `*axis->statusword`：对象 0x6041，每周期按**结果状态**整字重写
  - `axis->state`：次态
  - `axis->transition`：本周期迁移号，无迁移时为 `NO_TRANSITION(0)`
  - `axis->flags`：应用动作标志，按结果状态刷新
  - `axis->prevflags`：调用前的 flags 快照，供应用做动作沿检测

T13（`DRIVE_ERROR`）不经控制字触发：应用检测到错误后直接写 `axis->state = FAULT_REACTION_ACTIVE`，状态机随后自动经 T14 进入 FAULT。

## 2. 输入编码表（控制字命令）

判定方式：`(controlword & MASK) == COMMAND`，宏为 `CIA402_CONTROLWORD_*_MASK / *_COMMAND`：

| 命令 | 掩码 | 匹配值 | 等效位条件 (0x6040) |
|---|---|---|---|
| SHUTDOWN | 0x0087 | 0x0006 | bit0=0, bit1=1, bit2=1, bit7=0 |
| SWITCH_ON | 0x0087 | 0x0007 | bit0=1, bit1=1, bit2=1, bit7=0 |
| DISABLE_OPERATION | 0x008F | 0x0007 | 同 SWITCH_ON 且 bit3=0 |
| ENABLE_OPERATION | 0x008F | 0x000F | bit0–3 全 1, bit7=0 |
| SWITCH_ON_ENABLE | 0x008F | 0x000F | 与 ENABLE_OPERATION 完全同码 |
| DISABLE_VOLTAGE | 0x0082 | 0x0000 | bit1=0 且 bit7=0 |
| QUICK_STOP | 0x0086 | 0x0002 | bit1=1, bit2=0, bit7=0 |
| FAULT_RESET | 0x0080 | 0x0080 | bit7=1 |

**编码重叠**：SWITCH_ON 与 DISABLE_OPERATION 同值 0x07，仅掩码差 bit3；SWITCH_ON_ENABLE 与 ENABLE_OPERATION 完全同码。同一控制字模式在不同状态下解释为不同命令。

## 3. 状态转移表

 - 每个状态内部按优先级自上而下排列，**首个命中生效**。
 - "自持"表示保持当前状态不变；优先级"2a"表示在优先级 2 命中后追加判定。

| 当前状态 | 优先级 | 触发条件 | 次态 | 迁移号 |
|---|---|---|---|---|
| **NOT_READY_TO_SWITCH_ON** (0x00) | 1 | `ALstatus == 0x08`（自动） | SWITCH_ON_DISABLED | **T1** |
| | — | 其他 | NOT_READY_TO_SWITCH_ON（自持） | — |
| **SWITCH_ON_DISABLED** (0x40) | 1 | SHUTDOWN **或** `ALstatus == 0x08` | READY_TO_SWITCH_ON | **T2** |
| | — | 其他 | SWITCH_ON_DISABLED（自持） | — |
| **READY_TO_SWITCH_ON** (0x21) | 1 | DISABLE_VOLTAGE | SWITCH_ON_DISABLED | **T7** |
| | 2 | SWITCH_ON | SWITCHED_ON | **T3** |
| | 2a | SWITCH_ON_ENABLE | OPERATION_ENABLED | **T34**（T3 与 T4 直达） |
| | — | 其他 | READY_TO_SWITCH_ON（自持） | — |
| **SWITCHED_ON** (0x23) | 1 | SHUTDOWN | READY_TO_SWITCH_ON | **T6** |
| | 2 | ENABLE_OPERATION | OPERATION_ENABLED | **T4** |
| | 3 | DISABLE_VOLTAGE | SWITCH_ON_DISABLED | **T10** |
| | — | 其他 | SWITCHED_ON（自持） | — |
| **OPERATION_ENABLED** (0x27) | 1 | DISABLE_OPERATION | SWITCHED_ON | **T5** |
| | 2 | SHUTDOWN | READY_TO_SWITCH_ON | **T8** |
| | 3 | DISABLE_VOLTAGE **或** `ALstatus != 0x08`（掉线） | SWITCH_ON_DISABLED | **T9** |
| | 4 | QUICK_STOP | QUICK_STOP_ACTIVE | **T11** |
| | — | 其他 | OPERATION_ENABLED（自持） | — |
| **QUICK_STOP_ACTIVE** (0x07) | 1 | DISABLE_VOLTAGE | SWITCH_ON_DISABLED | **T12** |
| | 2 | ENABLE_OPERATION | QUICK_STOP_ACTIVE（自持） | — |
| | — | 其他 | QUICK_STOP_ACTIVE（自持） | — |
| **FAULT_REACTION_ACTIVE** (0x0F) | 0 | 无条件（自动） | FAULT | **T14** |
| **FAULT** (0x08) | 1 | FAULT_RESET（电平触发） | SWITCH_ON_DISABLED | **T15** |
| | — | 其他 | FAULT（自持） | — |
| **非法状态值** | 0 | 无条件（自动） | NOT_READY_TO_SWITCH_ON | —（并清全部 flags） |

## 4. 状态输出表

每次调用先 `statusword = 0`、`prevflags = flags`，再按**结果状态**写 statusword、刷新 flags：

| 状态 | statusword | config_allowed | axis_func_enabled | hv_power_applied | brake_applied |
|---|---|---|---|---|---|
| NOT_READY_TO_SWITCH_ON | 0x0000 | 保持 | 保持 | 保持 | 保持 |
| SWITCH_ON_DISABLED | 0x0040 | 1 | 0 | 0 | 1 |
| READY_TO_SWITCH_ON | 0x0021 | 1 | 0 | 0 | 1 |
| SWITCHED_ON | 0x0023 | 1 | 0 | 1 | 1 |
| OPERATION_ENABLED | 0x0027 | 0 | 1 | 1 | 0 |
| QUICK_STOP_ACTIVE | 0x0007 | 0 | 1 | 1 | 0 |
| FAULT_REACTION_ACTIVE | 0x000F | 保持 | 保持 | 保持 | 保持 |
| FAULT | 0x0008 | 1 | 0 | 0 | 0 |

注意：NOT_READY_TO_SWITCH_ON 与 FAULT_REACTION_ACTIVE 不刷新 flags；非法状态值清零全部 flags。

## 5. 行为要点与标准偏差

**与 EtherCAT ALstatus 的三处耦合**：

1. ALstatus 达到 0x08 时自动发生 T1，从 NOT_READY_TO_SWITCH_ON 迁移到 SWITCH_ON_DISABLED
2. ALstatus 为 0x08 时自动发生 T2，从 SWITCH_ON_DISABLED 迁移到 READY_TO_SWITCH_ON
3. ALstatus 离开 0x08 即掉线时自动发生 T9，从 OPERATION_ENABLED 迁移到 SWITCH_ON_DISABLED

**与 CiA402 标准图（IEC 61800-7-2）的偏差**：

| 条目 | CiA402 标准 | 本实现 |
|---|---|---|
| READY_TO_SWITCH_ON 下的 QUICK_STOP | T7，到 SWITCH_ON_DISABLED | 自持 |
| SWITCHED_ON 下的 QUICK_STOP | T10，到 SWITCH_ON_DISABLED | 自持 |
| FAULT_RESET 触发方式 | 0 到 1 上升沿 | 电平触发 |
| T16：QUICK_STOP_ACTIVE 收到 ENABLE_OPERATION | 迁移到 OPERATION_ENABLED，标注"不建议支持" | 禁用，保持自持 |
| statusword bit4（电压使能） | 可选位，按驱动实际状态置位 | 恒为 0 |
| 控制字 HALT 位（0x0100） | 置位时驱动减速停止并保持当前状态 | 未实现 |

**边界行为**：

- 控制字在 NOT_READY_TO_SWITCH_ON 与 FAULT_REACTION_ACTIVE 状态下不参与判定；FAULT_REACTION_ACTIVE 同时忽略 ALstatus
- 控制字 bit8–15（含 HALT）在任何状态都不参与判定
- 控制字为 0x00 即 DISABLE_VOLTAGE，在所有检查该命令的状态生效
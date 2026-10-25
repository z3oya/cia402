# CiA402 标准状态转换表调研报告

## 0. 术语与约定

| 术语 | 定义 |
|---|---|
| CiA 402 | 运动控制设备行规，源自 CANopen 行规 DSP 402，现行版本 v4.x，由 CiA 组织维护 |
| IEC 61800-7-200/-201/-204 | 可调速电力驱动系统标准系列，收编 CiA 402 的行规规范与通信映射 |
| PDS | Power Drive System，行规文本对驱动设备的称呼 |
| 控制字 0x6040h | 主站周期下发给从站的命令字，本报告仅涉及其 bit0–3 与 bit7 |
| 状态字 0x6041h | 从站上报的状态字，本报告涉及其 bit0–6 与 bit9 |
| 0x605Ah / 0x605Bh / 0x605Ch / 0x605Eh | Quick stop / Shutdown / Disable operation / Fault reaction 选项码对象 |

状态机由 8 个状态与 1 个 Start 伪初始态组成，迁移编号 T0–T16 共 17 条。表格中的 x 表示该位取值与状态判定无关；bit4 电压已使能与 bit9 Remote 为可选位。典型值一列引用常见从站实现的上报值：Not ready to switch on 阶段电压尚未使能，取 0x0000；其余状态按 bit4=1、bit9=1。

## 1. 状态与状态字编码

| 状态 | bit6 | bit5 | bit4 | bit3..0 | 典型值 |
|---|---|---|---|---|---|
| Not ready to switch on | 0 | x | x | 0000 | 0x0000 |
| Switch on disabled | 1 | x | x | 0000 | 0x0250 |
| Ready to switch on | 0 | 1 | x | 0001 | 0x0231 |
| Switched on | 0 | 1 | x | 0011 | 0x0233 |
| Operation enabled | 0 | 1 | x | 0111 | 0x0237 |
| Quick stop active | 0 | 0 | x | 0111 | 0x0217 |
| Fault reaction active | 0 | x | x | 1111 | 0x021F |
| Fault | 0 | x | x | 1000 | 0x0218 |

bit5 在 Quick stop active 状态强制为 0，在 Ready to switch on、Switched on、Operation enabled 状态强制为 1，其余状态为无关位。

## 2. 状态迁移表 T0–T16

| 迁移 | 源状态 | 事件 | 动作 | 目标状态 |
|---|---|---|---|---|
| T0 | 上电或复位 | 自动 | 自检与驱动初始化 | Not ready to switch on |
| T1 | Not ready to switch on | 自动，初始化完成 | 通信可激活 | Switch on disabled |
| T2 | Switch on disabled | Shutdown 命令 | 无 | Ready to switch on |
| T3 | Ready to switch on | Switch on 命令 | 接通主电压，如可行 | Switched on |
| T4 | Switched on | Enable operation 命令 | 驱动功能使能，清除内部设定值 | Operation enabled |
| T5 | Operation enabled | Disable operation 命令 | 按 0x605Ch 执行后禁止驱动功能 | Switched on |
| T6 | Switched on | Shutdown 命令 | 主电压可断开 | Ready to switch on |
| T7 | Ready to switch on | Quick stop 或 Disable voltage | 无 | Switch on disabled |
| T8 | Operation enabled | Shutdown 命令 | 按 0x605Bh 执行，主电压可断开 | Ready to switch on |
| T9 | Operation enabled | Disable voltage | 禁止驱动功能，主电压可断开 | Switch on disabled |
| T10 | Switched on | Disable voltage 或 Quick stop | 禁止驱动功能，主电压可断开 | Switch on disabled |
| T11 | Operation enabled | Quick stop 命令 | 按 0x605Ah 执行快停 | Quick stop active |
| T12 | Quick stop active | 快停完成自动触发，或 Disable voltage 命令 | 禁止驱动功能，主电压可断开 | Switch on disabled |
| T13 | 任意状态 | 故障信号，来源为驱动自身 | 按 0x605Eh 执行故障反应 | Fault reaction active |
| T14 | Fault reaction active | 自动，故障反应完成 | 禁止驱动功能，主电压可断开 | Fault |
| T15 | Fault | Fault reset，bit7 由 0 变 1 | 复位激活的故障；主站随后须将 bit7 清零 | Switch on disabled |
| T16 | Quick stop active | Enable operation 命令，仅当 0x605Ah 取值 5–8 | 驱动功能重新使能 | Operation enabled |

## 3. 控制字命令编码

| 命令 | bit7 | bit3 | bit2 | bit1 | bit0 | 典型值 | 触发迁移 |
|---|---|---|---|---|---|---|---|
| Shutdown | x | x | 1 | 1 | 0 | 0x06 | 2, 6, 8 |
| Switch on | x | 0 | 1 | 1 | 1 | 0x07 | 3 |
| Switch on + Enable operation | x | 1 | 1 | 1 | 1 | 0x0F | 3+4 |
| Disable voltage | x | x | x | 0 | x | 0x00 | 7, 9, 10, 12 |
| Quick stop | x | x | 0 | 1 | x | 0x02 | 7, 10, 11 |
| Disable operation | x | 0 | 1 | 1 | 1 | 0x07 | 5 |
| Enable operation | x | 1 | 1 | 1 | 1 | 0x0F | 4, 16 |
| Fault reset | 0 变 1 | x | x | x | x | 0x80 | 15 |

七条状态命令的 bit7 列按转载来源记为 x；Fault reset 定义为 bit7 上升沿，各命令典型值因此均取 bit7=0。Switch on 与 Disable operation 共享典型值 0x07，区分依据为 bit3，二者分别作用于 Ready to switch on 与 Operation enabled 状态。

## 4. Quick Stop 选项码 0x605Ah

| 值 | 定义 |
|---|---|
| 0 | 立即禁止驱动功能 |
| 1 | 按慢减速斜坡减速，随后迁移至 Switch on disabled |
| 2 | 按快停斜坡减速，随后迁移至 Switch on disabled |
| 3 | 按电流限值减速，随后迁移至 Switch on disabled |
| 4 | 按电压限值减速，随后迁移至 Switch on disabled |
| 5 | 按慢减速斜坡减速，保持 Quick stop active |
| 6 | 按快停斜坡减速，保持 Quick stop active |
| 7 | 按电流限值减速，保持 Quick stop active |
| 8 | 按电压限值减速，保持 Quick stop active |

取值 1–4 时快停完成即自动触发 T12；取值 5–8 时驱动停留在 Quick stop active，T12 仅由 Disable voltage 命令触发，T16 因此可用。

## CHANGELOG

- 第 1 轮：检索标准表格来源；两次 PDF 直抓返回 404，改用带正文的搜索接口，获得多份厂商手册的表格全文。
- 第 2 轮：交叉核对 Festo CMMT、Technosoft iPOS、Unidrive M753、B&R EPSG 指令、ZLG 五个来源，内容一致后成稿。
- 第 3 轮：规范审计修正——修复 T9 目标状态笔误，论证性内容移入本节，消除正文否定式定义与论证性括号。
- 第 4 轮：按最新要求停用状态缩写，全文改用完整状态名；移除工程对照附录，报告限定为标准内容。
- 第 5 轮：正确性审查。核对 Festo Tab 1031/1032 与 CODESYS 命令编码表后修正四处：T2、T7 动作对齐来源记为“无”；T9、T10、T12、T14 动作措辞由“断开主电压”改为“主电压可断开”；状态命令 bit7 列由 0 改为 x，并改写对应说明句；典型值约定句补充 Not ready to switch on 取 0x0000 的例外。Festo 两版手册将 T16 动作写为 deactivate，与 Technosoft、Unidrive 及其自身目标状态定义冲突，本报告采信 enable。

## 来源

- [Festo CMMT-AS-S3 手册 2026-02a，Tab 1031–1033](https://www.festo.com/net/en-gb_gb/SupportPortal/Downloads/678957/862361/CMMT-AS-_-S3_manual_2026-02a_8249083g1.pdf)
- [Technosoft iPOS EtherCAT 用户手册 P091.064，表 4.1.2](https://technosoftmotion.com/wp-content/uploads/P091.064.EtherCAT.iPOS_.UM_.1123.pdf)
- [Unidrive M753 控制用户指南，表 9-39](https://www.apator.com/sites/default/files/2025-12/unidrive-m753-control-ug-issue-3.book-digitax-hd-m753-control-user-guide-3-eng.pdf)
- [B&R POWERLINK Implementation Directive for CiA402 EPSG V0.0.4，Table 5](https://no-cdn.br-automation.com/fileadmin/POWERLINK_Implementation_Directive_for_CiA402_EPSG_V_0_0_4-f3550a2c.pdf)
- [ZLG ZMTC-EE1050 数据手册，表 6.2](https://www.zlg.cn/data/upload/software/Zio/ZMTC-EE1050_ds.pdf)
- [SEW Quick stop option code 参数文档](https://www.sew-eurodrive.de/os/oh/ProductDataFolder/ProductMoviSuite/de-DE/26728582667.html)

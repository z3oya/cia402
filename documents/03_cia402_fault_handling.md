# CiA402 从机端故障处理设计调研报告

## 0. 术语与约定

| 术语 | 定义 |
|---|---|
| EMCY | Emergency 报文，节点在错误出现或清除时发出的诊断报文，CiA 301 §7.2.7 定义 |
| 错误寄存器 0x1001h | 8 位按类汇总当前激活错误的对象，所有 CANopen/CoE 设备必备 |
| 错误码 0x603Fh | CiA 402 对象，保存最近一次故障的 IEC 61800 紧急码 |
| 错误历史 0x1003h / 诊断历史 0x10F3h | 故障码历史对象；CANopen 用 0x1003h，EtherCAT 用 0x10F3h |
| 0x605Eh | Fault reaction 选项码，定义 T13 故障反应动作 |
| 0x6007h | Abort connection 选项码，IEC 61800-7 行规级对象，定义通信故障时的驱动反应 |
| 动态型 / 保持型故障 | 动态型在触发条件消失后自动清除；保持型须 fault reset 复位 |

状态与迁移编号沿用 `02_cia402_standard.md`，状态字编码见其第 1 节；0x605Ah 取值表见其第 4 节。

## 1. 故障在状态机中的路径

| 迁移 | 路径 | 触发 | 动作与对象联动 | 结果状态字 |
|---|---|---|---|---|
| T13 | 任意 → Fault reaction active | 驱动自身故障信号 | 按 0x605Eh 执行故障反应 | 0x021F |
| T14 | Fault reaction active → Fault | 自动，故障反应完成 | 禁止驱动功能；**置位 0x1001h 与 0x603Fh** | 0x0218 |
| T15 | Fault → Switch on disabled | 控制字 bit7 由 0 变 1 | 复位故障；**清零 0x1001h 与 0x603Fh**；发送错误码 0x0000 的 EMCY；主站随后须将 bit7 清零 | 0x0250 |

状态字 bit3 在 Fault reaction active 与 Fault 两个状态均为 1；bit7（Warning）置位表示存在警告级异常。

**0x605Eh Fault reaction 选项码**

| 值 | 动作 |
|---|---|
| 0 | 禁止驱动功能，自由停车 |
| 1 | 按慢减速斜坡减速后禁止 |
| 2 | 按快停斜坡减速后禁止（maxon 出厂默认） |

快停斜坡对象为 0x6085h（Nanotec 在 0x605Ah 语境明示）。短路、过流类关键故障无条件直接禁止驱动功能（maxon 手册标注为强制 disable 类）。

**复位约束**

- 故障源须已消除，且已无其他激活故障，状态机才离开 Fault（KEB、Technosoft）
- fault reset 仅在 Fault 状态被接受；短路、过流故障发生后 20 秒内拒绝复位（KEB）
- 动态型故障在条件消失后自动清除，保持型故障须 fault reset；状态字 bit3 反映保持型故障（KEB）
- 条件未消除时重复执行 fault reset 有损坏驱动风险（KEB 警告）

## 2. 故障源分类

**内部故障**的紧急码按 CiA 301 约定由高字节标示类别，低字节细化。各厂商实现高度一致：

| 高字节 | 类别 | 常见码例 |
|---|---|---|
| 0x00xx | 错误复位 / 无错误 | 0x0000 |
| 0x10xx | 通用错误 | 0x1000 |
| 0x2xxx | 电流 | 0x2210 / 0x2220 峰值过流，0x2310 / 0x2340 过流 |
| 0x3xxx | 电压 | 0x3210 过压，0x3220 欠压，0x3130 缺相 |
| 0x4xxx | 温度 | 0x4110 环境过温，0x4310 驱动过温 |
| 0x5xxx | 设备硬件 | 0x5210 校验告警 |
| 0x6xxx | 软件 / 参数 | 0x6310 / 0x6320 参数与校验错误 |
| 0x7xxx | 模块 / 反馈 | 0x7305 / 0x7306 / 0x7310 编码器断线，0x7380 位置传感器破损 |
| 0x8xxx | 通信 | 0x8110 总线过载，0x8130 心跳错误，0x8141 bus-off |
| 0xFFxx | 行规 / 厂商特定 | 0xFF00 起厂商自定义 |

跟随误差 0x8611 为 CiA 402 常见厂商码（maxon 示例），类别由厂商自定义。

**通信故障**由 0x6007h Abort connection 选项码统一映射到驱动反应，触发面覆盖总线掉线、watchdog / heartbeat 超时、SYNC 丢失、RxPDO 长度错误、NMT / AL 离开 operational。对 EtherCAT 从机即 SM watchdog 超时与 AL 离开 OP 的场景：

| 值 | 反应 |
|---|---|
| 0 | 无动作，保持当前工作点 |
| 1 | 执行 0x605Eh 配置的故障反应 |
| 2 | Disable voltage，直接迁移到 Switch on disabled |
| 3 | 执行 0x605Ah 配置的快停 |

厂商扩展：Nanotec 增加值 -1（RxPDO 全部清零，控制字归零，自由停车）；Control Techniques 增加 -1 / -2 / -3（先执行对应反应，停稳后再触发 trip 进入 Fault）。

## 3. 诊断与上报

| 通道 | 内容 |
|---|---|
| 状态字 0x6041h | bit3 故障（0x021F / 0x0218），bit7 警告 |
| 0x1001h 错误寄存器 | 按类汇总当前激活错误，位定义见下表 |
| 0x603Fh 错误码 | 最近一次故障码，随 fault reset 自动清零 |
| 历史对象 | CANopen 用 0x1003h 错误历史，EtherCAT 用 0x10F3h 诊断历史，后者的 subindex 0x06–0x0A 为带详情的完整诊断消息。故障复位只清当前码，历史保留 |
| EMCY | CANopen 以默认 COB-ID 0x80+Node-ID 发送；EtherCAT 经邮箱以 CoE emergency 服务发送，每个故障仅发一条。帧固定 8 字节：错误码（2，小端）+ 错误寄存器副本（1）+ 厂商特定错误域（5） |

**0x1001h 位定义**

| 位 | 值 | 含义 |
|---|---|---|
| 0 | 0x01 | 通用错误 |
| 1 | 0x02 | 电流错误 |
| 2 | 0x04 | 电压错误 |
| 3 | 0x08 | 温度错误 |
| 4 | 0x10 | 通信错误 |
| 5 | 0x20 | 行规特定错误 |
| 6 | 0x40 | 保留 |
| 7 | 0x80 | 厂商特定错误 |

错误清除时按 CiA 301 §7.2.7 发送错误码 0x0000 的 error reset EMCY。主站侧标准读法：0x1001h 与 0x6041h 用于发现异常，定位根因须读历史对象（maxon 支持文档）。TI 对 AM243x EtherCAT 从站的行规一致性测试将 0x1001h 与 0x603Fh 列为必选检查项。

## 4. 恢复时序

从机侧：进入 Fault → 接受控制字 bit7 上升沿 → 校验故障源已消除、无复位锁定、无其他激活故障 → 清零 0x603Fh 与 0x1001h、发送 0x0000 EMCY → 迁移到 Switch on disabled。

主站侧工程实践（ZLG 例程）：检测到 Fault 后延迟约 2 秒再发 fault reset，防止驱动在故障与恢复间快速振荡损坏功率电路；恢复出力后再延迟约 2 秒才开始加速，兼顾抱闸动作时间。

## 5. 本仓库现状对照

| 环节 | 标准设计 | 本仓库现状 |
|---|---|---|
| 状态机骨架 | T13 / T14 / T15 | 已实现并有单测；T15 电平触发偏差已记录于 `01_state_transition_table.md` 第 5 节 |
| 故障检测源 | 应用置位故障信号 | `DRIVE_ERROR` 置位代码缺失，`cia402device.h:106` 注释将其约定给应用层 |
| 反应配置 | 0x605Eh / 0x605Ah / 0x6007h | 三个选项码对象缺失，故障反应固定 |
| 诊断上报 | 0x1001h / 0x603Fh / 0x10F3h / EMCY | 0x1001h 已映射只读（`esi.json:37-45`），写入代码缺失；其余通道缺失 |
| 通信故障联动 | 按 0x6007h 配置反应 | 固定等效于 0x6007h=2：ALstatus 离开 0x08 经 T9 迁移到 Switch on disabled |
| 协议栈钩子 | — | SOES 的 `pre/post_state_change_hook` 与 `APP_safeoutput` 均为 NULL（`ecatapp.c:31-35`）；watchdog 超时走 `ESC_ALstatusgotoerror(SafeOP|Error, ALERR_WATCHDOG)`，CiA 402 层仅靠下一周期的 T9 间接触发 |

SOES 的错误路径钩子设计可追溯至 OpenEtherCATsociety/SOES issue #45：栈内错误刻意经 `ESC_ALstatusgotoerror` 联动 `ESC_stopoutput`。对齐标准的接线位置在 `APP_safeoutput` 与状态钩子：按 0x6007h 分派 T13 / T9 / T11；应用层故障经错误码表写 0x1001h 与 0x603Fh 并置位 `DRIVE_ERROR` 走 T13；fault reset 时清两个诊断对象。

## CHANGELOG

- 第 1 轮：联网检索标准与厂商手册，勘察本仓库现状，成稿调研报告。
- 第 2 轮：规范审计修正三处——0x1001h 与 0x603Fh 的置位时机由 T13 改为 T14（SIGMATEK 表为原始出处）；EMCY error reset（0x0000）补 CiA 301 §7.2.7 来源；复位约束按来源拆分归属，"仅 Fault 状态接受复位"与"20 秒锁定"标为 KEB 单一来源。
- 第 3 轮：文风审计——正文否定式表述改写为肯定式，覆盖关键故障强制禁止、0x8611 类别归属、历史对象映射三处；现状对照表措辞对齐。全量检索核实的验证性陈述由正文移入本条：0x603Fh、0x605Ah、0x605Eh、0x6007h、0x10F3h、EMCY 在代码与对象字典中零命中。

## 来源

- [SIGMATEK FDD-3000 用户指南（T13/T14/T15 与 0x1001/0x603F 联动表）](https://www.sigmatek-automation.com/fileadmin/user_upload/downloads/FDD-3000-SI-VARAN-User-Guide-eng.pdf)
- [KEB GM 电机手册（动态/保持型故障、复位约束、20 秒锁定）](https://data.keb.de/fileadmin/media/Manuals/gm/ma_gm_it-motorb_en.pdf)
- [Technosoft iPOS 手册（T15 复位条件）](https://technosoftmotion.com/wp-content/uploads/P091.063.CANopen.UM_.1224_ESM2.pdf)
- [maxon EPOS4/IDX EtherCAT：错误码对象与 0x10F3 诊断历史](https://support.maxongroup.com/hc/en-us/articles/4404761465106-EPOS4-IDX-EtherCAT-Which-object-holds-the-error-codes)
- [maxon 手册（0x605E 取值表、0x605A 取值表）](https://www.maxongroup.co.jp/medias/sys_master/8825413107742.pdf)
- [canopen-rs EMCY 模块文档（引 CiA 301 §7.2.7：EMCY 帧结构与 0x0000 error reset）](https://docs.rs/canopen-rs/latest/canopen_rs/emcy/)
- [Aventics CANopen 模块手册（EMCY 编码表、error reset 帧）](https://www.emerson.com/documents/automation/instruction-manual-bdc-b-design-canopen-canopen-sb-aventics-en-6897592.pdf)
- [ABB FECA-01 适配器手册（EMCY 邮箱发送、每故障一条）](https://library.e.abb.com/public/b972c703e2aa42cd80ffd7fc6822f434/EN_FECA_01_UM_F_A4.pdf)
- [SMAC LCC 技术说明（0x1001 位定义、错误历史保留性）](https://www.smac-mca.com/documents/controllers/TechNote-ResetError-LCC.pdf)
- [LS Electric S100 EtherCAT 手册（紧急码表）](https://sol.ls-electric.com/uploads/document/17191913742770/S100%20EtherCAT_User%20Manual_EN_V1.1.pdf)
- [Parker TPD 手册（紧急码实例表）](https://www.parker.com/content/dam/Parker-com/Literature/Electromechanical-Europe/User-Guides/192-141230_manTPD_GB.pdf)
- [Novanta Summit：0x6007 取值](https://drives.novantamotion.com/summit/0x6007-abort-connection-option-code)；[Nanotec NP5 EtherCAT 手册（0x6007 值 -1 及 EtherCAT 行为）](https://www.nanotec-cn.cn/zh/products/manual/m/n5-ethercat-en/p/object_dictionary%2Fod_motion_0x6007.html)；[Control Techniques SI-EtherCAT 手册（-1/-2/-3 扩展）](https://www.manualslib.com/manual/3557902/Control-Techniques-Nidec-Si-Ethercat.html?page=66)；[Bonfiglioli CM-EtherCAT 手册（0x6007 触发面）](https://www.manualzz.com/doc/o/g1gft/ethercat---cod.-vec1007-r1-device-profile-objects--0x6nnn-)
- [ZLG PCIe 主站例程（主站侧延迟复位工程实践）](https://zlg.cn/data/upload/software/Ethercatio/PCIe_MasterCard_IO_C&C++_MotorCtrlApp.pdf)
- [SOES issue #45（watchdog → 错误状态钩子设计）](https://github.com/OpenEtherCATsociety/SOES/issues/45)
- [TI AM243x EtherCAT 从站一致性测试报告（0x1001/0x603F 必选检查）](https://downloads.ti.com/processor-industrial-sw/esd/ind_comms_sdk/am243x/latest/docs/am243x/ethercat_subdevice/report-am243x-evm.pdf)

# 以太网交换机芯片行业综述：博通与Marvell芯片架构深度分析

> **文档目标**：从芯片架构师视角，系统分析Broadcom和Marvell的交换机芯片产品线、IP模块构成、架构设计理念，结合行业格局和技术趋势做全面综述
>
> **配套**：与《包处理流水线详解》《整体架构》《包的完整旅程》《补充篇》《小白入门指南》构成完整知识体系

---

## 目录

1. [行业格局综述](#一行业格局综述)
2. [交换机芯片总体架构功能点](#二交换机芯片总体架构功能点)
3. [Broadcom 芯片架构深度分析](#三broadcom-芯片架构深度分析)
4. [Marvell 芯片架构深度分析](#四marvell-芯片架构深度分析)
5. [Broadcom vs Marvell 架构对比分析](#五broadcom-vs-marvell-架构对比分析)
6. [各IP模块构成与细节逐项对比](#六各ip模块构成与细节逐项对比)
7. [技术趋势与未来展望](#七技术趋势与未来展望)
8. [选型指南与总结](#八选型指南与总结)

---

## 一、行业格局综述

### 1.1 市场概览

以太网交换机芯片市场 2024-2025 年规模约 120-130 亿美元，数据中心市场占比超 55% 且仍在扩大。

```
市场格局（2024年估算）：
                    ┌─────────────────┐
                    │   Broadcom      │  数据中心 ~70%
                    │   (全线覆盖)     │  企业 ~45%
                    └────────┬────────┘     运营商 ~40%
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────┴────┐         ┌────┴────┐         ┌────┴────┐
   │ Marvell │         │ NVIDIA  │         │ 其他     │
   │ ~15-20% │         │(Mellanox)│         │ Realtek  │
   │         │         │  ~8-10% │         │ MediaTek │
   │企业+数据 │         │ HPC/AI  │         │  ~8-10%  │
   └─────────┘         └─────────┘         └─────────┘
```

### 1.2 主要玩家定位

| 厂商 | 数据中心份额 | 核心产品线 | 最大容量 | 关键客户 |
|------|------------|-----------|---------|---------|
| **Broadcom** | 68-70% | Tomahawk 5, Trident 4, Jericho 3 | 51.2Tbps | Google, AWS, Meta, Microsoft |
| **Marvell** | 12-15% | Teralynx 10, Prestera 98DX | 51.2Tbps | AWS(二供), 企业OEM |
| **NVIDIA (Mellanox)** | 8-10% | Spectrum-4, Quantum-2 | 51.2Tbps | HPC/AI自用+外售 |
| **Intel (Tofino)** | 3-5% | Tofino 3 (已终止) | 25.6Tbps | P4可编程用户(已退场) |
| **Realtek/MediaTek** | <1% | 企业L2/L3交换 | 1.6Tbps | SMB/低端企业 |

### 1.3 重要行业事件

- **2021**：Marvell 收购 Innovium（$11亿），获得 Teralynx 数据中心芯片线
- **2021**：Marvell 收购 Inphi（$100亿），获得 PAM4 DSP 和 SerDes 能力
- **2023**：Intel 关闭 Tofino/P4 可编程交换芯片线，团队解散
- **2024**：Broadcom Tomahawk 5 (51.2T) 量产，Marvell Teralynx 10 (51.2T) 跟进
- **2025**：800G 端口批量部署元年，AI 集群驱动 51.2T 芯片需求爆发
- **2026-27(e)**：102.4T 芯片（Tomahawk 6, 3nm）预计流片/量产

---

## 二、交换机芯片总体架构功能点

### 2.1 通用架构框图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         以太网交换机芯片（通用架构）                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      SerDes (高速串行接口)                            │   │
│  │   112G PAM4 / 224G PAM4 / 56G PAM4 / 28G NRZ  × N lanes            │   │
│  │   TX: FIR预加重 | RX: CTLE + DFE + CDR | DSP: FFE/MLSD             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                 PCS (Physical Coding Sublayer)                       │   │
│  │   64B/66B编解码 | RS-FEC(544,514)/(528,514) | 去扰码 | Lane对齐     │   │
│  │   FC-FEC(低延迟) | AM插入/提取 | 链路状态管理                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                 MAC (Media Access Control)                           │   │
│  │   802.3帧收发 | VLAN标签处理 | PFC(802.1Qbb) | MACsec(802.1AE)     │   │
│  │   1588v2 PTP硬件时间戳 | EEE(802.3az) | MIB统计计数器               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  Packet Processor / Pipeline                         │   │
│  │                                                                       │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │   │
│  │  │Packet   │→│VLAN/ACL │→│L2/L3    │→│Load    │→│Edit/    │      │   │
│  │  │Parser   │ │Lookup   │ │Forward  │ │Balance  │ │Encapsulate│     │   │
│  │  │(协议解析)│ │(TCAM)   │ │(哈希/LPM)│ │(Smart  │ │(隧道/    │      │   │
│  │  │         │ │         │ │         │ │ Hash)  │ │修改)    │      │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │            Traffic Manager / MMU (Memory Management Unit)            │   │
│  │                                                                       │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │   │
│  │  │Packet    │→│Queue Mgmt│→│Congestion│→│Scheduler │            │   │
│  │  │Buffer    │  │(VoQ/WFQ) │  │Control   │  │(SP/DWRR) │            │   │
│  │  │(SRAM/HBM)│  │(32K+队列)│  │(ECN/WRED)│  │(Shaper)  │            │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │              Fabric Interface / Switch Matrix                        │   │
│  │   Crossbar | Clos | HiGig(BCM) | Cell-bus | Credit-based Flow Ctrl │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        CPU Subsystem                                │   │
│  │   ARM Cortex-A72/R5 | PCIe | DMA | 硬件学习引擎 | 中断聚合 | I2C/SPI │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 芯片分级（按应用场景）

```
                   功能丰富度 →
                   低 ←──────────→ 高
                   
带宽 高     ┌────────────────────────────────┐
     ↑      │  Tomahawk 5          Trident 5  │
     │      │  51.2T, 64×800G      25.6T      │
     │      │  极简流水线          深度流水线  │
     │      │  最小Buffer          大Buffer    │
     │      │  数据中心Spine       企业ToR    │
     │      └────────────────────────────────┘
     │      ┌────────────────────────────────┐
     │      │  Teralynx 8          Prestera  │
     │      │  25.6T, P4可编程     98DX系列   │
     │      │  大Buffer+HBM        2.56T      │
     │      │  数据中心           企业汇聚    │
     │      └────────────────────────────────┘
     │      ┌────────────────────────────────┐
     │      │  Trident 3/4         Jericho 3 │
     │      │  12.8T              7.2T       │
     │      │  企业+数据中心      运营商路由  │
     │      └────────────────────────────────┘
     ｜      ┌────────────────────────────────┐
     ｜      │  BCM53xxx            Qumran    │
     ｜      │  家用/SOHO          MPLS运营商 │
     ｜      └────────────────────────────────┘
    低
```

### 2.3 各芯片定位矩阵

| 维度 | Tomahawk | Trident | Jericho | Prestera | Teralynx |
|------|----------|---------|---------|----------|----------|
| **目标市场** | DC Spine | DC ToR/企业 | 运营商路由 | 企业接入/汇聚 | DC Spine/Leaf |
| **带宽密度** | 最高 | 高 | 中 | 中低 | 高 |
| **功能深度** | 浅 | 深 | 最深(可编程) | 深 | 深(可编程) |
| **Buffer大小** | 小(~32MB) | 中(~64MB) | 极大(8GB HBM) | 中(32-64MB) | 大(256-512MB HBM) |
| **表项容量** | 小 | 中 | 极大 | 中 | 大 |
| **功耗** | 高(~350W) | 中(~200W) | 中高(~300W) | 低(~30-75W) | 中高(~250W) |

---

## 三、Broadcom 芯片架构深度分析

### 3.1 产品线总览

Broadcom（博通）通过四大产品系列实现对以太网交换市场的全面覆盖：

```
Broadcom 交换芯片产品族
│
├── Tomahawk 系列（数据中心高密度）
│   ├── TH1 (3.2T, 28nm)     → 2014
│   ├── TH2 (6.4T, 28nm)     → 2016
│   ├── TH3 (12.8T, 16nm)    → 2017
│   ├── TH4 (25.6T, 7nm)     → 2019
│   └── TH5 (51.2T, 5nm)     → 2022 ← 当前旗舰
│       └── TH6 (102.4T, 3nm) → 2027(e)
│
├── Trident 系列（企业级/ToR）
│   ├── T2 (1.28T, 28nm)     → 2012
│   ├── T3 (3.2T, 16nm)      → 2015
│   ├── T4 (12.8T, 7nm)      → 2019
│   └── T5/Warren (25.6T, 5nm) → 2023
│
├── Jericho 系列（运营商路由）
│   ├── Jericho (480G, 28nm)       → 2015
│   ├── Jericho+ (1T, 28nm)        → 2017
│   ├── Jericho 2 (2.4T, 16nm)     → 2019
│   ├── Jericho 2C+ (3.6T, 16nm)  → 2021
│   └── Jericho 3 (7.2T, 7nm)      → 2022 ← 旗舰
│       └── + Ramon 3 Fabric芯片扩展
│
├── Qumran 系列（MPLS运营商）
│   ├── Qumran (200G, 28nm)
│   ├── Qumran MX (1.2T, 28nm)
│   └── Qumran 2C (2.4T, 16nm)
│
└── DNX 系列（分布式Fabric路由器架构）
    └── 用于构建超大规模路由系统
```

### 3.2 Tomahawk 5 深度架构拆解

#### 3.2.1 规格概述

| 参数 | 规格 |
|------|------|
| **代号** | Tomahawk 5 (BCM56880系列) |
| **总带宽** | 51.2 Tbps（全双工） |
| **工艺** | TSMC 5nm (N5) |
| **端口配置** | 64×800GE / 128×400GE / 256×200GE / 512×100GE |
| **SerDes** | 112G PAM4（主流）+ 224G PAM4（部分chiplets） |
| **封装** | 2.5D Chiplet架构（主Die + SerDes chiplets） |
| **包Buffer** | ~128MB 片上SRAM |
| **转发延迟** | Cut-through ~450ns |
| **功耗** | ~300-400W（典型） |
| **AI优化** | RDMA CNP聚合、自适应路由、FC-FEC |
| **主要客户** | Google(Jaguar v4), AWS, Meta |

#### 3.2.2 内部架构IP构成

```
Tomahawk 5 的 Chiplet 架构：
┌─────────────────────────────────────────────────────────────────────┐
│                 2.5D CoWoS 封装基板（Silicon Interposer）            │
│                                                                      │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐        │
│  │  I/O Die 0   │      │  Central    │      │  I/O Die 1   │        │
│  │  (SerDes)   │←──→│  Switch Die │←──→│  (SerDes)   │        │
│  │  112G×N     │ UCIe│  (核心逻辑)  │ UCIe│  224G×N     │        │
│  └─────────────┘  die-to-die └─────────────┘  die-to-die └─────────────┘
│                          互联                              │
│                                                                      │
│  ┌─────────────┐                                     ┌─────────────┐│
│  │  I/O Die 2   │                                     │  I/O Die 3   ││
│  │  (SerDes)   │                                     │  (SerDes)   ││
│  │  112G×N     │                                     │  112G×N     ││
│  └─────────────┘                                     └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘

Central Switch Die 内部流水线（简化）：
┌─────────────────────────────────────────────────────────────────────┐
│  Ingress方向：                                                       │
│                                                                      │
│  Die-to-Die SerDes(Rx)                                              │
│       │                                                              │
│       ▼                                                              │
│  [Packet Parser] — 固定解析器，支持Ethernet/IP/TCP/UDP/VXLAN       │
│       │                                                              │
│       ▼                                                              │
│  [VLAN处理] — VLAN成员检查、STP状态、PVID映射                       │
│       │                                                              │
│       ▼                                                              │
│  [L2/L3 Lookup] — MAC表(哈希)+路由表(ALPM)+ECMP哈希                │
│       │     主要用算法查找（非TCAM，省功耗）                         │
│       ▼                                                              │
│  [Load Balance] — Smart-Hash / Flowlet-based负载均衡                │
│       │                                                              │
│       ▼                                                              │
│  [Egress方向]                                                         │
│  Traffic Manager — VoQ + ECN标记 + WRED                              │
│       │  包Buffer: ~128MB 片上SRAM                                   │
│       ▼                                                              │
│  [Egress Edit] — MAC头改写、TTL更新、FCS重新计算                    │
│       │                                                              │
│       ▼                                                              │
│  Die-to-Die SerDes(Tx) → 到 I/O Die → PCS → SerDes → 物理线路     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 3.2.3 设计哲学

> **"带宽密度优先，功能精简"**  
> Tomahawk 是为超大规模数据中心Spine层设计的。核心理念是：去掉不必要的功能，把所有晶体管用在带宽和端口密度上。

- 流水线极浅（约5级），只做必须的转发决策
- 无复杂ACL、无MACsec、无MPLS、无深度QoS
- 小Buffer（128MB片上SRAM vs Jericho的8GB HBM）
- 依赖上层ECN/PFC做拥塞控制，而非本地大Buffer
- 结果：51.2T带宽在5nm工艺下做到~300-400W

### 3.3 Trident 4 深度架构拆解

#### 3.3.1 规格概述

| 参数 | 规格 |
|------|------|
| **代号** | Trident 4 (BCM56690/BCM56680系列) |
| **总带宽** | 12.8 Tbps |
| **工艺** | TSMC 7nm (N7) |
| **端口配置** | 32×400GE / 64×200GE / 128×100GE |
| **SerDes** | 56G PAM4 + 112G PAM4 混合 |
| **包Buffer** | ~64MB 片上SRAM |
| **CPU** | ARM Cortex-A72 四核@2.0GHz |
| **功耗** | ~150-220W |
| **隧道能力** | VXLAN路由+Bridging + GENEVE + NSH |
| **RoCEv2** | 全硬件卸载: ECN/PFC/DCQCN/CNP |

#### 3.3.2 内部流水线（与Tomahawk的关键差异）

```
Trident 4 流水线（深度~12级，对比Tomahawk~5级）：

Ingress Pipeline:
  SerDes → PCS → MAC
    ↓
  Packet Parser（半可编程：支持GENEVE/NSH等自定义协议）
    ↓
  VLAN/DSCP映射 → 内部优先级分配
    ↓
  ACL Stage A-B-C（级联TCAM，~8K条目）
    ↓
  VFP (Flexible Parser) — 用户自定义解析
    ↓
  L2 MAC表查找（哈希+链，256K条目）
    ↓
  L3 ALPM路由查找（128K-512K IPv4）
    ↓
  负载均衡哈希（多级对称哈希 + DFA）
    ↓
  Policer/Meter（三色令牌桶）
    ↓
Traffic Manager:
  VoQ → WRED/ECN → QoS Shaper → 片上包Buffer(64MB)
    ↓
Egress Pipeline:
  Egress ACL → 包编辑 → Mirror/SFlow → MAC → PCS → SerDes
```

**与Tomahawk的核心差异**：

| 特性 | Tomahawk 5 | Trident 4 |
|------|-----------|-----------|
| 流水线深度 | 5级（精简） | 12级（完整） |
| Parser | 固定（有限协议） | 半可编程（CFG） |
| ACL | ≤4K TCAM | 8K+ TCAM + VFP |
| Tunnel | VXLAN only | VXLAN/GENEVE/NSH/NVGRE |
| MACsec | 不支持 | 全线速支持 |
| 应用场景 | Spine（最大带宽） | ToR/汇聚（功能完整） |

### 3.4 Jericho 3 深度架构拆解

#### 3.4.1 规格概述

| 参数 | 规格 |
|------|------|
| **代号** | Jericho 3 (BCM88800系列) |
| **总带宽** | 7.2 Tbps + Fabric扩展 |
| **工艺** | TSMC 7nm (N7) |
| **端口配置** | 36×400GE / 72×200GE / 144×100GE |
| **包Buffer** | **HBM2e 4-32GB**（片外，极大Buffer） |
| **表项规模** | >4M IPv4路由 / >2M IPv6 / >1M MPLS标签 |
| **QoS** | 三级H-QoS，10K+队列 |
| **功耗** | ~250-350W |
| **NPU** | 完全可编程微码引擎 |

#### 3.4.2 架构特色

```
Jericho 3 的核心差异化：NPS (Network Processor Switch) 架构

┌─────────────────────────────────────────────────────────────────┐
│                     Jericho 3 + Ramon 3 Fabric                   │
│                                                                  │
│  ┌──────────┐              ┌──────────┐                        │
│  │ Jericho 3│───HiGig3───▶│ Ramon 3  │───HiGig3───▶ │其他Jericho│
│  │ (NPU+TM) │◀───────────│ (Fabric) │◀────────────│           │
│  └──────────┘              └──────────┘              └──────────┘
│       │                        │                                  │
│  ┌────┴────┐             ┌────┴────┐                            │
│  │HBM2e    │             │  HBM2e  │                            │
│  │4-32GB   │             │  2-4GB   │                            │
│  └─────────┘             └─────────┘                            │
│                                                                  │
│ 典型配置：4×Jericho 3 + 2×Ramon 3 = 28.8Tbps 路由系统           │
└─────────────────────────────────────────────────────────────────┘
```

**Jericho 3 独有的能力**：
1. **HBM片外Buffer**：4-32GB超大包缓存，处理运营商级的流量突发
2. **完全可编程NPU**：通过microcode编程parser和转发逻辑（不仅配表，还能定义查表流程）
3. **三级H-QoS**：端口级→组级→队列级，支持每用户/每VP的独立QoS
4. **运营商级OAM**：硬件BFD(3.3ms)、Y.1731、1588v2(±4ns)、SyncE
5. **分层Fabric**：通过Ramon 3 fabric芯片构建多NPU集群

### 3.5 Broadcom IP模块清单

#### 3.5.1 SerDes IP

| 属性 | 描述 |
|------|------|
| **演进** | NRZ 28G(TH3) → PAM4 56G(TH4) → PAM4 112G(TH5/主流) → PAM4 224G(TH5部分/TH6) |
| **TX** | FIR滤波器(3-5 tap预加重) |
| **RX** | CTLE + DFE(12-16 tap) + FFE |
| **功耗/lane** | 28G NRZ ~100mW → 112G PAM4 ~250-350mW → 224G PAM4 ~450-600mW |
| **FEC协同** | RS-FEC后BER<1E-15 |
| **集成** | TH5: 主Die+SerDes chiplets; TH4/Trident4: 单Die集成 |

#### 3.5.2 PCS IP

| 功能 | 标准 | 说明 |
|------|------|------|
| RS-FEC(544,514) | IEEE 802.3bs | 100G/200G/400G, "KP-FEC" |
| RS-FEC(528,514) | IEEE 802.3cl | 25G/50G |
| FC-FEC | TH5专有 | 端到端低延迟FEC，适合AI/HPC |
| MLSD | TH5 | 最大似然序列检测提升PAM4链路容限~2dB |
| 64B/66B编解码 | IEEE 802.3 | 基准PCS功能 |

#### 3.5.3 MAC IP

- IEEE 802.3 帧收发
- PFC(802.1Qbb)：8个优先级独立反压
- MACsec(802.1AE)：Trident/Jericho支持，Tomahawk不支持
- 1588v2 PTP硬件时间戳
- EEE(802.3az)节能以太网
- >200个RMON统计计数器/端口

#### 3.5.4 包处理流水线IP

| Stage | 功能 | 说明 |
|-------|------|------|
| Parser | 协议解析 | Trident/Jericho可编程，Tomahawk固定 |
| VLAN/ACL | VLAN检查+TCAM查找 | Trident 8K+, Jericho 64K+ |
| L2 Switch | MAC地址表查找 | 全局哈希表，32K~512K条目 |
| L3 Route | ALPM最长前缀匹配 | Tree Bitmap，128K~4M路由 |
| Load Balance | Smart-Hash/Flowlet | Broadcom专利动态负载均衡 |
| Edit | MAC头重写/隧道封装 | VXLAN/MPLS Push/Pop |

#### 3.5.5 Traffic Manager / MMU IP

- **VoQ架构**：消除HOL阻塞
- **Buffer**：Tomahawk 16-128MB SRAM / Jericho 512MB-8GB HBM
- **调度**：SP + DWRR + WFQ + Shafer(1kbps步进)
- **拥塞**：WRED + ECN标记 + DTM(动态流量管理专利)
- **队列**：最多128K队列

#### 3.5.6 Fabric Interface IP

| 协议 | 速率 | 用途 |
|------|------|------|
| HiGig | 10G | 最早芯片堆叠 |
| HiGig+ | 20G | 扩展带宽 |
| HiGig2 | 25G/50G | Trident/Jericho代 |
| **HiGig3** | 100G/200G lane | TH5/Trident5/Jericho3 |

HiGig是Broadcom私有片间互联协议，在以太网帧前附加2-8字节HiGig头，携带源端口/目的端口/优先级/拥塞标记等。这是Broadcom构建多芯片系统的核心能力。

#### 3.5.7 CPU子系统IP

- **CPU核心**：ARM Cortex-A72/R5/M7（视系列不同）
- **硬件学习引擎**：自动MAC地址学习，无需CPU
- **DMA**：批量表项读写
- **PCIe**：Gen3/Gen4 x4/x8，外接CPU

### 3.6 Broadcom独有技术

#### HiGig 芯片互联协议
私有片间互联，20年迭代家族。使多片Broadcom芯片可组成更大交换系统（如4×TH5 = 204.8T系统）。

#### DTM (Dynamic Traffic Management)
专利拥塞管理机制，通过VoQ深度直接反压，比传统ECN反馈更精确（微秒级 vs 毫秒级）。

#### Smart-Hash / Flowlet
基于流间隙的智能哈希，在ECMP组内动态切换路径，解决大流负载不均问题。

#### ALPM (Algorithmic LPM)
使用算法化TCAM（非传统TCAM），功耗比纯TCAM低90%，密度高4倍。

#### FlexPort
同一SerDes lane可在不同速率间动态重配置，支持1G~800G任意端口拆分。

---

## 四、Marvell 芯片架构深度分析

### 4.1 产品线总览

Marvell（美满电子）通过收购整合形成多产品线格局：

```
Marvell 交换/网络产品族
│
├── Prestera 系列（企业级交换，自研）
│   ├── 98DX42xx (16-128G, 12nm)     → 接入层
│   ├── 98DX65xx (512G-1.28T, 12nm) → 数据中心ToR
│   ├── 98DX35xx/Alleycat5 (1.28-2.56T, 12nm) → 企业汇聚
│   ├── 98DX82xx (1.28-2.56T, 12nm) → 高密度100G汇聚
│   └── Alleycat6 (下一代)            → 企业旗舰
│
├── Teralynx 系列（数据中心，来自Innovium收购 2021）
│   ├── Teralynx 7/77x0 (12.8T, 7nm) → 2021 ← 旗舰
│   ├── Teralynx 8/78x0 (25.6T, 5nm) → 2023
│   └── Teralynx 10 (51.2T, 3nm)     → 2025(e)
│
├── LinkStreet 系列（低端SoC，自研）
│   └── 集成CPU+Switch+PHY，<5W，SOHO/消费
│
├── XP70xx/LiquidIO 系列（网络处理器，来自Cavium收购 2017）
│   └── 多核ARM/MIPS + 硬件加速引擎，SmartNIC/5G
│
└── Alaska 系列（SerDes IP，来自Inphi收购 2020）
    └── Alaska Gen6: 112G PAM4 DSP，全线SerDes核心
```

### 4.2 Prestera 98DX 系列深度架构

#### 4.2.1 规格概述（以98DX35xx为例）

| 参数 | 规格 |
|------|------|
| **代号** | Alleycat5 (98DX35xx/36xx) |
| **工艺** | 12nm FinFET |
| **交换容量** | 1.28-2.56 Tbps |
| **端口** | 48×25GE + 8×100GE 混合 |
| **CPU** | 双核ARM Cortex-A72 @2GHz |
| **Buffer** | 32-64MB eSRAM |
| **TCAM** | 32K-64K |
| **路由表** | 128K-512K IPv4 |
| **延迟** | ~1.5-2μs(64B) |
| **功耗** | 30-75W |

#### 4.2.2 Pipe 流水线架构（Marvell特有）

```
Prestera 的 Pipe 架构：

┌──────────────────────────────────────────────────────────┐
│                   Pipe 0 (处理端口0-3)                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │ Ingress │→│ Local   │→│ Crossbar│→│ Egress  │    │
│  │ Pipe    │  │ Memory  │  │ (交换)  │  │ Pipe    │    │
│  │(查表)   │  │(表项)   │  │         │  │(调度+   │    │
│  └─────────┘  └─────────┘  └─────────┘  │ 编辑)   │    │
│                                          └─────────┘    │
│                   Pipe 1 (处理端口4-7)                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │ Ingress │→│ Local   │→│ Crossbar│→│ Egress  │    │
│  │ Pipe    │  │ Memory  │  │ (交换)  │  │ Pipe    │    │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │
│                   ... (32-48个Pipe)                       │
│                                                           │
│  Pipe 间通过 Crossbar/Mesh 互联                           │
│  每个Pipe独立查表，独立调度                                │
└──────────────────────────────────────────────────────────┘

Pipe内部：
  Ingress Pipe Stages:
    Parser → VLAN → ACL → L2/L3 → Tunnel → Meter → Mirror
  
  Egress Pipe Stages:
    QoS Queuing → Shaper → Edit → Mirror → MAC → PCS → SerDes
```

**Pipe架构的特点**：
1. **表项独立**：每个Pipe有自己的MAC表/路由表副本 → 查表无竞争
2. **线性扩展**：增加Pipe数直接增加转发带宽
3. **物理隔离**：Pipe间通过Crossbar互联，互相不影响
4. **局限**：大路由表需在所有Pipe间同步，表项利用率低于全局共享

#### 4.2.3 Finger 技术（Marvell特有）

Finger是Marvell用于流识别和哈希的核心技术：

```
Packet → 字段提取(L2/L3/L4) → 字段选择器(可配置) →
哈希生成器(CRC/Murmur) → Finger(128/256bit) →
    │
    ├→ ECMP/LAG负载均衡
    ├→ Flow Tracking (sFlow/NetFlow)
    ├→ ACL匹配
    └→ Flow Cache快速转发
```

**独特之处**：
- 可自定义参与哈希的字段（甚至VXLAN内层头）
- 首包做全字段哈希得到Finger，同Flow后续包复用
- Flow Cache + Finger实现第二包后的线速转发

### 4.3 Teralynx 系列深度架构

#### 4.3.1 规格概述

| 参数 | Teralynx 7 (77x0) | Teralynx 8 (78x0) | Teralynx 10 |
|------|-------------------|-------------------|-------------|
| **工艺** | 7nm | 5nm | 3nm(e) |
| **带宽** | 12.8T | 25.6T | 51.2T |
| **端口** | 32×400GE | 64×400GE/32×800GE | 64×800GE |
| **SerDes** | 56G PAM4 | 112G PAM4 Alaska G6 | 112G/224G |
| **Buffer** | 128MB HBM2E | 256-512MB HBM2E | HBM3 |
| **P4引擎** | TeraPump v2 | TeraPump v3 + AI | TeraPump v4 |
| **延迟** | ~550ns | ~500ns | ~450ns |
| **功耗** | ~150W | ~200-250W | ~250-350W |
| **可用** | 2021 | 2023 | 2025(e) |

#### 4.3.2 TeraPump 可编程流水线

```
Teralynx 的核心差异：完全可编程 Match-Action 流水线

P4程序 → Compiler(p4c-teralynx) → Pipeline Config → 加载到芯片

┌─────────────────────────────────────────────────────────────┐
│                      TeraPump 流水线                          │
│                                                               │
│  ┌────────────┐  ┌─────────────────┐  ┌──────────────┐      │
│  │ 可编程Parser│→│ Match-Action    │→│ 可编程Deparser│      │
│  │ (任意协议)  │  │ Stage 0..N     │  │ (头重组)     │      │
│  └────────────┘  │ (级联深度可配)   │  └──────────────┘      │
│                   │ ┌──────┐ ┌────┐ │                         │
│                   │ │Match │→│Action││ (每个Stage)           │
│                   │ │TCAM/  │ │计数/││                        │
│                   │ │ALPM/EM│ │改头 ││                        │
│                   │ └──────┘ └────┘ │                         │
│                   └─────────────────┘                         │
│                                                               │
│  特点：                                                       │
│  - Match-Action数量和顺序可配置（非固定）                      │
│  - 转发和ACL共享资源池（无固定分区浪费）                      │
│  - P4_16 + Teralang扩展                                       │
│  - 支持运行中重载(慢重配)                                     │
└─────────────────────────────────────────────────────────────┘
```

#### 4.3.3 FLB（Fine-grained Load Balancing）

Teralynx的负载均衡技术，逐流+逐包粒度：

- 在ECMP组内实时监控每条路径利用率
- 大象流自动分裂到多条路径
- 使用Finger技术追踪流状态
- 目标：链路利用率>90%

**对比Broadcom DLB**: Marvell FLB更激进（追求90%+利用率，可能引入少量乱序），Broadcom DLB更保守（追求无损TCP体验，利用率75-85%）。

### 4.4 Marvell 收购整合路径

```
Marvell 的收购整合：
                    ┌─────────────────┐
                    │    Marvell       │
                    │  (自研: Prestera  │
                    │   LinkStreet)    │
                    └──┬──┬──┬──┬────┘
                       │  │  │  │
       ┌───────────────┘  │  │  └────────────────┐
       │            ┌─────┘  └──────┐            │
       ▼            ▼               ▼            ▼
┌────────────┐ ┌──────────┐ ┌────────────┐ ┌──────────┐
│ Cavium     │ │ Innovium │ │   Inphi    │ │ 其他     │
│ (2017,$60亿)│ │(2021,$11亿)││(2020,$100亿)│ │           │
├────────────┤ ├──────────┤ ├────────────┤ ├──────────┤
│网络处理器   │ │ 数据中心  │ │PAM4 DSP + │ │ PHY公司   │
│OCTEON/    │ │ 交换芯片  │ │ SerDes IP  │ │           │
│ThunderX   │ │ Teralynx │ │ Alaska     │ │           │
│XP70xx     │ │          │ │ Colorado   │ │           │
└────────────┘ └──────────┘ └────────────┘ └──────────┘
```

收购后的整合效果：
- **Innovium** → Teralynx填补12.8T+数据中心空白，P4可编程能力
- **Inphi** → Alaska SerDes成为全线高速互联基础，使其在400G/800G领域具备竞争力
- **Cavium** → XP70xx/LiquidIO SmartNIC + OCTEON网络处理器 + 安全加速引擎

### 4.5 Marvell独有技术

#### Oman 端口扩展
允许单个Prestera芯片管理的逻辑端口扩展到大量物理端口（1:8~1:16 Fan-out），类似HiGig的轻量级版本，但更侧重于SerDes复用。

#### Pipe 流水线架构
模块化Pipe设计，每个Pipe独立处理一组端口，通过Crossbar互联。优点：物理隔离、线性扩展；缺点：大表需Pipe间同步。

#### Finger 流指纹
自定义字段选择器+多层次哈希链，用于流识别、负载均衡和Flow Cache加速。

---

## 五、Broadcom vs Marvell 架构对比分析

### 5.1 设计理念根本差异

| 维度 | Broadcom | Marvell |
|------|----------|---------|
| **哲学** | "固化优化"——固定流水线深度优化 | "开放可编程"——P4/Microcode可定义转发 |
| **生态策略** | 封闭自洽——统一SDKLT/SDK6 + HiGig | 开放兼容——SAI/P4/SONiC |
| **产品策略** | 多系列精准定位(TH/TD/JC/QM) | 收购整合(Prestera/Teralynx/XP70) |
| **可编程度** | 低（配表）到中（微码Jericho） | 中（Prestera）到高（Teralynx P4） |
| **Buffer哲学** | 小Buffer+智能调度(DLB/DTM) | 大Buffer+HBM+精确ECN |
| **互联能力** | HiGig生态（多芯片堆叠成熟） | 无统一互联（依赖标准以太网） |

### 5.2 产品矩阵对比

```
          带宽密度   →   功能丰富度
               低           高
              
51.2T    Broadcom: Tomahawk 5
          Marvell:  Teralynx 10
              
25.6T    Broadcom: Tomahawk 4 / Trident 5
          Marvell:  Teralynx 8
              
12.8T    Broadcom: Tomahawk 3 / Trident 4
          Marvell:  Teralynx 7 / Prestera 98DX
              
 2.56T   Broadcom: Trident 3
          Marvell:  Prestera 98DX35xx (Alleycat5)
              
  640G   Broadcom: BCM56860
          Marvell:  Prestera 98DX65xx
```

### 5.3 各场景竞争力

| 场景 | 首选 | 次选 | 理由 |
|------|------|------|------|
| **DC Spine (400G/800G)** | Broadcom TH5 | Marvell Teralynx 10 | TH5生态成熟、性能领先 |
| **DC ToR (25G/100G)** | Broadcom Trident4 | Marvell Prestera98DX | Trident功能完整+生态 |
| **企业汇聚** | Marvell Prestera | Broadcom Trident3 | Prestera集成度高、功耗低 |
| **运营商路由** | Broadcom Jericho3 | N/A | Jericho生态+大Buffer+OAM |
| **AI集群** | Broadcom TH5 | Marvell Teralynx 8 | TH5中标率高、AI优化 |
| **P4可编程** | Marvell Teralynx | N/A (Tofino已退) | 唯一大规模P4交换芯片 |
| **低端SoHo** | Realtek | Marvell LinkStreet | 成本敏感市场 |

---

## 六、各IP模块构成与细节逐项对比

### 6.1 SerDes IP 对比

| 对比项 | Broadcom | Marvell (Alaska) |
|--------|----------|-------------------|
| **命名** | Broadcom自研SerDes | Alaska Gen5/Gen6 (源自Inphi) |
| **112G PAM4** | TH5/Trident5 量产 | Teralynx 8 量产 |
| **224G PAM4** | TH5部分chiplets | Teralynx 10计划中 |
| **DSP架构** | 自研混合信号DSP | Inphi Colorado DSP |
| **功耗效率** | ~2.5pJ/bit @112G | ~2.0pJ/bit @112G (略优) |
| **链路预算** | ~35dB插损 @112G | ~40dB插损 @112G (略优) |
| **MLSD** | TH5支持 | 计划中 |
| **核心优势** | 集成度高，系列内统一 | 独立DSP，光互联经验 |

### 6.2 PCS/MAC IP 对比

| 对比项 | Broadcom | Marvell |
|--------|----------|---------|
| **RS-FEC** | 全系列支持 | 全系列支持 |
| **FC-FEC** | TH5专有 | Teralynx 8支持 |
| **MACsec** | Trident/Jericho支持 | Prestera支持，Teralynx可选 |
| **1588v2精度** | ±4ns (Jericho) | <8ns (Prestera) |
| **FlexE** | 有限支持 | Prestera支持(5G承载) |
| **PFC** | 8优先级全硬件 | 8优先级全硬件 |

### 6.3 包处理流水线IP 对比

| 对比项 | Broadcom Tomahawk | Broadcom Trident | Broadcom Jericho | Marvell Prestera | Marvell Teralynx |
|--------|------------------|-----------------|-----------------|-----------------|-----------------|
| **流水线深度** | ~5级 | ~12级 | ~20级+可编程 | ~20-30级 | 可配置(Match-Action) |
| **Parser** | 固定 | 半可编程 | 可编程microcode | 半可编程 | P4完全可编程 |
| **TCAM ACL** | ≤4K | 8K-64K | 64K-128K | 32K-64K | 资源池共享 |
| **路由表 IPv4** | 128K | 512K | 4M+ | 512K | 1M+ |
| **隧道封装** | VXLAN only | VXLAN/GENEVE/NSH | 全面 | VXLAN/MPLS | P4自定义 |
| **编辑能力** | 基本 | 中级 | 高级 | 中级 | P4自定义 |

### 6.4 Traffic Manager / MMU IP 对比

| 对比项 | Broadcom | Marvell Prestera | Marvell Teralynx |
|--------|----------|-----------------|-----------------|
| **Buffer容量** | 16-128MB(SRAM) / 8GB(HBM) | 12-64MB(SRAM) | 128-512MB(HBM) |
| **Buffer位置** | 片上SRAM / 片外HBM(Jericho) | 片上eSRAM | 片外HBM |
| **队列数** | 128K | 8K-64K | 256K+ |
| **调度层级** | 4级 | 4级 | 8级 |
| **ECN** | WRED+ECN+DTM | WRED+ECN | 精确ECN+PFC |
| **Shaper精度** | 1kbps步进 | 8kbps步进 | 1kbps步进 |
| **关键技术** | DTM专利/小Buffer哲学 | 动态门限 | 大Buffer哲学 |

### 6.5 Fabric / 芯片互联 IP 对比

| 对比项 | Broadcom (HiGig) | Marvell |
|--------|-----------------|---------|
| **互联协议** | HiGig3（私有，20年迭代） | 标准以太网 |
| **带宽** | 100G/200G per lane | 100G/200G/lane 标准 |
| **特性** | VoQ over Fabric, ECN传递 | 依赖标准ECN/PFC |
| **多芯片组网** | 极强（HiGig Stack/NPS） | 有限（堆叠能力弱） |
| **生态锁定** | 强（只能用BCM芯片互联） | 弱（开放） |

**关键差异**：HiGig是Broadcom的护城河之一。构建大型交换系统时，HiGig提供跨芯片的VoQ/ECN一致性，使多芯片系统像一个芯片一样工作。Marvell没有对应技术。

### 6.6 CPU子系统 IP 对比

| 对比项 | Broadcom | Marvell Prestera | Marvell Teralynx |
|--------|----------|-----------------|-----------------|
| **片上CPU** | A72/R5/M7 | A72/A53 | 外接x86/ARM |
| **CPU性能** | 中（低端A72） | 强（高端A72@2GHz） | N/A（依赖外接） |
| **可直接跑NOS** | 一般（需外接CPU） | 可（片上跑Linux/FRR） | 不可 |
| **学习引擎** | 硬件自动 | 硬件自动 | 硬件自动 |
| **SDK接口** | SDKLT/SAI | SDK/SAI | P4+Open SDK |

**Prestera的一个优势**：片上CPU可直接运行完整Linux + FRR(路由协议栈)，单芯片即可构成一台完整的L3交换机，无需外接CPU。Broadcom Trident通常需要外接CPU。

---

## 七、技术趋势与未来展望

### 7.1 交换容量演进

```
2014 ─ TH1 ─ 3.2T ─── 28nm
2017 ─ TH3 ─ 12.8T ── 16nm
2019 ─ TH4 ─ 25.6T ── 7nm
2022 ─ TH5 ─ 51.2T ── 5nm
2027 ─ TH6 ─ 102.4T ─ 3nm

每~2.5-3年翻倍，但102.4T将面临SerDes(224G)和功耗(>500W)双重挑战
```

### 7.2 AI/ML驱动的需求

AI训练集群对交换机芯片的**特殊要求**：
- **无损网络**：PFC/ECN/DCQCN硬件卸载是必选项
- **超大Buffer**：AI流量微突发需要大Buffer吸收
- **负载均衡**：传统ECMP在大象流下失效，需DLB/FLB等动态方案
- **AllReduce卸载**：未来芯片可能需要硬件加速集合通信

### 7.3 CPO与Chiplet

- **CPO（共封装光学）**：Broadcom Bailly方案已发布，2027-28年规模量产
- **Chiplet**：102.4T必然走向Chiplet，单Die超过800mm²良率无法承受
- **UCIe/BoW**：Die-to-Die互联标准统一中

### 7.4 可编程路线

Intel关闭Tofino后，P4全可编程路线遭受重大打击。目前局面：
- **Broadcom**：固定流水线+有限可编程CFG（"足够可编程"策略）
- **Marvell Teralynx**：唯一的全P4可编程大规模交换芯片
- **结论**：全可编程在数据中心场景被证明"过杀"，但在特定场景(新协议试验/定制网络)仍有价值

### 7.5 白牌+SONiC冲击

- 白牌交换机在数据中心占比从2018年15% → 2024年45%
- SONiC成为数据中心NOS事实标准（仅次于Cisco IOS）
- SAI接口降低芯片切换成本，有利于Marvell挑战Broadcom
- **但实际切换成本远非零**——Broadcom的SAI实现最稳定

---

## 八、选型指南与总结

### 8.1 选型决策树

```
你需要的交换机芯片？
│
├── 数据中心Spine（最高带宽密度）
│   ├── 生态优先 → Broadcom Tomahawk 5 (51.2T)
│   └── 需要P4可编程 → Marvell Teralynx 10 (51.2T)
│
├── 数据中心ToR（功能+带宽平衡）
│   ├── 主流选择 → Broadcom Trident 4/5 (12.8-25.6T)
│   └── 集成度高/片上CPU → Marvell Prestera 98DX65xx
│
├── 企业汇聚（功能丰富+低功耗）
│   └── Marvell Prestera 98DX35xx (1.28-2.56T)
│       （片上CPU可跑FRR，BOM成本低）
│
├── 运营商路由（大表项+大Buffer+OAM）
│   └── Broadcom Jericho 3 + Ramon Fabric
│       （HBM 4-32GB Buffer，>4M路由）
│
├── AI集群（无损+负载均衡）
│   ├── 主流 → Broadcom Tomahawk 5 (生态成熟)
│   └── P4定制 → Marvell Teralynx 8/10
│
└── SOHO/低端
    └── Realtek / Marvell LinkStreet / Broadcom BCM53xxx
```

### 8.2 总结

| 评价维度 | Broadcom | Marvell |
|----------|----------|---------|
| **市场份额** | 65-70% ✅ | 15-20% |
| **产品线宽度** | 最全（1G-800G全覆盖）✅ | 较全（有缺口） |
| **最高性能** | 51.2T (TH5) ✅ | 51.2T (Teralynx 10) |
| **生态成熟度** | 最强（SDK+HiGig+ODM）✅ | 较弱 |
| **可编程性** | 有限（固话流水线） | Teralynx P4可编程 ✅ |
| **P4支持** | 仅Jericho微码 | Teralynx全P4 ✅ |
| **集成CPU** | 弱（需外接） | Prestera片上A72强 ✅ |
| **SerDes技术** | 自研，优秀 | Inphi Alaska，✅ 略优 |
| **多芯片互联** | HiGig生态 ✅ | 无（标准以太网） |
| **数据中心份额** | ~70% ✅ | ~12-15% |
| **企业份额** | ~45% | ~20-25% ✅(增长中) |

**Broadcom的核心优势**：
- 生态壁垒（SDK+HiGig+ODM全链路绑定）
- 产品线最宽（1G-800G全覆盖）
- HiGig多芯片互联能力
- 性能领先（每代领先对手约12-18个月）

**Marvell的突破口**：
- Teralynx P4可编程性（差异化竞争）
- Prestera片上CPU集成（降低系统BOM）
- Alaska SerDes（Inphi技术优势）
- 完整互联产品线（交换+PHY+光+网络处理器）

**一句话总结**：Broadcom是行业老大，生态护城河极深；Marvell通过收购拼图形成完整产品线，在可编程和集成度上找到差异化。多数场景选Broadcom最稳妥，但Marvell在特定场景提供了有竞争力的替代方案。

---

> **文档结束** — 本文从芯片架构师视角，系统分析了Broadcom和Marvell的交换机芯片产品线、IP模块构成、架构设计理念。结合行业格局和技术趋势，为选型和理解行业提供了全面参考框架。

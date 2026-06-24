# SIM 卡插卡 → 注册网络 → 信号强度 & 数据上网

> **适用版本**：Android 9 ~ 14 | **标签**：Telephony · 插卡 · 网络注册 · APN · PDP
>
> **版本说明**：Android 13 重构了数据连接模块。Android 9~12 数据连接由 `DcTracker` 管理；Android 13+ 改为 `DataNetworkController` + `DataNetwork` 管理。流程逻辑不变，类名变了。

---

## 〇、总览图

```
                   蜂窝网络注册（AT+COPS）
                  附着到基站 ← 这是地基
                         ↓
              ┌──────────┴──────────┐
              ↓                     ↓
     信号格数（能打电话）      数据上网（能上网）
```

> **关键认知**：通话信号和数据上网都依赖蜂窝网络。蜂窝网络注册成功 = 附着到基站，这是后续一切的前提。没有这个地基，后面两条路都走不通。

```
┌──────────────────────────────────────────────────────────────┐
│                     插卡检测（硬件中断）                        │
│                         ↓                                     │
│                    SIM 卡初始化                                │
│                 （读 IMSI、鉴权）                               │
│                         ↓                                     │
│              ┌────── 蜂窝网络注册 ──────┐                       │
│              │     AT+COPS 附着到基站      │   ← 公共链路         │
│              └────────────┬──────────────┘                       │
│              ┌────────────┴──────────────┐                       │
│              ↓                           ↓                       │
│    ┌─────────────────┐        ┌────────────────────┐            │
│    │  链路 A：通话信号  │        │   链路 B：数据上网    │            │
│    │                 │        │                    │            │
│    │  +CSQ 信号上报    │        │   APN 配置加载       │            │
│    │  显示信号格数      │        │   PDP 上下文激活      │            │
│    │  能打电话         │        │   显示 4G/5G 图标     │            │
│    │                 │        │   能上网             │            │
│    └─────────────────┘        └────────────────────┘            │
└──────────────────────────────────────────────────────────────┘
```

---

## 一、公共链路：插卡 → SIM 就绪 → 网络注册

### 1.1 插卡检测

```
┌────┐  ┌──────┐  ┌──────┐  ┌───────┐  ┌───────┐
│内核 │  │Modem │  │ RILD │  │RIL.java│  │IccCard│
└──┬──┘  └──┬───┘  └──┬───┘  └───┬───┘  └───┬───┘
   │ GPIO   │          │          │          │
   │中断    │          │          │          │
   │────────│          │          │          │
   │        │ SIM 状态 │          │          │
   │        │变化      │          │          │
   │        │──────────│          │          │
   │        │          │ socket   │          │
   │        │          │──────────│          │
   │        │          │          │ 状态上报  │
   │        │          │          │──────────│
```

### 1.2 SIM 卡初始化

```
IccCard 检测到 SIM 状态 = PRESENT
  ↓
读取 SIM 文件：EF_IMSI、EF_ICCID
  ↓ AT+CRSM / AT+CIMI
获取到 IMSI → SIM 卡就绪
  ↓
GsmCdmaPhone 收到 SIM 就绪通知
```

### 1.3 蜂窝网络注册

```
SIM 就绪 → Modem 根据 SIM 卡内 PLMN 优先级列表自动搜索网络
  ↓
Modem 搜索可用网络 → 选择最优 PLMN
  ↓ AT+COPS? 查询注册状态
Modem 发起网络 Attach（附着到蜂窝网络）
  ↓
注册成功 → Modem 上报：+COPS: 0,0,"China Mobile",7
  ↓
RIL.java → ServiceStateTracker 处理
  ↓
ServiceState：OUT_OF_SERVICE → IN_SERVICE
  ↓
Phone.notifyServiceStateChanged()
```

> **到此为止**：有信号格数，能打电话/发短信。
> 数据上网需要继续走后续的 APN + PDP 流程。

---

## 二、链路 A：通话信号（信号格数显示）

### 2.1 信号强度上报

```
注册成功后，Modem 周期性上报信号强度
  ↓ AT 通道上报：+CSQ: 25,99
RILD → RIL_onUnsolicitedResponse(RIL_UNSOL_SIGNAL_STRENGTH)
  ↓ socket
RIL.java → 通知所有 SignalStrength 监听者
  ↓
ServiceStateTracker 处理信号强度
  ↓
Phone → SignalStrength 对象更新
```

### 2.2 UI 显示

```
现代 Android（尤其 13+）走 TelephonyRegistry 回调机制：
  TelephonyRegistry.registerTelephonyCallback()
  ↓
SystemUI (MobileSignalController) 收到 TelephonyCallback 信号回调
  ↓
根据 SignalStrength 计算格数（0~5 格）
  ↓
StatusBar 更新信号图标
```

### 2.3 信号指标

| 指标 | 含义 | 单位 |
|------|------|------|
| **RSSI** | 接收信号强度指示 | dBm |
| **RSRP** | 参考信号接收功率（4G） | dBm |
| **RSRQ** | 参考信号接收质量（4G） | dB |

### 2.4 格数映射（典型值）

| 格数 | RSRP 范围 (4G) | RSSI 范围 (2G/3G) |
|:----:|:--------------|:-----------------|
| 0 格 | < -140 dBm | < -113 dBm |
| 1 格 | -140 ~ -115 dBm | -113 ~ -105 dBm |
| 2 格 | -115 ~ -105 dBm | -105 ~ -95 dBm |
| 3 格 | -105 ~ -95 dBm | -95 ~ -85 dBm |
| 4 格 | -95 ~ -85 dBm | -85 ~ -75 dBm |
| 5 格 | > -85 dBm | > -75 dBm |

> 具体阈值由运营商 `carrier_config` 决定。

### 2.5 +CSQ 解析

```
AT+CSQ 返回: +CSQ: <rssi>,<ber>

  rssi: 0~31（31=最强，0=-113dBm 以下，99=未知）
  ber:  0~7（误码率，99=未知）

  换算: RSSI(dBm) = -113 + 2 × rssi
  示例: rssi=25 → RSSI = -63 dBm（极强信号）
  示例: rssi=13 → RSSI = -87 dBm（约 3 格）
```

---

## 三、链路 B：数据上网（4G/5G 图标 → 能上网）

> **前置条件**：必须先完成 **蜂窝网络注册**（1.3 节），附着到基站。注册成功后，核心网才完成 IMSI 鉴权，才能为你分配数据通道。

### 3.1 APN 配置加载

```
蜂窝网络注册成功 → 核心网已认证 → APN 管理器启动
  Android 9~12：`DcTracker`（原 `DataConnectionTracker`）
  Android 13+：`DataNetworkController`
  ↓
读取 APN 配置源（优先级从高到低）：

  ① CarrierConfigManager → 运营商配置文件（XML）    ← 最高
     例：中国移动定制 ROM 内置的 carrier_config.xml

  ② SIM 卡 IMSI 推导运营商 → 查 apns-conf.xml       ← 中
     根据 IMSI 前缀 MCC+MNC 判断运营商（46000=移动）

  ③ 系统内置 apns-conf.xml 全球 APN 列表            ← 低
     Android 系统自带的全球运营商 APN 数据库

  ④ 用户在设置中手动选择/添加                       ← 最低
     设置 → 移动网络 → 接入点名称(APN) → 选择/新建

  ↓ 匹配成功
  名称：中国移动  APN：cmnet  类型：default,supl,dun,mms  协议：IPv4v6
```

> **APN 是什么**：Access Point Name，接入点名称。告诉手机"我要连到哪个网关去上网"。就像 WiFi 需要连到路由器，蜂窝网络需要连到 APN 网关才能访问互联网。

### 3.2 IA-APN（初始附着兜底）

```
如果上面的 APN 列表中没有 default 类型 APN → 触发 IA-APN 机制
  ↓
RIL 向核心网查询初始附着 APN → 核心网下发一个基础 APN
  ↓
用这个基础 APN 先建立数据通道（至少能联网）
  ↓
联网后下载完整的运营商 APN 配置 → 切换为完整 APN
```

> **IA-APN 作用**：设备刚开机或换卡时，手机里可能还没有这个运营商的 APN 配置。先用核心网下发的初始 APN 通网，再下载完整配置。相当于"先用临时通行证进门，再办正式证件"。

### 3.3 PDP 上下文激活（数据通道建立）

> **PDP 是什么**：Packet Data Protocol，分组数据协议。可以理解为手机和核心网之间建立的一条"数据隧道"。隧道建好后，你的所有上网流量都走这条隧道。

#### 第一步：配置 APN 参数

```
┌───────────────┐  ┌──────┐  ┌──────┐  ┌──────────┐
│ DataController │  │ RIL  │  │ RILD │  │ Modem/核心网 │
└───────┬───────┘  └──┬───┘  └──┬───┘  └──────┬───┘
        │              │          │              │
        │ setupDataCall          │              │
        │──────────│              │              │
        │          │ setupDataCall          │              │
        │          │──────────│              │              │
        │          │          │ AT+CGDCONT   │              │
        │          │          │───────────────────────────>│
        │          │          │              │ 参数已配置    │
        │          │          │<───────────────────────────│
```

#### 第二步：激活 PDP 上下文

```
┌───────────────┐  ┌──────┐  ┌──────┐  ┌──────────┐
│ DataController │  │ RIL  │  │ RILD │  │ Modem/核心网 │
└───────┬───────┘  └──┬───┘  └──┬───┘  └──────┬───┘
        │              │          │              │
        │              │          │ AT+CGACT     │
        │              │          │───────────────────────────>│
        │              │          │              │ IP 分配      │
        │              │          │<───────────────────────────│
        │              │          │ AT+CGACT:1,1   │
        │              │          │<───────────────────────────│
        │              │          │              │              │
        │              │          │ INACTIVE→ACTIVE  │              │
        │              │          │              │              │
```

**两条关键 AT 命令：**

| 命令 | 作用 | 类比 |
|------|------|------|
| `AT+CGDCONT=1,"IP","cmnet"` | 告诉 Modem 你要用什么 APN 连网 | 输入地址 |
| `AT+CGACT=1,1` | 激活 PDP 上下文，正式建立数据通道 | 点击"连接" |

### 3.4 UI 显示

```
数据通道 ACTIVE → ServiceStateTracker 更新数据网络类型（LTE/NR/UMTS/EDGE）
  ↓
TelephonyManager/DataNetworkController 通知数据连接状态变化（现代 Android 走回调机制）
  ↓
SystemUI (MobileSignalController) 收到回调
  ↓
根据网络类型渲染图标：5G / 4G(LTE) / 3G / E
  ↓
有上下行流量时 → 显示上下箭头动画
```

### 3.5 APN 类型速查

| 类型 | 用途 |
|:----:|------|
| `default` | 普通上网流量 |
| `mms` | 彩信专用 |
| `supl` | GPS 辅助定位（SUPL） |
| `dun` | 网络共享（热点） |
| `hipri` | 高优先级连接 |
| `fota` | OTA 升级专用 |
| `ims` | IMS 信令和媒体通道（VoLTE / ViLTE / RCS） |
| `ia` | Initial Attach（初始附着） |

---

## 四、完整时序图

```
Modem      RILD    RIL.java   GsmCdmaPhone  ServiceState  DataNetCtrl  SystemUI
  │          │        │           │             │            │           │
  │──SIM────→│         │           │             │            │           │
  │          │─socket─→│           │             │            │           │
  │──IMSI───→│─socket─→│──初始化───→│             │            │           │
  │          │        │           │─网络注册────→│            │           │
  │──注册成功→│         │           │             │            │           │
  │          │─socket─→│           │             │─IN_SERVICE │           │
  │          │        │           │             │            │           │
  │──+CSQ───→│─socket─→│           │─信号更新────→│            │           │
  │          │        │           │             │──回调─────→│           │──信号格数
  │          │        │           │             │            │─APN加载──→│
  │←─配APN───│←───────│←──────────│←────────────│←───────────│           │
  │←─激PDP───│←───────│←──────────│             │            │           │
  │──返回IP──→│─socket→│           │─数据变化────→│            │           │
  │          │        │           │             │──回调─────→│──4G/5G 图标
```

> DataNetCtrl = 数据连接管理器，Android 9~12 是 `DcTracker`，Android 13+ 是 `DataNetworkController`

---

## 五、网络注册状态

| 状态码 | 含义 | 状态栏显示 |
|:------:|------|-----------|
| **0** | 未注册，未搜索 | 无服务（×） |
| **1** | 已注册，归属网络 | 正常信号格数 |
| **2** | 搜索中 | 搜索中（...） |
| **3** | 注册被拒绝 | 无服务（×） |
| **4** | 未知 | 无服务（×） |
| **5** | 已注册，漫游 | 漫游图标 + 信号格数 |

---

## 六、排查速查表

### 6.1 通话信号类

| 现象 | 排查方向 | 关键日志 |
|------|---------|---------|
| 插卡无反应 | Kernel GPIO 中断 | `dmesg` |
| 显示"无 SIM 卡" | IccCard 状态 | IccCard 日志 |
| 显示"SIM 卡错误" | SIM 读取失败 | UiccController 日志 |
| 显示"正在搜索" | 网络搜索中 | ServiceState = SEARCHING |
| 显示"无服务" | 注册被拒绝 | ServiceState = OUT_OF_SERVICE |
| 注册后立刻掉网 | 信号太差 | 看 RSRP 值 |
| 仅 2G 无 4G | 网络模式设置 | preferred network type |
| 信号格数不准 | 阈值配置 | carrier_config |

### 6.2 数据上网类

| 现象 | 排查方向 | 关键日志 |
|------|---------|---------|
| 有信号但上不了网 | APN 缺失/错误 | APN 设置列表 |
| 显示 4G 但无箭头 | 数据连接未 ACTIVE | DataConnection 日志 |
| 显示 E/2G 不显示 4G | 信号差/基站不支持 | RSRP < -120dBm |
| PDP 激活失败 | AT+CGACT? 返回 | DataConnection 状态 |
| 核心网拒绝 | 欠费/漫游限制 | rillog 看网络侧信令 |

### 6.3 逐层排查清单

```
1. dmesg        → Kernel 是否检测到 SIM 插入？
2. RILD         → 是否收到 Modem 的 SIM 状态变化上报？
3. RIL.java     → UiccController 是否收到 SIM 就绪通知？
4. GsmCdmaPhone → 是否发起网络注册？
5. ServiceState → 注册状态是什么？（通话信号链路）
6. SignalStrength → RSRP/RSSI 值是多少？（通话信号链路）
7. DcTracker / DataNetworkController → APN 是否加载？数据连接是否 ACTIVE？（数据上网链路）
8. SystemUI     → MobileSignalController 是否收到回调？
9. 状态栏       → Android 11 及以下：SignalClusterView 是否更新了图标？
               Android 12+：通过 StatusBarSignalPolicy + IconController 体系更新图标
```

---

## 七、关键日志标识

| 阶段 | RIL 日志 | 其他日志 |
|------|---------|---------|
| 插卡检测 | `RIL_UNSOL_SIM_STATUS_CHANGED` | dmesg：GPIO 中断 |
| SIM 初始化 | `RIL_REQUEST_GET_IMSI` | UiccController：SIM 加载 |
| 网络搜索 | `RIL_REQUEST_OPERATOR` | ServiceStateTracker：搜索中 |
| 注册成功 | `+COPS: 0,0,"...",7` | ServiceState: `IN_SERVICE` |
| 信号上报 | `RIL_UNSOL_SIGNAL_STRENGTH` | SignalStrength 更新 |
| APN 加载 | — | DcTracker（9~12）/ DataNetworkController（13+）/ ApnContext 日志 |
| 数据连接请求 | `RIL_REQUEST_SETUP_DATA_CALL` | DataConnection 状态变化 |
| 配置 APN | `AT+CGDCONT` | RILD 日志 |
| 激活 PDP | `AT+CGACT` | RILD 日志 |
| 数据连接激活 | — | DataConnection: `ACTIVE` |
| UI 更新图标 | — | MobileSignalController 收到回调 |

---

## 八、蜂窝网络 vs 数据网络对比

> **关键区分**：蜂窝网络 = 附着到基站（能打电话）；数据网络 = 在蜂窝网络基础上建立数据通道（能上网）。

| 维度 | 蜂窝网络注册（信号格数） | 数据网络连接（4G/5G 图标） |
|:----:|----------------------|-------------------------|
| **目的** | 能打电话、发短信 | 能上网 |
| **前提** | SIM 就绪 + 有基站信号 | 蜂窝网络注册成功 + APN 配置 |
| **核心网交互** | IMSI 鉴权 → 附着到基站 | PDP 上下文激活 → 分配 IP |
| **AT 命令** | `AT+COPS` | `AT+CGDCONT` + `AT+CGACT` |
| **状态机** | ServiceState | DataConnection |
| **管理类** | ServiceStateTracker | DcTracker（9~12）/ DataNetworkController（13+） |
| **UI 显示** | 信号格数（0~5 格） | 4G/5G/E 图标 + 上下箭头 |
| **失败表现** | 无服务（×） | 有信号但无数据 |
| **WiFi 替代** | 无（打电话必须走蜂窝） | 有（WiFi 可替代蜂窝上网） |

---

## 九、面试高频考点

**Q1：插卡到能打电话、能上网经历了什么？**

硬件检测 SIM 插入 → 读 IMSI 鉴权 → 蜂窝网络注册（AT+COPS）附着到基站 → 注册成功后分两条路：① 信号强度上报（+CSQ），SystemUI 显示信号格数，能打电话；② 加载 APN 配置，建立 PDP 上下文（AT+CGDCONT / AT+CGACT），核心网分配 IP，显示 4G/5G 图标，能上网。

**Q2：信号格数是怎么算出来的？**

Modem 通过 +CSQ 上报原始信号值（rssi 0~31），RIL.java 解析成 SignalStrength 对象。现代 Android 通过 TelephonyRegistry 回调通知 SystemUI，MobileSignalController 根据运营商 carrier_config 中配置的阈值，将 RSRP/RSSI 映射为 0~5 格。

**Q3：APN 是什么？IA-APN 又是什么？**

APN 是接入点名称，定义了设备如何连接到移动网络的数据通道。IA-APN 是 Initial Attach APN，用于设备刚开机或换卡时还没有完整 APN 配置的场景，先用网络侧下发的初始 APN 通网，再下载完整配置。

**Q4：为什么有时候有信号但上不了网？**

最常见原因：APN 配置缺失或错误。其次是数据开关关闭、核心网拒绝（欠费/漫游限制）、信号质量差导致数据连接不稳定、PDP 激活失败等。

---

*本文档基于 AOSP 通用架构整理*

---

## 附：Telephony vs Telecom 概念速查

> 本文档主要涉及 **Telephony** 层（SIM 管理、网络注册、信号上报）。Telephony 与 Telecom 是 Android 通话架构中两个独立层级：

| | Telephony | Telecom |
|--|--|--|
| **职责** | 管硬件：SIM 卡、网络注册、信号强度、AT 命令通信 | 管电话：通话状态机、多路管理、音频路由 |
| **核心类** | `RIL`, `GsmCdmaPhone`, `CallTracker`, `ServiceStateTracker` | `CallsManager`, `ConnectionService`, `android.telecom.Call` |
| **本文档涉及的部分** | 全部（SIM 检测 → 网络注册 → 信号上报 → 数据连接） | 不涉及（纯通信基础设施层） |
| **桥梁** | **`ConnectionService`**：把 Telephony 的底层通话包装成 Telecom 的统一通话对象 |

```
Telephony（本文档内容）: SIM ↔ 网络注册 ↔ 信号/数据
Telecom（通话相关文档）: 来电/去电/挂断 ↔ 状态管理 ↔ InCallUI
```

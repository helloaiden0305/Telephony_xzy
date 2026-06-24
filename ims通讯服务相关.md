# IMS 通讯服务相关

## 什么是 IMS？

在 Android 手机中，IMS 是实现 VoLTE (Voice over LTE)、VoWiFi (Voice over Wi-Fi)、RCS (Rich Communication Services) 消息等高级通信功能的关键技术。

简单来说，让手机能够通过 4G/5G 网络进行高质量的语音通话，或者在 Wi-Fi 环境下打电话，发送更丰富的消息。

**IMS = 运营商的语音"服务器"和业务"控制平台"。**

手机只要注册上 IMS，就能使用上面所有业务能力（前提是运营商给你开通）。

## IMS 服务架构：从底层到上层

### 整体链路示意图

```
┌─────────────────────────────────────────────────────────┐
│                    应用层 (Application)                  │
│              拨号器、短信应用、第三方应用                   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Telephony Framework & 应用                   │
│         负责调用 ImsService，提供用户界面交互               │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│           Android 框架层：ImsService                      │
│  对上：通过 Binder/AIDL 接口供 Telephony Framework 调用   │
│  对下：将 IMS 请求转交给厂商实现 (imsdaemon)              │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              桥梁：RIL (Radio Interface Layer)           │
│  rilj (Java) ──► rild (Native) ──► 厂商 RIL 库 (.so)     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│        进程实现：imsdaemon (Vendor Layer 供应商层)         │
│   原生守护进程，处理 SIP 信令、RTP 媒体流，与 Modem 交互     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    Modem / 基带                           │
│              与运营商 IMS 核心网进行通信                    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                运营商 IMS 核心网                           │
│          VoLTE、VoWiFi、RCS 等业务的控制平台                │
└─────────────────────────────────────────────────────────┘
```

Android 系统中的 IMS 服务是一个复杂的系统，涉及多个进程和组件协同工作。大致分为以下几个层次：

### 1. ims.apk：IMS 服务的应用载体与预置配置

ims.apk 是一个应用程序包（APK），它通常包含了 imsdaemon 守护进程（或其客户端 Java 组件，用于与原生守护进程交互）以及实现 IMS 服务所需的其他资源。

它通常由芯片厂商（例如 org.codeaurora.ims 表明这可能是高通平台的实现）或设备制造商提供，作为一个预置的系统应用存在于设备中。它是设备上实现 VoLTE、VoWiFi、RCS 消息等 IMS 核心功能的具体载体。

ims.apk 作为一个系统级应用，必须在设备出厂时就被预装到系统分区。

如果这个配置在某个产品分支（例如用于 CTA 认证的分支）中被遗漏（"漏配"），那么在编译出的 ROM 中将不会包含 ims.apk。这将导致设备无法提供任何 IMS 服务。

**与 IMS 服务的关系：** ims.apk 是 IMS 服务在 Android 设备上的具体实现者。Android 框架层提供的 ImsService API 接口，实际通过 Binder 机制与 ims.apk 内部的组件（包括 imsdaemon）进行通信的。

ims.apk 负责接收来自框架的请求，执行与运营商 IMS 核心网的实际交互，管理 IMS 连接和会话，并向框架报告 IMS 的注册和能力状态。可以说，没有 ims.apk，上层的 IMS 相关功能将无法正常运行。

### 2. 进程实现：imsdaemon (Vendor Layer 供应商层)

imsdaemon (通常是 vendor.imsdaemon) 是 IMS 服务的核心实现，它是一个运行在用户空间（User Space）的**原生守护进程**。

这个进程通常由芯片厂商或设备厂商提供，私有化，Android无法统一，与底层的基带（Modem）进行紧密交互。

**主要职责：**

- **IMS 协议栈实现：** 负责处理 SIP (Session Initiation Protocol) 信令、RTP (Real-time Transport Protocol) 媒体流等 IMS 核心协议。
- **与 Modem 交互：** 通过 RIL (Radio Interface Layer) 或其他专有接口与基带通信，获取网络状态、注册 IMS 服务、管理语音/视频会话等。
- **提供 Binder/AIDL 接口：** imsdaemon 会对外暴露一个标准的 Binder/AIDL (Android Interface Definition Language) 接口，供上层 Android 框架调用。

**在链路中的位置：** 它是 IMS 服务的"大脑"和"执行者"，直接与网络侧进行交互。

### 3. 桥梁：RIL

RIL 是 Android 电话服务与基带（Modem）之间的抽象层。它将上层框架的请求翻译成基带能理解的命令，并将基带的响应和事件传递给上层。

- **与 IMS 的关系：** 尽管 imsdaemon 可能会有自己的方式与 Modem 交互，但 RIL 仍然是 Android 框架与 Modem 之间进行通用无线通信的关键。某些 IMS 相关的请求（例如 IMS 注册状态查询、VoLTE 开关控制等）可能仍然会通过 RIL 传递。
- **在链路中的位置：** 介于 Android 框架和 Modem 之间，负责无线通信的桥接。imsdaemon 不一定需要完全通过 RIL 才能与 Modem 通信，它通常有自己的更直接或专有的方式。

这里表示整体概念的 RIL，包括

**1. rilj (RIL Java)：**

- 它是 RIL 的 Java 部分，位于 Android 框架层 (com.android.internal.telephony.RIL)。
- 它是 Android 电话服务（Telephony Framework）与 RIL 交互的接口。上层框架通过 rilj 发送请求（例如拨号、发送短信、查询网络状态等），并接收来自 Modem 的响应和事件。
- rilj 不直接与 Modem 交互，它通过 Binder 机制与 rild 进行通信。

**2. rild (RIL Daemon)：**

- 它是 RIL 的 Native (C/C++) 守护进程 (/system/bin/rild)。
- 它是 rilj 与 厂商 RIL 库 之间的桥梁。rild 接收来自 rilj 的请求，然后加载并调用厂商提供的 RIL 库（通常是一个 .so 共享库）。
- rild 也负责将厂商 RIL 库从 Modem 接收到的响应和事件，通过 Binder 机制传递回 rilj。

**3. 厂商 RIL 库 (Vendor RIL Library)：**

- 这是由芯片厂商或 Modem 厂商提供的共享库。
- 它包含了与特定 Modem 硬件直接交互的实现代码，将通用的 RIL 请求翻译成 Modem 能理解的 AT 命令或其他专有命令，并解析 Modem 的响应。

### 4. Android 框架层：ImsService

ImsService 是 Android Framework 提供的一个系统级 Service 基类（SystemApi），位于 com.android.phone 进程（或独立的 IMS 进程）中。

ImsService 并不是抽象类，但它提供了一套标准的生命周期、Binder 接口以及 Feature 管理框架，厂商需要通过 override 关键方法来提供 MmTel、RCS 等 IMS 功能。

在职责上，ImsService 主要负责：

- **对上：** 通过标准化的 Binder / AIDL 接口，供 Telephony Framework 调用，
- **对下：** 将 Framework 的 IMS 请求转交给厂商实现，通常再由厂商代码与底层的 imsdaemon 或 native IMS 模块交互；
- **状态管理：** 维护 IMS 注册状态、能力变化，并向 Framework 回调。

在整体链路中，Android 框架与底层 imsdaemon 之间的"翻译官"和"代理人"，

暂时无法在HONOR E Link文档外展示此内容

**ImsService 的启动**

ImsService 不是常驻的，由于 Feature 属于 Service 的 子模块

由 ImsResolver 根据当前需要的 IMS Feature，动态决定是否拉起以及拉起哪个 ImsService。

ImsResolver 在 calculateFeatureConfigurationChange() 计算出需要创建的 Feature 后，会调用 bindImsServiceWithFeatures()，并通过 ImsServiceController 执行 bindService()，从而拉起对应的 ImsService 并建立 Binder 连接。

```
IMS 启动
ImsResolver
   ↓
计算需要的 Feature（逻辑配置）
   ↓
bindImsServiceWithFeatures()
   ↓
bindService() → 启动 ImsService
   ↓
ImsService.onCreate()
   ↓
ImsService.createMmTelFeatureInternal()
   ↓
new xxxxxxxFeature()
```

### 5. 最上层：Telephony Framework & 应用

Android 系统中负责电话、短信、数据连接等核心功能的框架层，以及最终使用 IMS 功能的各种应用程序（如拨号器、短信应用等）。

**主要职责：**

- **调用 ImsService：** 通过 ImsService 提供接口，实现 VoLTE 电话、VoWiFi 电话、RCS 消息等功能。
- **用户界面交互：** 提供用户界面，允许用户启用/禁用 VoLTE/VoWiFi，管理 IMS 相关设置。

**在链路中的位置：** 用户和应用直接交互的层面。

---

## 结合案例分析：imsdaemon 不稳定导致的问题

结合日志和问题描述，深入分析 imsdaemon 不稳定对整个 IMS 链路的影响。

### 问题现象：imsdaemon 被 init 杀死和拉起

**日志显示：**

- init 31: Sending signal 15 to service 'vendor.imsdaemon' (pid 5040) process group...
- init 31: Sending signal 9 to service 'vendor.imsdaemon' (pid 5040) process group...
- init 31: starting service 'vendor.imsdaemon'...

这表明 imsdaemon 进程并非自身崩溃，而是被 Android 系统的init 进程主动停止

先 signal 15 尝试关闭，后 signal 9 强制杀死，然后又被init进程根据配置重新拉起。

这是一个**服务生命周期管理问题**，而不是 imsdaemon 内部的崩溃

### 连锁反应：ImsService 长时间未 ready

日志中频繁出现：

- ImsService not up yet - timeout waiting for connection.

这是 imsdaemon 不稳定的直接后果。

当 imsdaemon 被 init 杀死时，它与上层 ImsService 之间的 Binder 连接就会断开。

当 imsdaemon 被重新拉起时，ImsService 需要重新建立连接并完成初始化。由于 imsdaemon 被反复杀死和拉起，导致 ImsService 始终无法建立一个稳定的、持续的连接，因此它会一直处于"未 ready"状态，并不断尝试连接，最终导致超时。

### 进一步影响：subId 频繁变化与 CarrierConfig 缓存问题

IMS 订阅 subId 在初始化阶段频繁变化（1 / 2 / -1 / 3）。

CarrierConfig 在多个临时 subId 下被缓存。

**1. subId 频繁变化：**

- subId** (Subscription ID)** 是 Android 系统用来唯一标识一个 SIM 卡或订阅的 ID。在多卡手机中，每个 SIM 卡都有一个 subId。IMS 服务是与特定的 subId 绑定的。
- 当 ImsService 无法稳定运行时，Android 电话框架可能无法正确地识别和管理当前哪个 subId 应该用于 IMS 服务。这可能是因为：
  - imsdaemon 的不稳定导致其无法向框架报告正确的 IMS 注册状态或能力。
  - 框架在 imsdaemon 不可用时，可能尝试在不同的 subId 上初始化 IMS，或者由于内部状态机混乱，导致 subId 频繁切换（例如从 1 切换到 2，再到 -1 表示无效，然后又到 3）。
  - 这种 subId 的频繁变化，进一步加剧了 IMS 服务的不稳定，因为它使得系统难以确定哪个 subId 才是当前应该提供 IMS 服务的有效订阅。

**2. CarrierConfig在多个临时 subId 下被缓存：**

- CarrierConfig(运营商配置)*是一组由移动运营商提供的配置参数，它决定了设备如何为特定运营商提供各种网络服务，包括 IMS。例如，是否启用 VoLTE、VoWiFi、特定的编解码器、各种定时器等，都由 CarrierConfig 控制。这些配置对于 IMS 服务的正确运行至关重要。
- 当 subId 频繁变化时，系统可能会为每一个"临时"或"错误"的 subId 去尝试获取并缓存 CarrierConfig。
- **问题所在：**
  - **资源浪费：** 不断为无效或临时 subId 缓存配置，浪费系统资源。
  - **配置混乱：** 更严重的是，如果系统最终使用了错误的 subId 对应的 CarrierConfig，或者在 subId 切换过程中配置未能正确更新，将导致 IMS 服务无法按照正确的运营商要求进行配置和运行。这可能表现为 VoLTE 无法注册、通话质量差、无法使用 VoWiFi 等问题。
  - **加剧"未 ready"：** 错误的 CarrierConfig 或配置过程中的混乱，会进一步阻碍 ImsService 达到"ready"状态，形成恶性循环。

---

## IMS 相关排查关键词（更新中）

```
vendor.imsdaemon
ImsService not up yet
timeout waiting for connection
Binder died
subId
CarrierConfig
IMS registration
IMS not registered
MmTel not ready
capability unavailable
VoLTE
VoWiFi
```

---

## ImsManager 类解读

`frameworks/opt/net/ims/src/java/com/android/ims/ImsManager.java`

提供了用于 MMTEL IMS 服务的 API，例如发起 IMS 呼叫，并提供了访问运营商 IMS 网络的能力。

本类是所有 IMS MMTEL 操作的起点。可以通过调用 {@link #getInstance getInstance()} 方法来获取它的实例。对于 RCS 服务，请使用 {@link RcsFeatureManager}。

**仅供内部使用！开发者改用 {@link ImsMmTelManager}**

### LazyExecutor 类

延迟初始化的执行器，只有在需要执行任务时才会创建其内部的单线程执行器，以避免不必要的线程启动。

### MmTelFeatureConnectionFactory 接口

工厂接口，定义了如何创建 MmTelFeatureConnection 实例的方法，用于建立与 MMTEL IMS 功能的连接。

### ImsStatsCallback 接口

定义了用于收集 IMS MMTEL 相关统计信息的事件回调。

### InstanceManager 类

负责管理每个 SIM 卡槽（per-slot）的 ImsManager 实例的生命周期和连接，通过 FeatureConnector 确保实例的更新和连接状态的维护。

- **InstanceManager 构造函数：** 初始化 InstanceManager，设置日志后缀，并创建一个 FeatureConnector 来管理 ImsManager 的连接。
- **getInstance() 方法：** 返回由 InstanceManager 管理的 ImsManager 实例。
- **reconnect() 方法：** 触发 FeatureConnector 重新连接 IMS 服务，并等待连接建立或超时。
- **connectionReady() 方法：** 当 ImsManager 与 IMS 服务的连接准备就绪时，此回调被调用，并释放等待连接的锁。
- **connectionUnavailable() 方法：** 当 ImsManager 与 IMS 服务的连接变得不可用时，此回调被调用，并根据原因更新连接状态。

```
===================================================================================================================

// IMS 服务实例获取与管理
getInstance(Context context, int phoneId)
// 获取一个 ImsManager 实例，如果不存在则为指定的 phoneId 创建一个，并可能在有限时间内阻塞以连接到相应的 IMS 服务。

isImsSupportedOnDevice(Context context)
// 检查设备是否支持 IMS 电话功能。

isTelephonyCallingSupportedOnDevice(Context context)
// 检查设备是否支持电话呼叫功能。

isServiceAvailable()
// 返回 IMS 服务是否可用。

isServiceReady()
// 返回 IMS 服务是否已准备好向底层发送请求。

open(MmTelFeature.Listener listener, ImsEcbmStateListener ecbmListener, ImsExternalCallStateListener multiEndpointListener)
// 打开 IMS 服务，用于进行呼叫和/或接收通用 IMS 呼叫，并注册 ECBM、多端点和 UT 的监听器。

close()
// 关闭在 open 方法中打开的连接，并移除相关联的监听器。

turnOnIms()
// 开启 IMS 服务。

turnOffIms()
// 关闭 IMS 服务。

getImsServiceState()
// 获取 IMS 服务的当前状态。

getImsServiceState(Consumer<Integer> result)
// 异步获取 IMS 服务状态，并通过 Consumer 回调返回结果。

// 呼叫管理
shouldProcessCall(boolean isEmergency, String[] numbers)
// 判断具有指定号码的呼叫应该通过 IMS 还是 CSFB 进行。

createCallProfile(int serviceType, int callType)
// 根据服务能力和 IMS 注册状态创建一个 ImsCallProfile。

makeCall(ImsCallProfile profile, String[] callees, ImsCall.Listener listener)
// 创建一个 ImsCall 对象以发起呼叫。

takeCall(IImsCallSession session, ImsCall.Listener listener)
// 创建一个 ImsCall 对象以接听来电。

getEcbmInterface()
// 获取 ECBM (Emergency Call Back Mode) 接口，用于请求退出 ECBM。

setTerminalBasedCallWaitingStatus(boolean enabled)
// 通知 IMS 服务呼叫等待用户设置的更改。

// 注册与能力管理
addRegistrationCallback(RegistrationManager.RegistrationCallback callback, Executor executor)
// 添加一个回调，当与此 ImsManager 关联的槽 ID 的 IMS 注册状态发生变化时会调用此回调。

removeRegistrationListener(RegistrationManager.RegistrationCallback callback)
// 移除之前通过 addRegistrationCallback 添加的注册回调。

addCapabilitiesCallback(ImsMmTelManager.CapabilityCallback callback, Executor executor)
// 添加一个回调，当 MMTel 能力状态（例如 IMS 语音或 IMS 视频的可用性）发生变化时会调用此回调。

removeCapabilitiesCallback(ImsMmTelManager.CapabilityCallback callback)
// 移除之前注册的 MMTel 能力回调。

reevaluateCapabilities()
// 评估 IMS 能力的状态，并将更新后的状态推送到 IMS 服务。它会根据 VoLTE、VoWiFi、视频通话、跨 SIM 卡呼叫、RCS、UT 和呼叫合成器等功能的配置来启用或禁用相应的 IMS 能力，并决定是否开启或关闭 IMS。

changeMmTelCapability(boolean isEnabled, int capability, int... radioTechs)
// 启用或禁用多个无线电技术的能力。

queryMmTelCapability(@MmTelFeature.MmTelCapabilities.MmTelCapability int capability, @ImsRegistrationImplBase.ImsRegistrationTech int radioTech)
// 查询指定能力和无线电技术是否已启用。

queryMmTelCapabilityStatus(@MmTelFeature.MmTelCapabilities.MmTelCapability int capability, @ImsRegistrationImplBase.ImsRegistrationTech int radioTech)
// 查询指定能力在当前注册技术下是否可用。

getRegistrationTech()
// 获取当前的 IMS 注册技术（例如 LTE、NR）。

isCapable(@ImsService.ImsServiceCapability long capabilities)
// 返回指定的所有能力是否都可用。

// SMS/消息管理
sendSms(int token, int messageRef, String format, String smsc, boolean isRetry, byte[] pdu)
// 发送短信。

setSmsListener(IImsSmsListener listener)
// 设置短信监听器。

onSmsReady()
// 通知短信服务已准备就绪。

// 配置与预置状态
getConfigInterface()
// 获取用于获取/设置服务/能力参数的配置接口 (ImsConfig)。

getConfigInt(int key)
// 获取整数类型的配置值，可以是本地配置或通过 IMS 配置接口获取。

setConfig(int key, int value)
// 设置整数类型的配置值，可以是本地配置或通过 IMS 配置接口设置。

updateImsServiceConfig(Context context, int phoneId, boolean force)
// 向 IMS 服务实现推送配置更新。

isVolteProvisionedOnDevice()
// 判断当前 SIM 卡槽是否已预置 VoLTE 功能。

isWfcProvisionedOnDevice()
// 判断当前 SIM 卡槽是否已预置 VoWiFi (WFC) 功能。

isVtProvisionedOnDevice()
// 判断当前 SIM 卡槽是否已预置视频通话 (VT) 功能。

isMmTelProvisioningRequired(int capability, int tech)
// 判断指定能力和技术的 MMTel 是否需要预置。

isRcsProvisioningRequired(int capability, int tech)
// 判断指定能力和技术的 RCS 是否需要预置。

// VoLTE/VoWiFi/视频通话/RTT 等功能开关
isEnhanced4gLteModeSettingEnabledByUser()
// 返回当前订阅用户配置的增强型 4G LTE 模式（VoLTE）设置。

setEnhanced4gLteModeSetting(boolean enabled)
// 更改当前订阅的增强型 4G LTE 模式的持久设置。

isWfcEnabledByUser()
// 返回用户对 VoWiFi (WFC) 设置的配置。

setWfcSetting(boolean enabled)
// 更改当前订阅的 VoWiFi (WFC) 的持久设置。

getWfcMode(boolean roaming)
// 获取用户配置的 VoWiFi (WFC) 首选模式，区分漫游和非漫游状态。

setWfcMode(int wfcMode, boolean roaming)
// 更改当前订阅的 VoWiFi (WFC) 的持久首选模式。

isVtEnabledByUser()
// 返回当前订阅用户配置的视频通话 (VT) 设置。

setVtSetting(boolean enabled)
// 更改当前订阅的视频通话 (VT) 的持久设置。

setRttEnabled(boolean enabled)
// 启用或禁用设备上的 RTT 配置。

setTtyMode(int ttyMode)
// 设置 TTY 模式，并根据新的 TTY 模式重新评估和更新 VoLTE、视频通话和 VoWiFi 的能力。

// 辅助方法
getSubId()
// 获取当前电话 ID 对应的订阅 ID。

factoryReset()
// 将 ImsManager 设置恢复到出厂默认值。
```

---

## 开机后 Ims 服务建立流程

```
请求 ImsManager →
通过 FeatureConnector 连接 ImsService 中的 MmTelFeature →
建立 IMS 业务能力通道。
```

`frameworks/opt/net/ims/src/java/com/android/ims/ImsManager.java`

```
ImsManager.getInstance()
  ↓
InstanceManager.reconnect()
  ↓
FeatureConnector.connect()
  ↓
ImsManager.registerFeatureCallback() 后续某一时刻触发回调
  ↓
Telephony / ImsService Binder
  ↓
MmTelFeature (ImsService 内)
  ↓
==================== Framework 边界 ====================
  ↓
厂商 IMS 实现（Java / JNI / native）
  ↓
imsdaemon / modem / IMS stack
```

### ImsManager.getInstance() → InstanceManager.reconnect()

**单例获取（按 phoneId）**

```java
@UnsupportedAppUsage
public static ImsManager getInstance(Context context, int phoneId) {
    InstanceManager instanceManager;
    synchronized (IMS_MANAGER_INSTANCES) {
        instanceManager = IMS_MANAGER_INSTANCES.get(phoneId);
        if (instanceManager == null) {
            ImsManager m = new ImsManager(context, phoneId);
            instanceManager = new InstanceManager(m);
            IMS_MANAGER_INSTANCES.put(phoneId, instanceManager);
        }
    }
    // If the ImsManager became disconnected for some reason, try to reconnect it now.
    instanceManager.reconnect();
    return instanceManager.getInstance();
}
```

**关键行**

```java
instanceManager.reconnect();
```

ImsManager.getInstance() 并不保证 IMS 已经就绪， 它只是确保触发一次连接流程，并在有限时间内等待连接结果。

不管是：

- 第一次创建
- 之前断开过
- 多线程同时调用

都会走到reconnect()：

- isConnectorActive 判断
- CountDownLatch 等待
- 单次触发 connect()

这是一个"懒连接 + 自动重连"的设计

- 返回的 ImsManager 对象 ≠ 一定已经可用
- 可用性取决于：
  - reconnect() 是否成功
  - 是否在 timeout 内连上

**可能存在问题为：IMS 服务挂了**

- getInstance() 被反复调用
- 每次都会进入 reconnect()
- 但：
  - isConnectorActive 没有被正确复位
  - 或 countDown() 没被调用

👉 就会出现：一直 timeout，但不再真正重连

[图片]

### 重连

```java
// xzy ImsService 重连
public void reconnect() {
    boolean requiresReconnect = false;
    synchronized (mLock) {
        if (!isConnectorActive) {
            requiresReconnect = true;
            isConnectorActive = true;
            mConnectedLatch = new CountDownLatch(1);
        }
    }
    if (requiresReconnect) {
        mConnector.connect();
    }
    try {
        // If this is during initial reconnect, let all threads wait for connect
        // (or timeout)
        if(!mConnectedLatch.await(CONNECT_TIMEOUT_MS, TimeUnit.MILLISECONDS)) {
            mImsManager.log("ImsService not up yet - timeout waiting for connection.");
        }
    } catch (InterruptedException e) {
        // Do nothing and allow ImsService to attach behind the scenes
    }
}
```

---

### FeatureConnector.connect()

**FeatureConnector<T> 是一个「通用的异步服务连接器」， 专门用来：连接 / 断开 相关的 Binder Service，并把连接过程标准化。**

在这里：

```java
private final FeatureConnector<ImsManager> mConnector;
```

意思是：

> 一个负责把 ImsManager 和底层 IMS Service 连接起来的工具对象"

**FeatureConnector<ImsManager> =** ImsManager 的"专职接线员"，负责把它连上 IMS Service，并在断线时通知它。

在整个 ImsManager 架构里的位置：

`frameworks/opt/net/ims/src/java/com/android/ims/ImsManager.java`

```
ImsManager.getInstance()
  ↓
InstanceManager.reconnect()
  ↓
FeatureConnector.connect()
  ↓
ImsManager.registerFeatureCallback() 后续某一时刻触发回调
  ↓
Telephony / ImsService Binder
  ↓
MmTelFeature (ImsService 内)
  ↓
==================== Framework 边界 ====================
  ↓
厂商 IMS 实现（Java / JNI / native）
  ↓
imsdaemon / modem / IMS stack
```

✅ ImsManager 不直接跟 Binder / Service 打交道

✅ 所有"连 / 断 / 回调"都交给 FeatureConnector

**关键方法**

```java
/**
 * xzy
 * Start the creation of a connection to the underlying ImsService implementation. When the
 * service is connected, {@link FeatureConnector.Listener#connectionReady} will be
 * called with an active instance.
 *
 * If this device does not support an ImsStack (i.e. doesn't support
 * {@link PackageManager#FEATURE_TELEPHONY_IMS} feature), this method will do nothing.
 */
public void connect() {
    if (DBG) log("connect");
    if (!isSupported()) {
        mExecutor.execute(() -> mListener.connectionUnavailable(
                UNAVAILABLE_REASON_IMS_UNSUPPORTED));
        logw("connect: not supported.");
        return;
    }
    // 创建 Framework 侧 Manager 例如：ImsManagerImpl / MmTelFeatureConnection
    synchronized (mLock) {
        if (mManager == null) {
            mManager = mFactory.createManager(mContext, mPhoneId);
        }
    }
    // 通过 Binder 把 mCallback 注册到 IMS Service
    mManager.registerFeatureCallback(mPhoneId, mCallback);
}
```

**和 reconnect() 是怎么配合的？**

```
调用方线程
   ↓
ImsManager.getInstance()
   ↓
ImsManager.reconnect()
   ├─ synchronized：创建 CountDownLatch
   ├─ mConnector.connect()        （开始异步）
   └─ await()                     （同步等待）

FeatureConnector.connect()
   ├─ createManager
   └─ registerFeatureCallback     （Binder 注册 mCallback）

IMS Service
   └─ 异步回调 Feature 相关状态，如：
        imsFeatureCreated
        imsStatusChanged
        imsFeatureRemoved

某一时刻 FeatureConnector.mCallback
   ├─ 状态机判断（READY / NOT_READY）
   └─ Executor → Listener 回调

ImsManager（Listener）
   ├─ 更新内部状态
   └─ mConnectedLatch.countDown() （唤醒 await）

reconnect() 返回
```

✅ reconnect() 是策略层

✅ FeatureConnector 是执行层

一个非常关键的认知点：FeatureConnector 本身是"异步"的， reconnect() 只是给它套了一层"有限同步等待"。

这也是为什么：

- reconnect() 可能返回
- 但 IMS 其实还没 ready
- 后面再慢慢 attach

---

### ImsService

见目录 4. Android 框架层：ImsService

---

### ImsManager.registerFeatureCallback()

```java
// xzy 注册 IMS Feature 回调（Framework → Telephony/IMS Service）
@Override
public void registerFeatureCallback(int slotId, IImsServiceFeatureCallback cb) {
    try {
        // ★ 关键阶段 1：
        // 监听 Framework 侧 callback Binder 的生死
        // 一旦 Binder 断开（进程挂 / 回调不可达）
        // 立即主动通知：IMS Feature 已不可用
        ITelephony telephony = mBinderCache.listenOnBinder(cb, () -> {
            try {
                // ★ 兜底失败回调，防止上层 await 永久阻塞
                cb.imsFeatureRemoved(
                        FeatureConnector.UNAVAILABLE_REASON_SERVER_UNAVAILABLE);
            } catch (RemoteException ignore) {}
        });

        if (telephony != null) {
            // ★ 关键阶段 2（核心动作）：
            // 真正通过 Binder 向 Telephony / IMS Service 注册
            // 将 slotId(phoneId) 对应的 IMS Feature 回调绑定进去
            telephony.registerMmTelFeatureCallback(slotId, cb);
        } else {
            // ★ 关键阶段 2-失败分支：
            // Telephony Service 尚未就绪或已死亡
            // 直接通知 IMS Feature 不可用
            cb.imsFeatureRemoved(
                    FeatureConnector.UNAVAILABLE_REASON_SERVER_UNAVAILABLE);
        }

    } catch (ServiceSpecificException e) {
        try {
            switch (e.errorCode) {
                case android.telephony.ims.ImsException.CODE_ERROR_UNSUPPORTED_OPERATION:
                    // ★ 关键阶段 3：
                    // 设备 / 构建不支持 IMS 能力
                    // 明确告知上层：无需重试，永久不可用
                    cb.imsFeatureRemoved(
                            FeatureConnector.UNAVAILABLE_REASON_IMS_UNSUPPORTED);
                    break;
                default:
                    // ★ 关键阶段 3-兜底：
                    // Service 抛出其他异常，按 Service 不可用处理
                    cb.imsFeatureRemoved(
                            FeatureConnector.UNAVAILABLE_REASON_SERVER_UNAVAILABLE);
            }
        } catch (RemoteException ignore) {
            // Binder 已死亡，无需再处理
        }

    } catch (RemoteException e) {
        // ★ 关键阶段 4（终极兜底）：
        // Binder 调用层级直接失败
        // 统一视为 IMS Service 不可用
        try {
            cb.imsFeatureRemoved(
                    FeatureConnector.UNAVAILABLE_REASON_SERVER_UNAVAILABLE);
        } catch (RemoteException ignore) {
            // Binder 已死亡，无需再处理
        }
    }
}
```

mCallback 会发送至 IMS Service ，根据状态回调

保证：

✅ 注册成功就能收到状态

✅ 注册失败一定立刻通知上层

✅ Binder 出事永不挂死调用方

---

### MmTelFeature

`frameworks\base\telephony\java\android\telephony\ims\feature\MmTelFeature.java`

MmTelFeature 属于 Framework（ims 模块），不是厂商私有类。是 ImsService 内部

承载"语音 / 视频 / 短信（VoLTE / ViLTE / SMS over IMS）能力的 Feature实例"， 是 Framework 认知中"可用 IMS 业务能力"的最小单元。

**MmTelFeature 是 Framework 视角下的"IMS 语音能力实体"，** 是 Telephony 操作 VoLTE / ViLTE / SMS over IMS 的唯一合法入口，而它之下发生的一切，Framework 都通过它来感知。

👉 **MmTelFeature READY ≠ IMS 注册成功**

MmTelFeature 不是"底层返回给 Framework 的对象"，而是由 ImsService（厂商实现）在 Framework 侧主动创建并提供的。

`frameworks/opt/net/ims/src/java/com/android/ims/ImsManager.java`

```
ImsManager.getInstance()
  ↓
InstanceManager.reconnect()
  ↓
FeatureConnector.connect()
  ↓
ImsManager.registerFeatureCallback() 后续某一时刻触发回调
  ↓
Telephony / ImsService Binder
  ↓
MmTelFeature (ImsService 内)
  ↓
==================== Framework 边界 ====================
  ↓
厂商 IMS 实现（Java / JNI / native）
  ↓
imsdaemon / modem / IMS stack
```

**它的职责可以拆成三块：**

**① 向 Framework"声明能力"**

MmTelFeature 对 Framework 来说，等价于：

当前 slot / phoneId 上，是否支持：

- VoLTE
- ViLTE
- SMS over IMS
- Emergency over IMS"

这些能力通过：

- getCapabilities()
- queryCapabilityConfiguration()
- notifyCapabilitiesStatusChanged()

向上汇报。

---

**② 承载 IMS 业务接口（非常关键）**

所有真正的 IMS 业务接口，都挂在 MmTelFeature 上：

- 创建呼叫会话
- 设置语音 / 视频开关
- 处理 IMS 短信
- 紧急呼叫

Framework 侧看到的是：

`IImsMmTelFeature` (Binder 接口)

**App / Telephony 永远不会直接操作 imsdaemon，它们只和 MmTelFeature 的 Binder 接口打交道。**

---

**③ 把"厂商实现"包进 Framework 形态**

MmTelFeature 本身：

- ✅ 定义了标准接口
- ✅ 定义了状态机
- ✅ 定义了回调模型

但：真正的逻辑几乎一定在厂商 override 的子类里

比如：

```java
class VendorMmTelFeature extends MmTelFeature {
    @Override
    public ImsCallSessionImpl createCallSession(...) {
        // 转给 native / imsdaemon
    }
}
```

---

**MmTelFeature 和 ImsService 的关系是：**

> **ImsService 是"容器 / 管理者"，** MmTelFeature 是"具体能力实例"。**

**类比一下（非常贴切）：**

| 概念 | 类比 |
|------|------|
| ImsService | USB Host |
| MmTelFeature | USB 设备（键盘 / 鼠标） |
| Feature | 一种功能类型 |
| FeatureConnector | USB 线 + 驱动 |

Android 的 IMS 是**多 Feature 设计**：

- MmTelFeature（语音 / 视频 / 短信）
- RcsFeature（消息 / Presence）
- 未来可能还有其他 Feature

👉 所以设计成：

```
ImsService
   ├── MmTelFeature
   ├── RcsFeature
   └── ...
```

---

### Ims 与 Slot 配置的关联

Slot 详见卡槽（slot）配置相关。

ImsManager 并不负责创建通信能力。所有 IMS 操作都建立在 Framework 已经成功创建并稳定运行的 slot（Phone）之上。

在 Framework 中，phoneId 本质是 slotid，因为phone是由 PhoneFactory 在系统启动阶段，根据 slotid 创建的

**slot 数量代表设备声明的通信能力，而非插卡状态。**

每一个 IMS Feature（如 MmTelFeature）都是按 slot 创建并存在的，Framework 通过 phoneId 将 Telephony、IMS Service 与底层能力进行关联。

一旦 slot / RIL / Radio HAL 初始化失败，SystemServer 将在启动阶段发生 ANR，在此情况下，ImsManager 永远不可能进入 ready 状态。

**关联图如下：**

```
Phone / RIL / Radio HAL  (卡槽篇)
  ↑
  │ phoneId (slotIndex)
  ↓
ImsManager / ImsService / MmTelFeature （本篇）
```

phoneId 作为 key，用于按需获取（或创建并缓存）与该 slot 关联的 ImsManager 访问入口。

ImsManager 再通过异步绑定的方式获取对应 slot 的 IMS Feature（如 MmTelFeature）。

**可以把 Telephony 和 IMS 想成两个互不认识的部门，**

**Telephony 只给 IMS 一张"工位号"（phoneId），**

**至于这个工位后面有没有人、设备齐不齐，**

**IMS 并不知道，也不会去检查，**

**它只负责"既然给了工位号，我就按这个号干活"。**

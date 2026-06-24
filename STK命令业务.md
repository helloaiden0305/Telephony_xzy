# STK 命令业务

## 1. 什么是STK？

在移动通讯领域，**STK (SIM Tool Kit)** 是一种标准，它允许SIM卡（或者更广义地说，UICC卡）与手机进行交互，并向用户提供各种增值服务。简单来说，STK就是SIM卡上的一个"小程序"平台，运营商可以通过它向用户推送菜单、消息、进行身份验证、启动浏览器等操作。

日常工作中，当提到STK时，通常是指由SIM卡触发的，与用户界面交互相关的业务流程和命令。

---

## 2. STK的工作原理与流程

### 2.1 STK命令的预设位置

STK命令并非预设在手机的操作系统或应用程序中，而是**预设在SIM卡（或UICC卡）内部**。SIM卡作为一张智能卡，包含了一个微型操作系统和应用程序（即STK应用）。运营商在发行SIM卡时，会将这些STK应用和相关的命令逻辑写入卡中。

### 2.2 STK命令的类型：主动上报与被动读取

STK命令可以分为两大类：

**Proactive Commands (主动命令/上报)：**

- 这类命令是由SIM卡**主动发起**的，旨在与手机用户进行交互或请求手机执行特定操作。
- 例如：`DISPLAY_TEXT` (显示文本)、`GET_INPUT` (获取用户输入)、`LAUNCH_BROWSER` (启动浏览器)、`SEND_SMS` (发送短信) 等。
- 当SIM卡需要执行某个操作时，它会生成一个主动命令并发送给手机。手机接收到后，会根据命令类型在用户界面上显示相应内容或执行相应操作。
- 对话中遇到的"锁屏预警通知"、"弹窗"和"LAUNCH_BROWSER"都是典型的 STK 主动命令触发的。

**Envelope Commands (信封命令/被动读取)：**

- 这类命令是由手机**被动发起**的，用于向SIM卡发送信息或请求SIM卡执行某些操作。
- 例如：手机向SIM卡报告网络状态、电池状态、位置信息等，或者请求SIM卡执行特定的文件操作。
- 这些命令通常不直接涉及用户界面交互，而是作为后台通信的一部分。

### 2.3 STK命令的通讯路径：SIM -> Modem -> RIL -> Android Framework -> Application

STK命令从SIM卡到手机应用，再到用户界面的完整通讯路径如下：

1. **SIM卡生成STK命令：**
   SIM卡内部的STK应用根据预设逻辑或特定事件（例如开机、网络注册、收到特定短信等）生成一个 Command。

2. **SIM卡将命令发送给Modem：**
   SIM卡通过其接口（通常是ISO 7816协议）将STK命令发送给手机的基带处理器（Modem）。Modem负责处理所有无线通信。

3. **Modem通过RIL上报给Android Framework：**
   - Modem接收到STK命令后，会通过**RIL (Radio Interface Layer，无线电接口层)** 将其上报给Android操作系统的上层。
   - RIL是Android系统与基带Modem之间的抽象层，它定义了一套标准接口，允许 Android Framework 与 Modem进行通信，而无需关心底层 Modem 的具体实现细节。
   - STK命令通常会通过RIL事件（如`RIL_UNSOL_STK_PROACTIVE_COMMAND`）的形式上报。

4. **Android Framework处理STK命令：**
   - Android Framework中的`TelephonyManager`、`TelecomManager`、`StkAppService`等组件会接收并解析RIL上报的STK命令。
   - Framework会根据命令的类型和内容，决定如何处理：
     - 如果是`DISPLAY_TEXT`，则可能通过通知管理器显示通知。
     - 如果是`LAUNCH_BROWSER`，则会生成一个Intent来启动浏览器应用。
     - 如果是`GET_INPUT`，则会启动一个对话框让用户输入信息。

5. **STK应用程序/系统UI响应：**
   - 最终，STK命令会被分发到相应的应用程序或系统UI组件进行处理。例如，`StkDialogActivity`或`NotificationManager`会负责显示弹窗或通知。
   - 在我们的对话中，`com.hihonor.contacts`（拨号应用）和`StkDialogActivity`就是处理STK命令的典型应用。

---

## 3. 关键类和核心流程的调用

提到了TelephonyManager、TelecomManager、StkAppService、StkDialogActivity、NotificationManager这些组件。

- 例如，当用户反馈"弹窗显示不正确"时，您需要知道StkDialogActivity是负责显示弹窗的，然后去检查它的相关代码和配置。
- 当用户反馈"通知优先级不对"时，您需要知道NotificationManager是管理通知的，然后去检查通知的importance和visibility设置。
- 当遇到Modem层面的问题时，了解RIL接口和RIL_UNSOL_STK_PROACTIVE_COMMAND这样的事件，能够帮助您在Modem log中定位问题。

---

## 4. STK业务常见问题案例分析

### 案例一：锁屏预警通知问题

**问题描述：**
在锁屏状态下，用户会收到来自STK的预警通知。这个通知的显示方式可能不符合预期，例如，它可能以较高的优先级显示，或者显示了本应隐藏的私密信息。

**原因分析：**
STK通过**Proactive Command**（例如`DISPLAY_TEXT`）将通知内容发送给手机。手机的`NotificationManager`接收到后，会根据STK命令中携带的通知属性（如`importance`和`visibility`）来决定通知的显示方式。

- `importance = 3` 通常表示"中等重要性"或更高，这可能导致通知在锁屏上以更显眼的方式出现。
- `vis = PRIVATE` 可能意味着通知内容在锁屏上不应该完全公开，但其显示方式可能仍需调整。

如果STK默认设置为较高的重要性或不恰当的可见性，就可能导致预警通知在锁屏上显示得过于突出或暴露了不应显示的信息。

**解决方案建议：**
根据产品需求调整STK通知的属性：

- 如果锁屏不应显示此类预警通知，应将`importance`设置为低于`3`的值（例如`importance < 3`）。
- 同时，要明确产品对这类通知在锁屏状态下的显示策略。

### 案例二：LAUNCH_BROWSER场景多发DISPLAY_TEXT命令

**问题描述：**
在进行运营商测试时，发现在锁屏状态下执行"启动浏览器 (LAUNCH_BROWSER)"的STK用例时，系统会额外发送一条"显示文本 (DISPLAY_TEXT)"的STK命令。而在亮屏状态下执行相同用例则无此异常。同时，其他对比机型（如LDY）在相同场景下也没有此额外命令上报。

**原因分析：**
平台为了**适配谷歌对锁屏状态下"启动浏览器 (LAUNCH_BROWSER)"的禁用策略**，**故意**在锁屏LAUNCH_BROWSER场景中，额外发送了一条"显示文本 (DISPLAY_TEXT)"命令。这是一种为了绕过谷歌禁用而采取的**特殊处理方式**。

---

## 5. 总结

STK业务作为运营商提供增值服务的重要途径，其在不同场景下的表现对用户体验和系统稳定性至关重要。作为通讯应用的新手，理解STK命令的**预设位置（SIM卡）**、**主动/被动类型**以及**通讯路径（SIM -> Modem -> RIL -> Android Framework -> Application）**。

在处理STK相关问题时，需要与应用团队和测试团队保持紧密沟通，共同分析问题。

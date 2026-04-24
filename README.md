============================================================
DxO ONE APK 修改说明(仅个人学习研究使用)
DxO ONE APK Modification Notes (For Personal Research and Study Only)
============================================================

修改目的：解决 Android 16 无法通过 WifiManager.getConnectionInfo().getSSID()
检测系统 WiFi SSID 的问题，导致应用卡在"连接到dxo-one..."界面。

============================================================
一、完整的连接流程（修改后）
============================================================

[用户点击"直接连接"]
    ↓
[WifiConnectionActivity] 启动，绑定 WifiConnectionService
    ↓
[WifiConnectionService] 状态: IDLE → WIFI_START
    ↓
[WifiStartState.onCreate()]
    ├── USB已连接 (CommunicationService.CONNECTED)
    │   └── 绑定 CameraService
    │       └── 通过USB发送命令: 相机开启 dxo-one 热点
    │           └── 相机返回 WifiCredentialsAcceptanceStatusEvent
    │               ├── 成功 → AUTO_SSID_CONNECTION
    │               └── 失败 → RECOVERABLE_ERROR
    │
    └── USB未连接
        └── 直接进入 AUTO_SSID_CONNECTION
    ↓
[AUTO_SSID_CONNECTION]
    ↓
[AutoSsidConnectionState.onCreate()]
    ├── 获取保存的 SSID (AppSettings.getWifiLastSsid)
    ├── 绑定网络状态广播 (android.net.wifi.STATE_CHANGE)
    └── 调用 connectToAccessPoint()
    ↓
[connectToAccessPoint()] —— Android 16 卡住的位置
    ├── 检查 WiFi 是否开启
    │   └── 关闭 → MANUAL_SSID_CONNECTION
    ├── getConnectionInfo().getSSID() ← Android 16返回null或<unknown ssid>
    │   ├── 匹配目标SSID 或 <unknown ssid> → ENDPOINT_CONNECTION ✓
    │   └── 不匹配 → 重试逻辑
    │       ├── 重试次数 <= 99 (修改前是3) → disconnect/addNetwork/reconnect
    │       └── 重试次数 > 99 → MANUAL_SSID_CONNECTION
    ↓
[onNetworkStateChange(WifiInfo)] —— WiFi状态变化回调
    ├── SSID 匹配 或 <unknown ssid> → ENDPOINT_CONNECTION ✓
    ├── 不匹配且重试<99次 → connectToAccessPoint() 再次尝试
    └── 不匹配且重试>=99次 → MANUAL_SSID_CONNECTION
    ↓
[ENDPOINT_CONNECTION] ← 用户点击按钮后进入这里
    ↓
[EndpointConnectionState.onCreate()]
    ├── 如果 CommunicationService 未 CONNECTED
    │   └── connectToWifiEndpoint()
    │       ├── 创建 WifiEndpoint (携带 WiFi token)
    │       └── CommunicationService.setMode(WIFI, endpoint) ← 建立通信端点
    │
    └── 如果已 CONNECTED
        └── 绑定 CameraService
    ↓
[EndpointConnectionState.onEvent(DeviceConnectionStateEvent)]
    ├── 连接成功 且 无USB → CONNECTION_ESTABLISHED
    ├── 连接成功 且 有USB → IDLE
    ├── 连接失败 且 无USB → RECOVERABLE_ERROR
    └── 其他 → UNRECOVERABLE_ERROR
    ↓
[CONNECTION_ESTABLISHED]
    ↓
[WifiConnectionActivity]
    ├── 播放断开USB动画 (performUnplugAnimation)
    └── 动画结束后 leaveConnectionActivity()
        ├── CommunicationService.CONNECTED → goToCapture() → 拍摄界面 ✓
        └── 其他 → goToMainActivity() ❌


============================================================
二、所有修改的代码
============================================================
2026.4.24新增bug修复：
1. 修复情况2：WiFi 没打开时卡住修改了 AutoSsidConnectionState 中 connectToAccessPoint() 的逻辑。原始代码在检测到 WiFi 关闭时，
直接跳转到 MANUAL_SSID_CONNECTION（手动连接提示界面），并且没有【已连接，进入下一步】按钮，导致卡住。修复：现在 WiFi 关
闭时，App 会自动调用 setWifiEnabled(true) 尝试打开 WiFi，然后继续自动连接流程，而不是进入手动模式。
2. 修复情况3：WiFi 打开但没连接网络时直接跳到"您现在可以远程控制..."并卡住这是两个 bug 叠加导致的：
Bug A：当 WiFi 打开但未连接任何网络时，WifiManager.getConnectionInfo().getSSID() 会返回 <unknown ssid>。原始代码把 <unknown ssid> 
当作"已连接到相机热点"处理，错误地直接进入 ENDPOINT_CONNECTION 状态。修复：删除了这个错误的特殊处理，现在即使返回 <unknown ssid>，
也会继续尝试连接相机热点。Bug B：进入 ENDPOINT_CONNECTION 后，如果端点连接失败（isConnected() == false）且断开原因不是 CONNECTION_FAILED，
原始代码直接 return，不做任何状态转换，导致状态永远停留在 ENDPOINT_CONNECTION，界面卡住。
修复：删除了这个 return，现在连接失败时会正确进入 RECOVERABLE_ERROR（可恢复错误）状态，用户可以点击重试按钮重新连接。
3. 修复连接成功后的跳转问题（之前的修改）
EndpointConnectionState：连接成功后不再检查 USB 设备，无条件设为 IDLE 状态。
WifiConnectionActivity：CONNECTION_ESTABLISHED 状态也会触发跳转到预览控制界面。
预期效果
现在三种情况应该都能正常工作了：
情况1（WiFi 已连接相机热点）：保持原有行为，正常工作。
情况2（WiFi 没打开）：App 会自动打开 WiFi，然后自动连接相机热点。
情况3（WiFi 打开但没连网络）：App 会正确尝试连接相机热点，而不是误判为已连接。
============================================================
2026.4.23日晚修改内容:
1.恢复原闪光灯图标显示(闪光灯auto按钮还是隐藏，因为没用)
2.隐藏测试时的闪光灯开关按钮，将功能映射到原闪光灯按钮开关。
============================================================
修改涉及3个文件：
1. WifiConnectionService.smali
2. WifiConnectionActivity.smali
3. AutoSsidConnectionState.smali

------------------------------------------------------------
文件1：WifiConnectionService.smali
位置：smali/com/dxo/one/app/wifi/connection/WifiConnectionService.smali
------------------------------------------------------------

【新增 manualConnect() 方法】（在 retry() 方法之后）

.method public final manualConnect()V
    .locals 1

    sget-object v0, Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;->ENDPOINT_CONNECTION:Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;

    invoke-direct {p0, v0}, Lcom/dxo/one/app/wifi/connection/WifiConnectionService;->setState(Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;)V

    return-void
.end method

【修改说明】
用户点击按钮后调用此方法，将状态设为 ENDPOINT_CONNECTION，
让 EndpointConnectionState 执行真正的WiFi端点建立逻辑（connectToWifiEndpoint()）。

------------------------------------------------------------
文件2：WifiConnectionActivity.smali
位置：smali/com/dxo/one/presentation/wifi/WifiConnectionActivity.smali
------------------------------------------------------------

【修改点A：onRetryButton()】

.method private final onRetryButton(Landroid/view/View;)V
    .locals 2

    iget-object p1, p0, Lcom/dxo/one/presentation/wifi/WifiConnectionActivity;->wifiConnectionServiceConnector:Lcom/dxo/infrastructure/utils/ServiceConnector;

    invoke-virtual {p1}, Lcom/dxo/infrastructure/utils/ServiceConnector;->getService()Landroid/app/Service;

    move-result-object p1

    check-cast p1, Lcom/dxo/one/app/wifi/connection/WifiConnectionService;

    if-eqz p1, :cond_0

    invoke-virtual {p1}, Lcom/dxo/one/app/wifi/connection/WifiConnectionService;->getState()Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;

    move-result-object v0

    sget-object v1, Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;->AUTO_SSID_CONNECTION:Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;

    if-ne v0, v1, :cond_retry

    invoke-virtual {p1}, Lcom/dxo/one/app/wifi/connection/WifiConnectionService;->manualConnect()V

    goto :cond_done

    :cond_retry
    invoke-virtual {p1}, Lcom/dxo/one/app/wifi/connection/WifiConnectionService;->retry()V

    :cond_done
    :cond_0
    return-void
.end method

【修改说明】
点击重试按钮时，先判断当前状态。如果是 AUTO_SSID_CONNECTION，
调用 manualConnect() 进入端点连接；否则执行原来的 retry()。


【修改点B：onWifiConnectionStateChange() 中 retry 按钮的可见性和文本】

    sget-object v4, Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;->RECOVERABLE_ERROR:Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;

    if-eq p1, v4, :cond_4

    sget-object v4, Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;->AUTO_SSID_CONNECTION:Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;

    if-ne p1, v4, :cond_4

    const/4 v5, 0x0

    :cond_4
    invoke-virtual {v3, v5}, Landroid/widget/Button;->setVisibility(I)V

    sget-object v4, Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;->AUTO_SSID_CONNECTION:Lcom/dxo/one/app/wifi/connection/WifiConnectionService$State;

    if-ne p1, v4, :cond_auto_text

    const-string v4, "\u5df2\u8fde\u63a5\uff0c\u8fdb\u5165\u4e0b\u4e00\u6b65"

    invoke-virtual {v3, v4}, Landroid/widget/Button;->setText(Ljava/lang/CharSequence;)V

    goto :cond_auto_done

    :cond_auto_text
    const v4, 0x7f0f00d0

    invoke-virtual {v3, v4}, Landroid/widget/Button;->setText(I)V

    :cond_auto_done

【修改说明】
- 可见性：AUTO_SSID_CONNECTION 状态下也显示 retry 按钮
  （原来是只在 RECOVERABLE_ERROR 显示）
- 文本：AUTO_SSID_CONNECTION 状态下按钮显示"已连接，进入下一步"
  （硬编码中文字符串）；其他状态显示原始文本 0x7f0f00d0

------------------------------------------------------------
文件3：AutoSsidConnectionState.smali
位置：smali/com/dxo/one/app/wifi/connection/states/AutoSsidConnectionState.smali
------------------------------------------------------------

【修改点A：connectToAccessPoint() 中的重试上限】

    :cond_4
    iget v2, p0, Lcom/dxo/one/app/wifi/connection/states/AutoSsidConnectionState;->numRetries:I

    const/4 v4, 0x1

    add-int/2addr v2, v4

    iput v2, p0, Lcom/dxo/one/app/wifi/connection/states/AutoSsidConnectionState;->numRetries:I

    iget v2, p0, Lcom/dxo/one/app/wifi/connection/states/AutoSsidConnectionState;->numRetries:I

    const/16 v5, 0x63       ← 修改前是 const/4 v5, 0x3

    if-le v2, v5, :cond_5

【修改点B：onNetworkStateChange() 中的重试上限】

    :cond_2
    iget p1, p0, Lcom/dxo/one/app/wifi/connection/states/AutoSsidConnectionState;->numRetries:I

    const/16 v0, 0x63       ← 修改前是 const/4 v0, 0x3

    if-ge p1, v0, :cond_3

【修改说明】
两处重试上限都从 3 改为 99（0x63），防止应用在用户还没来得及
点击按钮前就自动跳转到 MANUAL_SSID_CONNECTION。

============================================================
三、其它修改
============================================================
WiFi 连接后界面边缘显示绿色线框（USB 提示动画）且不会自动消失。修改文件：
1. WifiConnectionActivity.smali
在 AUTO_SSID_CONNECTION 状态下隐藏绿框：原代码中 wifi_connection_unplug_animated_view（绿框视图）在某些状态下一直显示。
修改后在 AUTO_SSID_CONNECTION 状态将其设为隐藏（visibility=0）。在 ENDPOINT_CONNECTION 状态调用 setShowIndicator(false)：
获取根布局 WifiIndicatorLayout 并调用 setShowIndicator(false)，停止绿框绘制。
2. WifiIndicatorLayout.smali（尝试后撤销）
曾尝试在 onDraw() 中增加 showIndicator 字段检查，但因导致闪退而撤销。最终通过 WifiConnectionActivity 中调用 setShowIndicator(false) 来隐藏绿框。
3. 闪光灯菜单控制
修改闪光灯相关 Smali 代码，使闪光灯菜单默认显示或改变其可见性逻辑。
4. 功能按钮迁移至设置面板
在 Smali 层面将某些功能按钮（可能与闪光灯或拍摄设置相关）移动到 CaptureSettingsFragment（设置面板）中。
5. App 闪退修复
修复了某些场景下的闪退问题（可能与绿框显示或 WiFi 连接状态切换相关）。
============================================================
三、修改后的使用流程
============================================================

1. USB连接相机，打开App，点击【直接连接】
2. App通过USB命令让相机开启 dxo-one 热点
3. 【手动连接手机到 dxo-one 热点】（系统设置里操作）
4. App界面显示"连接到dxo-one..."，下方出现【已连接，进入下一步】按钮
5. 点击按钮 → 状态变为 ENDPOINT_CONNECTION
   → connectToWifiEndpoint() 建立WiFi通信端点
   → 通信服务连接相机
   → 成功 → CONNECTION_ESTABLISHED → 进入拍摄界面


============================================================
四、重要提示
============================================================

1. 安装前必须卸载原App（签名不同，无法覆盖安装）
2. 点击按钮前必须确保手机真的已经连上了 dxo-one 热点

============================================================
五、打包来源文件清单
============================================================

- AndroidManifest.xml      (解码后的清单文件)
- apktool.yml              (apktool打包配置)
- resources.arsc           (原始二进制资源表)
- lib/                     (原生库，4个架构)
- smali/                   (反编译后的Smali代码)
- unknown/                 (Kotlin元数据、play-services配置、原始res)
- build/apk/               (apktool临时文件，无需备份)

主要修改的文件只有3个：
1. smali/com/dxo/one/app/wifi/connection/WifiConnectionService.smali
2. smali/com/dxo/one/presentation/wifi/WifiConnectionActivity.smali
3. smali/com/dxo/one/app/wifi/connection/states/AutoSsidConnectionState.smali

============================================================
测试时增加的功能:启用对手机闪光灯的控制，添加闪光灯延迟功能(点击切换延迟ms)

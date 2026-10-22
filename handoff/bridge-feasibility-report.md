# CarWith Bridge 靜態可行性報告

日期：2026-10-22

## 範圍與身份

- 唯一分析目標：`/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk`
- SHA-256：`ec213a90b9900d2d54b379ca8abf9c39b1556b86afec3e82a259c2bf9cee5bed`
- 分析方式：REA CLI + JADX headless 0.7.1；只做靜態分析，未執行 APK。
- 未分析 MyCadillac 或 OcarPlay；未修改、簽署、安裝 APK，也未連接手機或車機。

## 結論摘要

1. **Bridge 有自己的 App 入口和投屏顯示機制。** Manifest 有 launcher Activity 與 `CarlifeConnectService`；Bridge 的 Surface callback 會建立 `VirtualDisplay` 並顯示 Presentation。這支持「Bridge 自身包含投屏接收／呈現部分」，但不等於已證明完整 CarPlay 鏈路可單獨工作。
2. **MyCadillac 不提供 Bridge 所綁定的 service，且看起來是同 package 的另一個版本。** 經使用者批准的 manifest-only 檢查顯示 MyCadillac APK package 同樣是 `com.baidu.carlife.xiaomi`，版本為 `8.1.1-carplay.B`；其 manifest 沒有宣告 `com.miui.carlink/.CarlifeConnService`，只將 `com.miui.carlink` 放在 `<queries>`。Bridge 自己版本為 `574d7d2-11`，並明確 bind 該外部 component。因此，MyCadillac APK 不是這個 service 的提供者；一般 Android 安裝下，兩個相同 package name 的 APK 也不能作為兩個獨立 app 同時安裝，較像不同版本／變體。Bridge 仍依賴裝置上另一個 `com.miui.carlink` service。
3. **USB 連線尚未證實可獨立於 Bluetooth。** Service 接受小米 CarLink 的 `usb_type` 等 metadata，但其連線 coroutine 也讀取有效 Bluetooth MAC、要求 Bluetooth 開啟，再進入連線控制器。這可能是協定配對／控制所需，不能單憑靜態資料斷言 USB 不可用。
4. **Bridge 已有實際 PCM callback 處理路徑，不只是音頻介面宣告。** `ISystemAudioCallback` 收到 PCM 後會驗證格式並送入 Bridge 的音頻管線；另一方向也有向 `IRemoteAudioListener` 傳送 PCM 的程式。這表示待解問題未必是「完全沒有音頻程式碼」，更可能在音頻來源、CarLife/CarPlay 協商、路由或啟用條件；靜態證據不能定位實機故障點。
5. **目前不應直接編譯。** 尚未確定 Bridge 是否為唯一必要 APK，也沒有明確的 patch 範圍、重簽方案或 Android 相容性驗證。環境未找到 `apktool`、`apksigner`、`zipalign`；目前只找到 `jarsigner` shim，尚不足以確認可產出可安裝且相容的 APK。

## 發現與 Evidence

| 問題 | 靜態觀察與判斷 | Evidence ID |
|---|---|---|
| APK 身份與入口 | package 為 `com.baidu.carlife.xiaomi`；有 Bridge launcher Activity `dev.carwith.bridge.ui.activity.MainActivity` 和 exported `CarlifeConnectService`。自身入口存在，不代表所有連線依賴都在 APK 內。 | `ev_f74227ab995c6387dda7d79f1c77baac00cd67ebfdb6f9a42fbd03df3d216ddb` |
| MyCadillac APK 關係 | MyCadillac APK package 也是 `com.baidu.carlife.xiaomi`，version `8.1.1-carplay.B`；manifest 沒有宣告 `com.miui.carlink/.CarlifeConnService`。這是 manifest-only 檢查，未讀其程式碼。相同 package name 表示兩個 APK 看起來是替代版本，不是標準 Android 安裝下的兩個並存套件。 | `ev_b12984ee12ac7f422adb0391ed21164b7ca13ba8e5e6a7c5c645e00fb51ae4c8` |
| 投屏呈現 | Bridge 接收 Surface、尺寸與 frame rate；有效 Surface 上建立名為 `CarWith & Carplay Bridge` 的 `VirtualDisplay`，再顯示 Presentation。這是 app 內的呈現端證據，不是 runtime 或完整投屏鏈路驗證。 | `ev_a9e26fad0de95a97acc2ba92b0bc07dbc1ac8fe115415244ffdf925c694f1b2d`, `ev_cb9d148f15066ff68975e7d6c76bda55da94e5081b3017e85e0712f38592cf24` |
| 外部 service 依賴 | coroutine 對 `com.miui.carlink/com.xiaomi.ucar.carlife.CarlifeConnService` 呼叫 `bindService`；綁定後驗證 `ICarLifeCastManager` descriptor，註冊 cast callback 與 system-audio callback。MyCadillac 是否提供此 component 未知。 | `ev_98a2d1a18086178c60aaf7ee2866f4f556551dd4ba6e5af4012a6231c746ce0c`, `ev_149a414997ce5dcc51fcf91d88cc80aeff13bb6cead285a9e52dc99a8644042a`, `ev_54799bdb31c982e458742b8d9d005425a02ecbf2096b16d94e70e28a8bbb40b4` |
| 連線啟動 | Service 有 Bridge 自身的 `CONNECT`/`STOP` action；另外支援 `com.miui.carlink`、`CarlifeConnService`、`manufacturer=xiaomi`、`usb_type` 範圍與 `is_box` 等參數。啟動連線時，程式仍進入控制器流程。 | `ev_785369aed25b6c880c0ad18b9b6efea501632d96168226e5febf19bbf626f35e` |
| Bluetooth 條件 | 連線 coroutine 驗證 Bluetooth MAC，並檢查 Bluetooth adapter 是否啟用；零 MAC／占位 MAC 會被拒絕。這與 USB metadata 同時存在，因此 USB 是否需要 Bluetooth 配對仍未解。 | `ev_892d2fcb370f11e082fd20bf815388214be5831d8306ddcb4295ef6a834c16c7` |
| 系統音頻接收 | `onAudioStreamReady(byte[], …)` 驗證 PCM 格式、聲道／取樣率／資料長度，再把資料送入 `bf0.l(...)` 音頻緩衝管線。這是有 PCM 資料流的靜態證據；下游路由與實際車機播放尚未 runtime 驗證。 | `ev_648f1dcbf8cb55ca54855687851551cbdf8f35b4fefa7158954ec6cbfa203a4e`, `ev_fb604f462fbd2696f8f58fe0c168d25c0e37b814c7d56106f5f67ef189d48210` |
| 手機麥克風上行 | CarLife callback 可注入 `IRemoteAudioListener`；`vl.q` 將 PCM byte array 透過 `onReceiveAudioStream` 發送給該 listener。此處顯示的是手機音頻上行 callback，不能當成音樂下行已成功。 | `ev_65ad5ce40f8f5e9e14510aba500c3b2c3c7c25893dbe6b2f310e2c41bd3ae1bf`, `ev_3cbf77f7f3c068d9acd9dfb158d10c22b41c24765b903d9ee11c9729e07e5d31` |

## 已知限制

- JADX 靜態反編譯不能證明任何路徑在 Redmi K100 Pro Max / HyperOS 4 或 XPENG G6 上實際成功。
- APK provider 未驗證簽名、split APK、native libraries 或 Android runtime capture。
- 某些混淆 coroutine 的 decompiler 輸出有警告；本報告對關鍵入口優先採用可讀方法或 Smali 證據，不把巨大或失真的控制流當成確定語義。
- MyCadillac 僅檢查 manifest；未檢查其程式碼或執行行為。Manifest 未宣告 Bridge 所綁 service，因此不能把它視為該 service 的提供者。
- 未確認簽名、privileged 權限、原始安裝來源及重簽後與既有安裝的更新相容性。

## 建議的下一步

MyCadillac manifest 不提供 Bridge 所需的外部 service，無需為此再反編譯其程式碼。下一步應只在 Bridge 內追查音頻來源／播放路由和有線連線控制器，確認目前的 PCM 路徑為何沒有抵達 XPENG G6，以及是否存在可局部修補的缺口。確認具體修改點後，再提出精確 patch 範圍和簽署／輸出方式供審閱。在此之前不改檔、不重建、不安裝 APK，也不安裝額外 APK 工具。

# CarWith Bridge 靜態可行性報告

日期：2026-10-22

## 範圍與身份

- 唯一分析目標：`/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk`
- SHA-256：`ec213a90b9900d2d54b379ca8abf9c39b1556b86afec3e82a259c2bf9cee5bed`
- 分析方式：REA CLI + JADX headless 0.7.1；只做靜態分析，未執行 APK。
- MyCadillac 僅檢查 manifest，未分析其程式碼；未分析 OcarPlay。未修改、簽署、安裝 APK，也未連接手機或車機。

## 結論摘要

1. **Bridge 有自己的 App 入口和投屏顯示機制。** Manifest 有 launcher Activity 與 `CarlifeConnectService`；Bridge 的 Surface callback 會建立 `VirtualDisplay` 並顯示 Presentation。這支持「Bridge 自身包含投屏接收／呈現部分」，但不等於已證明完整 CarPlay 鏈路可單獨工作。
2. **MyCadillac 不提供 Bridge 所綁定的 service，且看起來是同 package 的另一個版本。** 經使用者批准的 manifest-only 檢查顯示 MyCadillac APK package 同樣是 `com.baidu.carlife.xiaomi`，版本為 `8.1.1-carplay.B`；其 manifest 沒有宣告 `com.miui.carlink/.CarlifeConnService`，只將 `com.miui.carlink` 放在 `<queries>`。Bridge 自己版本為 `574d7d2-11`，並明確 bind 該外部 component。因此，MyCadillac APK 不是這個 service 的提供者；一般 Android 安裝下，兩個相同 package name 的 APK 也不能作為兩個獨立 app 同時安裝，較像不同版本／變體。Bridge 仍依賴裝置上另一個 `com.miui.carlink` service。
3. **USB 連線尚未證實可獨立於 Bluetooth。** Service 接受 `usb_type` 為 0 或 1 的小米 CarLink metadata，符合條件後將模式旗標交給連線控制器。controller coroutine 中可見 Bluetooth MAC 驗證與 adapter 開啟檢查，但它們受另一個設定旗標控制；現有證據未完整追清 USB 模式旗標在後續 coroutine 的語義。因此不能斷言 USB 一定需要 Bluetooth，也不能斷言 USB 可免藍牙。
4. **Bridge 有音頻 callback contract 與編碼／封包元件，但端到端資料路徑未閉合。** Bridge 註冊 `ISystemAudioCallback`，其 API 提供 `onAudioStreamReady(byte[], …)`；另有接收音頻 frame、MediaCodec 處理及 packetizer 的程式碼。現有靜態證據尚未把 system-audio callback 的 bytes 連到可確認的音頻播放／網路輸出消費端，也未證明這些分支在目標連線中被呼叫或資料抵達車機。因此目前不應另造音頻管線或猜測修補點。
5. **目前不應直接編譯。** Manifest 顯示 MyCadillac 不是 Bridge 所綁定 service 的提供者，且與 Bridge 同 package，較像替代版本；Bridge 的完整 runtime 能力尚未驗證。尚無已定位的 patch 範圍或重簽相容性結論。環境未找到 `apktool`、`apksigner`、`zipalign`；目前只找到 `jarsigner` shim，尚不足以確認可產出可安裝且相容的 APK。

## 發現與 Evidence

| 問題 | 靜態觀察與判斷 | Evidence ID |
|---|---|---|
| APK 身份與入口 | package 為 `com.baidu.carlife.xiaomi`；有 Bridge launcher Activity `dev.carwith.bridge.ui.activity.MainActivity` 和 exported `CarlifeConnectService`。自身入口存在，不代表所有連線依賴都在 APK 內。 | `ev_f74227ab995c6387dda7d79f1c77baac00cd67ebfdb6f9a42fbd03df3d216ddb` |
| MyCadillac APK 關係 | MyCadillac APK package 也是 `com.baidu.carlife.xiaomi`，version `8.1.1-carplay.B`；manifest 沒有宣告 `com.miui.carlink/.CarlifeConnService`。這是 manifest-only 檢查，未讀其程式碼。相同 package name 表示兩個 APK 看起來是替代版本，不是標準 Android 安裝下的兩個並存套件。 | `ev_b12984ee12ac7f422adb0391ed21164b7ca13ba8e5e6a7c5c645e00fb51ae4c8` |
| 投屏呈現 | Bridge 接收 Surface、尺寸與 frame rate；有效 Surface 上建立名為 `CarWith & Carplay Bridge` 的 `VirtualDisplay`，再顯示 Presentation。這是 app 內的呈現端證據，不是 runtime 或完整投屏鏈路驗證。 | `ev_a9e26fad0de95a97acc2ba92b0bc07dbc1ac8fe115415244ffdf925c694f1b2d`, `ev_cb9d148f15066ff68975e7d6c76bda55da94e5081b3017e85e0712f38592cf24` |
| 外部 service 依賴 | Bridge 的連線入口使用 `com.miui.carlink` 與 `com.xiaomi.ucar.carlife.CarlifeConnService` 資訊；CarLife manager client 驗證 Binder descriptor，註冊 cast 與 system-audio callback。MyCadillac manifest 未宣告該 service，故不是它的 provider；Bridge 仍依賴裝置上的外部 CarLink service。 | `ev_785369aed25b6c880c0ad18b9b6efea501632d96168226e5febf19bbf626f35e`, `ev_54799bdb31c982e458742b8d9d005425a02ecbf2096b16d94e70e28a8bbb40b4`, `ev_b12984ee12ac7f422adb0391ed21164b7ca13ba8e5e6a7c5c645e00fb51ae4c8` |
| 連線啟動與 USB metadata | Service 要求 `bindPackageName=com.miui.carlink`、`bindClassName=com.xiaomi.ucar.carlife.CarlifeConnService`、`manufacturer=xiaomi`、`usb_type` 在 `[0, 2)` 且 `is_box=false`，才將外部 Intent 識別為 Xiaomi CarLink 入口並啟動 controller coroutine。這是入口辨識／模式選擇證據，不等同於證明底層 USB 傳輸已建立。 | `ev_785369aed25b6c880c0ad18b9b6efea501632d96168226e5febf19bbf626f35e` |
| Bluetooth 條件 | controller coroutine 的 Smali 顯示：在一個設定旗標成立且認證狀態符合時，會驗證兩個 Bluetooth MAC，拒絕零值／占位值，並要求 Bluetooth adapter 已啟用，才啟動後續 controller 協程。Service 傳入的模式布林值被帶入該後續協程，但其對這些 Bluetooth 條件的影響尚未完成追蹤；故 USB 與 Bluetooth 的必要關係仍未知。 | `ev_892d2fcb370f11e082fd20bf815388214be5831d8306ddcb4295ef6a834c16c7`, `ev_785369aed25b6c880c0ad18b9b6efea501632d96168226e5febf19bbf626f35e` |
| System audio callback | `ISystemAudioCallback` 定義 `onAudioStreamReady(byte[], …)`；Bridge manager client 註冊此 callback。現有 callback references 尚未證明其 payload 被送進哪個播放或網路輸出 sink。 | `ev_9bf404b10c673bd7fdd21f06dbb5a95335d651145a5ae4e6143a14664dafdd9b`, `ev_54799bdb31c982e458742b8d9d005425a02ecbf2096b16d94e70e28a8bbb40b4`, `ev_b5d84c59264c81f9b228bb6562e24a6c3b885697a62e8b5c1e3ff4dc19065f77` |
| 音頻編碼與封包 | `el.f(...)` 驗證音頻 frame 並送入 `MediaCodec` worker；worker 將資料排入 codec、取出編碼輸出，再由 packetizer 組成帶時間戳的 frame。這證明 APK 內有實質音頻編碼／封包機制，但輸出對端和與 system-audio callback 的 runtime 連接仍未知。 | `ev_d1c3f5f7174f2059913f41e64e681d434f8aec09cbcda7bb47c582cbf7cd531a`, `ev_eb9c7b5f8926f0fccf30944c9f75af078c5fc2d935f2e0038d0d6534bbc1edc9`, `ev_d5071972dbbfebcb02d72650fde0d89178401c27d39bca11c09c1a1a950b93b5`, `ev_28da6889272521e021a8092064613411262f67036145b5390a47aa53417e4cc8` |
| 音頻 stream 協商 | stream handler 解析 `audioType`、`audioFormat`、`audioLatencyMs` 及 data-stream/control-port 欄位，建立 socket-backed stream。該方法以 Smali 為可用表示且含 decompiler 警告；「AirPlay-like」僅是協定欄位線索，不作確定協定歸屬。 | `ev_ed6dbc58d988f26f9838bde5fc5e120d3fe0b8bb59a742d948075ed32c6cc715`, `ev_970a0c5e1e5101d5d2b6513700e816badc1a7e84307a44686bb3e0c8390fc822` |
| Cast 啟動控制 | `prepareCast` 呼叫只在 CarWith host 已綁定且 Surface 有效時發生，傳入 Surface 尺寸與 Surface；此配置偏向畫面 cast。另有 `invokeMethod` action `carlife_encode_format` 傳投屏尺寸，已追到的直接 action 未顯示音頻啟動命令。 | `ev_10342475bce946c58f8550423df264974680af3fb1b5aaccc790f7a8e8fdc73b`, `ev_029e0addc6f38a47c13af1cf477660b592159f145bb96d798a7b64b9d59fa677`, `ev_c26f4181cf5ca4a3c8ddfa720170aab7279eb7289157687cdc5c1c6214bb244d` |

## 已知限制

- JADX 靜態反編譯不能證明任何路徑在 Redmi K100 Pro Max / HyperOS 4 或 XPENG G6 上實際成功。
- APK provider 未驗證簽名、split APK、native libraries 或 Android runtime capture。
- 某些混淆 coroutine 的 decompiler 輸出有警告；本報告對關鍵入口優先採用可讀方法或 Smali 證據，不把巨大或失真的控制流當成確定語義。
- MyCadillac 僅檢查 manifest；未檢查其程式碼或執行行為。Manifest 未宣告 Bridge 所綁 service，因此不能把它視為該 service 的提供者。
- 未確認簽名、privileged 權限、原始安裝來源及重簽後與既有安裝的更新相容性。

## 建議的下一步

MyCadillac manifest 不提供 Bridge 所需的外部 service，無需為此再反編譯其程式碼。靜態分析找到音頻 callback contract、codec 與 frame packetizer，但未把 callback payload 連到可確認的輸出端；USB metadata 也尚未與後續 Bluetooth-gated coroutine 條件完全解耦。下一步若繼續只讀分析，應聚焦這兩個剩餘的資料流缺口；若要在 Redmi 上啟動 App、使用 ADB 或收集 log，須另行取得明確授權。尚未定位具體 patch 缺口，故不修改、不重建、不簽署 APK，也不安裝額外 APK 工具。

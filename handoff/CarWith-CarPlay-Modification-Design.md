# CarWith CarPlay 功能修改設計

狀態：供使用者審閱；靜態 APK 分析尚未開始。

## 目標

調查並嘗試改善小米 CarWith 配套 APK，使 Redmi K100 Pro Max（HyperOS 4、使用者回報 CarWith 4.0.2）能透過有線通道使用 CarPlay，並讓音頻也經 CarPlay 通道傳送。OcarPlay 0.8.44 是 Oppo 已實現案例，只作機制參考，不是修改目標。

使用者回報 XPENG G6 CarPlay 為測試接收端。使用者確認具有本地逆向分析與修改權限，並希望嘗試產出修改 APK。

## 輸入

- `/workspace/CarWith-CarPlay-My-Cadillac-release-1dafe966.apk`
- `/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk`
- `/workspace/OcarPlay-0.8.44.apk`

上述檔案路徑已在工作區確認存在。APK 實際 package/version、簽名身分及兩個 CarWith APK 的分工尚未驗證；「CarWith 4.0.2」是使用者提供資訊，不視為已由 APK 證實。

## 方法與階段

### 1. 唯讀基線與身分

- 為三個 APK 記錄大小、SHA-256、APK/package/version 資訊、manifest、DEX/資源/原生庫 inventory。
- 記錄簽名憑證的觀察值與簽名驗證狀態；不得將憑證資訊等同於信任判斷或平台授權。
- 保留 REA Evidence、來源 APK digest、工具/JADX 身分、coverage、diagnostics、limitations。
- 輸出與工作副本放在 `/workspace/handoff/`；絕不覆寫原始 APK。

### 2. 三包比較與功能路徑

- 比對兩個 CarWith APK 的套件身分、元件宣告、權限、共享身分/簽名關係與相互呼叫線索，以證據確認它們是否確為一組及各自職責。
- 在 CarWith 及 OcarPlay 中搜尋並追蹤 USB、Accessory/Host、Bluetooth、網路傳輸、CarPlay session、音頻 focus/routing/stream/codec 等相關類別與方法。
- 由類別 inventory 選擇精確方法與 overload，再檢查反編譯內容和 incoming static references；不依賴檔名、字串或單一命中就宣稱功能實作。
- 將每項結果標記為「觀察」「推論」或「未知」。反編譯源碼是衍生表示，不等於原始源碼或執行期行為。

### 3. 可行性閘門

靜態分析後先報告：

- 有線鏈路與音頻鏈路在 APK 中的實際入口、資料流和缺口。
- OcarPlay 的可比部分、不可移植依賴，以及尚待實機證據的部分。
- 修改應採源碼重建、局部 smali/resource patch，或無法在 APK 層解決。
- 重簽後可能失去的 signature-level permissions、共享 UID/套件信任、系統白名單或 Xiaomi 特權服務存取。
- 能否產出可安裝版本及需要的金鑰/系統簽署條件。不得繞過平台簽名或權限保護；若依賴不可取得的特權或原簽名條件，需停止改包並說明阻礙。

### 4. 有條件的修改與驗證

- 只有找到由證據支持、APK 層可修的具體缺口，才提出精確 patch；先讓使用者審閱結論和 patch 範圍，再操作工作副本。
- 修改版與所有中間產物均另存新路徑；保留原始 APK 不變，記錄 diff、輸入/輸出 digest、建置與簽署工具、簽署憑證指紋。
- 靜態驗收至少確認 APK 結構可讀、manifest/package 一致、修改位置與預期相符、簽名狀態和安裝限制被如實記錄。
- 安裝、啟動 APK、ADB 操作或連接車機均會改變設備狀態或執行目標；不包含在本階段授權內，須另行確認後才可進行。若未取得實機驗證，只能標示「靜態產物」，不能宣稱有線或音頻功能已工作。

## 成功標準

1. 清楚回答兩個 CarWith APK 是否配套、各自承擔什麼角色。
2. 以 Evidence 描述有線 CarPlay 與 CarPlay 音頻的程式入口、資料流、依賴和未知。
3. 說明 OcarPlay 機制哪些可比較、哪些依賴 Oppo/平台能力而不可直接移植。
4. 作出 APK 層修改可行性與簽名/權限阻礙結論。
5. 若進入改包，提供獨立輸出的 APK 與可重現的變更/簽署紀錄；實機成功只在實際完成授權測試後宣稱。

## 安全及範圍限制

- 靜態分析期間不執行或安裝 APK、不連接車機、不讓目標 APK 存取網路服務，也不改原始 APK。若環境沒有現成 REA/JADX，取得分析工具可能需要使用已設定的 npm/GitHub egress；這不會向目標 APK 的服務送出請求。
- 不繞過簽名驗證、平台權限、DRM/授權檢查，不做反除錯/反注入規避或隱蔽執行期 Hook。
- 不假設 OcarPlay 已知實作方式；須由其自身 Evidence 支持每項結論。
- 不把缺少靜態呼叫、字串或類別當成該功能不存在；反射、動態載入及系統服務邊界必須保留為未知。

## 已知限制與待確認事項

- 尚無三個 APK 的 SHA-256、REA Evidence、package/version/簽名事實。
- XPENG G6 車機／CarPlay 接收端版本與具體 USB 拓撲尚未提供；實機測試條件未知。
- 原 APK 簽名金鑰、系統簽署或特權安裝條件未知。
- 「有線」指 Redmi 直接 USB 到車機，還是經特定轉接器，尚需實機路徑確認。
- 目前沒有取得原始碼；能否以 APK patch 達成取決於靜態發現和必要系統能力。

# CarWith CarPlay 功能修改設計

狀態：使用者已確認分階段方案，並指定 CarPlay Bridge 優先；本版反映該範圍。靜態 APK 分析尚未開始。

## 目標

調查並嘗試改善小米 CarWith 的 CarPlay Bridge APK，使 Redmi K100 Pro Max（HyperOS 4、使用者回報 CarWith 4.0.2）能透過有線通道使用 CarPlay，並讓音頻也經 CarPlay 通道傳送。第一輪只分析 Bridge，先判斷它是否已能獨立提供投屏，不依賴 MyCadillac APK 啟動。若證據支持 Bridge 足以承擔目標功能，後續只對 Bridge 做修改／重建，不分析或編譯 MyCadillac。

OcarPlay 0.8.44 是 Oppo 已實現案例，只在 Bridge 的具體技術問題需要對照時作機制參考，不是修改目標。

使用者回報 XPENG G6 CarPlay 為測試接收端。使用者確認具有本地逆向分析與修改權限，並希望嘗試產出修改 APK。

## 輸入

- `/workspace/CarWith-CarPlay-My-Cadillac-release-1dafe966.apk`
- `/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk`
- `/workspace/OcarPlay-0.8.44.apk`

上述檔案路徑已在工作區確認存在。APK 實際 package/version、簽名身分、Bridge 是否依賴 MyCadillac 尚未驗證；「CarWith 4.0.2」是使用者提供資訊，不視為已由 APK 證實。MyCadillac 是條件式參考：只有 Bridge 的靜態證據顯示存在會影響結論的跨包依賴時，才先詢問使用者是否擴大範圍。

## 方法與階段

### 1. CarPlay Bridge 唯讀基線與身分

- 優先只對 CarPlay Bridge 記錄大小、SHA-256、APK/package/version 資訊、manifest、DEX/資源/原生庫 inventory。
- 記錄 Bridge 簽名憑證觀察值與簽名驗證狀態；不得將憑證資訊等同於信任判斷或平台授權。
- 保留 REA Evidence、來源 APK digest、工具/JADX 身分、coverage、diagnostics、limitations。
- 輸出與工作副本放在 `/workspace/handoff/`；絕不覆寫原始 APK。

### 2. 判斷 Bridge 的投屏完整性與依賴

- 檢查 Bridge manifest 的 launch/exported 元件、服務、receiver、provider、權限與 package visibility；在 DEX 中追蹤啟動路徑上的明確跨包呼叫、Intent action、authority、service binding 及 package-manager 查詢。
- 由 Bridge 類別 inventory 選擇精確類別、方法與 overload，檢查投屏 session 的入口、狀態和資料路徑。靜態結果只能說明「未觀察到特定依賴」或支持某個呼叫鏈，不能單靠沒有命中就證明 runtime 完全不依賴另一 APK。
- 在 Bridge 內搜尋 USB/Accessory/Host、Bluetooth、網路傳輸、CarPlay session、音頻 focus/routing/stream/codec 等路徑，判斷新增有線與音頻功能是否可能由這個 APK 承擔。
- 只在 Bridge 的特定缺口需要比較時，才分析 OcarPlay；不要因其存在就預先分析完整 APK。
- 若發現可能依賴 MyCadillac，且不檢視它就無法區分 launcher、服務提供者或必要授權關係，暫停並請使用者批准擴大分析，不自動讀取。
- 由類別 inventory 選擇精確方法與 overload，再檢查反編譯內容和 incoming static references；不依賴檔名、字串或單一命中就宣稱功能實作。
- 將每項結果標記為「觀察」「推論」或「未知」。反編譯源碼是衍生表示，不等於原始源碼或執行期行為。

### 3. 可行性閘門

靜態分析後先報告：

- Bridge 是否包含現有投屏流程，以及目前 Evidence 對「不依賴 MyCadillac 啟動」可支持到什麼程度。
- 有線鏈路與音頻鏈路在 Bridge 中的實際入口、資料流和缺口。
- 如有必要，OcarPlay 可比的部分、不可移植依賴，以及尚待實機證據的部分。
- 修改應採源碼重建、局部 smali/resource patch，或無法在 APK 層解決。
- 重簽後可能失去的 signature-level permissions、共享 UID/套件信任、系統白名單或 Xiaomi 特權服務存取。
- 能否產出可安裝版本及需要的金鑰/系統簽署條件。不得繞過平台簽名或權限保護；若依賴不可取得的特權或原簽名條件，需停止改包並說明阻礙。

### 4. 有條件的修改與驗證

- 若 Evidence 支持 Bridge 可自行承擔現有投屏，且有線/音頻缺口在 Bridge 內有具體、可修的程式路徑，先讓使用者審閱結論和 patch 範圍，再只對 Bridge 工作副本嘗試 patch／重建。
- 若 Bridge 不足以獨立投屏，或關鍵依賴仍未知，先停止；不自動分析或重建 MyCadillac。
- JADX 輸出是衍生源碼，不是可直接編譯回 APK 的原始專案。重建必須使用適合 APK/smali 的既有或使用者提供工具；不可為此默默安裝工具。若缺少必要的 build/sign 工具，回報阻礙，不偽稱已編譯。
- 修改版與所有中間產物均另存新路徑；保留原始 APK 不變，記錄 diff、輸入/輸出 digest、建置與簽署工具、簽署憑證指紋。
- 靜態驗收至少確認 APK 結構可讀、manifest/package 一致、修改位置與預期相符、簽名狀態和安裝限制被如實記錄。
- 安裝、啟動 APK、ADB 操作或連接車機均會改變設備狀態或執行目標；不包含在本階段授權內，須另行確認後才可進行。若未取得實機驗證，只能標示「靜態產物」，不能宣稱有線或音頻功能已工作。

## 成功標準

1. 以 Evidence 說明 Bridge 的投屏入口及它對 MyCadillac 的依賴證據與靜態分析界限。
2. 以 Evidence 描述 Bridge 有線 CarPlay 與音頻的程式入口、資料流、依賴和未知。
3. 只有在使用者同意擴查時，才對照 MyCadillac；OcarPlay 僅於有具體研究問題時比較。
4. 作出僅重建 Bridge 的可行性與簽名/權限阻礙結論。
5. 若成功進入改包，提供獨立輸出的 Bridge APK 與可重現變更/簽署紀錄；實機成功只在實際完成授權測試後宣稱。

## 安全及範圍限制

- 靜態分析期間不執行或安裝 APK、不連接車機、不讓目標 APK 存取網路服務，也不改原始 APK。若環境沒有現成 REA/JADX，取得分析工具可能需要使用已設定的 npm/GitHub egress；這不會向目標 APK 的服務送出請求。
- 不繞過簽名驗證、平台權限、DRM/授權檢查，不做反除錯/反注入規避或隱蔽執行期 Hook。
- 不假設 OcarPlay 已知實作方式；須由其自身 Evidence 支持每項結論。
- 不把缺少靜態呼叫、字串或類別當成該功能不存在；反射、動態載入及系統服務邊界必須保留為未知。

## 已知限制與待確認事項

- 尚無 CarPlay Bridge 的 SHA-256、REA Evidence、package/version/簽名事實。
- 僅靠靜態分析無法證明未觀察到的跨包依賴在 runtime 絕不存在。
- XPENG G6 車機／CarPlay 接收端版本與具體 USB 拓撲尚未提供；實機測試條件未知。
- 原 APK 簽名金鑰、系統簽署或特權安裝條件未知。
- 「有線」指 Redmi 直接 USB 到車機，還是經特定轉接器，尚需實機路徑確認。
- 目前沒有取得原始碼；能否以 APK patch 達成取決於靜態發現和必要系統能力。

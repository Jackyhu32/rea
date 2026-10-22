# CarWith Bridge 優先分析執行計畫

> **執行界線：** 本計畫先做 APK 靜態分析；不代表可安裝或執行 APK。遵守各階段檢查點與停止條件。

**Goal:** 判斷 CarPlay Bridge 是否自行承擔 CarWith 投屏，找出有線與音頻功能的證據鏈及可行修改點；只有證據支持且重建條件成立時，才嘗試重建 Bridge。

**Architecture:** 首先只對 Bridge 使用 REA 靜態 APK/Android 工作流程，完整 Evidence 存檔並以 APK digest 綁定。MyCadillac 為排除在外的條件式目標；只有觀察到會影響結論的跨包依賴，且先取得使用者同意後才分析。OcarPlay 僅在 Bridge 留下明確技術問題時作參考。

**Tech Stack:** `rea-agents@6.7.0` CLI、JADX headless MCP 0.7.1、現有 Temurin JDK 25（含 `jdk.compiler`）。APK 重建與簽名工具不假設已安裝，不得未經批准安裝。

---

## 範圍與界線

- 首要目標：`/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk`。
- 不讀取 `/workspace/CarWith-CarPlay-My-Cadillac-release-1dafe966.apk`，除非 Bridge 證據顯示會影響結論的外部依賴；此時先停下詢問。
- 不讀取 `/workspace/OcarPlay-0.8.44.apk`，除非 Bridge 留下具體且未解的有線、音頻或 session 問題。
- Evidence、記錄、工作副本與可能的重建產物一律放在 `/workspace/handoff/`；不覆寫原 APK。
- 不執行或安裝 APK、不連 XPENG G6、不用 ADB、不向目標服務送網路請求。這些都須另行確認。
- 不繞過簽名驗證、平台權限、授權或保護。若重建需要無法取得的簽署權限或特權，停止並回報。

## Task 1：綁定 Bridge 輸入並盤點封裝

**Files:**
- Read: `/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk`
- Create: `/workspace/handoff/bridge.sha256`
- Create: `/workspace/handoff/bridge-inventory.json`

- [ ] **Step 1：記錄來源 APK digest**

Run:

```bash
sha256sum "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" > /workspace/handoff/bridge.sha256
```

Expected: 一筆記錄，含所選 Bridge APK 的 SHA-256。

- [ ] **Step 2：保存 APK archive Evidence**

Run:

```bash
npx -y rea-agents@6.7.0 inspect-artifact "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" --json > /workspace/handoff/bridge-inventory.json
```

Expected: 完整 `inspect_artifact` Evidence 存檔。從 inventory 核對 APK root、`AndroidManifest.xml`、`classes*.dex`、原生 libraries 與相關資源；保留 Evidence ID、source digest、coverage 和 limitations。

如果 REA 認定不是支援的封裝，或觀察到的 digest 與 `bridge.sha256` 不同，停止；不得換用其他 APK。

## Task 2：唯讀確認 JADX 前置條件

**Files:**
- Create: `$HOME/.local/share/rea/android/jadx-headless-mcp-0.7.1-all.jar`
- Create: `/workspace/handoff/bridge-jadx-readiness.json`

- [ ] **Step 1：下載固定版本 JADX**

Run:

```bash
mkdir -p "$HOME/.local/share/rea/android"
curl -fL -o "$HOME/.local/share/rea/android/jadx-headless-mcp-0.7.1-all.jar" "https://github.com/1013503897/jadx-headless-mcp/releases/download/v0.7.1/jadx-headless-mcp-0.7.1-all.jar"
```

- [ ] **Step 2：驗證引擎 SHA-256**

Run:

```bash
cd "$HOME/.local/share/rea/android"
printf '%s  %s\n' '6e5eacf500b64292bfb73c49797c1958f6ee44646e43e868039ae7feb573ff75' 'jadx-headless-mcp-0.7.1-all.jar' | sha256sum -c -
```

Expected: `OK`。若 digest 不符，不可使用該 JAR。

- [ ] **Step 3：設定本 shell 的 JADX 路徑**

Run:

```bash
export REA_JADX_MCP_JAR="$HOME/.local/share/rea/android/jadx-headless-mcp-0.7.1-all.jar"
```

沿用現有 Temurin JDK 25 與 `JAVA_HOME`；不可為此安裝或更換 Java。

- [ ] **Step 4：保存唯讀 readiness 結果**

Run:

```bash
npx -y rea-agents@6.7.0 doctor --provider jadx --json > /workspace/handoff/bridge-jadx-readiness.json
```

Expected: provider diagnostics 顯示所選 JAR 與完整 JDK。若失敗，只按具體 diagnostics 處理；不可啟動 JADX 或修復無關設定。

## Task 3：檢視 Bridge 套件身分與入口元件

**Files:**
- Create: `/workspace/handoff/bridge-package.json`

- [ ] **Step 1：保存 Android package Evidence**

Run:

```bash
npx -y rea-agents@6.7.0 inspect-android-package "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" --json > /workspace/handoff/bridge-package.json
```

Expected: 完整 Evidence，包含 package/version 宣告、decoded manifest、permissions、SDK facts、類別/資源數、provider identity 與 APK SHA-256。

- [ ] **Step 2：核對 APK 身分一致性**

確認 `bridge-package.json` 的 APK digest 與 `bridge.sha256` 及 inventory Evidence 一致。記錄 package name、version、application class、啟動 Activity、services、receivers、providers、exported 狀態及 requested permissions。不可單憑名稱推定 launcher 或特權權限。

若不同操作間 APK digest 改變，停止並分別保留觀察結果，不得合併不同 bytes 的事實。

## Task 4：只由 Bridge 檢驗 MyCadillac 依賴假設

**Files:**
- Create: `/workspace/handoff/bridge-classes-*.json`
- Create: `/workspace/handoff/bridge-dependency-findings.md`

- [ ] **Step 1：搜尋候選類別**

在 Bridge 上執行有目標的類別搜尋，並各自保存完整結果：

```bash
npx -y rea-agents@6.7.0 search-android-classes "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" "carplay" --json > /workspace/handoff/bridge-classes-carplay.json
npx -y rea-agents@6.7.0 search-android-classes "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" "bridge" --json > /workspace/handoff/bridge-classes-bridge.json
npx -y rea-agents@6.7.0 search-android-classes "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" "usb" --json > /workspace/handoff/bridge-classes-usb.json
npx -y rea-agents@6.7.0 search-android-classes "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" "audio" --json > /workspace/handoff/bridge-classes-audio.json
```

只使用 REA 實際回傳的類別候選；某個 substring 零命中不代表功能或依賴不存在。若名稱混淆或這些搜尋不足以追蹤入口，才使用空 query 列出完整 class inventory，並保留輸出大小與 coverage。

- [ ] **Step 2：檢視精確類別成員**

對前一步實際回傳的候選類別執行：

```bash
npx -y rea-agents@6.7.0 inspect-android-class "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" "<類別搜尋實際回傳的完整類別名>" --json
```

由類別結果選取 method name 和 `overload_index`；不可猜 overload 或預設採用第一個同名方法。

- [ ] **Step 3：只反編譯與依賴問題相關的方法**

針對入口/session 路徑上的精確方法執行：

```bash
npx -y rea-agents@6.7.0 inspect-android-method "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" "<精確類別名>" "<精確方法名>" --json
```

檢視有無具體跨包機制，例如明確 package Intent、`getLaunchIntentForPackage`、`PackageManager` 查詢、具名 service binding、provider authority、broadcast action，或要求另一套件的 permission。保留 `body_status`、overload 身分、文本 coverage 與 Evidence ID。

- [ ] **Step 4：追蹤入口方法 incoming references**

先用實際 CLI help 確認旗標，再以類別清單中選定的精確 overload 執行：

```bash
npx -y rea-agents@6.7.0 trace-android-references --help
npx -y rea-agents@6.7.0 trace-android-references "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" "<精確類別名>" --method-name "<精確方法名>" --overload-index <類別清單中的索引> --json
```

Incoming references 是靜態 provider-node 關係，不是實際 runtime 呼叫。反射、動態載入與 unresolved invokes 保持未知。

- [ ] **Step 5：寫下有界依賴結論**

將結果歸為以下之一：

- **觀察到內部入口路徑：** Evidence 顯示 Bridge 自身宣告的入口進入投屏/session 路徑，且已檢查的方法內未觀察到啟動 MyCadillac 的呼叫。
- **觀察到跨包依賴：** Evidence 顯示明確 MyCadillac package/component/service/permission 依賴。此時停止，先詢問是否檢查 MyCadillac。
- **未能判定：** coverage 或動態／系統邊界不足以可靠判斷；記錄未知並詢問是否擴大範圍。

不得僅因有限靜態搜尋未找到 package reference，就宣稱已證明 runtime 完全獨立。

## Task 5：在 Bridge 中定位有線與音頻路徑

**Files:**
- Create: `/workspace/handoff/bridge-feature-paths.md`
- Optional: 只有在存在具名未解問題時才建立 `/workspace/handoff/ocarplay-reference-*.json`

- [ ] **Step 1：沿 Bridge 現有投屏/session 路徑追蹤**

使用 manifest 入口和 Task 4 精確類別/方法結果，只檢視畫出狀態轉換及資料邊界所需的方法。分別記錄 Bluetooth 路徑和任何 USB/Accessory/Host 候選。

- [ ] **Step 2：追蹤有線候選路徑**

只檢查 Bridge 實際回傳且與 USB 權限/裝置發現、Accessory/Host、socket/session 建立或協定交接相關的類別與方法。區分「宣告可用 Android API」與「完整有線 CarPlay 路徑已證實」。

- [ ] **Step 3：追蹤音頻候選路徑**

只檢查 Bridge 中使用 audio focus、routing、capture/playback stream、codec 或 CarPlay session 的候選方法。區分 Android 音頻 API 與「音頻已穿過 CarPlay 傳輸」的證據。

- [ ] **Step 4：必要時才使用 OcarPlay 作對照**

開啟 OcarPlay 前先寫明 Bridge 的具體未解問題（例如已確認的音頻 session 使用何種交接協定）。只檢視能回答該問題的 manifest/class/method。OcarPlay 相似性不能證明它與 Xiaomi service 或 XPENG G6 相容。

- [ ] **Step 5：評估 Bridge 重建與簽名可行性**

確認 patch 點是否局限在 Bridge、需要重新產生哪些 resources/smali，以及重簽可能失去哪些平台/簽名權限或共享身分。只檢查目前已存在的 APK 重建/簽名工具；若缺工具，不得自行安裝，需回報需要使用者提供的工具或授權。

## Task 6：改動 APK 前的決策檢查點

**Files:**
- Create: `/workspace/handoff/bridge-feasibility-report.md`

- [ ] **Step 1：先完成證據報告**

報告必須包含：

- Bridge 已檢查的入口/session 路徑是否支持「可不經 MyCadillac 啟動投屏」，並明示靜態證據的信心水位與邊界。
- 有線/音頻的 Evidence-backed 入口、候選 patch 點，以及未解的協定/系統依賴。
- APK digest、REA/JADX 身分、Evidence IDs、coverage、limitations、diagnostics。
- Bridge-only 重建的技術可行性、是否需要重簽、預期權限/安裝限制。
- 是否需要查看 MyCadillac 或對照 OcarPlay。

- [ ] **Step 2：等待使用者審閱具體 patch 範圍**

使用者審閱報告並核准具體修改邊界與簽署方式之前，不得編輯或重建 Bridge。若證據指出 MyCadillac 必要或關係仍具重大不確定性，先詢問是否查看；不得自行擴大範圍。

## Task 7：有條件地只修改並重建 Bridge

**Files:**
- Read-only source: `/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk`
- Create after approval: `/workspace/handoff/bridge-work/`
- Create after approval: `/workspace/handoff/CarWith-CarPlay-Bridge-modified.apk`
- Create after approval: `/workspace/handoff/bridge-rebuild-record.md`

- [ ] **Step 1：建立獨立工作副本**

在 `/workspace/handoff/bridge-work/` 建立新工作目錄，並再次核對來源 APK digest 與 `bridge.sha256` 相符。

- [ ] **Step 2：只修改核准的 Bridge method/resource**

按 readiness 結果使用已存在且合適的 APK/smali 建置工具，只套用經審閱的有線/音頻 patch。不得更改 package ID、簽名權限、反除錯保護或無關元件。

- [ ] **Step 3：只在允許條件下重建與簽署**

將產物寫入新輸出路徑。若沒有原簽名金鑰，僅能在使用者另行核准且靜態權限檢查確認重簽可行時使用新的開發簽名；不可宣稱重簽後仍保有平台特權。

- [ ] **Step 4：靜態驗證重建產物**

對新 APK 再跑 archive 與 Android package inspection。確認可讀，記錄新 SHA-256，對照 package/version/manifest 與預期變更元件，記錄簽名憑證身分及限制。不安裝、不執行。

## 預期檢查點

Task 1–6 完成後，先交付附 Evidence 的簡短可行性報告，再詢問是否執行具體 Bridge-only patch。Task 7 為條件式，不可提前執行。重建成功不代表有線 CarPlay 或音頻已可用；只有在另行授權後，實際於 Redmi K100 Pro Max 與 XPENG G6 測試，才能報告 runtime 成功。

## 自我檢查

- MyCadillac 明確排除：只有發現實際跨包依賴且使用者批准後才擴查。
- OcarPlay 是針對具體問題的選配參考，不當成 Xiaomi 相容性的證明。
- 所有結論都綁定精確 APK digest 和 REA Evidence；靜態缺席不作完整性宣稱。
- 原 APK 保持不變；修改、簽名、安裝、runtime 測試是不同且分開授權的操作。

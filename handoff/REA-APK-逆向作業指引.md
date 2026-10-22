# REA APK 逆向作業指引（交接用）

> 本檔是「已理解倉庫運作模式」的交接文件，**尚未執行任何逆向操作**。
> 後續模型請照本指引執行；所有結論需標明「觀察 / 推論 / 未知」並附 Evidence。

---

## 0. 現況檢查結果（本機實測，2026-10-22）

| 項目 | 狀態 | 說明 |
| --- | --- | --- |
| 目標 APK | ✅ 存在 | 見 §1，實際在倉庫根目錄，**不在** `attachments/` |
| `attachments/` 目錄 | ⚠️ 空 | 已建立但沒有檔案 |
| Node.js | ✅ v24.1.0 | REA 支援 22.19+/24.11+/26+ |
| npm | ✅ 11.4.2 | |
| Java | ✅ Temurin **OpenJDK 25**（`JAVA_HOME` 已設，含 `jdk.compiler`） | JADX 需要完整 JDK 17+ |
| `ps`（procps） | ✅ procps-ng 4.0.4，`ps -o` 可用 | Linux owned-process 前置條件已滿足 |
| 網路 | ✅ proxy `http://127.0.0.1:18080`；npm registry、github.com 可達 | 可下載套件與 JADX JAR |
| `rea` CLI | ❌ 不在 PATH | 用 `npx -y rea-agents@latest` 取代 |
| REA **MCP 工具** | ❌ 本 session 未註冊 | 走 CLI 路線最省事（見 §3） |
| 倉庫 `dist/`、`node_modules/` | ❌ 不存在 | 要從原始碼跑需先 `npm ci && npm run build:cached` |
| JADX JAR | ❌ 未下載 | 需自行取得（REA 不代裝），見 §5 |
| apktool | ❌ 未安裝 | 只有「資源解碼」才需要，可選 |

npm 上 `rea-agents` 最新版 = **6.7.0**（本倉庫 checkout 為 6.5.0）。Android 家族自 4.1.0 起就有，`@latest` 可直接用。

---

## 1. 目標檔案

```
/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk          (3.7 MB)
/workspace/CarWith-CarPlay-My-Cadillac-release-1dafe966.apk                   (10.8 MB)
```

**注意**：檔名含空格與中文，所有命令務必用雙引號包住完整路徑。
第一個動作建議先算 digest 綁定身分（Evidence 也會自帶 `target.sha256`）：

```bash
sha256sum "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" \
          "/workspace/CarWith-CarPlay-My-Cadillac-release-1dafe966.apk"
```

---

## 2. 倉庫（REA）的工作模式

REA = `rea-agents`，一個 CLI + 一個 MCP server，**共用同一套 application workflow 與 Evidence 契約**。
重點概念：

1. **無狀態、以 artifact 為中心**：每個 Android 查詢都自帶 `path`，不需要先「開 session」。
   檔案路徑本身即為 target 綁定；Evidence 會記錄 APK 的 SHA-256、原始請求/回應。
2. **CLI 與 MCP 等價**：CLI 的 `inspect-android-package` ↔ MCP 的 `inspect_android_package`（同一個 workflow）。
   本 session 沒有註冊 MCP，所以**用 CLI**。
3. **Evidence 語意（必須遵守）**：
   - `normalized_result` = 操作資料；`evidence_id` = 穩定引用。
   - 一定要區分 **observation / inference / unknown**；不可把靜態分析講成「執行過」。
   - 覆蓋率（`coverage`）與限制（`limitations`）必須保留；**分頁取盡 ≠ 完整**（尤其 manifest、method 文本會被截斷）。
   - `signature verification: not_performed`；APK 真實性、執行期行為皆為未驗證。
4. **絕不執行目標**：APK 不會被啟動、不需要模擬器。
5. **只讀**：不改原始 APK、不改倉庫原始碼、不 commit（見 §9）。

### 針對 APK 的四條路線（依「是否需要外部引擎」排序）

| 路線 | 命令 | 需要的引擎 | 用途 |
| --- | --- | --- | --- |
| A. 檔案/封裝盤點 | `inspect-artifact`、`extract-artifact` | 無 | APK 是 ZIP：成員清單、`.so`、`assets/`、簽章檔、完整性矛盾 |
| B. Flutter 判定 | `identify-flutter-build`、`inspect-dart-aot` | 無 | 純解析；判定是否 Flutter 與 Dart snapshot hash |
| C. **Java/Kotlin 反編譯（主力）** | `inspect-android-package` → `search-android-classes` → `inspect-android-class` → `inspect-android-method` → `trace-android-references` | **JADX headless MCP 0.7.1 JAR + 完整 JDK 17+** | Manifest 宣告、類別/方法、反編譯、incoming references |
| D. 資源解碼（選配） | `inspect-apktool-client`、`decode-android-resources` | apktool 2.7.0+ | 解出 manifest 文字、字串表、locale、版本資訊 |
| （進階）E. 引擎級反編譯 | JEB 家族 | 需自備 JEB | 非必要，先不做 |

另有 `project-android-application-graph`：吃 A 的 inventory Evidence，產出元件路徑、雜湊與 JNI bridge 假設（**推論**，不是執行觀察）。

---

## 3. 入口：用 CLI（推薦）

本 session 沒註冊 MCP，直接用 package runner。**一律加 `--json`**：

```bash
npx -y rea-agents@latest <command> <path> --json
```

- CLI 一次呼叫＝一個獨立 process；**結果不會自動留存**，要用的話自己存檔。
- CLI 的 Android 輸出是**完整**的（沒有 MCP 的 `detail: summary` 分流）。
- 大輸出**務必重導到檔案**再用 `jq` 讀，別直接灌進 context：

```bash
npx -y rea-agents@latest inspect-android-package "/abs/path.apk" --json > /workspace/handoff/pkg-evidence.json
```

（若之後決定改用 MCP：`npx -y rea-agents@latest setup --client <id> --dry-run --json` 先看計畫，
經同意才 `--yes`，然後重啟 client。**但本任務不需要動 MCP**。）

---

## 4. 第一階段：零依賴動作（先跑，用來分流）

### 4.1 封裝盤點（路線 A）

```bash
npx -y rea-agents@latest inspect-artifact "/workspace/CarWith-CarPlay-My-Cadillac-release-1dafe966.apk" --json > /workspace/handoff/inv1.json
npx -y rea-agents@latest inspect-artifact "/workspace/CarWith 与 Carplay Bridge_574d7d22-11_20261017_153037.apk" --json > /workspace/handoff/inv2.json
```

看點：`AndroidManifest.xml` 是否存在、`classes*.dex` 數量、`lib/<abi>/*.so`（有 `libflutter.so`/`libhermes.so`/`libreactnative*.so` 就代表不是純 Java）、`assets/`、`META-INF/`。

需要落地檔案時（例如抽 `.so` 或 `assets/`）：

```bash
npx -y rea-agents@latest extract-artifact "/abs/path.apk" --json
```

（REA 自選暫存目錄並在結果中回報路徑；呼叫者不能指定目的地。）

### 4.2 Flutter 判定（路線 B）

```bash
npx -y rea-agents@latest identify-flutter-build "/abs/path.apk" --json
npx -y rea-agents@latest inspect-dart-aot "/abs/path.apk" --abi arm64-v8a --json
```

若不是 Flutter，會直接說明；是 Flutter 則給 Dart snapshot hash 與各 ABI library digest（可對 Dart SDK 版本）。

### 4.3 Application graph 投影（可選，吃 4.1 的 Evidence）

```bash
npx -y rea-agents@latest project-android-application-graph '{"inventory_evidence":[<inv1.json 內容>]}' --json
```

---

## 5. 第二階段：JADX 靜態分析（主力）

### 5.1 準備 JADX JAR（REA 不會代裝）

固定版本 **0.7.1**，SHA-256 必須符合（來源：`src/android/JadxRelease.ts`）：

```
6e5eacf500b64292bfb73c49797c1958f6ee44646e43e868039ae7feb573ff75
```

```bash
mkdir -p "$HOME/.local/share/rea/android" && cd "$HOME/.local/share/rea/android" &&
curl -fL -o jadx-headless-mcp-0.7.1-all.jar \
  https://github.com/1013503897/jadx-headless-mcp/releases/download/v0.7.1/jadx-headless-mcp-0.7.1-all.jar &&
echo '6e5eacf500b64292bfb73c49797c1958f6ee44646e43e868039ae7feb573ff75  jadx-headless-mcp-0.7.1-all.jar' | sha256sum -c -
```

每個新 shell 都要設：

```bash
export REA_JADX_MCP_JAR="$HOME/.local/share/rea/android/jadx-headless-mcp-0.7.1-all.jar"
# JAVA_HOME 已指向 Temurin 25（含 jdk.compiler），可沿用
```

### 5.2 先用 doctor 確認引擎（唯讀）

```bash
npx -y rea-agents@latest doctor --provider jadx --json
```

看 `scope_checks` 是否通過（JAR 路徑、Java 版本/模組、`operation_timeout_ms`）。

### 5.3 標準問答流程

```bash
# 1) 套件身分 + manifest 宣告 + 權限 + 類別/資源數量
npx -y rea-agents@latest inspect-android-package "/abs/path.apk" --json > /workspace/handoff/and1-package.json

# 2) 找類別（空 query = 全部類別清單）
npx -y rea-agents@latest search-android-classes "/abs/path.apk" MainActivity --json
npx -y rea-agents@latest search-android-classes "/abs/path.apk" "" --json > /workspace/handoff/and1-classes.json

# 3) 看某類別成員（先用這個挑 overload）
npx -y rea-agents@latest inspect-android-class "/abs/path.apk" com.xiaomi.carwith.MainActivity --json

# 4) 反編譯某方法（Java 或 smali）
npx -y rea-agents@latest inspect-android-method "/abs/path.apk" com.example.Foo onCreate --json
npx -y rea-agents@latest inspect-android-method "/abs/path.apk" com.example.Foo select --overload-index 1 --json

# 5) 誰呼叫它（incoming static references）
npx -y rea-agents@latest trace-android-references "/abs/path.apk" com.example.Foo --json
npx -y rea-agents@latest trace-android-references "/abs/path.apk" com.example.Foo --method-name onCreate --json
```

**規則**：
- 方法多載時 **REA 會拒絕**而非猜第一個；務必先 `inspect-android-class` 取 `overload_index`。
- `overload_index` **綁定該 APK 與該引擎**，跨版本不穩定，不能當穩定 ID。
- **不要**對同一輸入重複同一呼叫；先用回傳的 Evidence 決定下一個聚焦問題。

### 5.4 解讀重點（避免誤判）

- Manifest 摘要欄位由上游引擎抽取；需要時對照保留的 decoded XML。
- Class 清單是「反編譯器視角」的表示，含 synthetic members；搜尋結果的 `duplicate_names` 表示同名多筆。
- `body_status: "not_available"` = native/abstract 無 body，**不可推論**是哪個 flag 造成。
- `<clinit>` 會以 smali 回傳並帶 `static_initializer_disassembly` 標記。
- Reference 結果是 **JADX node 間**的靜態關係：`reference_unit=using_node` → `total` 算的是「包含它的 node」數，**不是** invoke 指令數。
  `direct_invokes` 有值才算對到 raw DEX invoke；`[]` = 該 node 內沒對到；`null` = 無 raw code 或 node 非 method。
  反射、動態載入、virtual/interface 的實際實作 → **未知**。
- 「完整文字」不代表「完整語意」或可 byte-identical 還原。

---

## 6. 第三階段（選配）：apktool 資源解碼

只有需要**解碼 manifest 文字 / 字串表 / locale / versionCode** 時才做。REA 不代裝 apktool。

```bash
export REA_APKTOOL_COMMAND=/absolute/path/apktool   # 未設則從 PATH 找
npx -y rea-agents@latest inspect-apktool-client --json
npx -y rea-agents@latest decode-android-resources "/abs/path.apk" --json
npx -y rea-agents@latest decode-android-resources "/abs/path.apk" --locale zh-TW --json
```

- 設計上跳過 smali（`--no-src`）；Java 源碼恢復**屬於 JADX 路線**。
- Manifest 上限 512 KiB、字串表預設 2000 筆（超出標 partial）、decode 600 s / 4 MiB 上限。

---

## 7. 逾時與資源參數

| 變數 | 預設 | 說明 |
| --- | --- | --- |
| `REA_JADX_OPERATION_TIMEOUT_MS` | 120000 | 整個操作的上限（1–2147483647）。大型 multidex 冷載建議調大（例：600000）。MCP client 自己的 tool timeout 也要一起調。 |
| `REA_JADX_HEAP_MIB` | 無 | JVM heap（MiB）。 |
| `REA_JADX_ACTIVE_PROCESSOR_COUNT` | 無 | 僅影響 JVM，不改變 JADX 單一 worker。 |
| `REA_JADX_MCP_JAR` | 無 | **必填**，絕對路徑。 |
| `JAVA_HOME` | 系統 | 指向完整 JDK（否則 java on PATH）。 |

固定上限（是記憶體安全邊界，不是「未公開的結果上限」）：decoded manifest/方法文字 **1 MiB**；
協定單帧 **8 MiB**；單次操作 stdout+stderr **32 MiB**。Bridge startup handshake 30 s、上游反編譯 90 s。

Session 生命週期：**閒置 60 秒**、目標/設定變更、失敗、逾時、取消、MCP 斷線或 server 關閉都會回收。
CLI one-shot 一定會 join cleanup 後才返回。

---

## 8. 疑難排解對照

| 症狀 | 處理 |
| --- | --- |
| `capability_unavailable` / `provider_unavailable`（JADX） | 讀 `message` + `remediation`；先確認 `REA_JADX_MCP_JAR` 是絕對路徑且 digest 相符 |
| 缺 `jdk.compiler` / Java 啟動失敗 | `java --list-modules \| grep jdk.compiler`；用 `JAVA_HOME` 指向完整 JDK（本機 Temurin 25 已知可用） |
| 找不到 `ps`（Linux） | 本機已滿足；若換環境需 procps |
| 方法選擇 ambiguous | 用 `inspect-android-class` 取 `overload_index` 再呼叫 |
| 類別/方法找不到 | 用 `search-android-classes`（空 query 列全部）先對齊命名；class name 是反編譯器表示 |
| cleanup 未確認 | REA 會**保留** workspace 並回報路徑；該 provider instance 會擋住後續啟動，需人工處理後重啟 |
| `resource_constraint`（transport） | MCP 才有的情境；CLI 直接完整輸出。若在 MCP，改走 `inspect-analysis-view` 分頁 |
| npx 下載失敗 | 檢查 proxy（`HTTPS_PROXY=http://127.0.0.1:18080`）；npm registry 已驗證可達 |

---

## 9. 操作紀律（硬性）

1. **只讀目標**：不執行 APK、不需要 emulator、不改原始 APK bytes。
2. **不改倉庫、不 commit**：`/workspace` 是 REA 的 git checkout。不要 commit、
   不要新增 `dist/`、`node_modules/`、APK 或其他二進位（AGENTS.md 明訂）。
3. **不裝無關軟體**：REA 不代裝 Java/JADX/apktool；也不要為了修註冊而升級 Node/npm/Homebrew。
4. **輸出地點**：中間產物放 `/workspace/handoff/`（或既有 `attachments/`），別污染倉庫根目錄。
5. **結論紀律**：每個結論都要標「觀察 / 推論 / 未知」並引用 Evidence ID 或檔名；
   靜態分析不得暗示執行過；覆蓋率不足要明講。
6. **不要重複同一輸入的同一呼叫**：先讀已回傳的 Evidence 再決定下一個聚焦問題。

---

## 10. 建議的執行順序（給後續模型）

1. `sha256sum` 兩個 APK → 綁定身分。
2. `inspect-artifact` ×2 → 判斷是否 Flutter/RN/Hermes、dex 數、`.so`、`assets/`。
3. 若是 Flutter → `identify-flutter-build` / `inspect-dart-aot`；否則略過。
4. 下載並驗證 JADX JAR → `doctor --provider jadx --json`。
5. `inspect-android-package` ×2 → manifest 宣告、權限、SDK、類別/資源量。
6. 針對**使用者真正想問的功能**（carwith / CarPlay 連線邏輯）：
   `search-android-classes` → `inspect-android-class` → `inspect-android-method` → `trace-android-references`。
7. 需要資源字串/版本時才加做 apktool 路線。
8. 產出發現報告（觀察/推論/未知 + Evidence + 覆蓋率與限制）。

> **待使用者明確指定分析目標**：目前只知道「反編譯這兩個 APK」。建議先問清楚
> 「想知道什麼」（例如：CarPlay 連線協議、藍牙/Wi-Fi 通道、憑證、更新機制、與小米 CarWith 的差異），
> 再據此收斂成 checklist，避免無目的地把整個 APK 掃一遍。

---

## 11. 文件索引（要深入時再讀）

- `docs/android-analysis.md` — JADX 路線細節、Evidence 解讀、資源/生命週期限制
- `docs/apktool-resource-analysis.md` — 資源解碼
- `docs/cli.md` — CLI 用法、輸出/退出碼、snapshot、Evidence 匯入匯出比較
- `docs/installation.md` — readiness、MCP registry、Linux owned-process 前置
- `docs/mcp-contracts.md` — MCP 會話、generated catalog、retained Evidence
- `.agents/skills/reverse-engineer-anything/references/android-applications.md` — 技能版 Android 摘要
- `src/android/JadxRelease.ts` — JADX 版本/SHA-256/補救訊息（事實來源）

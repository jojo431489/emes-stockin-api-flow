# eMES ERP 整合雙版本支援架構計畫

## 專案資訊

| 項目 | 內容 |
|------|------|
| 專案名稱 | eMES ERP 整合雙版本支援（XML/JSON）|
| 建立日期 | 2026-04-20 |
| 預估工時 | 14H |
| GitHub | https://github.com/jojo431489/emes-stockin-api-flow |

---

## 1. 目標

支援同時運行 XML（現有 WFGP）和 JSON（新 ERP）兩種資料格式，透過設定切換，無需修改業務邏輯程式碼。

---

## 2. 架構設計

### 2.1 整體架構圖

```
┌─────────────────────────────────────────────────────────────┐
│                        eMES 業務層                           │
│  (GeneralUpdater, CallErp, etc.)                            │
└─────────────────────┬───────────────────────────────────────┘
                      │ sendToErp(param, serviceName)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                  XmlToERP_handler.java                       │
│                     (統一入口)                               │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   XML 模式   │  │ JSON-ESB 模式│  │JSON-DIRECT模式│       │
│  │  (現有邏輯)  │  │  (新增邏輯)  │  │  (新增邏輯)  │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
└─────────┼─────────────────┼─────────────────┼───────────────┘
          │                 │                 │
          ▼                 ▼                 ▼
    ┌───────────┐     ┌───────────┐     ┌───────────┐
    │   WFGP    │     │    ESB    │     │  ERP API  │
    │  (XML)    │     │  (JSON)   │     │  (JSON)   │
    └───────────┘     └───────────┘     └───────────┘
```

### 2.2 設定機制

#### 系統層級設定 (CONFIG_DEF)

| CD001 | CD002 | CD003 | CD004 |
|-------|-------|-------|-------|
| ErpDataFormat | ERP資料格式 | XML | XML=WFGP XML格式, JSON-ESB=JSON透過ESB, JSON-DIRECT=JSON直連ERP |

#### API 層級設定 (ESB_PARAMETER)

新增欄位 `EP_DATA_FORMAT`，支援針對特定 API 設定不同格式。

**設定優先順序**: API 層級 > 系統層級

---

## 3. 程式碼修改

### 3.1 SFTConfig.java

**檔案**: `SFT_base/src/com/dci/sft/data/SFTConfig.java`

```java
/**
 * 取得 ERP 資料格式設定
 * @return XML | JSON-ESB | JSON-DIRECT
 */
public static String getErpDataFormat() {
    String format = getConfigValue("ErpDataFormat");
    return (format == null || format.isEmpty()) ? "XML" : format;
}

/**
 * 取得特定 API 的資料格式（支援 API 層級覆蓋）
 * @param serviceName API 名稱
 * @return XML | JSON-ESB | JSON-DIRECT
 */
public static String getApiDataFormat(String serviceName) {
    // 1. 先查 ESB_PARAMETER 的 API 層級設定
    String apiFormat = getEsbParameterFormat(serviceName);
    if (apiFormat != null && !apiFormat.isEmpty()) {
        return apiFormat;
    }
    // 2. 回退到系統層級設定
    return getErpDataFormat();
}
```

### 3.2 XmlToERP_handler.java

**檔案**: `SFT_ERPIntegrate/src/com/dci/sft/erp/XmlToERP_handler.java`

```java
/**
 * 統一 ERP 送單入口 - 根據設定自動選擇格式
 * @param param 業務資料 (JSONObject)
 * @param serviceName API 名稱
 * @return 回傳結果
 */
public static Map<String, Object> sendToErp(JSONObject param, String serviceName) {
    String format = SFTConfig.getApiDataFormat(serviceName);

    Logger logger = Logger.getLogger("at.ERPSend");
    logger.info("[sendToErp] serviceName=" + serviceName + ", format=" + format);

    switch (format) {
        case "JSON-ESB":
            return sendAsJsonViaEsb(param, serviceName);
        case "JSON-DIRECT":
            return sendAsJsonDirect(param, serviceName);
        case "XML":
        default:
            return sendAsXmlViaWfgp(param, serviceName);
    }
}

// 現有 XML 邏輯封裝
private static Map<String, Object> sendAsXmlViaWfgp(JSONObject param, String serviceName) {
    return XmlToERP(param);
}

// 新增 JSON-ESB 邏輯
private static Map<String, Object> sendAsJsonViaEsb(JSONObject param, String serviceName) {
    // JSON 格式透過 ESB 發送
    // 實作細節待補充
}

// 新增 JSON-DIRECT 邏輯
private static Map<String, Object> sendAsJsonDirect(JSONObject param, String serviceName) {
    // JSON 格式直接發送到 ERP
    // 實作細節待補充
}
```

### 3.3 GeneralUpdater.java

**檔案**: `SFT_core/src/com/dci/sft/update/GeneralUpdater.java`
**位置**: 第 1636-1641 行, 第 1714-1719 行

```java
// 修改前
Map res = XmlToERP_handler.XmlToERP(param);

// 修改後
Map res = XmlToERP_handler.sendToErp(param, "stockin.data.create");
```

---

## 4. 資料庫變更

**檔案**: `SFT_web/sql/2-2. COMPANY_WF.sql`

```sql
-- ============================================
-- eMES ERP 雙版本支援設定
-- 建立日期: 2026-04-20
-- ============================================

-- 1. 系統層級設定
IF NOT EXISTS (SELECT 1 FROM CONFIG_DEF WHERE CD001 = N'ErpDataFormat')
BEGIN
    INSERT INTO CONFIG_DEF (CD001, CD002, CD003, CD004)
    VALUES (N'ErpDataFormat', N'ERP資料格式', N'XML',
            N'XML=WFGP XML格式, JSON-ESB=JSON透過ESB, JSON-DIRECT=JSON直連ERP');
    PRINT 'CONFIG_DEF.ErpDataFormat 已新增';
END
ELSE
BEGIN
    PRINT 'CONFIG_DEF.ErpDataFormat 已存在，跳過';
END

-- 2. API 層級設定欄位
IF NOT EXISTS (SELECT 1 FROM sys.columns
               WHERE object_id = OBJECT_ID('ESB_PARAMETER')
               AND name = 'EP_DATA_FORMAT')
BEGIN
    ALTER TABLE ESB_PARAMETER ADD EP_DATA_FORMAT NVARCHAR(20) NULL;
    PRINT 'ESB_PARAMETER.EP_DATA_FORMAT 欄位已新增';
END
ELSE
BEGIN
    PRINT 'ESB_PARAMETER.EP_DATA_FORMAT 欄位已存在，跳過';
END
```

---

## 5. 修改檔案清單

| # | 檔案 | 模組 | 修改內容 |
|---|------|------|----------|
| 1 | SFTConfig.java | SFT_base | 新增 getErpDataFormat(), getApiDataFormat() |
| 2 | XmlToERP_handler.java | SFT_ERPIntegrate | 新增 sendToErp() 統一入口 |
| 3 | GeneralUpdater.java | SFT_core | 呼叫端改用 sendToErp() |
| 4 | CallErp.java | SFT_ERPIntegrate | 其他 API 呼叫端改用 sendToErp() |
| 5 | 2-2. COMPANY_WF.sql | SFT_web | 新增設定表欄位 |

---

## 6. 編譯清單

- [ ] SFT_base
- [ ] SFT_ERPIntegrate
- [ ] SFT_core
- [ ] SFT_web

---

## 7. 測試驗證

| # | 測試情境 | 設定值 | 預期結果 |
|---|----------|--------|----------|
| 1 | 預設值測試 | ErpDataFormat=XML | 使用現有 WFGP XML 流程 |
| 2 | JSON-ESB 模式 | ErpDataFormat=JSON-ESB | 使用 JSON 格式透過 ESB |
| 3 | JSON-DIRECT 模式 | ErpDataFormat=JSON-DIRECT | 使用 JSON 格式直連 ERP |
| 4 | API 層級覆蓋 | 系統=XML, API=JSON-ESB | 該 API 使用 JSON，其他用 XML |
| 5 | 入庫單完整流程 | JSON-ESB | 正確寫入 ERP 並回壓單號 |
| 6 | 錯誤處理 | 模擬 ERP 回傳錯誤 | 正確解析並顯示錯誤訊息 |

---

## 8. 部署步驟

| 步驟 | 動作 | 說明 |
|------|------|------|
| 1 | 執行 SQL 腳本 | 新增 CONFIG_DEF 和 ESB_PARAMETER 設定 |
| 2 | 部署程式 | SFT_base, SFT_ERPIntegrate, SFT_core |
| 3 | 驗證預設值 | 確認 ErpDataFormat=XML（無需變更）|
| 4 | 啟用 JSON（選用）| 修改 CONFIG_DEF 或 ESB_PARAMETER |

---

## 9. 向後相容性

| 項目 | 說明 |
|------|------|
| 預設值 | XML（現有行為不變）|
| 現有客戶 | 無需任何設定變更 |
| 新客戶 | 可選擇 JSON 格式 |
| 混合環境 | 支援部分 API 用 JSON，部分用 XML |

---

## 10. 預估工時

| 項目 | 工時 |
|------|------|
| SFTConfig.java 修改 | 2H |
| XmlToERP_handler.java 統一入口 | 4H |
| GeneralUpdater.java 等呼叫端修改 | 2H |
| SQL 腳本 | 1H |
| 單元測試 | 2H |
| 整合測試 | 3H |
| **合計** | **14H** |

---

## 11. 注意事項

1. **JSON 格式需 ESB 支援**：若 ESB 不支援 JSON，需先升級 ESB
2. **回應格式統一**：無論發送格式，回應統一轉為 JSONObject 給業務層
3. **錯誤處理**：JSON 和 XML 的錯誤格式不同，需統一處理
4. **LOG 記錄**：記錄使用的格式類型，便於除錯

---

## 12. 相關文件

| 文件 | 連結 |
|------|------|
| API 規格書（XML 版）| https://jojo431489.github.io/emes-stockin-api-flow/stockin-api-flow-xml.html |
| API 規格書（JSON 版）| https://jojo431489.github.io/emes-stockin-api-flow/ |
| GitHub Repository | https://github.com/jojo431489/emes-stockin-api-flow |

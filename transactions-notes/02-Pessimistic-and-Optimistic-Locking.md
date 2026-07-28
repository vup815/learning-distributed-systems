# 悲觀鎖與樂觀鎖重點筆記

> 整理日期：2026-07-28  
> 來源：*Designing Data-Intensive Applications* Chapter 7 + 實務補充  
> **說明**：本文件包含 Mermaid 時序圖，GitHub 原生支援渲染。

---

## 1. 為什麼需要鎖？

在較弱的隔離層級（Read Committed、Snapshot Isolation）下，資料庫不會自動防止所有併發問題（尤其是 Lost Update 與 Write Skew）。  
應用層必須自己選擇適當的併發控制策略：

- **悲觀鎖（Pessimistic Locking）**：先鎖住，再操作（假設會衝突）
- **樂觀鎖（Optimistic Locking）**：先操作，寫入時再檢查（假設不會衝突）

---

## 2. 悲觀鎖（Pessimistic Locking）

### 2.1 核心語法

| 語法 | 鎖類型 | 說明 |
|------|--------|------|
| `SELECT ... FOR UPDATE` | 排他鎖（Exclusive / X Lock） | 我打算修改，別人都不准再拿鎖或修改 |
| `SELECT ... FOR SHARE` | 共享鎖（Shared / S Lock） | 我只讀，但要確保期間不被修改；多人可同時持有 |

### 2.2 SELECT FOR UPDATE vs FOR SHARE

| 項目 | `FOR UPDATE` | `FOR SHARE` |
|------|--------------|-------------|
| 鎖類型 | 排他鎖 | 共享鎖 |
| 其他事務能否再拿共享鎖 | ❌ | ✅ |
| 其他事務能否再拿排他鎖 | ❌ | ❌ |
| 其他事務能否 UPDATE/DELETE | ❌ | ❌ |
| 其他事務普通 SELECT | ✅ 可以 | ✅ 可以 |
| 主要用途 | 讀完後很可能要寫 | 只讀，但要保護資料不被改 |

**重要**：兩者都**不會阻擋普通 SELECT**（因為現代資料庫使用 MVCC）。

### 2.3 典型使用場景

**FOR UPDATE（最常用）**：
- 庫存扣減、餘額變更
- 搶票 / 秒殺
- 醫生下線前鎖住相關資料
- 任何 read-modify-write 且衝突代價高的場景

```sql
BEGIN;
SELECT * FROM inventory WHERE product_id = 100 FOR UPDATE;
-- 檢查庫存
UPDATE inventory SET stock = stock - 1 WHERE product_id = 100;
COMMIT;
```

**FOR SHARE**：
- 讀取主資料後插入子資料（確保主資料還在）
- 驗證關聯資料仍然存在
- 多筆資料一起做一致性檢查，但自己不修改

```sql
BEGIN;
SELECT * FROM orders WHERE id = 123 FOR SHARE;
INSERT INTO shipping_logs (order_id, ...) VALUES (123, ...);
COMMIT;
```

### 2.4 注意事項

- 鎖會持有到 `COMMIT` / `ROLLBACK` 才釋放 → 事務必須短小
- 容易產生死鎖（見 03-Deadlocks.md）
- 高併發下效能較差（排隊等待）

---

## 3. 樂觀鎖（Optimistic Locking）

### 3.1 核心思想

> 先假設不會有衝突，真的要寫入時再檢查。如果有人改過，就失敗並重試（或回傳錯誤）。

不依賴資料庫鎖，而是用**資料本身的版本資訊**來偵測衝突。

### 3.2 最常見實作：Version 欄位

```sql
CREATE TABLE doctors (
  id          SERIAL PRIMARY KEY,
  name        TEXT NOT NULL,
  on_call     BOOLEAN NOT NULL,
  version     INT NOT NULL DEFAULT 0   -- ★ 關鍵
);
```

**完整流程**：

```sql
-- 1. 讀取（不需加鎖）
SELECT id, on_call, version FROM doctors WHERE id = 1;
-- 假設讀到 version = 5

-- 2. 業務邏輯判斷...

-- 3. 寫入時帶上舊 version
UPDATE doctors
SET on_call = false,
    version = version + 1
WHERE id = 1
  AND version = 5;          -- 只有版本沒被改過才成功
```

- `rowcount = 1` → 成功  
- `rowcount = 0` → 衝突（中間有人改過）

### 3.3 為什麼能防止 Lost Update？

| 時間 | 事務 A | 事務 B | 結果 |
|------|--------|--------|------|
| T1 | 讀 version=5 | | |
| T2 | | 讀 version=5 | |
| T3 | | UPDATE ... version=5 成功 → version=6 | |
| T4 | UPDATE ... version=5 → **失敗** | | 衝突被偵測 |

### 3.4 應用層完整重試範例

```python
def go_off_call(doctor_id: int, max_retries: int = 3):
    for attempt in range(max_retries):
        doctor = db.query("SELECT * FROM doctors WHERE id = %s", [doctor_id])
        
        if not can_go_off_call(doctor):
            raise BusinessError("目前無法下線")
        
        rowcount = db.execute("""
            UPDATE doctors
            SET on_call = false, version = version + 1
            WHERE id = %s AND version = %s
        """, [doctor_id, doctor.version])
        
        if rowcount == 1:
            return  # 成功
        
        if attempt == max_retries - 1:
            raise ConcurrentModificationError("資料已被修改，請重新整理後再試")
        
        time.sleep(0.01 * (2 ** attempt))  # 指數退避
```

### 3.5 樂觀鎖一定要重試嗎？

**不一定必須**，但實務上幾乎都會做。

| 處理方式 | 說明 | 適用場景 |
|----------|------|----------|
| 自動重試 | 重新讀取 → 重新判斷 → 再更新 | 大多數 API、背景任務（推薦） |
| 直接回傳錯誤 | 告訴前端「請重新整理」 | 管理後台、衝突率極低 |
| 衝突合併 | 像 Git merge 一樣合併雙方修改 | 協作文件（複雜） |

樂觀鎖只負責**發現衝突**，重試負責**解決衝突**。

### 3.6 Version 欄位設計建議

- 型別：優先使用 `INT` 或 `BIGINT`（從 0 開始）
- 更新時一定要用 `version = version + 1`，不要寫死數字
- 設為 `NOT NULL DEFAULT 0`

其他變體：CAS（`WHERE value = ?`）、ETag / Hash、ORM 的 `@Version`。

---

## 4. 悲觀鎖 vs 樂觀鎖比較

| 項目 | 悲觀鎖（FOR UPDATE） | 樂觀鎖（Version） |
|------|----------------------|-------------------|
| 衝突偵測時機 | 讀取時就鎖住 | 寫入時才檢查 |
| 低衝突效能 | 較差（有鎖開銷） | **非常好** |
| 高衝突效能 | 較穩定（排隊） | 重試變多，效能下降 |
| 死鎖風險 | 有 | **幾乎沒有** |
| 事務時間限制 | 必須很短 | 可以較長 |
| 實作複雜度 | 較低 | 中等（需處理重試） |
| 能否防止 Write Skew | 可以（鎖對範圍） | **通常不行** |

**選型建議**：
- 衝突率低、一般業務資料 → **樂觀鎖**
- 庫存、餘額、熱點資源、秒殺 → **悲觀鎖** 或原子操作
- 有複雜不變式 → 悲觀鎖 + 物化衝突，或直接用 Serializable

---

## 5. 物化衝突（Materializing the Conflict）

解決 Write Skew 的實務技巧：把「分散在多個 row 的不變式」改成強制更新**同一筆 row**。

### Table 設計

```sql
CREATE TABLE doctors (
  id          SERIAL PRIMARY KEY,
  name        TEXT NOT NULL,
  shift_id    INT NOT NULL,
  on_call     BOOLEAN NOT NULL
);

-- ★ 物化計數器
CREATE TABLE shift_on_call_count (
  shift_id    INT PRIMARY KEY,
  count       INT NOT NULL CHECK (count >= 0)
);
```

### 下線 Transaction

```sql
BEGIN;

UPDATE shift_on_call_count
SET count = count - 1
WHERE shift_id = 123
  AND count > 1;          -- 關鍵條件

-- 檢查 rowcount，若為 0 則 ROLLBACK

UPDATE doctors
SET on_call = false
WHERE name = 'Alice' AND shift_id = 123;

COMMIT;
```

因為所有人都必須更新**同一筆計數器 row**，即使在 Snapshot Isolation 下也會產生寫寫衝突。

---

## 6. 快速複習清單

- [ ] 能清楚說明 `FOR UPDATE` 與 `FOR SHARE` 的差異
- [ ] 知道普通 SELECT 不會被這兩種鎖擋住
- [ ] 能寫出 Version 欄位的樂觀鎖 UPDATE 語句
- [ ] 知道樂觀鎖發生衝突後常見的三種處理方式
- [ ] 能解釋為什麼樂觀鎖擋不住 Write Skew
- [ ] 能說出物化計數器的用途與基本寫法

---

整理完成後可直接用於面試複習與實務決策。

# Transactions Notes（DDIA Ch.7）

單一節點事務的隔離層級、併發異常、鎖機制與死鎖筆記。

這是學習分散式事務（2PC / TCC / Saga）前的重要基礎。

## 檔案結構

| 檔案 | 內容 |
|------|------|
| [01-Transaction-Isolation.md](./01-Transaction-Isolation.md) | ACID、六大併發異常（含時序圖）、隔離層級對照表、MVCC / 2PL / SSI |
| [02-Pessimistic-and-Optimistic-Locking.md](./02-Pessimistic-and-Optimistic-Locking.md) | 悲觀鎖（FOR UPDATE / FOR SHARE）、樂觀鎖（Version）、物化衝突、選型比較 |
| [03-Deadlocks.md](./03-Deadlocks.md) | 死鎖原因、偵測、預防與應用層重試 |

## 核心重點速覽

### 併發異常

| 異常 | 最弱能防止的層級 | 備註 |
|------|------------------|------|
| Dirty Read | Read Committed | 幾乎所有系統都防 |
| Dirty Write | Read Committed | 幾乎所有系統都防 |
| Read Skew / Non-repeatable Read | Snapshot Isolation | SI 的主要價值 |
| Lost Update | SI（多數實作）或原子/樂觀鎖 | 應用層常需額外處理 |
| Write Skew | **Serializable** | SI 擋不住，最容易踩坑 |
| Phantom | Serializable | 與 Write Skew 相關 |

### 鎖與併發控制選型

| 場景 | 建議 |
|------|------|
| 一般業務、衝突率低 | 樂觀鎖（Version + 重試） |
| 庫存 / 餘額 / 熱點資源 | 悲觀鎖（FOR UPDATE）或原子操作 |
| 有複雜不變式 | 悲觀鎖 + 物化計數器，或 Serializable |
| 需要一致快照的報表 | Snapshot Isolation |

## 選型建議速記

- **一般業務** → Read Committed + 樂觀鎖
- **需要一致快照** → Snapshot Isolation
- **高競爭資源** → 悲觀鎖 或 原子操作
- **複雜不變式** → Serializable 或物化衝突
- **金融核心** → Serializable + 充分測試

---

整理日期：2026-07-28

# Transactions Notes（DDIA Ch.7）

單一節點事務的隔離層級、併發異常與解決方案筆記。

這是學習分散式事務（2PC / TCC / Saga）前的重要基礎。

## 檔案結構

| 檔案 | 內容 |
|------|------|
| [01-Transaction-Isolation.md](./01-Transaction-Isolation.md) | ACID、六大併發異常（含時序圖）、隔離層級對照表、MVCC / 2PL / SSI、實務選型建議 |

## 核心重點速覽

| 異常 | 最弱能防止的層級 | 備註 |
|------|------------------|------|
| Dirty Read | Read Committed | 幾乎所有系統都防 |
| Dirty Write | Read Committed | 幾乎所有系統都防 |
| Read Skew / Non-repeatable Read | Snapshot Isolation | SI 的主要價值 |
| Lost Update | SI（多數實作）或原子操作 | 應用層常需額外處理 |
| Write Skew | **Serializable** | SI 擋不住，最容易踩坑 |
| Phantom | Serializable | 與 Write Skew 相關 |

## 選型建議

- **一般業務** → Read Committed（效能優先）
- **需要一致快照** → Snapshot Isolation
- **有複雜不變式** → Serializable 或顯式 `FOR UPDATE`
- **計數器 / 庫存** → 原子操作 + 樂觀鎖重試

---

整理日期：2026-07-28

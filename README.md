# 學習分散式系統

這個倉庫用來整理學習分散式系統的相關筆記。

## 目前內容

### [distributed-lock-notes](./distributed-lock-notes/)

分散式鎖相關筆記（Redis / ZooKeeper / etcd + Raft）：

- [01-Redis-Distributed-Lock.md](./distributed-lock-notes/01-Redis-Distributed-Lock.md)
- [02-ZooKeeper-Distributed-Lock.md](./distributed-lock-notes/02-ZooKeeper-Distributed-Lock.md)
- [03-etcd-and-Raft.md](./distributed-lock-notes/03-etcd-and-Raft.md)

### [transactions-notes](./transactions-notes/)

單一節點事務隔離層級、併發異常、鎖機制與死鎖（DDIA Ch.7）：

- [01-Transaction-Isolation.md](./transactions-notes/01-Transaction-Isolation.md)
- [02-Pessimistic-and-Optimistic-Locking.md](./transactions-notes/02-Pessimistic-and-Optimistic-Locking.md)
- [03-Deadlocks.md](./transactions-notes/03-Deadlocks.md)

後續可以繼續新增其他主題（分散式事務 2PC/TCC/Saga、服務發現、一致性協議…等）。

---

整理開始日期：2026-07-28

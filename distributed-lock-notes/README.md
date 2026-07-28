# Distributed Lock Notes

Redis / ZooKeeper / etcd 分散式鎖與 Raft 共識演算法重點筆記

## 檔案結構

| 檔案 | 內容 |
|------|------|
| [01-Redis-Distributed-Lock.md](./01-Redis-Distributed-Lock.md) | Redis 分散式鎖、強一致性問題、可重入、Watchdog |
| [02-ZooKeeper-Distributed-Lock.md](./02-ZooKeeper-Distributed-Lock.md) | ZooKeeper 臨時順序節點機制 + 完整時序圖 |
| [03-etcd-and-Raft.md](./03-etcd-and-Raft.md) | etcd 鎖實作 + Raft 演算法詳解與選舉時序圖 |

## 快速對照

| 項目              | Redis                          | ZooKeeper                          | etcd                              |
|-------------------|--------------------------------|------------------------------------|-----------------------------------|
| **一致性模型**    | AP（最終一致 / 高可用）        | CP（強一致）                       | CP（強一致）                      |
| **共識演算法**    | 無（非同步複製）               | ZAB                                | **Raft**                          |
| **鎖的正確性**    | 高機率正確                     | 強一致                             | 強一致                            |
| **效能**          | 極高                           | 中等                               | 中高                              |
| **推薦框架**      | Redisson                       | Apache Curator                     | etcd concurrency / clientv3       |
| **典型場景**      | 高併發、秒殺、限流             | 金融、核心協調、選舉               | Kubernetes、雲原生                |

## 選型建議

- **高併發 + 可接受極少數衝突** → Redis (Redisson)
- **強一致性 + 已有 ZK 叢集** → ZooKeeper
- **強一致性 + Kubernetes / 雲原生** → etcd
- **正確性要求極高（金融級）** → ZooKeeper 或 etcd

---

整理日期：2026-07-28

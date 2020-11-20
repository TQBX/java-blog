| Feature              | euerka                       | Consul                 | zookeeper             |
| :------------------- | :--------------------------- | :--------------------- | :-------------------- |
| 服务健康检查         | 可配支持                     | 服务状态，内存，硬盘等 | (弱)长连接，keepalive |
| 多数据中心           | —                            | 支持                   | —                     |
| kv 存储服务          | —                            | 支持                   | 支持                  |
| 一致性               | —                            | raft                   | paxos                 |
| CAP                  | AP                           | CP                     | CP                    |
| 使用接口(多语言能力) | http（sidecar）              | 支持 http 和 dns       | 客户端                |
| watch 支持           | 支持 long polling/大部分增量 | 全量/支持long polling  | 支持                  |
| 自身监控             | metrics                      | metrics                | —                     |
| 安全                 | —                            | acl /https             | acl                   |
| 语言                 | Java                         | Go                     | Java                  |
| SpringCloud集成      | 已集成                       | 已集成                 | 已集成                |


## 背景介绍
PostgreSQL 主从复制的逻辑在具有大量数据表的 DDL 变更时会导致从库恢复数据极其缓慢。为了避免此类问题，TencentDB 针对此类场景进行了优化和修改。

## 原理介绍
PostgreSQL 是通过物理复制实现主从复制。日志同步到备机之后，备机会解析 wal 日志，来与主库保持数据一致。在 PostgreSQL 从库在恢复一条 drop 类语句时要做的操作如下：

1. 恢复系统表，例如 pg_class，pg_attrbute，pg_type 等，相当于移除表的元信息。
2. close 表对应的文件。
3. 遍历 buffer 中的页面，如果缓存的是该表的页面，则标记为 invalid，后面其他进程可以使用该页面。
4. 发异步失效消息给其他 backend，通知该表已删除。
5. 删除表对应的物理文件。

但在 PostgreSQL 内核中第3步将会执行 DropRelFileNodesAllBuffers 函数。其函数需要遍历整个 shard_buffer，以查看 buffer 是否缓存有将要删除的表的数据，将其标记为失效。而 PostgreSQL 中页面大小默认为8K，以 shard_buffer 大小16GB为例，则一共有16GB/8K = 200W个 page，每删除一个表这里需要循环200万次，如果表上面有索引，每个索引也要循环200万次。

所以从业务上看，当存在大量表变更并且快速删除表操作的时候，由于主库可以并发执行所以感觉不出性能的影响，但是因为 PostgreSQL 的备库是单进程的 recovery，就会出现主备同步日志堆积，数据延迟的问题。

## 问题解决
腾讯云数据库 PostgreSQL 在 recover drop table 操作时，将表信息写入一个共享的 hash 表中。当存在表文件删除场景下，不再直接进行物理删除，将删除实际文件的动作作为异步动作存储其中。

当 invalid buffer 结束时将表从 hash 表中移除，这样，如果在此过程中发生打开文件失败，检查是否存在此 hash 表中即可。并且如果在新创建文件的时候也去遍历一下此队列，若队列中存在同名文件正在 invalied buffer，则等待即可。
而 PostgreSQL 关于表文件命名是一个 uint32 整数保存，采用的是“全局分配，局部存储”的方式，即一个实例下的所有数据库使用一个计数器生成文件号，生成的文件保存在各自库的目录下，分配时，如果当前库下已有同名文件，则尝试下一个，直到没有冲突为止，计数器绕圈后重新开始。

经过优化后，可以明显发现同类场景下主从同步性能增强了3W多倍。

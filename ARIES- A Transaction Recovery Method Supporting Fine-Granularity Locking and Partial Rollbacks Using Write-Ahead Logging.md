- 支持 Tuple-Level Lock，只在 Page 上加 Latch（换句话说，支持 Early Lock Release 技术）

- 支持 Page-Oriented Redo，Logical Undo

- 支持 Fuzzy Checkpoint（下文会详细介绍）

- 支持 Partial Rollback（下文会详细介绍）

- 支持 Nested Top Actions（下文会详细介绍）

  ---

- **LSN**：全称是 Log Sequence Number。它是 Log 的编号，始终单调递增。
- **Log Header**：每一条 Log Record 都是由 Log Header 和具体的 Log 数据组成。Log Header 中一般有 Log Size，LSN，Transaction ID，Prev LSN（见下面介绍），LogType（见下面介绍） 等字段。
- **Prev LSN**：同一个事务中后一条 Log 会把前一条 Log 的 LSN 记录在 Log Header 的 Prev LSN 字段中，形成一个反向链表，便于 Undo 时逆序回溯事务的 Log。
- **Log Type**：一般有事务相关的 Type：Begin，Commit，Abort，End（End 有点特殊，下文会详细介绍）；普通 Log（包含 Redo Log，Undo Log 和 CLR（见下面介绍））的 Type：Insert，Update，MarkDelete，ApplyDelete，RollbackDelete 等；Checkpoint Log 的 Type：Checkpoint；为一个文件分配新 Page 的 Type：NewPage。
- **CLR**：全称是 Compensation Log Record，中文一般翻译为补偿日志。由于 ARIES 采用 Logical Undo，Undo 操作不是幂等的，不可以重复执行。我们通过为 Undo 操作记录 Redo Log 来物化 Undo 操作，同时记录 Undo 的进展（通过 UndoNextLSN 的实现，见下面介绍），保证已经 Undo 了的操作不会再被 Undo。Undo 产生的这些 Redo Log 就叫做 CLR。此外，CLR 是 **Redo-Only** 的，不支持 Undo，这一特点保证了 Undo 是有界的，Recovery 期间 Undo 过程中挂掉并不会增加 Undo 的工作量。这就是为什么 ARIES 要「Logging changes during undo」。
- **UndoNext LSN**：每条 CLR 中都会记录 UndoNext LSN，用来指示下一条需要 Undo 的 Log 的 LSN。如果 Undo 到一半数据库挂掉后重启，我们在重新执行 Undo 时，只需先取出最后一条 CLR Log 的 UndoNext LSN，就能继续之前的 Undo 工作。
- **Log Buffer**：内存中分配的用来临时存放 Log 的一段空间。事务执行期间写 Log 只写到 Log Buffer 中，直到事务提交的时候统一刷盘。这种做法有利于减少磁盘 IO，提升性能。（很方便我们实现 Group Commit 特性，本文不展开。）
- **Master Record**：磁盘上的一个文件，记录最近一次 Checkpoint Log 的 LSN。它能够帮助我们在 Recovery 期间快速找到最近一次 Checkpoint 的 Log。
- **Flushed LSN**：已经刷到磁盘上的 Log 中最大的 LSN。它是一个内存中的变量。
- **Page LSN**：对于一个 Page，最近一次更新操作对应的 Log 的 LSN。记录在 Page Header 中。
- **Rec LSN**：对于一个 Page，自从上一次刷盘以来，第一次更新操作对应的 Log 的 LSN。
- **Last LSN**：每个事务最近一次更新操作对应的 Log 的 LSN。
- **DPT**：全称是 Dirty Page Table。记录了所有的 Dirty Page 以及它们的 Rec LSN。它是一个内存中的数据结构。
- **ATT**：全称是 Active Transaction Table。记录了所有活跃事务（事务在执行中，还没有提交或回滚）以及它们的状态（Undo Candidate，Committed，下文会介绍）和 Last LSN。它也是内存中的一个数据结构。
## Lab 3: Fault-tolerant Key/Value Service
### 介绍
在本实验中，您将使用实验2中的 Raft库构建容错键/值存储服务 。您的键/值服务将是一个复制状态机，由使用Raft维护复制的几个键/值服务器组成。
只要大多数服务器都处于活动状态并且可以进行通信，即使存在其他故障或网络分区，您的键/值服务也应继续处理客户端请求。

该服务支持三种操作：Put（key，value）， Append（key，arg）和Get（key）。它维护着一个简单的键/值对数据库。
Put（）替换数据库中特定键的值，Append（key，arg） 将arg附加到键的值，而Get（）获取键的当前值。一个追加到一个不存在的关键应该像戴上。
每个客户都通过带有Put / Append / Get方法的职员与服务进行对话。一个店员管理与服务器的RPC相互作用。

您的服务必须为对Clerk Get / Put / Append方法的应用程序调用提供高度一致性。
这就是我们所说的强一致性。如果一次调用一次，则Get / Put / Append方法应像系统只有其状态的一个副本一样工作，
并且每次调用都应遵循对前面调用序列所隐含的状态的修改。对于并发调用，返回值和最终状态必须与这些操作以某种顺序一次执行一次相同。
如果调用在时间上重叠，则这些调用是并发的，例如，如果客户端X调用Clerk.Put（），则客户端Y调用Clerk.Append（），然后客户X的呼叫返回。
此外，呼叫必须观察呼叫开始之前已完成的所有呼叫的效果（因此，我们在技术上要求线性化）。

强大的一致性对应用程序很方便，因为这意味着非正式地，所有客户端都看到相同的状态，并且他们都看到最新的状态。对于单个服务器，提供强一致性相对容
易。如果要复制该服务，将更加困难，因为所有服务器必须为并发请求选择相同的执行顺序，并且必须避免使用最新状态来回复客户端。

本实验分为两个部分。在A部分中，您将实现服务，而不必担心Raft日志会无限增长。在B部分中，您将实现快照（本文的第7节），这将使Raft能够垃圾收集
旧的日志条目。请在相应的截止日期之前提交每个部分。

### 提示
- 本实验不需要您编写太多代码，但是您很可能会花费大量时间思考和盯着调试日志来弄清楚为什么您的实现不起作用。与Raft实验室相比，调试将更具挑战性，因为有更多组件彼此异步工作。尽早开始。
- 您应该重新阅读 扩展的Raft论文，尤其是第7和8节。要获得更广阔的视野，请看一下Chubby，Raft Made Live，Spanner，Zookeeper，Harp，
Viewstamped Replication和 Bolosky等。
- 您可以将字段添加到Raft ApplyMsg，也可以将字段添加到Raft RPC（例如AppendEntries）。但是请确保您的代码继续通过Lab 2测试。

### 合作政策
您必须编写您在6.824中提交的所有代码，但我们作为分配的一部分提供给您的代码除外。不允许您查看其他任何人的解决方案，不允许您查看前几年的代码，
也不允许您查看其他Raft实现。您可以与其他学生讨论作业，但不能查看或复制彼此的代码。

请不要发布您的代码或将其提供给现在或将来的6.824学生。 github.com仓库默认是公开的，因此除非您将仓库设为私有，否则请不要在其中放置代码。
您可能会发现使用MIT的GitHub方便 ，但是请确保创建一个私有存储库。

### 入门
我们为您提供src / kvraft中的框架代码和测试。您将需要修改kvraft / client.go，kvraft / server.go，甚至可能是kvraft / common.go。

要启动并运行，请执行以下命令:
```bash
$ cd ~/6.824
$ git pull
...
$ cd src/kvraft
$ GOPATH=~/6.824
$ export GOPATH
$ go test
...
$
```

### A部分：不进行日志压缩的键/值服务
您的每个键/值服务器（“ kvservers”）将具有一个关联的Raft对等方。职员将Put（），Append（）和Get（） RPC发送到与之相关的Raft是领导者的
kvserver。kvserver代码将Put / Append / Get操作提交给Raft，以便Raft日志保存一系列Put / Append / Get操作。所有kvserver都按顺序执行
Raft日志中的操作，并将这些操作应用于其键/值数据库；目的是让服务器维护键/值数据库的相同副本。
    
一个店员有时不知道哪个kvserver是筏领先地位。如果秘书将RPC发送到错误的kvserver，或者它无法到达kvserver，则秘书应通过发送到其他kvserver
重试。如果键/值服务将操作提交到其Raft日志（因此将操作应用于键/值状态机），则领导者将通过响应其RPC 向秘书报告结果。如果操作未能提交（例如，
如果更换了领导者），则服务器将报告错误，并且秘书将使用其他服务器重试。
> tasks

您的第一个任务是实现一个解决方案，该解决方案在没有丢弃的消息且没有故障的服务器时也可以使用。

你需要RPC-发送代码添加到店员认沽/追加/获取的方法client.go，并实现 PutAppend（）和get（）方法 RPC处理程序 server.go。这些处理程序应使
用Start（）在Raft日志中输入 Op；您应该在server.go中填写Op struct定义，以便它描述Put / Append / Get操作。当Raft提交时，每个服务器都
应执行Op命令，即它们出现在applyCh上。RPC处理程序应注意Raft提交其Op的时间，然后回复RPC。

当您可靠地通过了测试套件中的第一个测试：“一个客户端” 时，您就已经完成了此任务 。您可能还会发现，您可以通过“并发客户端”测试，具体取决于实现
的复杂程度。

> note：您的kvserver不应直接通信；他们只能通过Raft日志相互互动。

#### 提示
- 调用Start（）之后，您的kvservers将需要等待Raft完成协议。已商定的命令到达applyCh。您应该仔细考虑如何安排代码，以便它继续读取applyCh，
而PutAppend（）和Get（）处理程序使用Start（）将命令提交到Raft日志。在kvserver及其Raft库之间很容易实现死锁。
- 您的解决方案需要处理领导者为文员的RPC调用Start（）的情况，但是在将请求提交到日志之前失去其领导作用。在这种情况下，您应该安排服务员将请求
重新发送到其他服务器，直到找到新的领导者为止。一种实现方法是，服务器注意到Start（）返回的索引上出现了另一个请求，或者Raft的术语已更改，从
而检测到服务器失去了领导地位。如果前任领导者是自己分开的，就不会知道新任领导者。但是同一分区中的任何客户端也将无法与新的领导者对话，因此在这
种情况下，服务器和客户端可以无限期等待直到分区恢复正常。
- 您可能必须修改秘书，以记住原来是最后一个RPC的服务器的领导者，然后将下一个RPC首先发送到该服务器。这样可以避免浪费时间在每个RPC上搜索领导
者，这可以帮助您足够快地通过某些测试。
- 如果kvserver 不属于多数服务器，则它不应完成Get（） RPC（这样它就不会提供过时的数据）。一个简单的解决方案是在Raft日志中输入每个Get（）
（以及每个Put（） 和Append（））。您不必实现第8节中描述的只读操作的优化。
- 最好从一开始就添加锁定，因为避免死锁的需求有时会影响整体代码设计。使用go test -race检查您的代码是否运行 正常。

面对不可靠的连接和服务器故障， 秘书可能会多次发送RPC，直到找到可以肯定答复的kvserver。如果领导者在提交筏日志后刚失败，则秘书可能不会收到答
复，因此可能会将请求重新发送给另一个领导者。每次对Clerk.Put（）或Clerk.Append（）的调用都 只会导致一次执行，因此您必须确保重新发送不会导
致服务器两次执行请求。

> tasks

添加代码以应对重复的文员请求，包括以下情况：文员在一个任期内将请求发送给kvserver负责人，超时等待答复，然后在另一任期内将请求重新发送给新的
负责人。该请求应始终仅执行一次。您的代码应通过go测试-运行3A测试。

#### 提示
- 您将需要唯一地标识客户端操作，以确保键/值服务仅执行一次。
- 您的重复检测方案应快速释放服务器内存，例如，通过让每个RPC暗示客户端已经看到其先前RPC的答复。可以假设一个客户一次只打给一个职员一个电话。

您的代码现在应该通过Lab 3A测试，如下所示：
```bash
$ go test -run 3A
Test: one client (3A) ...
  ... Passed --  15.1  5 12882 2587
Test: many clients (3A) ...
  ... Passed --  15.3  5  9678 3666
Test: unreliable net, many clients (3A) ...
  ... Passed --  17.1  5  4306 1002
Test: concurrent append to same key, unreliable (3A) ...
  ... Passed --   0.8  3   128   52
Test: progress in majority (3A) ...
  ... Passed --   0.9  5    58    2
Test: no progress in minority (3A) ...
  ... Passed --   1.0  5    54    3
Test: completion after heal (3A) ...
  ... Passed --   1.0  5    59    3
Test: partitions, one client (3A) ...
  ... Passed --  22.6  5 10576 2548
Test: partitions, many clients (3A) ...
  ... Passed --  22.4  5  8404 3291
Test: restarts, one client (3A) ...
  ... Passed --  19.7  5 13978 2821
Test: restarts, many clients (3A) ...
  ... Passed --  19.2  5 10498 4027
Test: unreliable net, restarts, many clients (3A) ...
  ... Passed --  20.5  5  4618  997
Test: restarts, partitions, many clients (3A) ...
  ... Passed --  26.2  5  9816 3907
Test: unreliable net, restarts, partitions, many clients (3A) ...
  ... Passed --  29.0  5  3641  708
Test: unreliable net, restarts, partitions, many clients, linearizability checks (3A) ...
  ... Passed --  26.5  7 10199  997
PASS
ok      kvraft  237.352s
```

每次通过之后的数字是实时时间(以秒为单位),对等体数量，发送的RPC数量(包括客户端RPC)以及已执行的键/值操作的数量(文员Get/Put/Append调用)。

###B部分：具有日志压缩功能的键/值服务
现在，您的实验室代码已成为现实，重新启动服务器会重播完整的Raft日志，以恢复其状态。但是，长时间运行的服务器永远记住完整的Raft日志是不切实际
的.相反，您将修改Raft和kvserver以节省空间：不时kvserver将持久存储其当前状态的“快照”，而Raft将丢弃快照之前的日志条目。服务器重新启动时(
或远远落后于领导者，必须赶上来),服务器首先安装快照，然后从创建快照的那一刻开始重放日志条目。扩展的Raft论文的第7节 概述了该方案；您将必须设
计细节。

您应该花一些时间弄清楚Raft库和服务之间的接口,以便Raft库可以丢弃日志条目。想一想您的Raft在仅存储日志尾部的情况下将如何操作，以及它将如何丢
弃旧的日志条目。您应该以允许Go垃圾收集器释放并重新使用内存的方式丢弃它们；这要求没有对废弃日志条目的可达引用（指针）。

测试人员将maxraftstate传递给您的 StartKVServer（）。maxraftstate表示您的持久Raft状态允许的最大大小（以字节为单位）（包括日志，但不包
括快照）。您应该将maxraftstate与persister.RaftStateSize（）进行比较。每当您的键/值服务器检测到Raft状态大小接近此阈值时，它都应保存一个
快照，并告知Raft库它已快照，以便Raft可以丢弃旧的日志条目。如果maxraftstate为-1，则不必快照。

> tasks

您的raft.go可能会将整个日志保留在Go切片中。对其进行修改，以便可以为其指定日志索引，丢弃该索引之前的条目，并在仅存储该索引之后的日志条目的同
时继续操作。进行这些更改后，请确保您通过了所有的Raft测试。
> tasks

修改您的kvserver，以使其检测到持久化Raft状态何时变得太大，然后将快照交给Raft，并告诉Raft它可以丢弃旧的日志条目.
Raft应该使用persister.SaveStateAndSnapshot（）保存每个快照（不要使用文件）。kvserver实例重新启动时，应从持久性存储中还原快照。

#### 提示
- 您可以通过运行Lab 3A测试，同时人为地将maxraftstate设置为1，来测试Raft和kvserver使用修剪后的日志进行操作的能力，以及从kvserver快照和
持久化Raft状态的组合重新启动的能力 。
- 考虑一下kvserver何时应快照其状态以及快照应包含什么内容。Raft必须使用SaveStateAndSnapshot（）将每个快照以及相应的Raft状态存储在持久性
对象中 。您可以使用ReadSnapshot（）读取最新存储的快照。
- 您的kvserver必须能够跨检查点在日志中检测到重复的操作，因此，用于检测它们的任何状态都必须包含在快照中。请记住要大写存储在快照中的结构的所
有字段。
- 您可以在Raft中添加方法，以便kvserver可以管理整理Raft日志的过程并管理kvserver快照。
> tasks

修改您的Raft领导者代码，以在领导者丢弃跟随者所需的日志条目时将InstallSnapshot RPC发送给跟随者。当关注者收到InstallSnapshot RPC时,
您的Raft代码将需要将随附的快照发送到其kvserver。 为此，您可以通过向ApplyMsg添加新字段来使用applyCh。通过所有Lab 3测试后，您的解决方案
即告完成。

> Note:该maxraftstate限制适用于采空区编码的字节你筏传递给persister.SaveRaftState().

#### 提示
您应该在单个InstallSnapshot RPC中发送整个快照。您不必实现图13的偏移机制来拆分快照。
在继续进行其他Snapshot测试之前， 请确保您通过TestSnapshotRPC。
进行Lab 3测试的合理时间是400秒的实时时间和700秒的CPU时间。此外， 进行test -run TestSnapshotSize的实时时间应少于20秒。

您的代码应通过3B测试（如此处的示例）以及3A测试。


```bash
$ go test -run 3B
Test: InstallSnapshot RPC (3B) ...
  ... Passed --   1.5  3   163   63
Test: snapshot size is reasonable (3B) ...
  ... Passed --   0.4  3  2407  800
Test: restarts, snapshots, one client (3B) ...
  ... Passed --  19.2  5 123372 24718
Test: restarts, snapshots, many clients (3B) ...
  ... Passed --  18.9  5 127387 58305
Test: unreliable net, snapshots, many clients (3B) ...
  ... Passed --  16.3  5  4485 1053
Test: unreliable net, restarts, snapshots, many clients (3B) ...
  ... Passed --  20.7  5  4802 1005
Test: unreliable net, restarts, partitions, snapshots, many clients (3B) ...
  ... Passed --  27.1  5  3281  535
Test: unreliable net, restarts, partitions, snapshots, many clients, linearizability checks (3B) ...
  ... Passed --  25.0  7 11344  748

PASS
```
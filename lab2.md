## lab2 -raft
### 介绍
这是您构建容错键/值存储系统的系列实验中的第一个。在本实验中，您将实现Raft（复制状态机协议）。
在下一个实验中，您将在Raft之上构建键/值服务。然后，您将在多个复制状态机上“分片”服务，以提高性能。

复制服务通过将其状态（即数据）的完整副本存储在多个副本服务器上来实现容错功能。
即使服务的某些服务器出现故障（崩溃，网络故障或不稳定）,复制也可以使服务继续运行。关键点在于,故障可能会导致副本持有不同的数据副本。

Raft将客户请求组织成一个序列，称为日志，并确保所有副本服务器看到相同的日志。
每个副本均以日志顺序执行客户端请求，并将其应用于服务状态的本地副本。
由于所有活动副本都看到相同的日志内容，因此它们都以相同的顺序执行相同的请求，因此继续具有相同的服务状态。
如果服务器出现故障但后来又恢复了，Raft会确保其日志为最新状态。只要至少大多数服务器处于活动状态并且可以相互通信，Raft就会继续运行。
如果没有这样的多数，Raft将不会取得进展，但是一旦多数能够再次交流，它将从停下来的地方继续前进。

在本实验中，您将通过关联的方法将Raft实现为Go对象类型，该方法旨在用作更大服务中的模块。
一组Raft实例与RPC相互通信以维护复制的日志。您的Raft界面将支持不定编号的命令序列，也称为日志条目。
条目用索引号编号。具有给定索引的日志条目最终将被提交。届时，您的Raft应该将日志条目发送到较大的服务，以便其执行。

您应该按照扩展的Raft论文中的设计进行操作 ，尤其要注意图2。您将实现论文中的大部分内容，包括保存持久状态并在节点发生故障然后重新启动后读取它。
您将不会实现集群成员资格更改（第6节）。您将在以后的实验中实现日志压缩/快照化（第7节）。

您可能会发现本 指南 很有用，以及有关锁定 和 并发结构的建议 。
从更广泛的角度来看，请查看Paxos，Chubby，Paxos Made Live，Spanner，Zookeeper，Harp，Viewstamped Replication和 Bolosky等。

该实验分为三个部分。您必须在相应的截止日期之前提交每个零件。

### 合作政策
您必须编写您在6.824中提交的所有代码，但我们作为分配的一部分提供给您的代码除外。
不允许您查看其他任何人的解决方案，不允许您查看前几年的代码，也不允许您查看其他Raft实现。
您可以与其他学生讨论作业，但不能查看或复制他人的代码，也不得让他人查看您的代码。
请不要发布您的代码或将其提供给现在或将来的6.824学生。 github.com仓库默认是公开的，因此除非您将仓库设为私有，否则请不要在其中放置代码。
您可能会发现使用MIT的GitHub方便 ，但是请确保创建一个私有存储库。

### 入门
如果您已经完成了实验1，那么您已经有了该实验源代码的副本。
如果没有，您可以在Lab 1说明中找到通过git获取源代码的指导。

我们为您提供的框架代码的src /筏/ raft.go。
我们还提供了一组测试，您应使用这些测试来推动实现工作，并使用它们来对提交的实验进行评分。
测试位于src / raft / test_test.go中。

要启动并运行，请执行以下命令。不要忘记使用git pull获得最新软件。
```bash
$ cd ~/6.824
$ git pull
...
$ cd src/raft
$ GOPATH=~/6.824
$ export GOPATH
$ go test
Test (2A): initial election ...
--- FAIL: TestInitialElection2A (5.04s)
        config.go:326: expected one leader, got none
Test (2A): election after network failure ...
--- FAIL: TestReElection2A (5.03s)
        config.go:326: expected one leader, got none
...
$
```

### 代码
通过将代码添加到raft / raft.go来实现Raft 。在该文件中，您将找到框架代码，以及有关如何发送和接收RPC的示例。
您的实现必须支持以下接口，测试人员和（最终）您的键/值服务器将使用该接口。您可以在raft.go的评论中找到更多详细信息。

```bash
// create a new Raft server instance:
rf := Make(peers, me, persister, applyCh)

// start agreement on a new log entry:
rf.Start(command interface{}) (index, term, isleader)

// ask a Raft for its current term, and whether it thinks it is leader
rf.GetState() (term, isLeader)

// each time a new entry is committed to the log, each Raft peer
// should send an ApplyMsg to the service (or tester).
type ApplyMsg
```
一个服务调用Make（peers，me，...）创建一个Raft对等体。peers参数是用于RPC的Raft对等方（包括该对等方）的网络标识符的数组。
在 我的观点是这样的同行同行数组中的索引。Start（命令）要求Raft开始处理，以将命令附加到复制的日志中。Start（） 应该立即返回，
而不必等待日志追加完成。该服务需要您的实现发送 ApplyMsg每个新提交的日志条目添加到 applyCh通道参数做（） 。

raft.go包含示例代码，该示例代码发送RPC（sendRequestVote（））并处理传入的RPC（RequestVote（））。
您的Raft对等方应该使用labrpc Go软件包（src / labrpc中的源）交换RPC 。
测试人员可以告诉labrpc延迟RPC，对其重新排序，然后丢弃它们以模拟各种网络故障。
尽管您可以临时修改labrpc，但是请确保您的Raft与原始的labrpc兼容，因为这就是我们将用来测试和分级实验室的功能。
您的Raft实例必须仅与RPC交互；例如，不允许他们使用共享的Go变量或文件进行通信。

随后的实验将在此实验的基础上进行，因此给自己足够的时间来编写可靠的代码很重要。

### 第2A部
>tasks

实现raft领导者选举和心跳（无日志条目的AppendEntries RPC）。第2A部分的目标是选举一位领导者，如果没有失败，
则由该领导者继续担任领导者；如果老领导者失败，或者如果去往/来自老领导者的数据包被接收，则由新领导者接任丢失。
运行go test -run 2A来测试您的2A代码。
#### 提示
    您无法轻松地直接运行Raft实现；相反，您应该通过测试仪来运行它，即进行test-run 2A。
    遵循本文的图2。此时，您关心的是发送和接收RequestVote RPC，与选举有关的服务器规则以及与领导人选举有关的状态，
    在raft.go中 的Raft结构中添加图2的领导者选举状态。您还需要定义一个结构来保存有关每个日志条目的信息。
    填写RequestVoteArgs和 RequestVoteReply结构。修改 Make（）以创建后台goroutine，该后台goroutine将在有一段时间没有收到其他对等方的请求时通过发出RequestVote RPC来定期启动领导者选举。这样，同伴将了解谁是领导者（如果已经有领导者），或者成为领导者本身。实现RequestVote（） RPC处理程序，以便服务器将彼此投票。
    要实现心跳，请定义一个 AppendEntries RPC结构（尽管您可能还不需要所有参数），并让领导者定期将其发送出去。编写一个 AppendEntries RPC处理程序方法，该方法将重置选举超时，以便在已经选择一台服务器时，其他服务器不再作为领导服务器前进。
    确保不同对等方的选举超时不会总是同时触发，否则所有对等方将只为自己投票，而没有人会成为领导者。
    测试人员要求领导者每秒发送心跳RPC的次数不超过十次。
    测试人员要求您的Raft在旧领导者失败的五秒钟内（如果大多数同龄人仍然可以沟通）选出一位新领导者。但是，请记住，在投票分裂的情况下，领导人选举可能需要进行多轮投票（如果丢包或候选人不幸地选择了相同的随机退避时间，则可能会发生这种情况）。您必须选择足够短的选举超时（因此也要选择心跳间隔），即使需要进行多轮选举，选举也很可能在不到五秒钟的时间内完成。
    论文的5.2节提到选举超时范围为150到300毫秒。仅当领导者发送心跳的频率大大超过每150毫秒一次的频率时，此范围才有意义。由于测试仪将您的心跳限制为每秒10个心跳，因此您将必须使用比纸张的150到300毫秒大的选举超时时间，但不能太大，因为那样的话，您可能会在5秒钟内无法选举领导者。
    您可能会发现Go的 rand 非常有用。
    您需要编写定期或在时间延迟后执行操作的代码。最简单的方法是创建一个带有调用time.Sleep（）的循环的goroutine 。不要使用Go的time.Timer或time.Ticker，这是很难正确使用。
    阅读有关锁定和 结构的建议 。
    如果您的代码无法通过测试，请再次阅读论文的图2；否则，请参见图2。领导者选举的全部逻辑分布在图中的多个部分。
    不要忘记实现GetState（）。
    当测试人员永久关闭实例时，它会调用您的Raft的rf.Kill（）。您可以检查是否 杀（）被调用使用rf.killed（） 。您可能希望在所有循环中都这样做，以避免死掉的Raft实例打印出混乱的消息。
    调试代码的一种好方法是在对等方发送或接收消息时插入打印语句，然后使用go test -run 2A> out将输出收集到文件中 。然后，通过研究out文件中消息的踪迹，可以确定实现偏离所需协议的位置。您可能会发现util.go中的DPrintf 对调试各种问题时打开和关闭打印很有用。
    Go RPC仅发送名称以大写字母开头的结构字段。子结构还必须具有大写的字段名称（例如，数组中的日志记录的字段）。该labgob包会警告你这一点; 不要忽略警告。
    使用go test -race检查您的代码，并修复它报告的所有问题。
    
    
在提交第2A部分之前，请确保您通过了2A测试，以便您看到类似以下内容的内容：
```bash
$ go test -run 2A
Test (2A): initial election ...
  ... Passed --   4.0  3   32    9170    0
Test (2A): election after network failure ...
  ... Passed --   6.1  3   70   13895    0
PASS
ok      raft    10.187s
$
```

每个“通过”行均包含五个数字；这些是测试花费的时间（以秒为单位），Raft对等体的数量（通常为3或5），测试期间发送的RPC的数量，RPC消息中的字节总数以及Raft记录的日志数量报告已提交。
您的电话号码将与此处显示的号码不同。您可以根据需要忽略这些数字，但是它们可以帮助您理智地检查实现发送的RPC的数目。
对于所有实验2、3和4，如果所有测试（go test）花费的时间超过600秒，或者单个测试花费的时间超过120秒，则评分脚本将使您的解决方案失败。


### 第2B部

>tasks

实现领导者和跟随者代码以附加新的日志条目，以便go test -run 2B测试通过。
#### 提示
    运行git pull以获取最新的实验室软件。
    您的首要目标应该是通过TestBasicAgree2B（）。首先实现Start（），然后编写代码以通过AppendEntries RPC 发送和接收新的日志条目，如图2所示。
    您将需要实施选举限制（本文第5.4.1节）。
    在早期的Lab 2B测试中未能达成协议的一种方法是，即使领导者还活着，也要举行重复的选举。在选举计时器管理中查找错误，或者在赢得选举后不立即发出心跳信号。
    您的代码可能具有循环，该循环反复检查某些事件。不要让这些循环不间断地连续执行，因为这会使您的实现速度降低到足以使测试失败的程度。使用Go的 条件变量，或在每次循环迭代中插入 time.Sleep（10 * time.Millisecond）。
    请帮自己将来的实验室做些好事，并编写（或重写）干净清晰的代码。对于想法，您可以重新访问我们的 结构， 锁定和 指南 页面。

    
    
如果代码运行太慢，即将进行的实验室测试可能会使您的代码失败。您可以使用time命令检查您的解决方案使用了多少实时时间和CPU时间。这是典型的输出：
```bash
$ time go test -run 2B
Test (2B): basic agreement ...
  ... Passed --   1.6  3   18    5158    3
Test (2B): RPC byte count ...
  ... Passed --   3.3  3   50  115122   11
Test (2B): agreement despite follower disconnection ...
  ... Passed --   6.3  3   64   17489    7
Test (2B): no agreement if too many followers disconnect ...
  ... Passed --   4.9  5  116   27838    3
Test (2B): concurrent Start()s ...
  ... Passed --   2.1  3   16    4648    6
Test (2B): rejoin of partitioned leader ...
  ... Passed --   8.1  3  111   26996    4
Test (2B): leader backs up quickly over incorrect follower logs ...
  ... Passed --  28.6  5 1342  953354  102
Test (2B): RPC counts aren't too high ...
  ... Passed --   3.4  3   30    9050   12
PASS
ok      raft    58.142s

real    0m58.475s
user    0m2.477s
sys     0m1.406s
$
```

“确定筏58.142s”表示Go测得的2B测试时间为实际（挂钟）时间58.142秒。“用户0m2.477s”意味着代码消耗了2.477秒的CPU时间，即实际执行指令所花费的时间（而不是等待或休眠）。
如果您的解决方案在2B测试中使用的时间超过一分钟，或者CPU时间超过5秒，那么您以后可能会遇到麻烦。查找睡眠或等待RPC超时所花费的时间，不睡眠或等待条件或通道消息而运行的循环或发送的大量RPC。


### 第2B部

>tasks

实现领导者和跟随者代码以附加新的日志条目，以便go test -run 2B测试通过。
#### 提示
    运行git pull以获取最新的实验室软件。
    您的首要目标应该是通过TestBasicAgree2B（）。首先实现Start（），然后编写代码以通过AppendEntries RPC 发送和接收新的日志条目，如图2所示。
    您将需要实施选举限制（本文第5.4.1节）。
    在早期的Lab 2B测试中未能达成协议的一种方法是，即使领导者还活着，也要举行重复的选举。在选举计时器管理中查找错误，或者在赢得选举后不立即发出心跳信号。
    您的代码可能具有循环，该循环反复检查某些事件。不要让这些循环不间断地连续执行，因为这会使您的实现速度降低到足以使测试失败的程度。使用Go的 条件变量，或在每次循环迭代中插入 time.Sleep（10 * time.Millisecond）。
    请帮自己将来的实验室做些好事，并编写（或重写）干净清晰的代码。对于想法，您可以重新访问我们的 结构， 锁定和 指南 页面。

    
    
如果代码运行太慢，即将进行的实验室测试可能会使您的代码失败。您可以使用time命令检查您的解决方案使用了多少实时时间和CPU时间。这是典型的输出：
```bash
$ time go test -run 2B
Test (2B): basic agreement ...
  ... Passed --   1.6  3   18    5158    3
Test (2B): RPC byte count ...
  ... Passed --   3.3  3   50  115122   11
Test (2B): agreement despite follower disconnection ...
  ... Passed --   6.3  3   64   17489    7
Test (2B): no agreement if too many followers disconnect ...
  ... Passed --   4.9  5  116   27838    3
Test (2B): concurrent Start()s ...
  ... Passed --   2.1  3   16    4648    6
Test (2B): rejoin of partitioned leader ...
  ... Passed --   8.1  3  111   26996    4
Test (2B): leader backs up quickly over incorrect follower logs ...
  ... Passed --  28.6  5 1342  953354  102
Test (2B): RPC counts aren't too high ...
  ... Passed --   3.4  3   30    9050   12
PASS
ok      raft    58.142s

real    0m58.475s
user    0m2.477s
sys     0m1.406s
$
```

“确定筏58.142s”表示Go测得的2B测试时间为实际（挂钟）时间58.142秒。“用户0m2.477s”意味着代码消耗了2.477秒的CPU时间，即实际执行指令所花费的时间（而不是等待或休眠）。
如果您的解决方案在2B测试中使用的时间超过一分钟，或者CPU时间超过5秒，那么您以后可能会遇到麻烦。查找睡眠或等待RPC超时所花费的时间，不睡眠或等待条件或通道消息而运行的循环或发送的大量RPC。


### 第2C部分
如果基于Raft的服务器重新启动，则应从中断的位置恢复服务。这就要求Raft保持持久状态，使其在重启后仍然有效。本文的图2提到了哪个状态应该保持不变。

实际的实现会在每次更改后将Raft的持久状态写入磁盘，并在重新引导后重新启动时从磁盘读取状态。您的实现将不使用磁盘。相反，它将从Persister对象保存和恢复持久状态（请参阅persister.go）。
调用Raft.Make（）的人都会提供一个Persister ，该持久性最初保持Raft的最新持久状态（如果有）。Raft应该从该Persister初始化其状态 ，并且每次状态更改时都应使用它保存其持久状态。
使用Persister的 ReadRaftState（）和SaveRaftState（）方法
>tasks

完成功能 坚持（） 和 readPersist（）在raft.go 加入代码保存和恢复持久状态。您需要将状态编码（或“序列化”）为字节数组，以将其传递给Persister。使用labgob编码器；请参阅persist（）和readPersist（）中的注释。 labgob类似于Go的gob编码器，但是如果您尝试使用小写的字段名编码结构，则会输出错误消息。

在您的实现更改持久状态的点上 插入对persist（）的调用。完成此操作后，您应该通过其余的测试
> note：
为了避免内存不足，Raft必须定期丢弃旧的日志条目，但是在下一个实验之前，您不必为此担心。
#### 提示
  运行git pull以获取最新的实验室软件。
  许多2C测试涉及服务器故障以及网络丢失RPC请求或答复的情况。
  您可能需要进行一次最多备份一个索引的nextIndex优化。查看从第7页底部和第8页顶部开始的扩展的Raft纸（用灰色线标记）。该论文的细节含糊不清；您可能需要在6.824 Raft讲座的帮助下填补空白。
  全套Lab 2测试（2A + 2B + 2C）消耗的合理时间是4分钟的实时时间和1分钟的CPU时间。
    
    
您的代码应通过所有2C测试（如下所示）以及2A和2B测试。

```bash
$ go test -run 2C
Test (2C): basic persistence ...
  ... Passed --   7.2  3  206   42208    6
Test (2C): more persistence ...
  ... Passed --  23.2  5 1194  198270   16
Test (2C): partitioned leader and one follower crash, leader restarts ...
  ... Passed --   3.2  3   46   10638    4
Test (2C): Figure 8 ...
  ... Passed --  35.1  5 9395 1939183   25
Test (2C): unreliable agreement ...
  ... Passed --   4.2  5  244   85259  246
Test (2C): Figure 8 (unreliable) ...
  ... Passed --  36.3  5 1948 4175577  216
Test (2C): churn ...
  ... Passed --  16.6  5 4402 2220926 1766
Test (2C): unreliable churn ...
  ... Passed --  16.5  5  781  539084  221
PASS
ok      raft    142.357s
$ 
```

“确定筏58.142s”表示Go测得的2B测试时间为实际（挂钟）时间58.142秒。“用户0m2.477s”意味着代码消耗了2.477秒的CPU时间，即实际执行指令所花费的时间（而不是等待或休眠）。
如果您的解决方案在2B测试中使用的时间超过一分钟，或者CPU时间超过5秒，那么您以后可能会遇到麻烦。查找睡眠或等待RPC超时所花费的时间，不睡眠或等待条件或通道消息而运行的循环或发送的大量RPC。
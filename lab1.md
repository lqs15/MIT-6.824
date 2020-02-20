## lab1 - MapReduce
### 介绍
在本实验中，您将构建一个`MapReduce`系统。您将实现一个`worker`进程，该`worker`进程调用应用程序的`Map`和`Reduce`函数并处理读写文件，
以及一个`master`进程，该`master`进程将任务分发给工作人员并应对失败的工作人员。您将构建类似于`MapReduce`论文的内容。
### 合作政策
您必须编写6.824上交的所有代码,但我们作为分配的一部分提供给您的代码除外。
不允许您查看其他任何人的解决方案,也不允许您查看前几年的解决方案。您可以与其他学生讨论作业，但不能查看或复制彼此的代码.
制定此规则的原因是,我们相信您将自己设计和实施实验室解决方案,从而从中学到最多的知识.
    
请不要发布您的代码或将其提供给现在或将来的6.824学生.github.com仓库默认是公开的,因此除非您将仓库设为私有,否则请不要在其中放置代码。
您可能会发现使用MIT的GitHub方便,但是请确保创建一个私有存储库。
### 引入字数统计
按照步骤执行
```bash
# 生成动态库
$ cd src/mrapps
$  go build -buildmode=plugin -o wc.so wc.go

# 运行
$ cd src/main
$ rm mr-out *
$ go run mrsequential.go ../mrapps/wc.so pg*.txt
$ more mr-out #查看
A 509
ABOUT 2
ACT 8
...
```
我们可以从`mrapps/wc.go`和`mrsequential.go`代码,理解`mapreduce`的运行原理

### 实验
实现一个分布式`MapReduce`,它包含两种程序实现(`master`和`worker`),运行时只有一个`master`,有一个或者多个`worker`并行执行.在生产环境中,
`worker`将在许多不同的机器上运行,但是对于本实验，将全部在单个机器上运行`worker`.`worker`将通过RPC与`master`通信。每个`worker`进程都会向`master`请求一个任务，
`worker`从一个或多个文件中读取内容的输入,执行程序,并将任务的输出写入一个或多个文件。`master`应注意一个`worker`是否在合理的时间内没有完成任务（在本实验中，使用十秒钟）,超时将同一任务交给另一个`worker`。

### 入门
我们给了您一些代码代码实现,让您开始。`master`和`worker`程序的“主要”例程位于`main/mrmaster.go`和`main/mrworker.go`中；
不要更改这些文件。您应该将实现放在`mr/master.go`,` mr/worker.go`和`mr/rpc.go`中。

### 运行
首先,请确保单词计数插件是全新构建的,单词计数MapReduce应用程序上运行代码的方法。
```bash
$ go build -buildmode=plugin -o wc.so wc.go
```

在`main`目录中,运行`master`.
```bash
$ rm mr-out*
$ go run mrmaster.go pg-*.txt
```
`mrmaster.go` 的`pg-*.tx`t参数是输入文件;每个文件对应一个“拆分”,是一个Map任务的输入.

在一个或多个其他`terminal`，运行一些`worker`进程：
```bash
$ rm mr-out*
$ go run mrmaster.go pg-*.txt
```
当`worker`和`master`运行完成后,请查看输出到`mr-out- *`中的内容.完成实验后,输出文件的排序联合应与顺序输出匹配,如下所示：
```bash
$ cat mr-out-* | sort | more
A 509
ABOUT 2
ACT 8
...
```
我们提供了一个测试的脚本`main/test-mr.sh`,测试在给定`pg-xxx.txt`文件作为输入时,检查`wc`和`indexer MapReduce`应用程序是否产生正确的输出.
这些测试还检查您的实现是否并行运行`Map`和`Reduce`任务，以及您是否有实现恢复`worker`在执行运行任务时崩溃.

如果您现在运行测试脚本，则它将挂起，因为主脚本永远不会完成：
```bash
$ cd ~/6.824/src/main
$ sh test-mr.sh
*** Starting wc test.
```
您可以在`mr/master.go`的`Done`函数将`ret：= false`更改为`true`,`master`将立即执行退出.输出如下：
```bash
$ sh ./test-mr.sh
*** Starting wc test.
sort: No such file or directory
cmp: EOF on mr-wc-all
--- wc output is not the same as mr-correct-wc.txt
--- wc test: FAIL
$ 
```
每个`reduce`执行,测试脚本希望在名为`mr-out-X`的文件中看到程序的输出.在`mr/master.go` 和`mr/worker.go`的空实现(或执行其他任何操作)不会生成这些文件,
因此测试失败.当你完成后,测试脚本输出应如下所示：
```bash
$ sh ./test-mr.sh
*** Starting wc test.
--- wc test: PASS
*** Starting indexer test.
--- indexer test: PASS
*** Starting map parallelism test.
--- map parallelism test: PASS
*** Starting reduce parallelism test.
--- reduce parallelism test: PASS
*** Starting crash test.
--- crash test: PASS
*** PASSED ALL TESTS
$
```
您还将从Go RPC软件包中看到一些类似于如下,可忽略的消息
```bash
2019/12/16 13:27:09 rpc.Register: method "Done" has 1 input parameters; needs exactly three
```
### 一些规则：
 - `map`阶段应将中间键(`intermediate keys`)划分为桶存储于`nReduce`用于`reduce`任务,其中`main/mrmaster.go`将`nReduce`传递给`MakeMaster()`作为参数。
 - `worker`的实现应将第`X`个`reduce`任务的输出放入文件`mr-out-X`中。
 - 每个`Reduce`函数使用键和值进行调用，以`Go “％v％v” `格式生成一行输出,输出内容存于一个名为`mr-out-X`文件中.在`main/mrsequential.go`中查看注释为`“this is the correct format”`的行。
   如果您的实现偏离此格式太多，则测试脚本将失败。
 - 您可以修改`mr/worker.go`,`mr/master.go`,和`mr/rpc.go`。您可以临时修改其他文件以进行测试，但是请确保您的代码可以与原始版本一起使用；我们将使用原始版本进行测试。
 - 生成中间`map`的`worker`应将生成的中间`Map`输出放置在当前目录中的文件中，生成`reduce`的`worker`以后可以在其中读取它们,作为`Reduce`任务的输入。
 - 当`worker`的`job`完全完成后，`worker`进程应退出；一种可能是`master`给他们一个“请退出”的伪任务。`worker`在`tasks`完全完成之前不应退出。
 - `main/mrmaster.go`期望`mr/master.go`实现 `Done()`方法,该方法在`MapReduce`任务完全完成时返回`true`;此时,`mrmaster.go`将退出。
> 提示
  - 一种入门方法是修改`mr/worker.go`，以便 `Worker()`反复向主服务器发送`RPC`询问任务。主机应回答`Map`任务（输入文件名和Map任务号）或`Reduce`任务（`reduce`任务号）的描述。
    对于`Map`任务，工作人员应阅读输入文件，并将其交给应用程序`Map`函数，如`mrsequential.go`中所示。`worker`应在每个`reduce`任务中将`Map by key`返回的数组拆分为一个中间文件。对于`Reduce`任务，
    `worker`应读取该任务的所有中间文件，然后（如`mrsequential.go`中所示），按键排序，将每个键的值交给`Reduce`函数，然后将所有输出放入`mr-out-X`文件。
  - 使用`Go plugin`在运行时从名称以`.so`结尾的文件中加载应用程序`Map`和`Reduce`函数。
  - 如果您在`mr/`目录中进行了任何更改，则可能必须重新构建您使用的所有`MapReduce`插件，例如`go build -buildmode = plugin ../mrapps/wc.go`
  - 该实验依靠`worker`共享文件系统。当所有`worker`程序都在同一台计算机上运行时，这很简单，但是如果工作程序在不同的计算机上运行，​​则需要像`GFS`这样的全局文件系统。
  - 中间文件的合理命名约定是`mr-XY`，其中`X`是`Map`任务号，`Y`是`reduce`任务号。
  - `worker`的`map`任务代码将需要一种方法，以在`reduce`任务期间可以正确读取的方式在文件中存储中间键/值对。一种可能性是使用Go的`encoding/json`包。要将键/值对写入`JSON`文件：
  ```bash
  enc := json.NewEncoder(file)
  for _, kv := ... {
    err := enc.Encode(&kv)
  ```
   并读回这样的文件：
  ```bash
  dec := json.NewDecoder(file)
  for {
    var kv KeyValue
    if err := dec.Decode(&kv); err != nil {
      break
    }
    kva = append(kva, kv)
  }
  ```
  - 您的`worker`的`map`部分可以使用`ihash（key）`函数（在`worker.go`中）为给定的`key`选择`reduce`任务。
  - 您可以从`mrsequential.go`窃取一些代码，以读取`Map`输入文件，对`Map`和`Reduce`之间的中间键/值对进行排序，以及将`Reduce`输出存储在文件中。
  - `master`（作为`RPC`服务器）将是并发的；不要忘记锁定共享数据。
  - 使用`Go`的竞赛检测器，以及`go build -race和go run -race`。 `test-mr.sh`上有一条注释，向您展示了如何为测试启用种族检测器。
  - `Go`在其自己的线程中运行每个`RPC`处理程序，因此处理程序可以在返回之前等待事情发生。您可以使用`Go`的条件变量 `sync.Cond`。
  - `master`无法可靠地区分崩溃的`worker`人，活着的但由于某种原因停工的`worker`和执行但速度太慢而无法使用的`worker`。您能做的最好的事情就是让`master`服务器等待一段时间，然后放弃并将任务重新发布给其他`worker`。
    在本实验中，让`master`等待十秒钟；之后，`master`应假定`worker`已经死亡（当然，可能没有死亡）。
  - 要测试崩溃恢复，可以使用`mrapps/crash.go `应用程序插件。它在`Map`和`Reduce`函数中随机退出。
  - 为确保在崩溃时不会有人看到部分写入的文件，`MapReduce`论文提到了使用临时文件并在完全写入后自动重命名它的技巧。您可以使用 `ioutil.TempFile`创建一个临时文件，并使用`os.Rename` 原子地对其进行重命名。
  - `test-mr.sh`运行子目录`mr-tmp`中的所有进程 ，因此如果出现问题，并且您想查看中间文件或输出文件，请在此处查看。
  
如果您在入门时遇到麻烦，可以采用以下一种方法将实验室分为三个步骤：
  - 一是实施任务调度。`worker`应向`master`长要求任务；现在，`worker`在接到任务后可以立即告诉`master`他们已经完成了任务。`master`应首先分配所有地图任务，同时跟踪已分配的任务和已完成的任务。在完成所有地图任务后，
   `master`应分配缩小任务，在完成所有`reduce`任务后，`worker`应退出。当没有可用任务时，`worker`可以请求任务-`master`服务器应等待直到任务可用或任务完成后再答复（使用`sync.Cond`）。通过在`worker`进程中打印任务参数
    (包括映射任务输入文件名并减少任务编号)来验证您的实施是否正常。
  - 其次，实现映射并减少工作进程中的任务执行（请参见上面的第一个提示）。
  - 最后，要解决崩溃问题，请在十秒钟后将尚未完成的任务重新安排给其他`worker``。您可能会发现与同一个`sync`交互会很有帮助。
  
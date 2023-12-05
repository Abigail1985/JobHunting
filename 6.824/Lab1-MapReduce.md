当然，我将按照Markdown格式将提供的文本翻译成中文，并标记标题、重点、代码及代码段。

# 6.5840 - 2023年春季

## 6.5840 实验室1：MapReduce

**截止日期：** 2月17日星期五 23:59ET（麻省理工时间）

**合作政策 // 提交实验 // 设置 Go // 指导 // Piazza**

### 简介

在这个实验中，你将构建一个 MapReduce 系统。你将实现一个工作进程，该进程调用应用程序的 Map 和 Reduce 函数，并处理文件的读写，以及一个协调器进程，该进程分发任务给工作进程并处理失败的工作进程。你将构建类似于 MapReduce 论文中描述的系统。（注意：本实验使用“协调器”而不是论文中的“主节点”。）

### 开始

你需要设置 Go 来完成实验。

使用 git（一个版本控制系统）获取初始实验软件。要了解更多关于 git 的信息，请查看 Pro Git 书籍或 git 用户手册。

```bash
$ git clone git://g.csail.mit.edu/6.5840-golabs-2023 6.5840
$ cd 6.5840
$ ls
Makefile src
```

我们为你提供了一个简单的顺序 MapReduce 实现，在 `src/main/mrsequential.go` 中。它一次一个地在单个进程中运行 maps 和 reduces。我们还为你提供了几个 MapReduce 应用程序：`mrapps/wc.go` 中的词频统计和 `mrapps/indexer.go` 中的文本索引器。你可以按以下方式顺序运行词频统计：

```bash
$ cd ~/6.5840
$ cd src/main
$ go build -buildmode=plugin ../mrapps/wc.go
$ rm mr-out*
$ go run mrsequential.go wc.so pg*.txt
$ more mr-out-0
A 509
ABOUT 2
ACT 8
...
```

`mrsequential.go` 将其输出留在 `mr-out-0` 文件中。输入来自命名为 `pg-xxx.txt` 的文本文件。

随意借用 `mrsequential.go` 中的代码。你还应该看看 `mrapps/wc.go`，了解 MapReduce 应用程序代码的样子。

对于本实验和所有其他实验，我们可能会发布我们提供的代码的更新。为了确保你可以使用 `git pull` 轻松获取这些更新并合并它们，最好将我们提供的代码保留在原始文件中。你可以根据实验指南中的指示添加到我们提供的代码中；只是不要移动它。在新文件中放置你自己的新函数是可以的。

### 你的任务（中等/困难）

你的任务是实现一个分布式 MapReduce，由两个程序组成，协调器和工作进程。将只有一个协调器进程，以及一个或多个并行执行的工作进程。在真实系统中，工作进程将在许多不同的机器上运行，但对于本实验，你将在单台机器上运行它们所有。
工作进程将通过 RPC 与协调器通信。
每个工作进程将向协调器请求任务，从一个或多个文件中读取任务的输入，执行任务，并将任务的输出写入一个或多个文件。
如果工作进程在合理的时间内（对于本实验，使用十秒）没有完成其任务，协调器应该将同样的任务交给另一个工作进程。

我们给你一些代码来开始。协调器和工作进程的 "main" 例程分别在 `main/mrcoordinator.go` 和 `main/mrworker.go` 中；不要更改这些文件。你应该将你的实现放在 `mr/coordinator.go`，`mr/worker.go`，和 `mr/rpc.go` 中。

以下是如何在词频统计 MapReduce 应用程序上运行你的代码。首先，确保词频统计插件是最新构建的：

```bash
$ go build -buildmode=plugin ../mrapps/wc.go
```

在主目录中，运行协调器。

```bash
$ rm mr-out*
$ go run mrcoordinator.go pg-*.txt
```

`pg-*.txt` 参数是 `mrcoordinator.go` 的输入文件；每个文件对应一个 "分片"，并且是一个 Map 任务的输入。

在一个或多个其他窗口中，运行一些工作进程：

```bash
$ go run mrworker.go wc.so
```

当工作进程和协调器完成后，查看 `mr-out-*` 中的输出。当你完成实验后，输出文件的排序联合应该与顺序输出匹配，如下所示：

```bash
$ cat mr-out-* | sort | more
A 509
ABOUT 2
ACT 8
...
```

我们为你提供了一个测试脚本，在 `main/test-mr.sh` 中。测试检查当给定 `pg-xxx.txt` 文件作为输入时，wc 和 indexer MapReduce 应用程序是否产生正确的输出。测试还检查你的实现是否并行运行 Map 和 Reduce 任务，并且你的实现是否从运行任务时崩溃的工作进程中恢复过来。


如果你现在运行测试脚本，它将会挂起，因为协调器永远不会完成：

```bash
$ cd ~/6.5840/src/main
$ bash test-mr.sh
*** Starting wc test.
```

你可以在 `mr/coordinator.go` 中的 `Done` 函数里将 `ret := false` 改为 `true`，这样协调器将会立即退出。然后：

```bash
$ bash test-mr.sh
*** Starting wc test.
sort: No such file or directory
cmp: EOF on mr-wc-all
--- wc output is not the same as mr-correct-wc.txt
--- wc test: FAIL
$
```

测试脚本期望在名为 `mr-out-X` 的文件中看到输出，每个 reduce 任务一个文件。`mr/coordinator.go` 和 `mr/worker.go` 的空实现不会产生这些文件（或做任何其他事情），所以测试失败。

当你完成时，测试脚本的输出应该是这样的：

```bash
$ bash test-mr.sh
*** Starting wc test.
--- wc test: PASS
*** Starting indexer test.
--- indexer test: PASS
*** Starting map parallelism test.
--- map parallelism test: PASS
*** Starting reduce parallelism test.
--- reduce parallelism test: PASS
*** Starting job count test.
--- job count test: PASS
*** Starting early exit test.
--- early exit test: PASS
*** Starting crash test.
--- crash test: PASS
*** PASSED ALL TESTS
$
```

你可能会看到来自 Go RPC 包的一些错误，看起来像这样：

```
2019/12/16 13:27:09 rpc.Register: 方法 "Done" 有 1 个输入参数；需要恰好三个
```

忽略这些消息；将协调器注册为 RPC 服务器会检查它的所有方法是否适合 RPC（有 3 个输入）；我们知道 Done 不是通过 RPC 调用的。


### 一些规则：

- **Map 阶段** 应该将中间键分为 `nReduce` 个 reduce 任务的桶，其中 `nReduce` 是 reduce 任务的数量——这是 `main/mrcoordinator.go` 传递给 `MakeCoordinator()` 的参数。每个映射器应该为 reduce 任务创建 `nReduce` 个中间文件。

- **工作进程的实现** 应该将第 `X` 个 reduce 任务的输出放入文件 `mr-out-X` 中。

- 一个 `mr-out-X` 文件应该包含每个 `Reduce` 函数输出的一行。这行应该使用 Go 的 `"%v %v"` 格式生成，调用时带上键和值。可以在 `main/mrsequential.go` 中查看注释为“这是正确的格式”的行。如果你的实现与这种格式相差太大，测试脚本将会失败。

- 你可以修改 `mr/worker.go`、`mr/coordinator.go` 和 `mr/rpc.go`。你可以临时修改其他文件进行测试，但请确保你的代码与原始版本兼容；我们将使用原始版本进行测试。

- 工作进程应该将中间 Map 输出放在当前目录的文件中，这样你的工作进程稍后可以将它们作为输入读取给 `Reduce` 任务。

- `main/mrcoordinator.go` 期望 `mr/coordinator.go` 实现一个 `Done()` 方法，当 `MapReduce` 作业完全完成时返回 `true`；此时，`mrcoordinator.go` 将退出。

- 当作业完全完成时，工作进程应该退出。实现这一点的一种简单方法是使用 `call()` 的返回值：如果工作进程无法联系到协调器，它可以假设协调器已经退出，因为作业已经完成，所以工作进程也可以终止。根据你的设计，你可能也会发现有一个“请退出”的伪任务，协调器可以给工作进程，这会有所帮助。


### 提示

- **指导页面** 上有一些关于开发和调试的提示。

- 开始的一种方法是修改 `mr/worker.go` 的 `Worker()`，让它发送一个 RPC 到协调器请求一个任务。然后修改协调器以响应尚未开始的 map 任务的文件名。接着修改工作进程来读取该文件并调用应用程序的 Map 函数，如 `mrsequential.go` 中所示。

- 应用程序的 Map 和 Reduce 函数在运行时使用 Go 插件包加载，从文件名以 .so 结尾的文件中加载。

- 如果你在 `mr/` 目录中更改了任何内容，你可能需要重新构建你使用的任何 MapReduce 插件，比如使用 `go build -buildmode=plugin ../mrapps/wc.go`

- 这个实验依赖于工作进程共享文件系统。当所有工作进程在同一台机器上运行时这很简单，但如果工作进程在不同的机器上运行，则需要一个全局文件系统，如 GFS。

- 中间文件的合理命名规范是 `mr-X-Y`，其中 `X` 是 Map 任务编号，`Y` 是 reduce 任务编号。

- 工作进程的 map 任务代码需要一种方法将中间键/值对存储在文件中，以便在 reduce 任务期间可以正确地读回。一种可能性是使用 Go 的 `encoding/json` 包。将键/值对以 JSON 格式写入打开的文件：

```go
enc := json.NewEncoder(file)
for _, kv := ... {
  err := enc.Encode(&kv)
}
```

读回这样的文件：

```go
dec := json.NewDecoder(file)
for {
  var kv KeyValue
  if err := dec.Decode(&kv); err != nil {
    break
  }
  kva = append(kva, kv)
}
```


- 你的工作进程的 map 部分可以使用 `ihash(key)` 函数（在 `worker.go` 中）来为给定的键选择 reduce 任务。

- 你可以从 `mrsequential.go` 偷一些代码来读取 Map 输入文件，对 Map 和 Reduce 之间的中间键/值对进行排序，以及将 Reduce 输出存储在文件中。
- 作为 RPC 服务器的协调器将是并发的；不要忘记锁定共享数据。
- 使用 Go 的竞争检测器，带 `go run -race`。`test-mr.sh` 在开头有一个注释，告诉你如何用 `-race` 运行它。当我们给你的实验打分时，我们不会使用竞争检测器。然而，如果你的代码有竞争，即使没有使用竞争检测器，我们测试时也很可能会失败。
- 工作进程有时需要等待，例如在最后一个 map 完成之前 reduce 不能开始。一种可能性是让工作进程定期向协调器请求工作，在每次请求之间用 `time.Sleep()` 休眠。另一种可能性是让协调器中相关的 RPC 处理程序有一个等待循环，用 `time.Sleep()` 或 `sync.Cond`。Go 为每个 RPC 在其自己的线程中运行处理程序，因此一个处理程序的等待不需要阻止协调器处理其他 RPC。
- 协调器无法可靠地区分崩溃的工作进程、因某种原因停滞的存活工作进程，以及执行速度太慢而无用的工作进程。你能做的最好的是让协调器等待一段时间，然后放弃并将任务重新分配给另一个工作进程。对于这个实验，让协调器等待十秒钟；之后，协调器应该假设工作进程已经死亡（当然，它可能没有）。
- 如果你选择实现备份任务（第3.6节），请注意，我们测试你的代码在工作进程执行任务时没有崩溃时不安排额外的任务。备份任务应该只在相对较长的时间后（例如，10秒）安排。
- 要测试崩溃恢复，你可以使用 `mrapps/crash.go` 应用程序插件。它在 Map 和 Reduce 函数中随机退出。
- 为了确保在崩溃时没有人观察到部分写入的文件，MapReduce 论文提到了使用临时文件并在完全写入后原子地重命名的技巧。你可以使用 `ioutil.TempFile` 创建临时文件，使用 `os.Rename` 原子地重命名它。
- `test-mr.sh` 在 `mr-tmp` 子目录中运行所有进程，因此如果出了问题并且你想查看中间或输出文件，请在那里查看。随意临时修改 `test-mr.sh` 以在失败的测试后退出，这样脚本不会继续测试（并覆盖输出文件）。
- `test-mr-many.sh` 连续多次运行 `test-mr.sh`，你可能希望这样做以发现低概率的错误。它接受运行测试次数作为参数。你不应该并行运行几个 `test-mr.sh` 实例，因为协调器将重用相同的套接字，导致冲突。
- Go RPC 只发送其名称以大写字母开头的结构字段。子结构也必须具有大写的字段名称。
- 调用 RPC 的 `call()` 函数时，回复结构应该包含所有默认值。RPC 调用应该像这样：

```go
reply := SomeType{}


call(..., &reply)
```

在调用前不设置回复的任何字段。如果你传递的回复结构包含非默认字段，RPC 系统可能会默默返回错误的值。
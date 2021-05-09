## 测试Go程序性能

*Russ Cox, July 2011; updated by Shenghou Ma, May 2013*
*24 June 2011*

​		在2011年Scala Days上，Robert Hundt发表了一篇题为[《C ++ / Java / Go / Scala中的循环识别》](http://research.google.com/pubs/pub37122.html)的论文。本文实现了一种特定的循环查找算法，例如您可以在C ++，Go，Java，Scala的编译器的流分析过程中使用，然后使用这些程序得出关于这些语言中典型性能问题的结论。该论文中介绍的Go程序运行速度非常慢，这是一个极好的机会，可以演示如何使用Go的性能分析工具来执行缓慢的程序并使之更快。

​		通过使用Go的性能分析工具来识别和纠正特定的瓶颈，我们可以使Go循环查找程序的运行速度提高一个数量级，并减少6倍的内存。 （更新：由于`gcc`的`libstdc++`最新优化，现在的内存减少为3.7倍。）

​		Hundt的论文没有指定他使用的C ++，Go，Java和Scala工具的版本。在此博客文章中，我们将使用`6g` Go编译器的最新每周快照以及`g++`Ubuntu Natty发行版附带的版本。（我们不会使用Java或Scala，因为我们不熟练使用这两种语言编写高效的程序，因此进行比较是不公平的。由于C ++是本文中最快的语言，因此此处与C ++的比较就足够了。 ）（更新：在此更新的帖子中，我们将使用amd64上Go编译器的最新开发快照以及`g++`2013年3月发布的-4.8.0的最新版本。）

```bash
$ go version
go version devel +08d20469cc20 Tue Mar 26 08:27:18 2013 +0100 linux/amd64
$ g++ --version
g++ (GCC) 4.8.0
Copyright (C) 2013 Free Software Foundation, Inc.
...
```

​		这些程序在具有3.4GHz Core i7-2600 CPU，16 GB RAM和3.8.4-gentoo的Gentoo Linux内核的计算机上运行。机器正在通过以下方式禁止CPU频率缩放运行。

```bash
$ sudo bash
# for i in /sys/devices/system/cpu/cpu[0-7]
do
    echo performance > $i/cpufreq/scaling_governor
done
```

​		我们采用 了C ++和Go中的[Hundt基准测试程序](https://github.com/hundt98847/multi-language-bench)，将每个[程序](https://github.com/hundt98847/multi-language-bench)合并到一个源文件中，并删除了仅一行输出。我们将使用Linux `time`实用程序对程序进行计时，其格式应显示用户时间，系统时间，实际时间和最大内存使用量：

```bash
$ cat xtime
#!/bin/sh
/usr/bin/time -f '%Uu %Ss %er %MkB %C' "$@"
$

$ make havlak1cc
g++ -O3 -o havlak1cc havlak1.cc
$ ./xtime ./havlak1cc
# of loops: 76002 (total 3800100)
loop-0, nest: 0, depth: 0
17.70u 0.05s 17.80r 715472kB ./havlak1cc
$

$ make havlak1
go build havlak1.go
$ ./xtime ./havlak1
# of loops: 76000 (including 1 artificial root node)
25.05u 0.11s 25.20r 1334032kB ./havlak1
```

​		C ++程序运行了17.80秒，并使用700 MB的内存。Go程序运行了25.20秒，并使用1302 MB的内存。（这些测量结果很难与论文中的结论吻合，但是本文的重点是探讨如何使用`go tool pprof`，而不是复制论文中结论。）

​		要开始调试Go程序，我们必须启用性能分析。如果代码使用[Go测试包](https://golang.org/pkg/testing/)的基准测试支持，则可以使用gotest的标准`-cpuprofile`和`-memprofile` 标志。在这样的独立程序中，我们必须导入`runtime/pprof`并添加几行代码：

```go
var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to file")

func main() {
    flag.Parse()
    if *cpuprofile != "" {
        f, err := os.Create(*cpuprofile)
        if err != nil {
            log.Fatal(err)
        }
        pprof.StartCPUProfile(f)
        defer pprof.StopCPUProfile()
    }
    ...
```

​		新代码定义了一个名为的标志`cpuprofile`，调用 [Go标志库](https://golang.org/pkg/flag/)以分析命令行标志，然后，如果`cpuprofile`已在命令行上设置了该标志 ，则将[启动CPU配置](https://golang.org/pkg/runtime/pprof/#StartCPUProfile) 文件重定向到该文件。探查器需要最终调用以 [`StopCPUProfile`](https://golang.org/pkg/runtime/pprof/#StopCPUProfile)在程序退出之前刷新对文件的所有未完成写操作。

​		添加该代码后，我们可以使用新`-cpuprofile`标志运行程序，然后运行`go tool pprof`以解释概要文件。

```bash
$ make havlak1.prof
./havlak1 -cpuprofile=havlak1.prof
# of loops: 76000 (including 1 artificial root node)
$ go tool pprof havlak1 havlak1.prof
Welcome to pprof!  For help, type 'help'.
(pprof)
```

 		该`go tool pprof`程序是[Google `pprof`C ++ profiler](https://github.com/gperftools/gperftools)的略微变体 。最重要的命令是`topN`，它显示了概要文件中的前`N`个样本：

```bash
(pprof) top10
Total: 2525 samples
     298  11.8%  11.8%      345  13.7% runtime.mapaccess1_fast64
     268  10.6%  22.4%     2124  84.1% main.FindLoops
     251   9.9%  32.4%      451  17.9% scanblock
     178   7.0%  39.4%      351  13.9% hash_insert
     131   5.2%  44.6%      158   6.3% sweepspan
     119   4.7%  49.3%      350  13.9% main.DFS
      96   3.8%  53.1%       98   3.9% flushptrbuf
      95   3.8%  56.9%       95   3.8% runtime.aeshash64
      95   3.8%  60.6%      101   4.0% runtime.settype_flush
      88   3.5%  64.1%      988  39.1% runtime.mallocgc
```

​		启用CPU性能分析后，Go程序每秒停止约100次，并在当前正在执行的goroutine堆栈上记录一个由程序计数器组成的样本。该配置文件有2525个样本，因此运行了25秒多一点。在`go tool pprof`输出中，样本中出现的每个函数都有一行。前两列以原始计数和占总样本的百分比的形式显示了函数在其中运行的样本数（而不是等待调用的函数返回）。该`runtime.mapaccess1_fast64`功能在298个样本中运行，即11.8％。`top10`输出按此样本计数排序。第三列显示列表中的运行总计：前三行占样本的32.4％。第四和第五列显示了函数出现（运行或等待被调用函数返回）的样本数。该`main.FindLoops`函数在10.6％的示例中运行，但是在84.1％的示例中的调用堆栈上（正在调用的函数或函数正在运行）。

​		要按第四和第五列进行排序，请使用`-cum`（用于累积）标志：

```bash
(pprof) top5 -cum
Total: 2525 samples
       0   0.0%   0.0%     2144  84.9% gosched0
       0   0.0%   0.0%     2144  84.9% main.main
       0   0.0%   0.0%     2144  84.9% runtime.main
       0   0.0%   0.0%     2124  84.1% main.FindHavlakLoops
     268  10.6%  10.6%     2124  84.1% main.FindLoops
(pprof) top5 -cum
```

​		事实上`main.FindLoops`与`main.main`的调用总数本来应该100％，但每个栈的事例只包括底部100的堆栈帧; 在大约四分之一的样本中，递归`main.DFS`函数的深度比100帧深，`main.main`因此完整的trace被截断了。

​		堆栈跟踪样本包含的有关函数调用关系的有趣数据比文本列表所显示的要多。该`web`命令以SVG格式写入配置文件数据的图形，然后在Web浏览器中将其打开。（还有一个`gv`编写PostScript并在Ghostview中打开它的命令。对于这两个命令，都需要安装[graphviz](http://www.graphviz.org/)。）

```bash
(pprof) web
```

[完整图的](https://rawgit.com/rsc/benchgraffiti/master/havlak/havlak1.svg)一小段 看起来像：

![img](img/havlak1a-75.png)

​		图表中的每个方框都对应一个函数，方框的大小根据运行该函数的样本数确定。从框X到框Y的表示X调用Y；边缘的数字是调用在样本中出现的次数。如果一次调用在单个样本中多次出现（例如在递归函数调用期间），则每次出现都会计入边缘权重。因此`main.DFS`做了21342次的自我调用。

​		乍一看，我们可以看到该程序将大量时间花费在哈希运算上，这与Go `map`值的使用相对应。我们可以告诉我们`web`仅使用包含特定方法的样本如`runtime.mapaccess1_fast64`，该样本会 清除图表中的一些噪声：

```bash
(pprof) web mapaccess1
```

![img](img/havlak1-hash_lookup-75.png)

​		如果斜视一下，我们可以看到`runtime.mapaccess1_fast64`的调用主要是`main.FindLoops`和`main.DFS`方法。

​		现在，我们对总体情况有了一个大概的了解，是时候放大特定功能了。让我们`main.DFS`先来看一下，因为它是一个较短的函数：

```bash
(pprof) list DFS
Total: 2525 samples
ROUTINE ====================== main.DFS in /home/rsc/g/benchgraffiti/havlak/havlak1.go
   119    697 Total samples (flat / cumulative)
     3      3  240: func DFS(currentNode *BasicBlock, nodes []*UnionFindNode, number map[*BasicBlock]int, last []int, current int) int {
     1      1  241:     nodes[current].Init(currentNode, current)
     1     37  242:     number[currentNode] = current
     .      .  243:
     1      1  244:     lastid := current
    89     89  245:     for _, target := range currentNode.OutEdges {
     9    152  246:             if number[target] == unvisited {
     7    354  247:                     lastid = DFS(target, nodes, number, last, lastid+1)
     .      .  248:             }
     .      .  249:     }
     7     59  250:     last[number[currentNode]] = lastid
     1      1  251:     return lastid
(pprof)
```

​		清单显示了该`DFS`函数的源代码（确实是每个与正则表达式匹配的函数`DFS`）。前三列是在运行该行时获取的样本数，在运行该行或从该行调用的代码中获取的样本数以及文件中的行号。相关命令`disasm`显示了该函数的反汇编而不是源列表。如果有足够的样本，这可以帮助您了解哪些指令性能损耗较大。该`weblist`命令混合了两种模式：它显示 [源清单，在其中单击一行将显示反汇编](https://rawgit.com/rsc/benchgraffiti/master/havlak/havlak1.html)。

​		由于我们已经知道主要时间消耗是在哈希运行时函数实现的映射查找中，因此我们最关心第二列。`DFS`正如递归遍历所预期的那样，大部分时间都花在对（第247行）的递归调用上。除了递归，程序似乎需要花费大量时间在第242、246和250行上访问 Map。对于该特定查找，Map不是最有效的选择。就像在编译器中一样，类basic block具有分配给它们的唯一索引。`map[*BasicBlock]int`我们可以使用 `[]int`代替，它是由块号索引的切片。当数组或切片可以使用时，没有理由使用Map。

​		从Map更改为切片在程序中编辑第七行使其运行时间减少了近两倍：

```bash
$ make havlak2
go build havlak2.go
$ ./xtime ./havlak2
# of loops: 76000 (including 1 artificial root node)
16.55u 0.11s 16.69r 1321008kB ./havlak2
```

​		(See the [diff between `havlak1` and `havlak2`](https://github.com/rsc/benchgraffiti/commit/58ac27bcac3ffb553c29d0b3fb64745c91c95948))

​		我们可以再次运行分析器，以确认`main.DFS`不再是运行时间的重要部分：

```bash
$ make havlak2.prof
./havlak2 -cpuprofile=havlak2.prof
# of loops: 76000 (including 1 artificial root node)
$ go tool pprof havlak2 havlak2.prof
Welcome to pprof!  For help, type 'help'.
(pprof)
(pprof) top5
Total: 1652 samples
     197  11.9%  11.9%      382  23.1% scanblock
     189  11.4%  23.4%     1549  93.8% main.FindLoops
     130   7.9%  31.2%      152   9.2% sweepspan
     104   6.3%  37.5%      896  54.2% runtime.mallocgc
      98   5.9%  43.5%      100   6.1% flushptrbuf
(pprof)
```

​		该条目`main.DFS`不再出现在配置文件中，并且程序的其余部分也已删除。现在，该程序将大部分时间都花在分配内存和垃圾回收上（`runtime.mallocgc`同时分配和运行定期垃圾回收的时间占54.2％）。为了找出为什么垃圾收集器运行如此之多，我们必须找出正在分配内存的代码。一种方法是向程序添加memory profiling。我们将安排如果`-memprofile`提供了该标志，则程序在循环查找一次迭代后停止，写入memory profile，然后退出：

```go
var memprofile = flag.String("memprofile", "", "write memory profile to this file")
...

    FindHavlakLoops(cfgraph, lsgraph)
    if *memprofile != "" {
        f, err := os.Create(*memprofile)
        if err != nil {
            log.Fatal(err)
        }
        pprof.WriteHeapProfile(f)
        f.Close()
        return
    }
```

​		我们调用带有`-memprofile`标志的程序来编写配置文件：

```bash
$ make havlak3.mprof
go build havlak3.go
./havlak3 -memprofile=havlak3.mprof
```

​	（请参阅[havlak2](https://github.com/rsc/benchgraffiti/commit/b78dac106bea1eb3be6bb3ca5dba57c130268232)的[差异](https://github.com/rsc/benchgraffiti/commit/b78dac106bea1eb3be6bb3ca5dba57c130268232)）

​		我们使用`go tool pprof`完全相同的方式。现在我们正在检查的样本是内存分配，而不是时钟滴答。

```go
$ go tool pprof havlak3 havlak3.mprof
Adjusting heap profiles for 1-in-524288 sampling rate
Welcome to pprof!  For help, type 'help'.
(pprof) top5
Total: 82.4 MB
    56.3  68.4%  68.4%     56.3  68.4% main.FindLoops
    17.6  21.3%  89.7%     17.6  21.3% main.(*CFG).CreateNode
     8.0   9.7%  99.4%     25.6  31.0% main.NewBasicBlockEdge
     0.5   0.6% 100.0%      0.5   0.6% itab
     0.0   0.0% 100.0%      0.5   0.6% fmt.init
(pprof)
```

​		该命令`go tool pprof`报告`FindLoops`已分配了约86.3 MB的使用中的56.3；`CreateNode`占另外17.6 MB。为了减少开销，内存探查器仅记录分配的每半兆字节大约一个块的信息（“ 1/524288的采样率”），因此这是实际计数的近似值。

​		为了找到内存分配，我们可以列出这些功能。

```bash
(pprof) list FindLoops
Total: 82.4 MB
ROUTINE ====================== main.FindLoops in /home/rsc/g/benchgraffiti/havlak/havlak3.go
  56.3   56.3 Total MB (flat / cumulative)
...
   1.9    1.9  268:     nonBackPreds := make([]map[int]bool, size)
   5.8    5.8  269:     backPreds := make([][]int, size)
     .      .  270:
   1.9    1.9  271:     number := make([]int, size)
   1.9    1.9  272:     header := make([]int, size, size)
   1.9    1.9  273:     types := make([]int, size, size)
   1.9    1.9  274:     last := make([]int, size, size)
   1.9    1.9  275:     nodes := make([]*UnionFindNode, size, size)
     .      .  276:
     .      .  277:     for i := 0; i < size; i++ {
   9.5    9.5  278:             nodes[i] = new(UnionFindNode)
     .      .  279:     }
...
     .      .  286:     for i, bb := range cfgraph.Blocks {
     .      .  287:             number[bb.Name] = unvisited
  29.5   29.5  288:             nonBackPreds[i] = make(map[int]bool)
     .      .  289:     }
...
```

​		看起来当前的瓶颈与最后一个瓶颈相同：使用Map，而不是简单的数据结构。 `FindLoops`正在分配约29.5 MB的地图。

顺便说一句，如果我们`go tool pprof`使用该`--inuse_objects`标志运行，它将报告分配计数而不是大小：

```bash
$ go tool pprof --inuse_objects havlak3 havlak3.mprof
Adjusting heap profiles for 1-in-524288 sampling rate
Welcome to pprof!  For help, type 'help'.
(pprof) list FindLoops
Total: 1763108 objects
ROUTINE ====================== main.FindLoops in /home/rsc/g/benchgraffiti/havlak/havlak3.go
720903 720903 Total objects (flat / cumulative)
...
     .      .  277:     for i := 0; i < size; i++ {
311296 311296  278:             nodes[i] = new(UnionFindNode)
     .      .  279:     }
     .      .  280:
     .      .  281:     // Step a:
     .      .  282:     //   - initialize all nodes as unvisited.
     .      .  283:     //   - depth-first traversal and numbering.
     .      .  284:     //   - unreached BB's are marked as dead.
     .      .  285:     //
     .      .  286:     for i, bb := range cfgraph.Blocks {
     .      .  287:             number[bb.Name] = unvisited
409600 409600  288:             nonBackPreds[i] = make(map[int]bool)
     .      .  289:     }
...
(pprof)
```

​		由于〜200,000个map占29.5 MB，因此初始Map分配大约需要150个字节。当使用Map来保存键值对时，这是合理的，但是当映射被用作简单集合的替代者时（如此处所示），这是不合理的。

​		我们还可以使用简单的切片来列出元素而不是Map。不得已使用Map时，该算法将不能插入重复元素。在剩下的一种情况下，我们可以编写一个`append`内置函数的简单变体：

```go
func appendUnique(a []int, x int) []int {
    for _, y := range a {
        if x == y {
            return a
        }
    }
    return append(a, x)
}
```

​		除了编写该函数外，将Go程序更改为使用切片而不是Map仅需要更改几行代码。

```bash
$ make havlak4
go build havlak4.go
$ ./xtime ./havlak4
# of loops: 76000 (including 1 artificial root node)
11.84u 0.08s 11.94r 810416kB ./havlak4
$
```

​	（请参阅[havlak3](https://github.com/rsc/benchgraffiti/commit/245d899f7b1a33b0c8148a4cd147cb3de5228c8a)的[差异](https://github.com/rsc/benchgraffiti/commit/245d899f7b1a33b0c8148a4cd147cb3de5228c8a)）

现在的速度比开始时快2.11倍。让我们再次查看CPU配置文件。

```bash
$ make havlak4.prof
./havlak4 -cpuprofile=havlak4.prof
# of loops: 76000 (including 1 artificial root node)
$ go tool pprof havlak4 havlak4.prof
Welcome to pprof!  For help, type 'help'.
(pprof) top10
Total: 1173 samples
     205  17.5%  17.5%     1083  92.3% main.FindLoops
     138  11.8%  29.2%      215  18.3% scanblock
      88   7.5%  36.7%       96   8.2% sweepspan
      76   6.5%  43.2%      597  50.9% runtime.mallocgc
      75   6.4%  49.6%       78   6.6% runtime.settype_flush
      74   6.3%  55.9%       75   6.4% flushptrbuf
      64   5.5%  61.4%       64   5.5% runtime.memmove
      63   5.4%  66.8%      524  44.7% runtime.growslice
      51   4.3%  71.1%       51   4.3% main.DFS
      50   4.3%  75.4%      146  12.4% runtime.MCache_Alloc
(pprof)
```

​		现在，内存分配和后续的垃圾回收（`runtime.mallocgc`）占了我们运行时间的50.9％。查看系统为何进行垃圾收集的另一种方法是查看导致收集的分配，这些分配大部分时间用于`mallocgc`：

```
(pprof) web mallocgc
```

![img](img/havlak4a-mallocgc.png)

​		很难说出该图中发生了什么，因为有许多节点的小样本数掩盖了大样本。我们可以告诉`go tool pprof`忽略那些不占样本至少10％的节点：

```bash
$ go tool pprof --nodefraction=0.1 havlak4 havlak4.prof
Welcome to pprof!  For help, type 'help'.
(pprof) web mallocgc
```

![img](img/havlak4a-mallocgc-trim.png)

​		现在，我们可以轻松地跟随粗箭头，以了解这`FindLoops`正在触发大多数垃圾收集。如果我们列出，`FindLoops`则可以看到其中的大部分是在开始时：

```bash
(pprof) list FindLoops
...
     .      .  270: func FindLoops(cfgraph *CFG, lsgraph *LSG) {
     .      .  271:     if cfgraph.Start == nil {
     .      .  272:             return
     .      .  273:     }
     .      .  274:
     .      .  275:     size := cfgraph.NumNodes()
     .      .  276:
     .    145  277:     nonBackPreds := make([][]int, size)
     .      9  278:     backPreds := make([][]int, size)
     .      .  279:
     .      1  280:     number := make([]int, size)
     .     17  281:     header := make([]int, size, size)
     .      .  282:     types := make([]int, size, size)
     .      .  283:     last := make([]int, size, size)
     .      .  284:     nodes := make([]*UnionFindNode, size, size)
     .      .  285:
     .      .  286:     for i := 0; i < size; i++ {
     2     79  287:             nodes[i] = new(UnionFindNode)
     .      .  288:     }
...
(pprof)
```

​		每次`FindLoops`调用时，它都会分配一些相当大的簿记结构。由于基准测试调用了`FindLoops`50次，因此这些操作会增加大量垃圾，因此垃圾收集器需要进行大量工作。

​		拥有垃圾收集语言并不意味着您可以忽略内存分配问题。在这种情况下，一种简单的解决方案是引入缓存，以便每个调用`FindLoops` 在可能的情况下重用前一个调用的存储。（实际上，在Hundt的论文中，他解释说Java程序只需要进行此更改即可获得合理的性能，但是他在其他垃圾收集的实现中没有进行相同的更改。）

​		我们将添加一个全局`cache`结构：

```
var cache struct {
    size int
    nonBackPreds [][]int
    backPreds [][]int
    number []int
    header []int
    types []int
    last []int
    nodes []*UnionFindNode
}
```

​		然后`FindLoops`访问它以替代分配：

```go
if cache.size < size {
    cache.size = size
    cache.nonBackPreds = make([][]int, size)
    cache.backPreds = make([][]int, size)
    cache.number = make([]int, size)
    cache.header = make([]int, size)
    cache.types = make([]int, size)
    cache.last = make([]int, size)
    cache.nodes = make([]*UnionFindNode, size)
    for i := range cache.nodes {
        cache.nodes[i] = new(UnionFindNode)
    }
}

nonBackPreds := cache.nonBackPreds[:size]
for i := range nonBackPreds {
    nonBackPreds[i] = nonBackPreds[i][:0]
}
backPreds := cache.backPreds[:size]
for i := range nonBackPreds {
    backPreds[i] = backPreds[i][:0]
}
number := cache.number[:size]
header := cache.header[:size]
types := cache.types[:size]
last := cache.last[:size]
nodes := cache.nodes[:size]
```

​		当然，这样的全局变量是不好的工程实践：这意味着并发调用`FindLoops`现在是不安全的。目前，我们正在进行尽可能小的更改，以了解对程序性能至关重要的方面。此更改很简单，并且反映了Java实现中的代码。Go程序的最终版本将使用一个单独的`LoopFinder`实例来跟踪此内存，从而恢复了并发使用的可能性。

```bash
$ make havlak5
go build havlak5.go
$ ./xtime ./havlak5
# of loops: 76000 (including 1 artificial root node)
8.03u 0.06s 8.11r 770352kB ./havlak5
```

（请参阅[havlak4](https://github.com/rsc/benchgraffiti/commit/2d41d6d16286b8146a3f697dd4074deac60d12a4)的[差异](https://github.com/rsc/benchgraffiti/commit/2d41d6d16286b8146a3f697dd4074deac60d12a4)）

​		我们还可以做更多的工作来清理程序并使之更快，但是它们都不需要我们尚未展示的分析技术。内部循环中使用的工作清单可以在迭代和对的调用中重复使用`FindLoops`，并且可以与在该过程中生成的单独的“节点池”结合使用。同样，循环图存储可以在每次迭代中重用，而不是重新分配。除了这些性能更改外， [最终版本](https://github.com/rsc/benchgraffiti/blob/master/havlak/havlak6.go) 还使用惯用的Go风格，数据结构和方法编写。样式更改对运行时间影响很小：算法和约束条件不变。

​		最终版本的运行时间为2.29秒，使用的内存为351 MB：

```bash
$ make havlak6
go build havlak6.go
$ ./xtime ./havlak6
# of loops: 76000 (including 1 artificial root node)
2.26u 0.02s 2.29r 360224kB ./havlak6
```

​		这比我们开始使用的程序快11倍。即使我们禁用了对生成的循环Map的重用，以便唯一的缓存内存是循环查找布尔值，程序仍然比原始程序运行速度快6.7倍，并且使用的内存少1.5倍。

```
$ ./xtime ./havlak6 -reuseloopgraph=false
# of loops: 76000 (including 1 artificial root node)
3.69u 0.06s 3.76r 797120kB ./havlak6 -reuseloopgraph=false
$
```

​		当然，将此Go程序与原始C ++程序进行比较不再公平，后者使用像`set`s这样效率低下的数据结构，而`vector`s更合适。作为健全性检查，我们将最终的Go程序翻译为 [等效的C ++代码](https://github.com/rsc/benchgraffiti/blob/master/havlak/havlak6.cc)。它的执行时间类似于Go程序的执行时间：

```
$ make havlak6cc
g++ -O3 -o havlak6cc havlak6.cc
$ ./xtime ./havlak6cc
# of loops: 76000 (including 1 artificial root node)
1.99u 0.19s 2.19r 387936kB ./havlak6cc
```

​		Go程序的运行几乎与C ++程序一样快。由于C ++程序使用自动删除和分配而不是显式缓存，因此C ++程序更短，更容易编写，但并不是那么容易：

```
$ wc havlak6.cc; wc havlak6.go
 401 1220 9040 havlak6.cc
 461 1441 9467 havlak6.go
```

（请参阅[havlak6.cc](https://github.com/rsc/benchgraffiti/blob/master/havlak/havlak6.cc) 和[havlak6.go](https://github.com/rsc/benchgraffiti/blob/master/havlak/havlak6.go)）

​		基准测试和测量的程序一样好。我们曾经`go tool pprof`研究过效率低下的Go程序，然后将其性能提高了一个数量级，并将其内存使用量减少了3.7倍。随后与等效优化的C ++程序进行的比较表明，当程序员注意内部循环会产生多少垃圾时，Go可以与C ++竞争。

​		[GitHub上](https://github.com/rsc/benchgraffiti/)的[benchgraffiti项目](https://github.com/rsc/benchgraffiti/)中提供了程序源，Linux x86-64二进制文件以及用于撰写本文的配置文件。

​		如上所述，[`go test`](https://golang.org/cmd/go/#Test_packages)已经包括以下这些分析标志：定义一个 [基准函数](https://golang.org/pkg/testing/)，一切就绪。还有一个用于分析数据的标准HTTP接口。在HTTP服务器中，添加

```
import _ "net/http/pprof"
```

将在下安装一些URL的处理程序`/debug/pprof/`。然后，您可以`go tool pprof`使用单个参数运行-服务器配置文件数据的URL，它将下载并检查实时配置文件。

```
go tool pprof http://localhost:6060/debug/pprof/profile   # 30-second CPU profile
go tool pprof http://localhost:6060/debug/pprof/heap      # heap profile
go tool pprof http://localhost:6060/debug/pprof/block     # goroutine blocking profile
```

goroutine阻止配置文件将在以后的文章中进行解释。敬请关注。
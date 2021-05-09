## Redis-PipeLine

正常我们使用redis的时候，使用的是 命令->结果，命令->结果的形式。

然而这样会产生一些不必要的损失，主要有以下两个：

*1.RTT （往返时间）*

*2.系统损失（多次发送 使用socket 会产生多次陷入内核）*

这样以来，流水线的性能和单个请求性能对比如下图所示：

e![pipeline_iops](img\pipeline_iops.png)


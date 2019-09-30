---
layout: single
title:  "一些Linux平台性能分析工具"
categories:
  - Written in Chinese
  - data structure
tags:
  - etcd
  - btree
toc: true
excerpt_separator: <!--more-->
---

本文记录一些linux平台很好用的性能指标收集和分析工具。以下命令皆运行在ubuntu 16.04上。

<!--more-->

## dstat

以下介绍摘自man page。

> Dstat is a versatile replacement for vmstat, iostat and ifstat. Dstat overcomes some of the limitations and adds some extra features.

安装：

```
sudo apt-get install dstat
```

不加任何参数，直接运行dstat，它会默认使用-cdngy这几个选项。这几个选项的含义是： c : cpu stats d : disk stats n : network stats g : page stats y : system stats。

输出：
<figure>
  <img src="/assets/images/linux_perf_tools/dstat_default.png">
<figcaption>boltdb pages</figcaption>
</figure>

展示使用cpu最高的进程，最慢的进程，使用内存最多的进程。这里的latency是指调度的延迟，参考：<https://www.kernel.org/doc/Documentation/scheduler/sched-stats.txt>

```bash
dstat --top-cpu-adv --top-latency --top-mem
```

输出：
<figure>
  <img src="/assets/images/linux_perf_tools/dstat_top_cpu_and_latency_and_mem.png">
<figcaption>boltdb pages</figcaption>
</figure>


可以将dstat的输出保存在csv文件当中。例如如下命令，每2秒统计一次相关的指标，一共统计10次，将结果存入report.csv中。

```
dstat --time --cpu --mem --load --output report.csv 2 10
```

输出：
<figure>
  <img src="/assets/images/linux_perf_tools/dstat_save_output_to_file.png">
<figcaption>boltdb pages</figcaption>
</figure>


参考:
<https://hostpresto.com/community/tutorials/how-to-install-and-use-dstat-on-ubuntu-16-04/>

TODO

- iostat
- systat
- tsar
- sar
- <http://www.brendangregg.com/perf.html>
- <https://github.com/raboof/nethogs>

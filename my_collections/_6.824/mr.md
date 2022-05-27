---
date: 2022-05-26
categories: mapreduce 6.824
title: "Lab 1: MapReduce"
---

# 摘要

6.824 的第一个实验，已经充分展示了分布式系统的理念，分布式系统由多个主机组成，之间通过网络沟通协作构成一个系统。

其实分布式系统可以类比成公司，公司的人员之间进行协作，一起完成项目，分布式系统更像是社会，单体应用则像个人。

下面的话简单讲解下我个人 [mapreduce(mr) lab](https://github.com/hexinatgithub/6.824-2022/tree/solution/src/mr) 的代码。

## 源码

[lab](http://nil.csail.mit.edu/6.824/2022/labs/lab-mr.html) 页面和[论文](http://nil.csail.mit.edu/6.824/2022/papers/mapreduce.pdf) 里面所说，
`mr` 主要由一个 `coordinator` 和 多个 `worker` 组成。`coordinator` 把 `task` 分配给 `worker` ，待 `worker` 完成所有任务后，`mapreduce` 结束。

`worker` 接受任务并完成任务，`worker` 既可以是 `map task` 又可以是 `reduce task` ，接到对应的任务调用 `mapf` 或 `reducef` 即可。
由于 `mr` 在 `map tasks` 没有完成时，是不可以进入到 `reduce` 阶段的，所以需要增加些任务类型，`standby task` 和 `exit task` 。
`standby task` 让 `worker` 进入到短暂的休眠阶段，`exit task` 告知 `worker` `mapreduce` 结束了，进程退出。

`coordinator` 则主要是跟踪和分配任务，为此定义两个 RPC 接口，`GetTask` 和 `Report` ，前者获取任务，后者汇报 `map` 或 `reduce` 任务的结果，如文件地址，
任务 ID 等，`coordinator` 记录状态。`coordinator` 还需要有个定时器定时查看过期任务，并回收任务，分配给其他 `worker` ，由于 `worker` 函数是幂等性的，`Report`
的重复任务可以忽略。

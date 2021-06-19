---
title: React源码细读-深入了解scheduler"多任务时间管理大师"
date: 2021-06-19 16:43:43
categories:
	- Web
	- JavaScript
	- React
	- Scheduler
tags:
	- React
	- JavaScript
	- Scheduler
---

> 此文基于 `scheduler v0.20.1` 分析，[仓库传送门](https://github.com/facebook/react)

# Scheduler 多任务时间管理大师

在 `react` 提出 `Fiber` 之前，复杂耗时任务会阻塞页面的渲染，降低页面的响应速度，为了缓解耗时任务与渲染等其他进程之间资源争夺的情况，`react` 增加了 `scheduler` 这一模块，通过 `scheduler` 更好的调度多任务，控制任务的执行顺序

`scheduler` 通过划分任务优先级, 时间切片, 任务中断、任务恢复等机制来保证高优任务的执行，只占用每一帧中尽可能短的时间用于处理任务，把主要资源在有必要的情况下交还给其他线程，以此提高页面响应速度，提升用户体验

能做到的这些不得不说是“多任务时间管理大师”了😁

# React & Scheduler 缠缠绵绵

# Scheduler 如何调度

# Scheduler 如何执行

# 参考

[postmessage & scheduler](https://www.yuque.com/docs/share/8c167e39-1f5e-4c6d-8004-e57cf3851751)

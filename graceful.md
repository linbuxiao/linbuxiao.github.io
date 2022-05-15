---
layout: post
title: HTTP Server 优雅关闭
---

当你需要关闭服务器时，还存在着大量的活跃连接，直接关闭会使大量逻辑中断，未事务化的部分则会导致数据不一致（这也是事务化的重要之处）。

此时我们需要在关闭服务器前，进行以下步骤：
1. 拒绝将要到来的连接
2. 完成还在进行的连接
3. 完成其余连接关闭（如数据库连接）
4. 停止服务

服务的优雅关闭同时也是线上服务热更新的前提：在无宕机的情况下升级服务，这个我们之后讨论。

在 Golang 中，可以通过以下方式实现。

## [sync.WaitGroup](./bywaitgroup/main.go)

我们知道，每一次的访问都会让 `http.Server` 调度一个 `goroutine` 去处理。

那么我们可以在 `Handler` 中对事先声明的 `sync.WaitGroup` 去 `Add(1)`，
并在逻辑处理结束后调用 `wg.Done()`。
最后我们在最外层去 `wg.Wait()`，这样就能够保证服务在所有逻辑结束后关闭。

这样的实现确实能达成我们的需求。在现实场景中，我们可以使用一个全局的 `WaitGroup` 抑或是 `ErrGroup` 去对连接数量进行控制。

## [connState Hook](./byconnstate/main.go)

为了能更好的控制 `waitGroup` 的计数，我们可以使用 `http.Server` 下的 `ConnState` Hook。

它可以在连接的每个生命周期提供回调，然后我们根据回调周期进行计数，可以从 `conn` 维度控制当前活跃连接状态。

## [Shutdown()](./byshutdown/main.go)

最后我们引出 `http.Server` 自身的实现：`srv.Shutdown()`。

它采用了先关闭所有的 `listener`，然后关闭停用的 `conn`，最后轮询，无限期等待活跃连接关闭的方式。

由于 `conn` 不会因为 `listener` 的停用而直接关闭，所以在你调用该方法时，`http.ListenAndServe` 方法会首先停用，并抛出服务停止的信息。
而 `Shutdown` 则会在服务完全停止后再返回信息。所以你可能需要预先阻塞主线程，并在方法结束后再放开。
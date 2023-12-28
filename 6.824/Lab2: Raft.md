一个`follower`在给别人投票后，自己还能成为`candidate`吗？

> 当它收到一个投票请求后，它的`isActive`属性将会被设为`true`，因此，除非在对应时间内没有收到消息，否则它的状态会一直是`follower`，只有超时后，才会发起一轮新的选举，自己成为`candidate`

只有`leader`提交并执行了命令之后，它才会对`client`进行回复。如果`client`端没有接收到响应，应该在一段时间后再次发送请求。

只有日志比大多数人都新的`candidate`才能成为`leader`

当一个`raft`实例投票给另一个实例后，这个投票什么时候失效呢？也就是说，什么时候可以投新的票呢？

> 目前的代码逻辑是：如果一个`candidate`拥有更新的`term`，那么只要它有和投票者至少同样新的日志，那么投票者就会给这个`candidate`投票，如果一个`leader`给这样的`candidate`投票，那么这个`leader`会直接变为`follower`

如果一个`raft`实例收到一个具有更新`term`的投票请求，但是这个`candidate`的日志没有这个实例新，也就是说不会给这个`candidate`投票，这种情况下，需要更新实例的`term`吗？

> 应该是需要的：If RPC request or response contains term `T > currentTerm`: set `currentTerm = T`, convert to follower (§5.1)

实现一个定时器来精确控制超时选举

在一个`term`中，`votedFor`应该只被改变一次。

如果一条记录已提交，意味着状态机可以安全地执行该记录

一个`leader`可能被一个具有更新的`term`的实例影响后下台，之后又马上成为这个更新的`term`的`leader`，因此我们需要特别注意。

当一个`leader`向其他实例发送`AppendEntries`时，应该首先根据`nextIndex`把自己未提交的日志都给发送过去。

为什么一个过时的`leader`再次和其他的实例连接上时，可以发送自己的未提交`log`给到其他实例并覆盖该实例的已提交log?

> 当一个过时的`leader`重新回到集群中时，由于我们使用锁的原因，造成在发送`AppendEntries`时，可能已经不是`leader`的身份了，因此，此时我们应该直接停止那些还未发送的`rpc`，已发送的`rpc`带着的是老的`term`会被直接拒绝掉

2023/09/21 19:06:36 [1] claimed to be the leader, index is 102, term is 4
2023/09/21 19:06:36 [4] refuse [1]'s append entries with leader term: 4 currentTerm: 19
2023/09/21 19:06:36 [1]'s term is 19, [3]'s term is 18
2023/09/21 19:06:36 [1] to [3] entry index 0, term 4, data 202
2023/09/21 19:06:36 [2] refuse [1]'s append entries with leader term: 4 currentTerm: 19
2023/09/21 19:06:36 [1]'s term is 19, [0]'s term is 19

对于`leader`，我们需要一个`goroutine`来接受并发送`client`的消息，以及一个`goroutine`来发送心跳。
对于每一个实例，我们都需要一个`goroutine`来提交日志到`applyCh`，以及一个`goroutine`来进行超时选举

我们可以用一个缓冲区来一直接收来自`chan`的数据，如果时间到了或者收集到了特定量的数据，就触发一个`cond.signal()`，这样的话可以避免每接收一个数据就要`sleep`的问题

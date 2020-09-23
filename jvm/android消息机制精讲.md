这篇文章一起来学习几个问题：

1. Handler、Looper、Message之间的关系？
2. Handler如何实现线程切换？
3. Handler内存泄露的本质？
4. 延迟消息是如何实现的？
5. 主线程的Looper死循环为什么不会卡顿？（EPoll）

## Handler、Looper、Message之间的关系？


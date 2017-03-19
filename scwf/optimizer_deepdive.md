http://sqlblog.com/blogs/paul_white/archive/2012/04/28/query-optimizer-deep-dive-part-1.aspx

貌似sqlserver里面filer逻辑算子叫select

Sqlserver 单独把Simplification拿出来作为一个独立阶段，做常数折叠，predict push down，矛盾检查，join简化，感觉是对应spark sql优化器里面的 rbo阶段

Trivial Plan 用一个阶段区分是否要进入cbo框架，我们的框架也需要考虑。



CBO 并非是基于代价找到最优的plan，而是尽快的找到一个不错的plan。

Simplification的结果作为CBO的输入

optimizer places strict limits on itself to avoid spending more time on optimization than it saves on a single execution.  





![屏幕快照 2017-01-16 下午3.53.47](/Users/wangfei/Documents/屏幕快照 2017-01-16 下午3.53.47.png)
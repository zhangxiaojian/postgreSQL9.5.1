问题：改进vacuum垃圾回收工作进程。允许vacuum回收大于当前最小事务快照或未结束事务号的不需要的tuple版本.


####一 问题分析
由于多版本并发控制的影响，PG中对于删除的tuple采用标记删除的方式，删除或更新产生的垃圾元组由vacuum机制负责回收，对于元组的删除，只是标记了 HeapTupleFields 中的t_xmax,表示删除这个tuple的事务号，但是vacuum负责删除ItemIdData中标记了dead的tuple。ItemIdData是在page内指向元组的指针

![page](http://www.zhangxiaojian.name/wp-content/uploads/2016/03/page.png)

Linp为ItemIdData类型的数据，指向对应的tuple。所以，一个进程删除掉自己的tuple之后，只是更改tuple内的HeapTupleFields中的t_xmax，对应的Linp并没有任何改变。而vacuum机制是根据Linp中的flage，判断元组的状态，如果为dead就标记为unuse，然后物理删除，并整理page内的空闲空间。

Linp的标记改变是在vacuum中完成的，具体是在heap_page_prune中，它负责处理tuple的Hot链，并且遍历tuple，判断tuple是否可以认为已经dead。而判断原则是否为dead的函数为：HeapTupleSatisfiesVacuum，这个函数有一个重要的参数:OldestXmin，表示当前事务（也就是执行vacuum的事务）启动的时候，数据库其它进程所有的活动事务中Xmin最老的值。也就是最早启动的事务。Vacuum认为如果是在OldestXmin事务之后删除的tuple，也就是t_xmax < OldestXmin, 这种tuple还有可能被当前获得事务访问，所以还不能被vacuum掉。所以即使其它事务时read cmmitted级别，delete之后就无法在读取元组，也就是不在需要被删除的元组，vacuum仍然选择不删除。

分析到这，问题的关键就是修改获得OldestXmin这个值得策略，要获得隔离级别大于repeatableread的所有活动事务的的OldestXmin，用这个值来判断一个tuple是否该标记为dead被vacuum回收。

####二 解决策略
GetOldestXmin这个函数负责返回OldestXmin值，需要修改其中的策略。原有策略比较简单，遍历共享内存变量中的 allProcs 和 allPgXact，取出每一个进程，判断进程访问的数据库是否是vacuum当前要清理的数据库，取出每一个活动的事务，找出访问当前数据库的最老的一个xid，作为GetOldestXmin的返回值。

问题关键在于除了获得活动事务的xid，还要获得事务的隔离级别，小于repeatable read的事务可以忽略不计。但是原本的PGPROC 和 PGXACT 结构中并没有包含隔离级别的信息，需要自己添加。按照常理，应该放到PGXACT结构中，毕竟隔离级别是与每个事务有关。可是看源码中关于PGXACT的注释，貌似用到了比较黑的科技，可以充分利用多核CPU来加速（唐成老师貌似提到过），特别注明要非常慎重的修改这个结构。于是就打算放到PGPROC中，一个进程每个时刻也只会有一个顶层事务运转。添加之后，引起一连串错误，甚至initdb命令都会受到影响，波及范围太广。这时候灵光一闪：发现PGXACT结构中的vacuumFlags是一个unit8类型的变量，但是它的标记位只用到了前5位，我可以利用后面的3位来标记一个事务的隔离级别！

在每一个postgres进程都有一个从共享内存变量中分配的MyPgXact全局变量，需要在事务处理的时候，获得隔离级别，然后set进MyPgXact，一般开启一个事务的时候会调用StartTransaction函数，里面会将隔离级别初始化为默认级别（read committed），如果语句中指定隔离级别的，在portal->run之后，事务提交之前，会改变表示隔离级别的全局变量。最初的想法是一旦全局的隔离级别变量XactIsoLevel改变，就修改MyPgXact中的位标记，但是在函数assign_XactIsoLevel中添加代码之后，会出现奇怪的错误。就把改变隔离级别的动作放到portal->run之后。这个部分我觉得是有待改变的。

好了，每个进程通过共享内存set隔离级别位，vacuum的时候根据隔离级别判断OldestXmin，传给HeapTupleSatisfiesVacuum之后，就可以返回没有用的dead tuple，标记Linp为dead，vacuum后续根据这个dead标记，标记unuse，进行内存整理，就完成了一次优化过vacuum。

####三 效果
![result](http://www.zhangxiaojian.name/wp-content/uploads/2016/03/result.png)
按照顺序，read cmmitted级别的可以vacuum掉，repeatable read的不能vacuum。
####四 修改的文件
|Functions						              |File				   |line number    |
|-----------------------------|:---------:|:-------------:|
|exec_simple_query		          |postgres.c	|(1111 ~ 1127)  |
|								                     |proc.h			  |(48 ~ 53)      |
|StartTransaction			          |xact.c			  |(1873~1875)    |
|GetOldestXminVacuumOptimized	|procArray.c|(1298~1499)    |
|      									              |procArray.h|(55)           |
|vacuum_set_xid_limits				    |vacuum.c		 |(492)          |

####五 不足之处
1. 测试用例有限，在GetOldestXmin中有很多处理分支，包括replication_slot_xmin等参数都没有搞明白是干什么的。
2. 设置隔离级别的时机，在exec_simple_query中不一定合适。
3. 其它过程的函数都有很多处理分支并未完全明白，也许会有影响

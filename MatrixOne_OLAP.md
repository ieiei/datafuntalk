好大家好
呃很荣幸代表我们团队
今天做这么一个分享
呃
今天主要是分享一下我们
这个HTAP数据库的OLAP方面的设计和实现。


先介绍一下我们公司，
矩阵起源是一家数据库的创业公司，我们的产品叫Matrix One。
Matrix One是一个云原生的HTAP数据库，使用go语言开发，每一行代码我们都是从头开始写的。

Matrix One可以同时支持OLTP、 OLAP、streaming等不同的工作负载。
用户可以在公有云或者私有云，或者是自建的比如HDFS上面存储自己的数据。
可以无缝的在这上面部署和运行。

我今天沿这个分享
大概分这么几个部分：
第一部分是matrix one的整体的架构
第二部分是matrix one里面Olap引擎的一些设计
第三部分是一个测试的结果


先说matrix one整体的架构，大概在一年前，在架构上面做了一个比较大的调整。
之所以做这么调整的话，是因为之前的旧有的架构会有一些痛点无法解决。
我先介绍一下我们早期的架构是个什么样子的。
matrix one早期的架构就是一个比较典型的share nothing架构，就是说数据是存放在一个Multi Raft集群上面。

数据的每一个切片存在一个Raft上面，不同的Raft和group之间的数据是互相是完全没有重叠的

那么这么一个架构的话
就大概分成这么几个部分吧
就是下面存储层
存储层的话我们之前是打算要做
几个不同的存储引擎
一个是AOE，AOE只能支持AP查询
他没有事务的能力
AOE他只能是往后面追加的添加写数据，包括删除也不支持
然后之后为了支持TP查询的话
我们之前打算要新加一个存储引擎，
叫TAE就是说它会带有完整的事务来
在这个存储层之上的话就是分布式框架。
分布式框架是我们自己做的一个
multi raft的一个架构
就是来来做这些存储
存储节点的调度
在上面的话就是计算层
计算层是一个比较经典的MPP的执行的架构。
那这早期的架构有有什么问题呢
就是说它会有几个痛点我们之前一直无法解决
一个是扩展性，因为他是share nothing架构
每扩展一个节点的话，就要同时扩展存算的资源，因为计算和存储并没有完全的分开
而且每扩展一个节点的话，会做大量的数据迁移工作。
还有一个问题是，每份数据至少要保存3个副本，因为raft协议，每一份数据的都要保持 3个副本

从扩展阶段到完成的话时间会非常的久，每扩展一个节点可能要搬迁大部分的数据。
还有一个问题，是性能上，因为Raft协议的话我们都知道
他的写操作需要复制到所有节点上面去
但是他独操作的话
每个妈每个Raft group
他只能在leader节点上面去做读取
这样的话就容易会造成
leader节点成为一个热点然后成为性能的瓶颈
嗯还有还有就是说
嗯因为我们之前是打算设计多种各种不同的引擎
然后这些引擎的话它的性能也是非常不一样的
就无法有效的应对htap的场景
比如说用户他既需要有AP的查询，又需要做TP的查询的话，这种扩展性也是非常不好
然后的话就是成本的问题，还是那个raft本身的问题，因为数据要保存三副本。
实际上那个数据冗余是非常大的。
用户自己来，如果我们数据是放在用户自己的私有云或者是HDFS上的话。
随着节点规模上去的话，成本就不断的攀升，
而且由于存算不分离，存储也是放在用户自己的机房里面，
所以说需要让用户用非常好的SSD，才能很好的发挥数据库预期的性能。
这是几个痛点的话就是我们之前思考很久也没法解决。
所以在去年大概一年前的时候吧
时候吧就是推出了一个新的整体的架构
新的架构的话整体来说就是这个样子会有几个部分
最上面是计算层
计算层里面每一个单位叫做cn
就是computation note计算节点。

在计算下面的话有数据节点datanode，再下面的话是file service。
fire service的话就是支持
各种不同的文件系统
呃我和后面再详细的对每个部分，分别介绍一下。

呃先从最底下说起FIle service
FIle service的话它就是说会支持各种不同的文件系统
呃比如说你用户自己的本地的实盘
也可以然后NFS，HDFS都也也好
然后或者是对象存储，比如现在各种公有云的对象存储，量大又非常便宜。

file service对上层的话会提供一个统一的接口，对用户来说他并不需要关心最底下的那个文件系统
数据本身是存在什么样的介质上面
这边有个log service，log service存的就是那个
因为我们知道file service我们
就是我们插入数据的时候
我们肯定是只能一整块一整块的往往下面写
特别像S3它一个object就是非常大
这样的话我们
我们写数据时候不可能是一行一行都
都往那个fast里面写
这fast的话
我们一般是会积累到一定的量再往里面写一个整块
那么这些还没形成整块的部分的话，会放在一个单独log service里面去存放
所以说log service的话它还是几个Multi  Raft的一个集群
就是他就说我们已经形成整块的部分
他的数据完整性和一致性的话是用S3或者HDF是他们自己的功能来保证
然后还没有形成单个block的这些数据的话
它的数据一致性完整性的话，就通过Multi Raft group来保证。

在上面的话是存储节点
就存储节点这里存的是什么数据。
存的只有元数据信息，比如说你每一个表分成大的叫segment，每个segment会分成很多小的block
这元数据存放的就是说他每个block他放在具体比如说我们用S3来存放的话
就会每个block对应的哪个s3 object
它存的是这么一个元素去进行
呃还还有其他元素性信息比如说row map
就或者说是
像
自己索引bloom filter
这些信息也会存在DN上，这里可以看到有多个DN
这多个DN它是怎么分布呢
就是说
比如说我们一张表的话它有一个主键
那么我们可以
根据主键来做这么一个分区
呃我们目前是暂时只说了按hash来做这么一个分区，也可以按range来做都是没有问题的。

那如果用户的数据量很小的话
就说对大大量的这个
小客户来说其实一个DN就足够了
就是只是存放元数据的话一个DN就已经足够了

然后再上面的话就是cn节点
这cn就是具体的执行计算任务的节点
我们看的话
这里面CN实际上是可以分成各种不同类型
比如说专门做TP查询用的
做他有专门做AP查询用的用的实验节点
还有专门做streaming用的
还有一些数据库自己后台的一些任务，是对用户不可见的这一部分
还有一个组件的话是叫HA keeper，他可能长得跟那个zookeeper有点像，功能的话也是类似的
就是说在节点之间互相通知上线下线这些信息，然后维护了整个集群集群的可用信息。

那这个就是我们Matrix one，在大概一年前迁移到这么一个新的整体架构上面。

它的好处是什么呢
就说是看我现在是
我们从之前的share nothing架构，迁移到share storage架构。
DN节点的话它不保存任何的数据，它只保存元数据信息。
然后这样的话就不会让单个DN成为一个瓶颈，如果你要做弹性扩缩容的话，比如说DN要新增一个节点或者是要删除一个节点。
他只需要做数据迁移的话只需要交换一些元数据的信息
不需要把所有数据都做搬迁的操作。
这大大简化并加快扩缩容的效率。

然后的话
我们之前的架构的话
就说数据的一致性
完全是通过Raft协议来保证
就是那个三副本的那个架构来保证
那现在的话我们的数据的完整一致性主要是通过S3，
S3它本身就有这样的功能，并且从成本上来看的话也是非常的低，比用户自己去搭建个Multi Raft集群的话它的实际上它的成本会低很多。


然后计算的任务现在也可以很好的把它给去掉开
就是比如说TP和AP的计算
我们可以放在不同的CN节点上去做
因为现在数据的话已经不跟DN绑定在一起
所以说计算的节点也可以完全的解耦。
我们看一下一个典型的查询里面的话
他的读数据的操作是怎么样的
就读数据的话
我们比如说一个查询来到一个CN节点这里
他会首先去访问DN的信息，读元信息，判断某张表在哪些block，甚至会先通过过滤条件会对这些
用row map来做过滤啊
剩下的实际上需要去读的那些block信息
拿到之后再去直接就是先直接去那个file service下面去读取数据

在DN这里其实他就只提供一个元数据信息他就
基本上不会成为一个性能瓶颈
因为这个更多的数据化
他是直接cn去直接向File service去读的
CN上面的话它本身也会维护一个metadata cache。
假设这个数据新鲜度还没有过期的话，可能根本就不需要去访问DN，就直接去file service下面去拉取数据。
上面这就是Matrix ONE现在更新后一个整体架构。

先介绍一下这个我现在的存储引擎叫做TAE
就是说
就是说TAE和a分别就是TP和AP的那个
TAE和a分别就是TP和AP的那个
t和a就是它既有事物的能力也有
也可以处理
很好的处理分析性查询
就说用同一套引擎就同时支持AP和TP
那么它的结构是什么样呢


我们实际上还是可以把它看成
看成是基于列存
首先不同的列之间，它数据仍然是单单独分开存放
然后每每一列的话我们会
按照8192号给它分成一个小的block出来。
为什么选择8,192行的话

因为大多数的数字类型就可以在以个block就可以直接
在在LED开启里面就可以装得下
这样子在后面
嗯
在后面的那个叫PP r
计算的时候会对计算引擎是比较友好
然后
我们可能会有很多个block会组成一个segment
这个segment它会有什么作用呢
就是说假设我们
这个表它有主键或者排序键的情况下
一个segment内部它是会
通过排序键和主键去做这么一个排序

我们这数据存储的话在每一个segment内部是保持有序的，但segment之间的话那可能是会有重叠，这会跟LSM的存储有一些相似的地方。就是在不同的那个segment之间数据是有重叠的
呃但在我们之后如果做了partition功能之后的话
可能会把一个partition的所有的这个问题
也会去做这么一个
呃叫做compaction操作吧
就是说把它们重新拿出来
然后重新做一个归变排序再放进去。
我们现在存储架构的话就是大概就是这么一个样子。


下面会介绍一下MO的OLAP的计算引擎

我们的计算引擎分为四个部分：
1. 第一个是parser
2. 第二个是planner
3. 第三个是optimizer
4. 最后是execution
那parser是什么呢，就是把一个sql语句解析成一个AST树
然后planer就是把AST树转化成一个逻辑计划
然后optimizer把逻辑计划通过各种优化器规则或者是通过一些基于代价估算的方式把它转化成更好的那个逻辑计划。
最后就是执行器，把具体的逻辑计划转化成可执行的plan，然后去具体CPU上面去执行。
呃我后面可能嗯这几个部分的话
我先介绍下我们的parser，
parser的话因为我们是
呃
就说其实各大开源数据扩大
大多数来说都是
不会用不会去手写这么一个parser
说至少说像mysql或者说PG
比如说比以WDB为例的话，他就直接是用的postgres的那个parser的代码
然后我们即使不直接照办的话
也可以用一些YACC的工具去生成这么一个parser
在我们测试之后发现用YACC生成的一个parser的话，他并不比
并不成为一个呃新的贫穷
因为他耗时非常少，所以我们没有必要去手写一个parser，其实像clickhouse的parser是手写的。
呃然后就是它是
生成AST树之后的话
会通过一个逻辑计划器把parser转换成一个可以执行的逻辑计划。逻辑计划器主要是包含两个部分：
一个就是i
然后我我们我们这个逻辑计划的话
因为
哎我们的计算引擎呢就是说
如果要支持指查询的话
因为我们并不支持
像那个sql Server一样

比如说子查询他会转换成Apply join
或者像mysql一样他就是
完全的是从父查询里面拿出一行，然后再带入子查询里面
把子查询完那个完整的执行一次
这样的话子查询是完整不同的
不停的去执行
考虑到在AP查询的场景，这样的一个执行计划其实是不可忍受的。

我们就干脆就完全不支持这种和py
的这种方式
所以说我们只在PY的这一步
就把这查询特效数给做掉
然后是优化器的话就是一般来说
通常来说话就会有一个RBO就说基于规则这个优化
基于规则的话在对大部分查询来说已经是够用
你们优化器通常来说分为两种吧
一种是减少数据IO的，它确实会实际的减少从磁盘从文件系统读取的数据量
还有一种是在计算的过程中减少计算的代价
那么减少磁盘IO的这种的话通常来说就是RBO
所以还
那CPU的话
一般来说就是
是在减少这个实际计算的代价。
我后面我也会具体举一些例子来说明来说明我们这个Amboza是怎样设计这一部分。

那之后的话是从逻辑计划到实际可执行的PIPELINE这一部分
执行器的好坏对OLAP系统的执行效率影响是非常大，后面会介绍一下。
先介绍执行器的部分，因为执行器的话通常来说会有最经典的一个火山模型
这个可能大家都知道就说
呃对于每一行
它是它是一个典型的一个pull模型
就是说从最上层的那个计算的那个operator开始
每次去调用一个NEXT函数去从下面的节点去拿一行新的数据出来
然后做了计算之后，再等待更上层的那个计算节点去调用next从它这里取走。
那么这火山模型它还会有什么问题呢？

首先它是并没有做批量处理并没有做并行化。
它是一行一行的处理，而且每一行的话其实
呃他那个不同程序的节点之间做这么一个调用的话
可能实际上的虚函数的开销。
因为next在不同编程语言肯定是主要是这是要做这种函数重载
就算是虚函数开销也很很有可能是比你
实际计算的是开销还还大
即使不大的话也是会占相当的比例吧
而且这个
就说对
对于缓存来说他也是非常不友好
因为他是他是一行数据他会
跑这么多个不同的operation
但是可能你去下一行的时候
这是他的缓存已经被清洗掉了
那么我们matrix one的执行器是基于一个push模型
基于push模型的话它是
就说可能某几个连续的operator，它可以组成一个流水线。
组成流水线的话就是说，而且每这个流水线里面流动的数据，它并不是一行一行的数据
而是我们刚才说到那个
嗯就是TAE就存储引擎里面的
它一个block
一个block的话就是8,192行

一般来说数字类型的话是可以放进Le Cache里面
对缓存是非常友好的
就是每每一个operator他每一次是
呃要处理完这8192号
才会会给下一个operator
再加上调度的话是
从最下面的那个实际读取每一张表的那个table scan 那个节点开始
从那个节点开始往上面去push
就说这么一个Push模型。
它是以数据为中心而不是以operator为中心
那么他生成他的那个生成过程的话
就是说对那个生成的那个
对我们上一部那个planner和optimizer
生成的逻辑计划，作一个后续遍历
最后续便利之后就可以得到一个基础的pipeline结构

那么这个基础和他的结构的话
他还并没有带上像
呃比如说你回他信息
有多少个CPU和这样的信息
或者说你要需要在多少个cn节点上去执行这么一个信息
那么在后面实际执行的时候
再动态的根据这些信息去做这么一个扩展

举一个简单例子吧，假设有一个简单的查询，有R S T三张表做join。
那我们假设R表是最大的一个表，然后S表和T表相对比较小。
因为他们每个表都有过滤条件

这样的一个典型的hash join
就是说我们会把s这个
S和T这两张表去
构建hash表
然后R表在这两个hash表上面去依次去做探测（Probe）操作，得到join之后的数据。

这么一个逻辑计划的话至少需要插上这么三个pipeline
就说s这张表它的数据读取之后
它做了过滤之后再再建完hash表它它就在这里终止它
T表也是
它建完hash表之后就在这
这join这个算上面
终结掉但是R表这个就最大的
这张表它始终是做probe的
这张表的话他会这pipeline也可以就可以往上走很多步
比如说他在做完滤然后再在第一个join这个表join之后
他输出的仍然也是一个批量
然后这个批量的话会继续往下走
这个在下一个join的话
只能在同一个pipeline的下一个Operator里面，再跟T表做一个join
所以说一个3表join的话，通常来说就是会拆成3个pipeline出来。
在右边的话还包括了那个数据变形的一个信息
比如说S表我们会可能会给像go的话就是给三个协程并行的去读数据

然后在这里会做一个合并的操作，
合并之后构建T标啊体标的话也是会
呃
用三个写成去并行的去读
并行读之后
然后送到这里的构建hash表
R表比较大的话
可能我们可以给更多的scan的一个
就是会这个Pipeline会展开出更多的实际的pipeline出来
那么我们可以看到
就是R表这个pipeline
他他是不会被被阻断的
他通过这个hash这个opportunity之后
他会又继续进到下一个join这个节点

有人会感兴趣就是说我这pipeline假设中间会有一些异常的情况。比如说S表读数据出现了异常，发生错误
那么怎么样能够正常的把整个pipeline计算任务一起给结束掉
任务给一起给结束掉
就说因为我们的也是
使用go语言它本身的一个context的一个类似的数据结构吧
我们R表这个pipeline它会在
在join地方他会从一个嵌入表里面等这个hash表的数据的时候会
同时他会去监听一个context，context是适合这个s这这
这这
这张表它它的这个这些pipeline它的context
它是一个共享的
就如果这边
呃发生异常退出的话这边context会cancel掉，然后这边R表的这个pipeline他也会接收到这个信息
比如他也知道他也可以退出
那么就可以提早的退出然后
当然那a表退出之后的话
他也会把这个退出的信息又同时又通过另外一个context传递到T表上面
这样T表的话也就不用再读
这样整个计算任务就可以直接
直接cancel退出

我们的pipeline大概会做了这些算子。
比较典型的有像聚合，分组啊各种不同专业操作

然后是像构建hash表他这边有很多merge，
merge和这个collection dispatch的话这个把它颜色标识的不一样
有这这些个和其他的
它有什么不一样呢
就是说
对因为其他的算子的话他
他都是在就只能在一个pipeline的中间
他接受的数据是从上一个算子传过来
他发送的数据的话
就直接会发送给下一个算子去做后续的计算
那么这个标成灰色这一部分的话他是
就说他是在这么一个pipeline的数据的
那个source或者sink就是入口或者出口这个地方
像merge的话它会去其他的pipeline去接收数据
把就是说把不同pipeline的数据合并成一个
比如说我们
最终最终输出接口的那个那个
Pipeline的它会就一定会有个MERGE操作
就是会把所有的那个Pipeline计算的结构给被合并成一个
然后当一个用户
那像group或者是说像分组或者排序的话
也会这这么一个merge操作啊
然后发送这边的话也会有有这么两个
一个collect的话就是一对一的发送
然后还有一个dispatch算子的话是
就是会一对多的发送
他们dispatch的话会有很多不同的模式吧
就说你会有广播的模式比如说比如说像那个hash表构建这里会有一个广播的模式
就是说假设S表是一个很小的表，他构建完hash表之后
他其实是会把hash表广播到这个啊
这张表在不同的pipeline出来去去做计算

那还会有一种dismax嘛就是做做shuffle
就是说
假如说我们
这个S表和R表都比较大的话
因为要做shuffle join
那么直接说的会通过一个下发的指标去算是把数据给
发送到那个不同的对应的一个pipeline上面去


通常虽然OLAP的本身
它从语义上来说的话是跟sql本身是没什么关系
但是我们现在的OLAP系统的话，它必须要就是要具备sql查询的能力

OlAP的话就说他分析性查询一般来说是比较复杂的这种这计算任务
就说你有一些sql能力是必须具备的，比如说像多表join
这样子查询像窗口函数，还有CTE 和Recursive CTE，或者是用户自定义函数这些不同的sql能力
然后目前的话Matrix One已经都具备这些能力

下面会简单介绍一下matrix one的优化器规则。
优化器规则一般来说是分成这么两个部分：
一个是减少IO的，减少IO的话就是有列裁剪，列裁剪的话就是说
我是去读一张表
比如说他这个表有很多列
但我实际上数据我只用到其中一列
其他的列是不用读取的一个很好理解的一个规则
还有还有是谓词下推
就是说下推的话比如说你
我们会把一些过滤条件给直接
推到读取数据这一部分
然后这样的话就是
呃尽量少的读取数据
还有一个叫做谓词推断
谓词推断的话我们会
他主要会影响这个TPC型里面的Q7和Q19
这个后面会在具体
哦会详细介绍一下
然后一个是runtime filter ，runtime filter就是说
就是我还是我们刚才做join的操作
比如说你大表和小表做join
但是小表构建完hash表之后
可能他的hash表的计数非常小
这样的话我们可以直接通过那个
hash表里面不同的词去
去大表里面通过runtime或者说元数据信息去进行去做这么一个过滤
这样的话
呃在运行时的时候就把要需要读取的这个大表的这个block
需要读取的这个大表的这个block数量
数量给大大的减小
那这也是减一个减少IO的一个优化规则
优化规则还有一部分就是说是减少计算的
就说他并不能减少你实际从磁盘里面读取的数据，但是他会在计算的过程中大大的减少计算量
或者说减少这个
中间结果的一个数据量
其中一个就是说join order的一个连接顺序
那连接顺序的话就说在就
通常我们做那个哎OLAP 的benchmark的话
用那个TPC起的话
它会影响很多的不同的查询
就说就用的做的不好的话
这些查询他都可能会以数量级的变慢
还有一个就是说一个聚合函数下推和上拉的操作

假设我们聚合函数在一个join的上面
我们先先做一个join再再去做聚合
然后再在join这里的话
可能他数据会膨胀的非常多
但是如果把把这个聚合函数
他如果可以假设可以推到join下面去做
就是说在做join之前的话那个数据就已经减少很多的话
这也是可以大大的减少这个计算。

简单介绍谓词推断，谓词推断就是说可能我们我们之前说谓词下推的话
那个是已经显示的可以下推的一个位置
但谓词推断呢可能是用户需要去做一些逻辑上的变化之后
才能得到一些新的谓词
这些新的谓词也可以下推下去
比如说TPC Q19的话它会
它的那个过滤条件是三个很长的这么一个谓词然后用or来连接起来
但实际上我们仔细观察的话
其实说他这三个or里面其实有很多共同的部分
咱们可以把它共同的部分给提取出来
提取出来之后就变成这个样子
那这样的话我们可以观察他
他首先他有
就是说part
这张表上面有一个可以下推的谓词
然后那个
lineitem这张表上面他也有两个可以下推谓词
这样的话
这两个位置他首先是可以下推到
每个表的table scan上面去
然后还多出多出来一个位置的话
这个Part和lineitem上面那个
用主键做连接的一个谓词
这样的话就说
如果
原来这个执行计划你不做优化的话直接去执行
他可能会先做一个笛卡尔积
再去做这么过滤这个
这个效率非常低的
那现在的话
我们可以把变成一个join操作

那在简单介绍一下我们mo使用的那个join order的一个算法吧
因为join order的话在算法上也没什么秘密
就是说各大开源数据库
大概也就是贪心法和那个动态规划吧
动态规划有很多有几种不同方法，也有很多论文，大家都可以看。
但说但是动态规划的话
就是他会一个问题说当你表的数量稍微多一点的话
这个状态搜索空间就会急剧的膨胀
可能是以指数的形式膨胀
就是说呃相比说Starrocks
它的文档里面写的话就说10个表以上的话就没办法使用动态规划来算
就没办法使用动态规划来算
就只能使用弹性算来算
呃但是那个就是在我们在
matrix one的话我们在这里做一个思考

就是说可能在呃即使大于10张表的话
可能有问题
也可以先用贪心的方法来来做这么一个
剪枝操作
就是让搜索空间大大的减小
然后再在贪心法之后再做这么一个动态规划
那么这个贪心法的话大概分这么三步吧
第一步就是说确定事实表和维度表
因为一般来说OlAP查询也好
这就是OlAP数据会啊
他通常会把表分成事实表和维度表
就是无论是新型的那个框架
还是说那个雪花框架
他都是会分事实表和维度表
那事实表维度表之间的话使用维度表的
主键去做这么一个join
那我们首先是确定呃
就是拿到一个查询之后
我们可以把事实表和维度表给找出来
那下一步的话是说事实表先有维度表先join成子树
因为事实表维度表之间
他始终是通过那个
维度表的主键去做join
所以说做这样的join的话他的
他的结果的函数始终是不大于他
本来输入的函数
所以做这么一个join的话他的
也就是并不会造成很多io开销
还
会造成这种数据膨胀的这么一个操作
所以说实话
这样的join先做是没有问题的
那么
呃那么
在做这个事实表维度表join的过程中的话
肯定我们会先考虑事实表会先与过滤性好的维度表做join


就优化器的话
首先就是越早的减少这个数据量是越好
所以说先先会以过滤性好为主
表现做决定
那最后一步的话就是这指数之间
指数之间就说比如说像TPC DS的话
你会有多个维度表
这维度表互相之间
都不是以主键来做决定
那么子树之间的话
我们再使用经典的设计均有的算法
就是说比如说像动态规划这些
这样的话就是说
我们把
把这个Join order的算法就是说
假设之前是你只能考只能10表以下做
做这个动态规划
我们现在可以把它扩展到10个
事实表之下都可以用动态规划来做

举个例子，TPS Q5的会有customer orders这么六张表，
它是一个比较典型的一个星型的结构。

然后这个orders和region这两个表是比较红的
因为它上面是有过滤条件的
这过滤条件呢会用需要单独考虑
然后的话是把那个join条件
他本身的join条件
包括给他画了一些标了一些线
像这有箭头的
这个实现的话就是说从事实表到维度表
他用维度表的主键做join，这么一个条件
大家可以看到，一共有5个条件都是用主键来做join
那么其中还有比较特殊的一个条件
用画的虚线就是他
两边互相是都不是用主键来做的join
嗯那么这我们这个join算法下一步的话
就是还会有一个联合会话
就是说最后就可以跟谓词推断联合起来使用的话会有一个新的优化。
因为我们可以看到supplier
customer这张表他会有一个join条件
然后supplier跟nation他也有一个join条件
用的都是同一个类型T来做在决定

我们就可以推探出来一个类型和
customer之间
他也可以有一个
以类型key来做交易的一个条件
因为我用黄色的选项表示
那么最后实际
生成的最优的join顺序的话

实际上是从region先跟nation join
然后再跟customer join
然后跟orders join
再和lineitem join
那最后再跟supplier join
就是说
我们会用上这个新推断出来的条件
那么为什么是走这么一个路线
而不是说先把上面这这一条路join完成后再join下面这边的表
呃因为因为我们考虑到order的时候
微信这两只表他都有过滤条件
那么两个都有过滤条件的表的话
他放在一起的过滤的效果会就会更加的好
这样会让lineitem这张表他的行数减少的那个速率更加的快




Matrix One从去年开始整个架构重构一直到现在差不多接近一年的时间。
可以看给大家看一下TPC-H 100G的性能测试的结果。
我们用的是亚马逊EC2上面的机器做的测试环境，用的是比较一般的机器。
每个机器16核和64G的内存啊
一共用了四台机器，每个机器16核和64G的内存啊
有三个计算节点CN，一个数据节点DN
啊可以看
大家可以看一下这边的这个benchmark结果
前两列的话是在Snowflake上面做测试的
就是Snowflake的一个呃
就价格跟我们这个配置差不多的一个配置
然后呢数据呃
就是跑第一遍的那个时间啊跑第二遍的时间
那后面是Matrix one的跑第一遍和
跑第二遍的时间
可以看到我们现在差不多能达到snowflake60%的性能。
为什么要跟snowflake比呢，是因为snowflake和我们的架构是差不多的，都是云原生，数据完全存算分离。
然后数据的话是单独存储，不会跟实际的计算节点在一起


这个OLAP数据库的话
大概做了这么一年时间吧
现在这个结果的话我觉得应该还是还是很不错，性能上达到了snowflake的60%的水平。


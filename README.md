# ElasticSearch
《ElasticSearch权威指南》读书笔记

### 基础入门 
1. 数据输入和输出
   1. 处理冲突
      1. 同时更新某条数据，如更新库存
         1. **悲观并发控制**：这种方法被关系型数据库广泛使用，它假定有变更冲突可能发生，因此阻塞访问资源以防止冲突。 一个典型的例子是读取一行数据之前先将其锁住，确保只有放置锁的线程能够对这行数据进行修改。
         2. **乐观并发控制**：Elasticsearch 中使用的这种方法假定冲突是不可能发生的，并且不会阻塞正在尝试的操作。 然而，如果源数据在读写当中被修改，更新将会失败。应用程序接下来将决定该如何解决冲突。 例如，可以重试更新、使用新的数据、或者将相关情况报告给用户。
            1. Elasticsearch 是分布式的。当文档创建、更新或删除时， 新版本的文档必须复制到集群中的其他节点。Elasticsearch 也是异步和并发的，这意味着这些复制请求被并行发送，并且到达目的地时也许 顺序是乱的 。 Elasticsearch 需要一种方法确保文档的旧版本不会覆盖新的版本。每个文档都有一个 _version （版本）号，当文档被修改时版本号递增。 Elasticsearch 使用这个 _version 号来确保变更以正确顺序得到执行。如果旧版本的文档在新版本之后到达，它可以被简单的忽略。
            2. 我们可以利用 _version 号来确保 应用中相互冲突的变更不会导致数据丢失。 获取数据时拿到Version，更新是带上Version，若此时库中Version不同会提示更新失败并返回当前Version。
2. 分布式文档存储
   1. 路由一个文档到一个分片中
      1. 路由算法：`shard = hash(routing) % number_of_primary_shards` 。routing 是一个可变值，默认是文档的 _id ，也可以设置成一个自定义的值。
   2. 取回一个文档
      1. 当客户端通过ID查询一个文档时，接收到请求的Master节点计算出当前文档所在分片，把请求发送到对应节点，对应节点返回数据给Master，Master再返回给客户端
      2. 在处理读取请求时，协调结点在每次请求的时候都会通过轮询所有的副本分片来达到负载均衡。
      3. 在文档被检索时，已经被索引的文档可能已经存在于主分片上但是还没有复制到副本分片。 在这种情况下，副本分片可能会报告文档不存在，但是主分片可能成功返回文档。 一旦索引请求成功返回给用户，文档在主分片和副本分片都是可用的。
   3. 局部更新文档
      1. 客户端向协调节点发送修改请求，协调节点将请求转发到主分片所在节点，主节点更新成功后，将文档并行转发到副本重新建立索引，所有副本返回成功，则向协调节点返回成功，协调节点向客户端返回成功
      >当主分片把更改转发到副本分片时， 它不会转发更新请求。 相反，它转发完整文档的新版本。请记住，这些更改将会异步转发到副本分片，并且不能保证它们以发送它们相同的顺序到达。 如果Elasticsearch仅转发更改请求，则可能以错误的顺序应用更改，导致得到损坏的文档。 
3. 搜索-最基本的工具
   1. 分页过深性能差
   >理解为什么深度分页是有问题的，我们可以假设在一个有 5 个主分片的索引中搜索。 当我们请求结果的第一页（结果从 1 到 10 ），每一个分片产生前 10 的结果，并且返回给 协调节点 ，协调节点对 50 个结果排序得到全部结果的前 10 个。现在假设我们请求第 1000 页—​结果从 10001 到 10010 。所有都以相同的方式工作除了每个分片不得不产生前10010个结果以外。 然后协调节点对全部 50050 个结果排序最后丢弃掉这些结果中的 50040 个结果。可以看到，在分布式系统中，对结果排序的成本随分页的深度成指数上升。这就是 web 搜索引擎对任何查询都不要返回超过 1000 个结果的原因。
4. 执行分布式检索
   1. 当一个搜索请求被发送到某个节点时，这个节点就变成了协调节点。 这个节点的任务是广播查询请求到所有相关分片并将它们的响应整合成全局排序后的结果集合，这个结果集合会返回给客户端。
   2. 分片返回一个轻量级的结果列表到协调节点，它仅包含文档 ID 集合以及任何排序需要用到的值，例如 _score 。
   3. 协调节点确定那些文档需要返回后，像相关分片提交多个GET请求拿到需要数据，组合后返回给客户端
   4. 深分页：深分页非常耗费CPU、内存和宽带，正常人一般就查看签名两三页，很少翻页到几百页后
   5. 游标查询Scroll适合使用再深分页上。游标查询会取某个时间点的快照数据。 查询初始化之后索引上的任何变化会被它忽略。 它通过保存旧的数据文件来实现这个特性，结果就像保留初始化时的索引 视图 一样。
5. 分片内部原理
   [参考链接](https://www.elastic.co/guide/cn/elasticsearch/guide/current/inside-a-shard.html)
   >**索引与分片的概念**：被混淆的概念是，一个 Lucene 索引 我们在 Elasticsearch 称作 分片 。 一个 Elasticsearch 索引 是分片的集合。 当 Elasticsearch 在索引中搜索的时候， 他发送查询到每一个属于索引的分片(Lucene 索引)，然后像 执行分布式检索 提到的那样，合并每个分片的结果到一个全局的结果集。
   1. 使文本可被搜索
      1. 倒排索引被写入磁盘后是**不可改变**的，永远不会修改。不变性有重要的价值：
         - 不需要锁。如果你从来不更新索引，你就不需要担心多进程同时修改数据的问题。
         - 一旦索引被读入内核的文件系统缓存，便会留在哪里，由于其不变性。只要文件系统缓存中还有足够的空间，那么大部分读请求会直接请求内存，而不会命中磁盘。这提供了很大的性能提升。
         - 其它缓存(像filter缓存)，在索引的生命周期内始终有效。它们不需要在每次数据改变时被重建，因为数据不会变化。
         - 写入单个大的倒排索引允许数据被压缩，减少磁盘 I/O 和 需要被缓存到内存的索引的使用量。
   2. 动态更新索引
      1. 对于文档的修改，是通过新增文档、修改索引来反应。再查询时新的文档和旧的文档都会被查询出来，但最终被合并后只剩最新文档
      2. 删除和更新
         >段是不可改变的，所以既不能从把文档从旧的段中移除，也不能修改旧的段来进行反映文档的更新。 取而代之的是，每个提交点会包含一个 .del 文件，文件中会列出这些被删除文档的段信息。

         >当一个文档被 “删除” 时，它实际上只是在 .del 文件中被 标记 删除。一个被标记删除的文档仍然可以被查询匹配到， 但它会在最终结果被返回前从结果集中移除。
         
         >文档更新也是类似的操作方式：当一个文档被更新时，旧版本文档被标记删除，文档的新版本被索引到一个新的段中。 可能两个版本的文档都会被一个查询匹配到，但被删除的那个旧版本文档在结果集返回前就已经被移除。 
   3. 近实时搜索
      1. Lucene 允许新段被写入和打开—​使其包含的文档在未进行一次完整提交时便对搜索可见。内存索引缓冲区的文档会被写入一个新段中，这些新段会放到缓存，这一步代价较低，稍后才会写入磁盘，这一步代价较高。只要文件在缓冲中，就可以像其他文件一样被打开和缓存。
      2. **Refresh API**:在 Elasticsearch 中，写入和打开一个新段的轻量的过程叫做 refresh 。 默认情况下每个分片会每秒自动刷新一次。这就是为什么我们说 Elasticsearch 是 近 实时搜索: 文档的变化并不是立即对搜索可见，但会在一秒之内变为可见。也可手动设置，每次更新数据触发一次Refresh,除非特殊需求不建议这样做，毕竟对性能有影响。使用ES，应该知道ES的缺点，并非实时搜索，而是近实时搜索。
   4. 持久化变更
      1. 如何保证持久化：Elasticsearch 增加了一个 **translog** ，或者叫事务日志，在每一次对 Elasticsearch 进行操作时均进行了日志记录。translog 提供所有还没有被刷到磁盘的操作的一个持久化纪录。每隔一段时间或translog越来越大，执行一次全量提交，提交成功后删除translog。当 Elasticsearch 启动的时候， 它会从磁盘中使用最后一个提交点去恢复已知的段，并且会重放 translog 中所有在最后一次提交后发生的变更操作。
      2. translog 的目的是保证操作不会丢失。
      3. translog 也被用来提供**实时CRUD**。当你试着通过ID查询、更新、删除一个文档，它会在尝试从相应的段中检索之前， 首先检查 translog 任何最近的变更。这意味着它总是能够实时地获取到文档的最新版本。
      4. 默认情况下，translog每30分钟或内存达到512M会执行flush。完成以下事情：
         - 所有在内存缓冲区的文档都被写入一个新的段。
         - 缓冲区被清空。
         - 一个提交点被写入硬盘。
         - 文件系统缓存通过 fsync 被刷新（flush）。
         - 老的 translog 被删除。
      5. translog设置：默认translog5秒同步一次到硬盘。默认情况下5秒内数据可能会丢失的。 如果index.translog.durability被设置为async的话，Elasticsearch每5秒钟同步并提交一次translog，5秒内如果服务器挂了，5秒内的数据会丢失。或者如果被设置为request（默认）的话，每次index，delete，update，bulk请求时就同步一次translog。更准确地说，如果设置为request, Elasticsearch只会在成功地在主分片和每个已分配的副本分片上fsync并提交translog之后，才会向客户端报告index、delete、update、bulk成功。
   5. 段合并
      1. 由于自动刷新流程每秒会创建一个新的段 ，这样会导致短时间内的段数量暴增。而段数目太多会带来较大的麻烦。 每一个段都会消耗文件句柄、内存和cpu运行周期。更重要的是，每个搜索请求都必须轮流检查每个段；所以段越多，搜索也就越慢。
      2. Elasticsearch通过在后台进行段合并来解决这个问题。小的段被合并到大的段，然后这些大的段再被合并到更大的段,而那些旧的索引段会被抛弃并从磁盘中删除。
### 深入搜索  
1. 结构化搜索
   1. 结构化搜索（Structured search） 是指有关探询那些具有内在结构数据的过程。比如日期、时间和数字都是结构化的：它们有精确的格式，我们可以对这些格式进行逻辑操作。比较常见的操作包括比较数字或时间的范围，或判定两个值的大小。
   2. 精确值查找
      1. 当进行精确值查找时，我们会使用过滤器（filters）,执行速度快，不会执行评分相关逻辑，易被缓存
         ```
         curl -X GET "localhost:9200/my_store/products/_search?pretty" -H 'Content-Type: application/json' -d'
         {
            "query" : {
               "constant_score" : { 
                     "filter" : {
                        "term" : { 
                           "price" : 20
                        }
                     }
               }
            }
         }
         ```
   3. 缓存。使用过滤器查询时，会涉及到缓存bitset，
      1. Elasticsearch 会基于使用频次自动缓存查询。如果一个非评分查询在最近的 256 次查询中被使用过（次数取决于查询类型），那么这个查询就会作为缓存的候选。但是，并不是所有的片段都能保证缓存 bitset 。只有那些文档数量超过 10,000 （或超过总文档数量的 3% )才会缓存 bitset 。因为小的片段可以很快的进行搜索和合并，这里缓存的意义不大。
      2. 一旦缓存了，非评分计算的 bitset 会一直驻留在缓存中直到它被剔除。剔除规则是基于 LRU 的：一旦缓存满了，最近最少使用的过滤器会被剔除。
 ### 聚合
 1. Doc Values And FieldData
    1. Doc Values
    2. 深入理解Doc Values
       1. Doc Values 是在索引时与 倒排索引 同时生成。也就是说 Doc Values 和 倒排索引 一样，基于 Segement 生成并且是不可变的。同时 Doc Values 和 倒排索引 一样序列化到磁盘，这样对性能和扩展性有很大帮助。
       2. Doc Values利用操作系统内存而不是JVM Heap，所以针对聚合较多的使用场景JVM Heap可以分配较少，给操作系统预留多一些

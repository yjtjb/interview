# * redis
## 为什么使用redis？
在做美食点评项目的时候遇到了瓶颈，在秒杀订单的业务中我使用Jmeter进行高并发测试时数据库基本上扛不住了，这时候就需要缓存中间件来加入，现在比较常用的中间件有redis和memcached,并且redis支持更丰富的数据类型，能持久化，支持集群模式、发布订阅模型、lua脚本、事务等功能比memcached更加适合这个项目的业务逻辑，所以最后选择了redis
## redis有哪些数据结构以及使用场景是什么？
redis基本的数据结构有 String List Hash set Zset
String 使用SDS实现，**可以储存二进制数据和文本数据**，用一个len来记录字符串长度所以获取字符串长度的时间复杂度为O(1),而C语言m没有储存数组的长度需要遍历O(n)，并且**SDS解决了缓冲区溢出的问题**，在每次凭借字符串时候会检查剩下的长度够不够，如果不够就进行自动扩容，所以不会产生溢出。字符串对象有三种内部编码，int，raw，embstr，redisobject由type，编码，ptr指针组成，如果字符串保存的是int，就会讲int转化成long放在ptr中，将编码设置为int，如果字符串比较短会使用embstr，直接分配一块连续的地址保存redisobject和sds，如果字符串比较长会使用raw编码，在内存中开辟一块新的地址保存sds，并把ptr指针指向sds。embstr的编码模式因为是分配在连续内存中的所以如果长度增加重新分配的成本会很高，所以embstr编码的字符串对象是**只读**的，如果要进行修改会把embstr变成raw再进行增加长度。
使用场景：redis是单线程的所以执行命令的过程有原子性用来进行**常规计数**（点赞数，访问，转发）、**用setNX实现分布式锁**如果key不存在返回true表示获得锁成功，如果key存在返回false获得锁失败，**共享session信息**不同的服务器单独储存session会出现重复登录等问题，用redis对session信息进行统一的储存和管理。

List由**双向链表**或**压缩列表**实现，比较小的列表用**压缩列表**，压缩列表是连续内存块组成的顺序性数据结构，表头由zlbytes占用内存字节数，zltail偏移量，zllen节点数量组成，中间就是由表节点组成，最后用zlend当作结束符255,表节点由prevlen前一个节点长度，encoding编码这个节点数据类型长度，和data数据组成，表节点会根据数据的类型和大小选择不同的空间编码分配内存空间，如果前一个节点长度小于254，prevlen用1个字节储存，大于254用5个字节，如果数据是整数用1个字节进行编码，如果是字符串根据大小用1/2/5字节的空间进行编码，压缩列表在查询表头表尾的元素只需要O(1)的时间，查询其他节点地方需要O(n)复杂度，并且在新增某个元素或者修改某个元素的时候可能会产生**连锁更新**的问题（就比如一堆253长度的节点，突然头部插入一个255长度的节点，所有节点都要更新prevlen），所以只能用于储存节点数量不多的场景。最新的redis用**listpack**代替压缩列表和压缩列表不同的是列表头只有总字节数，和元素数量，而表结点中只舍弃了prevlen，只记录当前节点长度就不会产生连锁更新的问题**双向链表**比普通的链表添加了指向链表头和链表尾的指针，并且添加了链表节点数量len，并且用指针保存节点值增加了节点值复制、释放、比较函数可以保存各种不同类型的值。双向链表**内存开销大**，并且**地址不连续**。redis5.0之后使用quicklist来进行链表的实现，quicklist可以理解为多个压缩列表的列表，每个quicklist的节点都指向一个压缩列表，如果新增数据会检查当前节点的压缩列表能不能容纳，如果不能容纳就新增一个新的节点。
使用场景：消息队列，list因为是先进先出的数据结构可以用Lpush、rpop保证消息的顺序到达，通过设计全局唯一ID并在消费者地方记录处理过的id保证不处理重复消息，并且可以用BRPOPLPUSH命令进行备份的list记录保证消息可靠性。

Hash是用健值对集合，其中比较小的hash是用**压缩列表**组成健和值分成两个节点相连的保存在一起，如果比较大用**哈希表**进行实现，**哈希表**能以O(1)的复杂度快速查询数据，采用链式哈希解决哈希冲突，如果哈希表数据过大会进行渐进rehash重建，每个哈希结构包含两个哈希表，刚开始只对hash表1进行操作，当出发rehash的时候，哈希表2的长度为哈希表1的2倍，每次对哈希表进行增删改查操作时候会同时对哈希表2也进行相同的操作，同时会顺序的把哈希表1的数据迁移到哈希表2上，在新增数据时只对哈希表2进行操作，在rehash结束之后会把哈希表2改成哈希表1，哈希表1变成哈希表2，rehash的触发条件为负载因子为1时候，没有持久化命令时进行，或者负载因子超过5强制执行。
使用场景：一般用于储存对象，例如购物车等。

set就是唯一的健值集合，是由哈希表或整数集合实现的，如果数量比较小使用整数集合，由编码方式，元素数量和整数集合组成，当新加入的元素类型大于集合中原有的元素类型时，会进行整数集合升级，先计算出需要多分配的空间，从后向前把原来的元素一一升级，哈希表就是采用value=null的特殊哈希表。
应用场景：用来做只能出现一次的场景，比如共同关注，抽奖活动，点赞等。

Zset比set多了一个权重，是一个有序的set，是由压缩列表和跳表实现的，zset中有哈希表和跳表保证单点查询和范围查询，跳表头节点有多个层级的头指针，分别指向不同层级的节点，而每个跳表的节点包含指向头尾的指针，跳表的长度和跳表的最大层数，并包含一个对应的层数，和对应每层指向下一个相同层数的指针，跳表的查询首先从最高层的开始遍历找到离权重最近的节点，再从下一层进行遍历，直到找到和权重相等的节点，然后在进行储存的SDS类型数据的比较，如果该节点储存的SDS数据类型小于要查找的数据则访问该层的下一个节点，反之则继续下沉。，跳表比平衡树的优点就是从内存占用上比较跳表的平均使用指针数比平衡树要少，做范围查找时跳表只要找到对应的值然后顺序遍历，而平衡书还需要继续中序遍历，并且平衡树还需要实现插入删除的左旋右旋，比跳表实现更麻烦。
应用场景：主要是用在需要进行排序的场景，比如排行榜、姓名电话排序。

同时redis还有新增的数据比如BitMap、HyperLogLog、GEO、Stream还有一些redis-module模块比如布隆过滤器（找到redismoudle.h头文件，实现Onload方法）

bitmap就是位图一串连续的1或0数组，用String实现
应用场景：用来进行只有两种状态场景的储存，比如签到统计。

HyperLogLog主要用来统计一个集合中不重读的元素个数，不是百分百准确，可以合并，所占内存特别小只要12kb
应用场景：在百万级的数据统计上很有用

GEO:主要用来储存地理位置信息，用来实现搜索附近地区的需求
应用场景：附近搜索

Stream:专门为消息队列设计的数据类型，可以完美的实现消息队列，支持消息的持久化，支持消费组模式等功能，让消息队列更加稳定可靠。

## redis的线程模型是什么？
首先redis的单线程指的是 接受客户端请求，解析请求，进行数据读写，发送数据给客户端这个过程是单线程完成的，这个也叫redis的文件事件处理器，它采用I/O多路复用机制同时监听多个socket，将产生事件的socket加入到内存队列中，用事件分派器根据socket上的事件类型来选择对应的事件处理器进行处理。而后来的redis4.0之后增加了三个后台线程异步处理进行关闭文件、AOF刷盘、释放redis内存业务，这三个后台线程不断轮询对应的任务队列拿到任务就去执行对应的方法，这三个任务比较消耗时间。
## redis为什么采用单线程模型？
因为redis的大部分操作都在内存中完成处理速度快，而限制redis速度的并不是cpu而是网络I/O，既然限制不是cpu就可以用单线程进行处理，这样可以避免多线程之间的竞争以及多线程切换带来的时间和性能上的开销，并且redis是使用**I/O复用机制**处理大量客户端的socket请求，这使得一个redis线程能处理多个I/O流。
## 为什么redis后来引入的多线程？
因为随着网络硬件性能的提升，redis的瓶颈主要出在网络I/O的处理上，所以要采用多线程的方式处理网络I/O，但对于命令的执行仍然采用的是单线程。
## redis为什么这么快？
redis是单进程单线程的非关系性数据库，大部分的操作都是在**内存**上实现的，而mysql要根据索引进行多次的磁盘IO，**数据结构简单并且为了加快速度专门设计**，采用的单线程避免了不必要的上下文切换成本和资源竞争，也不存在进行多进程的切换消耗的cpu，也不用考虑各种锁的问题以及加锁操作，使用了多路复用I/O模型而不是阻塞IO，并且Redis自己实现了一个解析快速的自定义协议进行通信。
### redis如何实现持久化？
主要通过AOF日志和RDB快照，






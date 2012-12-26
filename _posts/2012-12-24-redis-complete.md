---
layout: post
title: redis源码阅读
category: linux
tags: [redis, nosql,  linux]
---

阅读的代码版本为Redis 2.6.7,只关注了其中重点的部分，对于很多细节性的代码进行了忽略
<https://github.com/antirez/redis>  
代码行数:  

    find src/ -name "*.[c|h]" |xargs wc -l
    37216 total

* 
{:toc}

Binary safe
------------

<http://en.wikipedia.org/wiki/Binary-safe>

二进制安全是一个针对字符串处理函数的计算机编程术语，一个二进制安全函数的最基本要求就是它把输入完全当作未处理的无任何格式要求的数据流。
假定8个比特位的字符，那么二进制安全函数就必须能够处理所有可能的256个值。


Redis协议
----------

<http://redis.io/topics/protocol>

请求以及应答都是通过\r\n结束  
请求协议:  

    *<参数数量> CRLF $<第一个参数字节数> CRLF <参数数据> CRLF ... $<第N个参数字节数> CRLF <参数数据> CRLF

应答协议:

1. 单行应答：使用+开始，后面跟应答字符串，\r\n结尾
2. 错误应答：使用-开始
3. 整数应答：使用:开始
4. Bulk应答：使用$开始 $<datalen>\r\n<data>\r\n (GET)
5. 批量应答: 使用与请求协议相同的格式

旧的协议：  
key不是二进制安全的，协议中通过空白字符以及回车换号符号进行分隔，key为不包含空格或者\n符号的字符串

1. inline command:     通过空白分隔请求参数
2. bulk command:     最后一个参数为字节数量，为二进制安全

Config
------

启动的时候可以指定配置文件，配置文件支持include指令。
配置可以热更新，可以通过CONFIG GET/SET指定进行配置项的获取以及更新

启动参数：

* --test-memory memtest 内存测试
* --sentinel     支持sentinel模式
<http://antirez.com/news/43>

启动：

1. Generate Redis "Run ID" (SHA1-sized random number) 唯一确认redis实例，用于作replication
通过/dev/urandom生成随机数或者通过time/pid进行xor生成随机数
<http://en.wikipedia.org/wiki//dev/random>
2. 进行配置读取
3. 通过initServer进行初始化，注册信号处理函数，创建共享对象 createSharedObject，开启端口并添加到事件循环
4. scriptingInit，slowlogInit, bioInit
...

DataStruct
-----------

####*hashtable*

特性：1）支持自动expand      2）支持迭代

使用结构dict表示，其中对于两个dictht，主要用于在expand的0/1切换。使用dictType抽象出不同类型的hashtable类型。dictht是只要的hashtable数据结构，使用链地址法解决冲突，每个项存储在dictEntry结构里。

dictAdd:

1. 通过dictAddRaw添加key到hashtable返回dictEntry,然后根据value类型赋值
2. dictAddRaw首先判断是否正在rehash，否则执行一步rehash（无迭代器正在迭代）
     通过通过_dictKeyIndex返回插入的slot的位置，分配dictEntry插入到指定位置
3. _dictKeyIndex首先通过_dictExpandIfNeeded进行必要的expand，初始分配的容量为DICT_HT_INITIAL_SIZE(4).  
   然后通过key的hash值判断是否存在，存在则返回-1，不存在则返回空闲slot的index (如果在rehash则返回第二个hashtable的空闲slot)

rehash:  
只有在dict_can_resize以及elements/buckets >5的时候才开始扩容，在add/find/delete 以及iterator为0 的时候都会进行rehash. 如果配置文件里配置了activerehashing，则会进行增量rehash 1ms.

iterator:  
通过dictGetIterator获取迭代器，然后通过dictNext获取下一个元素。通过dictGetSafeIterator获取的迭代其保证在迭代的过程中不进行rehash.

hash function:  
key为sds，使用hash函数为MurmurHash2


####*sds (string datastruct)*  
sds结构其实就是char *，字符串函数都可以使用。只是在分配字符串前添加一个字符串前缀记录其使用的大小(len)以及空闲的大小(free)，则其整个结构的大小为使用的空间+空闲的空间+8

...

Network/EventLoop
-------------------

<http://redis.io/topics/internals-rediseventlib>

Eventloop支持不同的平台，对于linux使用的是epoll实现。只要支持两种事件（File以及Timer）  
实现的比较简单，注意的是其timer通过链表实现，对于redis还好，其只使用了一个timer，所有的异步操作都在这个timer里完成。

可以设置beforesleep，在每次处理时间之前进行的操作。redis在1s内调用REDIS_HZ次serverCron方法,serverCron处理很多异步操作。时间控制通过REDIS_HZ配置  
<https://github.com/antirez/redis/commit/94343492361a04301a48fc56490d6113ff97aba9>

* accept  
redis对于通过`acceptTcpHandler`或者`acceptUnixHandler`处理客户端连接事件，对于获取每个连接建立一个`redisClient`对象，其通过`readQueryFromClient`读取数据

* receive
1. readQueuryFromClient每次尝试读取`REDIS_IOBUF_LEN`（1024 *16）的数据到querybuf，然后调用processInputBuffer处理数据。
2. processInputBuffer通过数据的第一个字节判断是否为*判断为Multibulk协议还是inline协议。inline协议通过processInlineBuffer进行协议解析，multibulk协议通过processMultibulkBuffer进行协议解析。  
3. 解析成功后调用processCommand执行命令。如果buffer里有数据，并能解析执行成功，则一直处理。


* reply  
通过addReply相关方法发送应答给client。通过client-output-buffer-limit可以限制output buffer的大小

* addReply
1. 通过prepareClientToWrite设置client的写的时间处理函数为aendReplyToClient
2. 避免COW，则优先通过_addReplyBuffer添加到buffer里，其次通过_addReplyObjectToList添加到list里
3. _addReplyToBuffer判断是否在reply链表上有数据以及可用空间是否足够存放，满足则将数据memcpy到buf里
4. _addReplyObjectToList将output分为chunk，chunk大小为16k。如果list后面最后一个chunk可以存放则存放，否则将robj添加到reply链表尾
5. 通过asyncCloseClientOnOutputBufferLimitReached判断是否output buffer限制达到。如果达到则添加到clients_to_close里，在cron通过freeClientsInAsyncFreeQueue释放
6. sendReplyToClient为了防止其他连接饥渴，在一次事件循环里最多发送64k数据


* cron  
serverCron周期性调用clientsCron用于处理客户端相关的异步操作  

1. 计算每次迭代的客户端数量，每次处理1/(REDIS_HZ * 10)条连接，最少50条连接
2. 通过listRotate交替head与tail，处理head client 以及clientsCronResize
3. 通过clientsCronHandlerTimeout处理客户端最大超时时间
4. 通过clientsCronResizeQueryBuffer处理客户端的querybuf的大小。
     如果querybuf > 32k以及为querybuf_peak的两倍或者buffer > 1k以及空闲超过2s则通过sdsRemoveFreeSpace释放空闲的空间

LRU
----
<http://antirez.com/post/redis-as-LRU-cache.html>

通过maxmemory设置最大使用的内存，通过maxmemory-policy设置踢数据的策略，支持

1. volatile-lru            踢除有过期设置的数据通过lru算法
2. volatile-random     随机踢除有过期设置的数据
3. volatile-ttl             剔除最快需要过期的数据  
4. allkeys-lru             所有的key通过lru算法
5. allkeys-random     随机踢除所有的key
6. noeviction             不踢除数据，返回客户端错误

redis的lru以及ttl算法不是精确，是通过取样判断需要踢除的数据，可以通过maxmemory-samples设置取样的大小。

在`processCommand`里，如果设置了maxmory，则通过freeMemoryIfNeeded释放可以释放的内存

1. 减去slave以及aof使用的buf，判断是否超过了内存限制
2. 计算需要释放的内存量，迭代所有的db，根据算法选择合适的key进行释放
     volatile-random/allkeys-random随机选择key，volatile-lru/allkeys-lru在随机选择maxmemory-samples个数据，选择空闲时间的最大的key  
     volatile-ttl类似，在samples个数据里选择最快过期的数据

*time*  
为了减少time调用，对于时间准确性不高的地方通过server.unixtime获取系统时间，其在serverCron每10ms更新一次

redis为了节省内存，robj的lru通过22位保存，精度通过REDIS_LRU_CLOCK_RESOLUTION控制，默认为10s.在serverCron通过updateLRUClock进行更新。

    void updateLRUClock(void) {
        server.lruclock = (server.unixtime/REDIS_LRU_CLOCK_RESOLUTION) &
                                                    REDIS_LRU_CLOCK_MAX;                                                                                                                                               
    }

PUB/SUB
--------
<http://redis.io/topics/pubsub>

发布者发布消息到指定的channel，不关注订阅者。订阅者订阅channel，并接收数据。  
订阅者进入SUB模式后，只能使用(P)SUBSCRIBE / (P)UNSUBSCRIBE /QUIT命令，而且支持pattern-model （glob）的订阅方式。注意：如果pattern sub符合相同的channel多次，则该channel上的消息会接收多次。

glob可以参考：  
<http://en.wikipedia.org/wiki/Glob_(programming)>

publish返回接收的client的数量，subscribe通过multi-bulk reply返回，格式为<message kind><channel name><channel number>。注意subscribe/unsubscribe针对每个channel返回应答。

对于client端redisClient 通过dict pubsub_channel保存订阅的channel，通过list  pubsub_pattern保存订阅的pattern模式  
对于server端redisServer通过dict pubsub_channel保存订阅的client，通过list pubsub_pattern保存订阅的pattern模式

* subscribe  
通过pubsubSubscribeChannel处理subscribe命令  
1、迭代client发送的channel，将其加入到client的pubsub_channel里。将client添加到server的pubsub_channel里对应channel的client链表上. 复杂度为O(1)

* unsubscribe  
如果无参数则调用pubsubUnsubscribeAllChannels取消所有的订阅，否则迭代调用pubsubUnsubscribeChannel取消指定的订阅。过程与subscribe类似，先删除client pubsub_channel的值，然后删除server pubsub_channel指定channel 对于的client链表上的值. 复杂度为O(n)，对于Channel的client的数量

* psubscribe  
通过pubsubSubscribePattern处理psubscribe命令
1、通过pubsubPattern保存pattern以及其对于的client，添加到server pubsub_patterns链表

* publish   
通过pubsubPublishMessage处理publish命令
1. 通过server pubsub_channels获取指定channel的client list. 迭代所有的client发送应答
2. 迭代server pubsub_patterns链表，模式匹配pattern，匹配成功的发送到client

punsubscribe 与unsubscribe类似

Transcation
------------
<http://redis.io/topics/transactions>  
<https://groups.google.com/forum/?fromgroups=#!topic/redis-db/a4zK2k1Lefo>

简单的事务，保证一个client发起的命令可以连续执行，中间不会插入其他client的命令。而且能保证命令要么全部执行要么全部回滚。  
*注意：事务只能保证执行的时候顺序执行，而不能保证中间的某个命令执行错误而回滚。*

使用multi命令进入transaction上下文，所有的command进入队列，使用discard放弃所有入队的command  
通过Watch命令实现了类似CAS的乐观锁。如果Watch的key在事务执行前被修改了则整个事务被回滚，Exec返回nil reply(\*-1)。当事务结束后所有被Watch的key会被UnWatch.

redis script执行的都是事务性的，可以使用script处理事务

* Watch
1. 迭代通过watchForKey保存watchedKey。获取在db的dict watched_keys（key修改后通知所有的client）上对于key的client list，将当前client添加到client list，在当前client的watched_keys链表添加watch key.

* Multi
1. 处理很简单，在client的flag上添加REDIS_MULTI进入事务上下文。在processCommand的时间判断是否进入事务上下文，否则通过queueMultiCommand将命令保存在client的mstate里

* Exec
1. 在执行前判断是否watchedKey是否已经修改或者queue命令的时候是否失败。修改key的时候通过touchWatchedKey，通知watch该key的所有client REDIS_DIRTY_CAS.或者在queue命令失败的时候。通过flagTransaction通知client REDIS_DIRTY_EXEC
2. 在执行前发送MULTI命令到AOF以及Slave，然后循环执行queue里的命令

* Discard  
1. 通过discardTransaction释放掉queue的命令以及unwatch所有的key

Persistence
------------

<http://redis.io/topics/persistence>  
<http://oldblog.antirez.com/post/redis-persistence-demystified.html>  
<http://code.google.com/p/redis/issues/detail?id=150>

一般的write可以简化为下面几个步骤:

1. database write data to kernel buffer     (write)
2. kernel flush data to disk control     (默认30s,一般通过fsync等可以强制内核写到disk control)
3. disk control write to physical media （不可控）

数据恢复可以使用如下几种方式:  

1. 使用Replica恢复
2. 使用log恢复
3. 使用AOF,不会发生冲突

####Snapshot  
<https://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-RDB-Dump-File-Format>

数据库某一时刻的镜像，根据save point将其dump到磁盘。某一段时间的数据可能丢失，而且不能增量dump。  
可以通过bgsave以及save命令启动Snapshot。

相关配置：

    rdb filename: rdb保存的文件名
    rdb compression: 是否启用LZF压缩
    rdb checksum: 是否使用checksum

相关流程：

1. 在serverCron里根据saveparamslen调用rdbSaveBackground dump数据，fork一个子进程调用rdbSave dump数据。
2. 根据进程名称新建临时文件，并写入magic number(REDIS .. REDIS_RDB_VERSION  9bytes)
3. 迭代db，先写入select db的OpCode (REDIS_RDB_OPCODE_SELECTDB 254  1bytes)，OpCode不进行encoding, 原始写入。然后写入db的编号（根据rdbSaveLen编码，(0-16) 1bytes）
4. 迭代dict里所有的key,调用rdbSaveKeyValuePair存储kv(expire time,type,key,value)

        [REDIS_RDB_OPCODE_EXPIRETIME_MS 252][ExpireTime 8bytes][Value Type][Key][Value]
    
        其中存储数字以及OpCode调用rdbSaveLen进行存储:
        1、存储长度与ziplist类似，根据MSB的不同编码，不同数量bit存储
        00|000000                    表示6位存储长度
        01|000000 00000000           表示14位存储长度
        10|000000 [32bit]            表示32位存储长度，数据需要转换为big endian存储
        11|000000                    后6位指定编码方式
        
        [Key]:
        int:     [encoding type 1bytes][data]
        string:  [string len][data]
        其中保存key的时候，如果小于INTMAX，数字类型按照区间设置不同的编码（8/16/32位)，否则按照字符串进行编码
        如果字符串长度>20并启动压缩则LZF压缩后存储
        
        [Value]: (rdbSaveObject)
        
        List/Set/Hash: 如果为Encoded类型(例如ziplist，intset等，具体可以参考DataType)则直接保存原始数据，否则迭代保存
    
5. 保存（REDIS_RDB_OPCOFE_EOF 255）标志结束，如果开启checksum则写入cksum(big endian)存储。然后fsync到OS buffer,rename到rdb filename.

Example:

保存key1,value1通过save命令保存，然后通过

    hexdump -C dump.rdb 显示如下:
    
    00000000  52 45 44 49 53 30 30 30  36 fe 00 00 04 6b 65 79  |REDIS0006....key|
    00000010  31 06 76 61 6c 75 65 31  ff 38 c9 db e7 ad 48 42  |1.value1.8....HB|
    00000020  2c                                                |,|

* 前9个字符为Magic Number，标识RDB文件版本号。  
* fe为Select DB操作码 
* 00为DB Num. 
* 00为Value的类型，表示为字符串类型。  
* 04为key1的长度   
* 6b 65 79 31为key1   
* 06为value1的长度   
* 76 61 6c 75 65 31为value1   
* ff为结束符号   
* 38 c9 db e7 ad 48 42 2c为checksum    

总大小为33个字节

通过dump命令可以导出指定key的值，其记录格式与rdb类似

####AOF

记录所有更改数据库记录的命令，记录方式和客户端发送的类似，可能会改变命令，例如incr变为set。随着AOF的增加，redis通过rewrite缩小AOF的大小，rewrite fork新进程根据内存里的数据建立临时的aof文件。

可以通过bgrewriteaof重建AOF,serverCron调用rewriteAppendOnlyFileBackground()方法执行rewrite。

启动的时候通过bioInit启动后台线程，使用bio处理文件删除，unlink/aof操作可能会阻塞当前进程，开启线程处理job queue，现在提供两种job （`REDIS_BIO_CLOSE_FILE`以及`REDIS_BIO_AOF_FSYNC`）（libeio提供异步接口）

控制命令：  
config appendonly no|yes  
bgrewriteaof  

*feed*  
与replication类似，在执行command的时候调用call()，然后调用feedAppendOnlyFile将命令添加到aof_buf.  
在rewrite的时候，也需要将其加入到aof_rewrite_buf.  

aof_rewrite_buf与aof_buf的实现不同，rewrite buf每次分配一定大小（10M）的block，通过链表连接起来，而不是通过append buf来实现。其由于耗时比较长，realloc的开销比较大。

1. 如果db id不同，则增加一条select <id>的语句
2. 进行命令转换，例如将Expire相关命令转换为PExpireat，超时时间总是绝对的，不能保存相对时间
3. 将命令append到aof_buf，如果开启rewrite则也需要append到rewrite_buf

*flush*

1. 在每次事件循环之前都会调用beforeSleep，其会调用flushAppendOnlyFile()
2. 如果在flush启动的时候上次flush依然未结束，则延迟write，设置aof_flush_postponed_start并返回(flush的时候会阻塞当前write,最多延迟2s)。  
否则single write server.aof_buf的数据到aof文件。如果设置了aof_no_fsync_on_rewrite，则在rewrite的时候不调用fsync。如果是AOF_FSYNC_EVERYSEC则调用bio线程处理fsync，否则直接在当前线程调用fsync.

*rewrite*

1. fork后台进程调用rewriteAppendOnlyFile 重写aof文件到临时文件，主进程禁止dict resize，防止COW大量的页拷贝
2. 根据db size进行迭代所有的db，根据db dict所有object类型dump command到文件，类似于dump sql.为了缩小命令数量，大量使用RPUSH/SADD/ZADD命令。迭代结束后会调用fsync到磁盘。
3. 主进程在serverCron里会check 后台进程是否结束，成功完成后调用backgroundRewriteDoneHandler处理善后
4. 将rewrite 期间的buf输出到文件。rename临时dump的文件为aof filename，注意的是关闭旧文件的逻辑，rename后关闭最后一个引用的fd将导致unlink，会阻塞当前进程，通过bio去处理

*load*

1. 在启动的时候会调用loadDataFromDisk从磁盘加载数据，如果不是sentinel模式，而且开启了aof则调用loadAppendOnlyFile加载aof文件，否则调用rdbLoad加载rdb文件。
2. 创建一个fackeClient顺序执行aof文件的命令，每执行1000条命令则执行事件循环，用于处理在loading期间能执行的命令。loading数据期间能执行的命令有：info、sub/pub相关命令

LOG
---------
*slowlog*  
记录慢日志，可以通过slowlog-log-slower-than以及slowlog-max-len配置slowlog的执行时间的下限以及记录的数量

    slowlog get <num> （返回指定条数的slowlog，包含unique id/log time/execute time/command args）
    slowlog len               (获取当前slowlog的数量）
    slowlog reset            (清空slowlog)

支持logfile以及syslog

DEBUG
----------

<http://redis.io/topics/latency>

可以通过Sofeware watchdog 分析延迟，主要是通过SIGALARM信号进行处理，timer超期后通过ucontext打印backtrace

DataType
--------------
<http://redis.io/topics/data-types>

DB默认为16个，通过redisDB定义

    typedef struct redisDb {
        dict *dict;                 /* The keyspace for this DB */                                                                                                                                                     
        dict *expires;              /* Timeout of keys with a timeout set */
        dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP) */
        dict *ready_keys;           /* Blocked keys that received a PUSH */
        dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
        int id;
    } redisDb;

dict为keyspace，key为sds string, value为redis object。dictType为dbDictType。所有的数据都保存在dict里.
expires为有过期时间的keyspace

    typedef struct redisObject {
        unsigned type:4;            （数据类型，
        unsigned notused:2;     /* Not used */
        unsigned encoding:4;       （编码方式，ENCODING_RAW|INT
        unsigned lru:22;        /* lru time (relative to server.lruclock) */ 精度为分钟级别
        int refcount;           (引用计数)
        void *ptr;               (引用的数据)
    } robj;    

redis对象用于保存string/list/set的值，sharedObjectsStruct 用于共享的redis对象，减少内存消耗以及加速读取

注意的是:

* 如type是REDIS_STRING类型的,则其值如果是数字,就可以编码成 REDIS_ENCODING_INT,以节约内存.
* 如type是REDIS_HASH类型的,如果其 entry 小于配置值: hash-max-ziplist-entries 或 value字符串的长度小于 hash-max-ziplist-value, 则可以编码成 REDIS_ENCODING_ZIPLIST 类型存储,以节约内存. 否则采用 Dict 来存储.
* 如type是REDIS_LIST类型的,如果其 entry 小于配置值: list-max-ziplist-entries 或 value字符串的长度小于 list-max-ziplist-value, 则可以编码成 REDIS_ENCODING_ZIPLIST 类型存储,以节约内存; 否则采用 REDIS_ENCODING_LINKEDLIST 来存储.
* 如type是REDIS_SET类型的,如果其值可以表示成数字类型且 entry 小于配置值set-max-intset-entries, 则可以编码成 REDIS_ENCODING_INTSET 类型存储,以节约内存; 否则采用 Dict类型来存储.
* 如type是REDIS_ZSET类型的,如果其 entry 小于配置值zset-max-ziplist-entries或 value字符串的长度小于zset-max-ziplist-value , 则可以编码成 REDIS_ENCODING_ZIPLIST 类型存储,以节约内存; 否则采用类型来存储.REDIS_ENCODING_SKIPLIST 


Encoding
--------------
<http://redis.io/topics/memory-optimization>

通过object命令可以查看robj的状态，例如编码方式，其有raw/int/hashtable/linkedlist/ziplist/intset/skiplist

####Ziplist

使用字符串表示双向链表，可以用于存储string以及integer，内存使用效率更高

*整体结构*

    <zlbytes><zltail><zlen><entry><entry><zlend>
    
* zlbytes:uint32_t       ziplist 分配的字节数
* zltail: uint32_t       最后一个entry的偏移量，类似tail指针
* zlen:   uint16_t       entry的数量
* zlend:  char           list结尾符，等于255
* entry:                 每个节点
    
其中integer通过little endian存储
    
*Entry结构*

    [previous entry length][encode and type][data]
    
* prevlen 的存储
如果 prevlen 小于 ZIP_BIGLEN(254),则用一个字节来保存; 否则需要 5 个字节来保存,第 1 个字节存 ZIP_BIGLEN,作为标识符.
    
* encode and type(length):
    
前两个位区分为string还是int，11为int类型，其余为string类型

string根据不同的长度占用不同的字节数

    00pppppp              (占用一个字节，长度为6位）
    01pppppp|qqqqqqqq     (占用两个字节，长度为14位）
    10----------|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt     (占用5个字节，长度为32位）

int类型都只占用一个字节,可以编码为不同的不同的长度（2、4、8、3、1字节，以及0000-1101 4位符号，用于表示0-12）
    
*Function*

    unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where);
    在ziplist的尾端或头部添加一个结点  
    zl是ziplist的指针  
    s是待添加结点的值  
    slen是待添加结点的值长度  
    返回最新的ziplist的指针


每次操作都需要realloc大小，对于数据量太大就不太适合了,相比adlist主要节省在：

1. 对于存储dual link list，不需要存储prev,next指针，在64位系统需要16bytes，现在一般情况下只需要1byte
2. 对于存储integer类型比较节省空间

####Set

无序集合列表，通过sinterGenericCommand可以获取集合的交集

1. 先判断是否传递的参数都是set类型以及是否存在，如果有个不存在则返回empty bulk(\*0)
2. 然后集合元素的数量对sets进行升序排列，然后迭代第一个set的所有的元素，判断是否在其他的set存在

intset

    typedef struct intset {
        uint32_t encoding;
        uint32_t length;
        int8_t contents[];
    } intset;

设计比较简单，对于int类型进行定长升序存储，encoding有int16/32/64三种，读取的时候根据encoding设定步长。数字都是小端存储

*intsetadd*   
1、根据插入的值判断encoding，如果大于当前encoding则需要调用intsetUpgradeAndAdd进行升级。否则判断是否已经存在，存在则返回，否则将数据按照升序插入到合适的位置。  
   根据encoding resize当前的容量大小，然后从后到前的重新设置已经插入的值到当前encoding.

####Zset
有序集合列表，按照指定的score排序。如果通过skiplist编码，则通过一个dict保持element到score的映射以及通过一个skiplist保存score到element的映射

*skiplist*

<http://www.dirinfo.unsl.edu.ar/eda/EsDaAl/teorias/skip2-11.pdf>  
<http://blog.nosqlfan.com/html/3041.html>  
<http://rdc.taobao.com/blog/cs/?p=1327>  

特性：

1. 为probabilistic  balancing算法而不是类似平衡树，强制进行自平衡。插入的时候根据一定概率随机选择层数
2. 每层都是一个有序链表，level1有所有的key
3. key X如果存在level i，则存在level k (k < i)

        typedef struct zskiplistNode {
            robj *obj;
            double score;
            struct zskiplistNode *backward;
            struct zskiplistLevel {
                struct zskiplistNode *forward;
                unsigned int span;
            } level[];
        } zskiplistNode;
        
        typedef struct zskiplist {
            struct zskiplistNode *header, *tail;
            unsigned long length;
            int level;
        } zskiplist;
        
        typedef struct zset {
            dict *dict;
            zskiplist *zsl;
        } zset;


1. redis的skiplist结构对应zskiplist，设置ZSKIPLIST_MAXLEVEL为32，ZSKIPLIST_P为0.25.与标准的skiplist主要有如下几点区别：
    * score可以相同，如果score一致则比较element的大小      
    * level 1为一个双向链表，用于ZREVRANGE命令
2. 节点通过zskiplistNode表示，每个节点根据所在的层有level数组有不同的大小，在底层的有高层的所有数据。通过span存储了在每层跳的间隔，用于RANK命令
3. 通过zslInsert插入值，先从最高层开始查找比较，查找所有level合适的位置。然后通过zslRandomLevel获取随机的level x，在新建node插入到所有level小于x的层中

Replication
--------------

Redis的HA有如下几个特性:

1. master 可以拥有多个 slave
2. 多个 slave 可以连接同一个 master 外，还可以连接到其他 slave
3. 主从复制不会阻塞 master，在同步数据时，master 可以继续处理 client 请求，slave sync的时候阻塞

对于slave而言，通过server.repl_state表示Replication的状态  
对于master而言，每个slave的状态存储在redisClient.replstate里，每个slave对于master而言为一个client

*master更新流程*

1. master 收到syncCommand，调用rdbSaveBackground dump数据到磁盘(REDIS_REPL_WAIT_BGSAVE_START -> REDIS_REPL_WAIT_BGSAVE_END)
2. dump完成后，调用updateSlavesWaitingBgsave处理等待的slave sync，打开rdb并添加fd到FileEvent(REDIS_REPL_WAIT_BGSAVE_END->REDIS_REPL_SEND_BULK）
3. Eventloop调用sendBulkToSlave发送，在发送rdb数据前，先发送repldbsize到slave，然后分批读取数据发送到后端，发送完成添加sendReplyToClient到FileEvent (REDIS_REPL_SEND_BULK->REDIS_REPL_ONLINE)
4. redis执行command的时候，调用call()，根据flag参数判断是否需要propagate到AOF以及replication link，
根据cmd flags以及是否修改数据为参数调用propogate -> replicationFeedSlaves传递数据到slave

        call flag:
        #define REDIS_CALL_NONE 0
        #define REDIS_CALL_SLOWLOG 1
        #define REDIS_CALL_STATS 2
        #define REDIS_CALL_PROPAGATE 4
        #define REDIS_CALL_FULL (REDIS_CALL_SLOWLOG | REDIS_CALL_STATS | REDIS_CALL_PROPAGATE)
        
        propagate flag:
        #define REDIS_PROPAGATE_NONE 0
        #define REDIS_PROPAGATE_AOF 1
        #define REDIS_PROPAGATE_REPL 2

5. replicaationFeedSlaves调用addReplyMultiBulkLen以及addReplyBulk将command数据添加到output buffer,
然后通过Eventloop调用sendReplyToClient发送到slave

*slave更新流程*

1. 启动的时候server.repl_state为REDIS_REPL_CONNECT状态，在replicationcron的时候调用connectWithMaster() 
（REDIS_REPL_CONNECT -> REDIS_REPL_CONNECTING)

2. 连接成功后调用syncWithMaster，如果状态为CONNECTING 阻塞发送PING到master，检测master能正常工作 
（REDIS_REPL_CONNECTING -> REDIS_REPL_RECEIVE_PONG)

3. 发送PING后，阻塞读取PONG。如果需要验证发送验证信息。发送REPLCONF listening-port给master。  
发送sync命令给master。创建临时文件用于存储发送过来的rdb文件信息。（以上的send/read都为blocking）  
添加readSyncBulkPayload到Eventloop （REDIS_REPL_RECEIVE_PONG-> REDIS_REPL_TRANSFER）

4. readSyncBulkPayload首先同步读取发送的rdb size。然后读取发送过来的数据，同时调用sync_file_range，防止传输结束后fsync导致的延迟。  
传输结束后调用rdbLoad加载rdb文件,创建Redisclient master 并重新rewrite AOF （REDIS_REPL_TRANSFER -> REDIS_REPL_CONEECTED）

*ReplicationCron*

在serverCron里，每1000ms调用replicationCron对master进行重连以及检测传输失败

slave:

1. 检测是否是否连接超时或者receive pong超时（超时时间配置在repl_timeout里)，如果超时，调用undoConnectWithMaster() (REDIS_REPL_CONNECTING \| REDIS_REPL_RECEIVE_PONG)
2. 检测是否是否接收bulk data超时，如果超时，调用replicationAbortSyncTransfer() (REDIS_REPL_TRANSFER)
3. 判断是否master发送数据超时，如果超时，释放master连接 （REDIS_REPL_CONNECTED)
4. 判断是否需要重连，如果需要，调用connectWithMaster进行重连

mater:

1. 根据repl_ping_slave_period，定期的发送PING 命令给slave


通过slaveof命令可以停止replication，转为master.通过sync发送同步请求给master

LUA Script
------------
<http://redis.io/commands/eval>

lua是一个短小精悍的脚本语言，很多服务器端程序嵌入其作为扩展，nginx lua模块就很强悍。

命令格式为

    eval <script> <keynum> <keys> <args>

其中keynum为keys的数量，在lua里可以通过KEYS全局变量进行获取，args为除keys以为所有的参数可以通过ARGV全局变量进行获取。

其中lua脚本与redis通过redis.call以及redis.pcall进行交互，两者之间的区别就是对于错误的不同处理，call命令将错误抛出给调用者，不会继续向下执行。而pcall捕获错误然后继续执行，执行结束后将error通过table返回

lua脚本可以通过return语句返回值，返回值通过redis协议返回给客户端。经测试可以返回integer/string/list，对于table不返回。而且对于多重返回值，只返回第一个值

*协议转换* 

通过redisProtocolToLuaType进行redis到lua协议的转换，通过luaReplyToRedisReply进行lua到redis的协议转换

    lua numer <-> redis integer reply
    lua string  <-> redis bulk reply
    lua table(array) <-> redis mult bulk reply
    lua table.ok     <-> redis status reply
    lua table.err     <-> redis error reply
    lua boolean false <-> redis nil reply
    lua boolean true -> redis integer reply 1

*注意*

> 对于float类型返回int，返回的table如果包含nil则其后的值不会被返回
> 对于replication以及AOF存储的是执行的脚本而不是命令，需要保证在不同的时间相同的输入其输出是相同（例如脚本里使用时间、随机数或者外部状态进行求值），对此redis对于lua脚本的执行的命令进行了限制，而且redis不容许建立全局变量

使用eval命令需要每次都传递script body，可以通过evalsha命令节省带宽，传递script body的sha1摘要，如果服务器端不存在则返回特定错误。   
对于最大执行时间可以通过lua-time-limit进行限制，如果超过限制，则可以接收客户端发送script kill命令终止脚本执行

*redis嵌入lua* 

启动的时候调用scriptingInit初始化环境

1. 通过lua_open新建lua环境，然后通过通过lua_call注册类库，支持的有base/table/string/math/debug/cjson/struct/cmsgpack
2. 通过lua_setglobal设置禁止的函数名为nil禁止执行
3. 初始化dict server.lua_scripts,用于缓存script
4. 注册redis的命令列表，支持call/pcall/log/sha1hex/error_reply/status_reply
5. 替换math.random以及math.randomseed功能
6. 通过luaL_loadbuffer以及lua_pcall注释实现一个__redis__compare_helper方法，用于对redis返回的大批量数据排序。
7. 初始化server.lua_client，用于在lua环境执行命令。
8. 调用scriptingEnableGlobalsProtection设置_G的metatable，防止对全局变量进行修改

*lua访问redis* 

通过luaRedisGenericCommand执行redis.call/pcall的功能

1. 通过lua_State stack获取command args
2. 判断command是否容许执行。例如在slave上以及执行过非决断的命令不容许执行写操作等
3. 然后执行command并将结果返回给lua 环境。如果需要对结果进行排序则通过luaSortArray对lua stack的结果进行排序...

*eval/evalsha* 

通过evalGenericCommand处理命令

1. 通过redisSrand48重置seed，获取keys.
2. 通过sha1hex获取script的sha1并添加前缀为f_组成funcname.如果不存在则通过luaCreateFunction创建lua func
3. 设置KYES,ARGV全局变量，通过lua_sethook设置钩子，防止执行时间太长。
4. 调用lua_pcall执行lua func，执行成功后将结果返回给客户端并调用lua_gc。evalsha发送给replication以及aof将转为eval命令

Memory Overhead
------------------

不考虑Replication、AOF、Slowlog、Transcation、Blocking等消耗的资源, 其中几个关键对象的大小为

    sizeof(redisClient) = 16648
    sizeof(sds) = 8
    sizeof(robj) = 16
    sizeof(dictEntry) = 24

*Connection*

每条连接需要建立一个redisClient对象，其中主要是redisClient->buf为16k数据。redisClient->querybuf，初始通过sdsMakeRoomFor分配32k的大小。  
如果对于10k活跃连接，其分配的为（16k + 32k）* 10k = 480M

*Hash* 

假设key为16个字符，value为32个字符，通过set命令进行存储.
key的存储为sizeof(sds) = 8 + 16 = 24个字节。  
value的存储为sizeof(robj) = 16 + 32 = 48个字节。  
由于都是存储在dict里，则每个节点的大小为sizeof(dictEntry) = 24 + 24(key) + 48(value) = 96个字节

则对于1,000k的数据，其存储的大小为 96 * 1000k = 96M，利用率为48/96 = 0.5

Redis VS MemCached
--------------------

<http://redis.io/topics/benchmarks>

Redis Model
---------------

<http://redis.io/topics/twitter-clone>
<http://coolshell.cn/articles/7270.html>
<http://blog.codingnow.com/2011/11/dev_note_2.html>

对于redis建模需要将关系型数据库里的二维的数据变为一维的，而且需要自己维护索引。  
对于大批量的数据，如果内存比较敏感，可以对数据进行分段，然后使用hash结构代替kv存储  
<http://instagram-engineering.tumblr.com/post/12202313862/storing-hundreds-of-millions-of-simple-key-value-pairs>

twitter clone:

    table user
    
    global:nextUserId                         用于获取userid
    user:<userid>:name                    指定userid的用户名
    user:<userid>:email                    指定userid的email信息
    如果需要通过name获取id，则可以添加
    user:<name>:id          获取指定name的userid
    
    table user-followers （1:n）对于1对n的关系可以通过set来保存（无序）或者list（有序）
    user:<userid>:followers    ->(set of followers)
    user:<userid>:following    ->(set of following)
    
    table user-posts
    user:<userid>:post     ->(list of posts，可以使用lrange进行分页操作)

Reids TODO
-----------------

<http://oldblog.antirez.com/post/short-term-redis-plans.html>   
<http://oldblog.antirez.com/post/redis-sentinel-beta-released.html>   
<http://www.antirez.com/news/45>

Redis 应用以及运维
-------------------

*监控*

<https://github.com/kumarnitin/RedisLive>  
<https://github.com/steelThread/redmon>   
<http://www.percona.com/doc/percona-monitoring-plugins/cacti/redis-templates.html>  

*Partition* 

<http://redis.io/topics/partitioning>

*应用*

* 排行榜（可以以时间List或者值Set 为权重）
* 消息队列（PUB/SUB）
* id生成器
* 分布式锁
...

*运维*

1. 一台机器多实例，这样能充分发挥多核CPU的性能，而且防止每个redis实例的内存太大，防止COPY-ON-WRITE的时候占用内存太多
2. 通过twemproxy做前端proxy,进行sharding以及load balance
3. 进行读写分离，slave端主要负责read以及snapshot

....

*实例分析*

假设一个user有uid,mobileno以及email信息，现在需要通过uid查询到mobileno以及email，数据量在1亿条左右，update/select的比重基本为1：5. 需要可靠性高以及延时低.

*方案1* 

可以通过hset进行存储，例如

    hset <uid> m <mobileno>
    hset <uid> e <email>

假设每个uid平均占用9个字符，email平均占用30个字节，则使用的空间为

    ZIPLIST_HEADER_SIZE = 10
    filed m : <pre> 1byte + <len> 1byte + <value> 1byte = 3
    filed value: <pre> 1byte + <len> 1byte + <mobile> 8byte = 10
    field e : 3
    field email:  <pre> 1byte + <len> 1byte +<email> 30 = 32
    tail:1
    
    (9+4) * 100,000k + (16 + 24 + 59) * 100,000k = 11200M内存

如果对于email的后缀进行映射，应该可以使占用的空间更小，单机可以容纳，通过HA方式进行处理，可以使用keepalived进行主从切换

*方案2*

可以通过set进行存储，例如

    set u:<uid>:m <mobileno>
    set u:<uid>:e <email>

这样占用的空间为（13+ 4) * 2 * 100,000k + 40 * 100,000k + 70 * 100,000k = 14400M内存

*方案3*

可以通过类似的将uid进行分段，然后通过hset进行储存，其中mobileno以及email可以储存在不同的db

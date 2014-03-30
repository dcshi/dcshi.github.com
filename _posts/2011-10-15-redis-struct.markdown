---
layout: post
status: publish
published: true
title: Redis内存存储结构分析
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 12
wordpress_url: http://www.dcshi.com/?p=12
date: '2011-10-15 10:25:18 +0000'
date_gmt: '2011-10-15 10:25:18 +0000'
categories:
- redis
tags:
- redis
- "内存"
---
<p align="left">转自：<a href="http://www.searchtb.com/2011/05/redis-storage.html" target="_blank">http://www.searchtb.com/2011/05/redis-storage.html</a></p>
<p align="left">Redis: A persistent key-value database with built-in net interface written in ANSI-C for Posix systems</p>
<p align="left"><strong>1 Redis </strong><strong>内存存储结构</strong><strong></strong></p>
<p align="left">本文是基于 Redis-v2.2.4 版本进行分析.</p>
<p align="left"><strong>1.1 Redis </strong><strong>内存存储总体结构</strong><strong></strong></p>
<p align="left">Redis 是支持多key-value数据库(表)的,并用 RedisDb 来表示一个key-value数据库(表). redisServer 中有一个 redisDb *db; 成员变量, RedisServer 在初始化时,会根据配置文件的 db 数量来创建一个 redisDb 数组. 客户端在连接后,通过 SELECT 指令来选择一个 reidsDb,如果不指定,则缺省是redisDb数组的第1个(即下标是 0 ) redisDb. 一个客户端在选择 redisDb 后,其后续操作都是在此 redisDb 上进行的. 下面会详细介绍一下 redisDb 的内存结构.</p>
<p align="left">redis 的内存存储结构示意图</p>
<p align="left">redisDb 的定义:</p>
<p align="left">typedef struct redisDb</p>
<p align="left">{</p>
<p align="left">dict *dict;                 /* The keyspace for this DB */</p>
<p align="left">dict *expires;              /* Timeout of keys with a timeout set */</p>
<p align="left">dict *blocking_keys;    /* Keys with clients waiting for data (BLPOP) */</p>
<p align="left">dict *io_keys;              /* Keys with clients waiting for VM I/O */</p>
<p align="left">dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */</p>
<p align="left">int id;</p>
<p align="left">} redisDb;</p>
<p align="left">struct</p>
<p align="left">redisDb 中 ,dict 成员是与实际存储数据相关的. dict 的定义如下:</p>
<p align="left">typedef struct dictEntry</p>
<p align="left">{</p>
<p align="left">void *key;</p>
<p align="left">void *val;</p>
<p align="left">struct dictEntry *next;</p>
<p align="left">} dictEntry;</p>
<p align="left">typedef struct dictType</p>
<p align="left">{</p>
<p align="left">unsigned int (*hashFunction)(const void *key);</p>
<p align="left">void *(*keyDup)(void *privdata, const void *key);</p>
<p align="left">void *(*valDup)(void *privdata, const void *obj);</p>
<p align="left">int (*keyCompare)(void *privdata, const void *key1, const void *key2);</p>
<p align="left">void (*keyDestructor)(void *privdata, void *key);</p>
<p align="left">void (*valDestructor)(void *privdata, void *obj);</p>
<p align="left">} dictType;</p>
<p align="left">/* This is our hash table structure. Every dictionary has two of this as we</p>
<p align="left">* implement incremental rehashing, for the old to the new table. */</p>
<p align="left">typedef struct dictht</p>
<p align="left">{</p>
<p align="left">dictEntry **table;</p>
<p align="left">unsigned long size;</p>
<p align="left">unsigned long sizemask;</p>
<p align="left">unsigned long used;</p>
<p align="left">} dictht;</p>
<p align="left">typedef struct dict</p>
<p align="left">{</p>
<p align="left">dictType *type;</p>
<p align="left">void *privdata;</p>
<p align="left">dictht ht[2];</p>
<p align="left">int rehashidx; /* rehashing not in progress if rehashidx == -1 */</p>
<p align="left">int iterators; /* number of iterators currently running */</p>
<p align="left">} dict;</p>
<p align="left">dict 是主要是由 struct dictht 的 哈唏表构成的, 之所以定义成长度为2的( dictht ht[2] ) 哈唏表数组,是因为 redis 采用渐进的 rehash,即当需要 rehash 时,每次像 hset,hget 等操作前,先执行N 步 rehash. 这样就把原来一次性的 rehash过程拆散到进行, 防止一次性 rehash 期间 redis 服务能力大幅下降. 这种渐进的 rehash 需要一个额外的 struct dictht 结构来保存.</p>
<p align="left">struct dictht 主要是由一个 struct dictEntry 指针数组组成的, hash 表的冲突是通过链表法来解决的.</p>
<p align="left">struct dictEntry 中的 key 指针指向用 sds 类型表示的 key 字符串, val 指针指向一个 struct redisObject 结构体, 其定义如下:</p>
<p align="left">typedef struct redisObject</p>
<p align="left">{</p>
<p align="left">unsigned type:4;</p>
<p align="left">unsigned storage:2;   /* REDIS_VM_MEMORY or REDIS_VM_SWAPPING */</p>
<p align="left">unsigned encoding:4;</p>
<p align="left">unsigned lru:22;        /* lru time (relative to server.lruclock) */</p>
<p align="left">int refcount;</p>
<p align="left">void *ptr;</p>
<p align="left">/* VM fields are only allocated if VM is active, otherwise the</p>
<p align="left">* object allocation function will just allocate</p>
<p align="left">* sizeof(redisObjct) minus sizeof(redisObjectVM), so using</p>
<p align="left">* Redis without VM active will not have any overhead. */</p>
<p align="left">} robj;</p>
<p align="left">//type 占 4 bit,用来表示 key-value 中 value 值的类型,目前 redis 支持: string, list, set,zset,hash 5种类型的值.</p>
<p align="left">/* Object types */</p>
<p align="left">#define REDIS_STRING 0</p>
<p align="left">#define REDIS_LIST 1</p>
<p align="left">#define REDIS_SET 2</p>
<p align="left">#define REDIS_ZSET 3</p>
<p align="left">#define REDIS_HASH 4</p>
<p align="left">#define REDIS_VMPOINTER 8</p>
<p align="left">// storage 占 2 bit ,表示 此值是在 内存中,还是 swap 在硬盘上.</p>
<p align="left">// encoding 占 4 bit ,表示值的编码类型,目前有 8种类型:</p>
<p align="left">/* Objects encoding. Some kind of objects like Strings and Hashes can be</p>
<p align="left">* internally represented in multiple ways. The 'encoding' field of the object</p>
<p align="left">* is set to one of this fields for this object. */</p>
<p align="left">#define REDIS_ENCODING_RAW 0     /* Raw representation */</p>
<p align="left">#define REDIS_ENCODING_INT 1     /* Encoded as integer */</p>
<p align="left">#define REDIS_ENCODING_HT 2      /* Encoded as hash table */</p>
<p align="left">#define REDIS_ENCODING_ZIPMAP 3  /* Encoded as zipmap */</p>
<p align="left">#define REDIS_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */</p>
<p align="left">#define REDIS_ENCODING_ZIPLIST 5 /* Encoded as ziplist */</p>
<p align="left">#define REDIS_ENCODING_INTSET 6  /* Encoded as intset */</p>
<p align="left">#define REDIS_ENCODING_SKIPLIST 7  /* Encoded as skiplist */</p>
<p align="left">/* 如 type 是 REDIS_STRING 类型的,则其值如果是数字,就可以编码成 REDIS_ENCODING_INT,以节约内存.</p>
<p align="left">* 如 type 是 REDIS_HASH 类型的,如果其 entry 小于配置值: hash-max-zipmap-entries 或 value字符串的长度小于 hash-max-zipmap-value, 则可以编码成 REDIS_ENCODING_ZIPMAP 类型存储,以节约内存. 否则采用 Dict 来存储.</p>
<p align="left">* 如 type 是 REDIS_LIST 类型的,如果其 entry 小于配置值: list-max-ziplist-entries 或 value字符串的长度小于 list-max-ziplist-value, 则可以编码成 REDIS_ENCODING_ZIPLIST 类型存储,以节约内存; 否则采用 REDIS_ENCODING_LINKEDLIST 来存储.</p>
<p align="left">*  如 type 是 REDIS_SET 类型的,如果其值可以表示成数字类型且 entry 小于配置值set-max-intset-entries, 则可以编码成 REDIS_ENCODING_INTSET 类型存储,以节约内存; 否则采用 Dict类型来存储.</p>
<p align="left">*  lru: 是时间戳</p>
<p align="left">*  refcount: 引用次数</p>
<p align="left">*  void * ptr : 指向实际存储的 value 值内存块,其类型可以是 string, set, zset,list,hash ,编码方式可以是上述 encoding 表示的一种.</p>
<p align="left">* 至于一个 key 的 value 采用哪种类型来保存,完全是由客户端的指令来决定的,如 hset ,则值是采用REDIS_HASH 类型表示的,至于那种编码(encoding),则由 redis 根据配置自动决定.</p>
<p align="left">*/</p>
<p align="left"><strong>1.2 Dict </strong><strong>结构</strong><strong></strong></p>
<p align="left">Dict 结构在&lt;1.1Redis 内存存储结构&gt;; 已经描述过了,这里不再赘述.</p>
<p align="left"><strong>1.3 zipmap </strong><strong>结构</strong><strong></strong></p>
<p align="left">如果redisObject的type 成员值是 REDIS_HASH 类型的,则当该hash 的 entry 小于配置值: hash-max-zipmap-entries 或者value字符串的长度小于 hash-max-zipmap-value, 则可以编码成 REDIS_ENCODING_ZIPMAP 类型存储,以节约内存. 否则采用 Dict 来存储.</p>
<p align="left">zipmap 其实质是用一个字符串数组来依次保存key和value,查询时是依次遍列每个 key-value 对,直到查到为止. 其结构示意图如下:</p>
<p align="left">为了节约内存,这里使用了一些小技巧来保存 key 和 value 的长度. 如果 key 或 value 的长度小于ZIPMAP_BIGLEN(254),则用一个字节来表示,如果大于ZIPMAP_BIGLEN(254),则用5个字节保存,第一个字节为保存ZIPMAP_BIGLEN(254),后面4个字节保存 key或value 的长度.</p>
<ol start="1">
<li>初始化时只有2个字节,第1个字节表示 zipmap 保存的 key-value 对的个数(如果key-value 对的个数超过 254,则一直用254来表示, zipmap 中实际保存的 key-value 对个数可以通过 zipmapLen() 函数计算得到).</li>
</ol>
<ol>
<li>hset(nick,wuzhu) 后,</li>
</ol>
<ol start="2">
<ul>
<li>第1个字节保存key-value 对(即 zipmap 的entry 数量)的数量1</li>
<li>第2个字节保存key_len 值 4</li>
<li>第3~6 保存 key “nick”</li>
<li>第 7 字节保存 value_len 值 5</li>
<li>第 8 字节保存空闭的字节数 0 (当 该 key 的值被重置时,其新值的长度与旧值的长度不一定相等,如果新值长度比旧值的长度大,则 realloc 扩大内存; 如果新值长度比旧值的长度小,且相差大于 4 bytes ,则 realloc 缩小内存,如果相差小于 4,则将值往前移,并用 empty_len 保存空闲的byte 数)</li>
<li>第 9~13字节保存 value 值 “wuzhu”</li>
</ul>
<li>hset(age,30)插入 key-value 对 (“age”,30)</li>
<li>hset(nick,tide)插入 key-value 对 (“nick”,”tide”), 后可以看到 empty_len 为1 ,</li>
</ol>
<p align="left"><strong>1.4 ziplist </strong><strong>结构</strong><strong></strong></p>
<p align="left">如果redisObject的type 成员值是 REDIS_LIST 类型的,则当该list 的 elem数小于配置值: hash-max-ziplist-entries 或者elem_value字符串的长度小于 hash-max-ziplist-value, 则可以编码成 REDIS_ENCODING_ZIPLIST 类型存储,以节约内存. 否则采用 Dict 来存储.</p>
<p align="left">ziplist 其实质是用一个字符串数组形式的双向链表. 其结构示意图如下:</p>
<ol start="1">
<li>ziplist header由3个字段组成:</li>
<ul>
<li>ziplist_bytes: 用一个uint32_t 来保存, 构成 ziplist 的字符串数组的总长度,包括 ziplist header,</li>
<li>ziplist_tail_offset: 用一个uint32_t 来保存,记录 ziplist 的尾部偏移位置.</li>
<li>ziplist_length: 用一个 uint16_t 来保存,记录 ziplist 中 elem 的个数</li>
</ul>
<li>ziplist node 也由 3 部分组成:</li>
<ul>
<li>prevrawlen: 保存上一个 ziplist node 的占用的字节数,包括: 保存prevarwlen,currawlen 的字节数和elem value 的字节数.</li>
<li>currawlen&amp;encoding: 当前elem value 的raw 形式存款所需的字节数及在ziplist 中保存时的编码方式(例如,值可以转换成整数,如示意图中的”1024”, raw_len 是 4 字节,但在 ziplist 保存时转换成 uint16_t 来保存,占2 个字节).</li>
<li>(编码后的)value</li>
</ul>
</ol>
<p align="left">可以通过 prevrawlen 和 currawlen&amp;encoding 来遍列 ziplist.</p>
<p align="left">ziplist 还能到一些小技巧来节约内存.</p>
<ul>
<li>len 的存储: 如果 len 小于 ZIP_BIGLEN(254),则用一个字节来保存; 否则需要 5 个字节来保存,第 1 个字节存 ZIP_BIGLEN,作为标识符.</li>
<li>value 的存储: 如果 value 是数字类型的,则根据其值的范围转换成 ZIP_INT_16B, ZIP_INT_32B或ZIP_INT_64B 来保存,否则用 raw 形式保存.</li>
</ul>
<p align="left"><strong>1.5 adlist </strong><strong>结构</strong><strong></strong></p>
<p align="left">typedef struct listNode</p>
<p align="left">{</p>
<p align="left">struct listNode *prev;</p>
<p align="left">struct listNode *next;</p>
<p align="left">void *value;</p>
<p align="left">} listNode;</p>
<p align="left">typedef struct listIter</p>
<p align="left">{</p>
<p align="left">listNode *next;</p>
<p align="left">int direction;</p>
<p align="left">} listIter;</p>
<p align="left">typedef struct list</p>
<p align="left">{</p>
<p align="left">listNode *head;</p>
<p align="left">listNode *tail;</p>
<p align="left">void *(*dup)(void *ptr);</p>
<p align="left">void (*free)(void *ptr);</p>
<p align="left">int (*match)(void *ptr, void *key);</p>
<p align="left">unsigned int len;</p>
<p align="left">} list;</p>
<p align="left">常见的双向链表,不作分析.</p>
<p align="left"><strong>1.6 intset </strong><strong>结构</strong><strong></strong></p>
<p align="left">intset 是用一个有序的整数数组来实现集合(set). struct intset 的定义如下:</p>
<p align="left">typedef struct intset</p>
<p align="left">{</p>
<p align="left">uint32_t encoding;</p>
<p align="left">uint32_t length;</p>
<p align="left">int8_t contents[];</p>
<p align="left">} intset;</p>
<ul>
<li>encoding: 来标识数组是 int16_t 类型, int32_t 类型还是 int64_t 类型的数组. 至于怎么先择是那种类型的数组,是根据其保存的值的取值范围来决定的,初始化时是 int16_t, 根据 set 中的最大值在 [INT16_MIN, INT16_MAX] , [INT32_MIN, INT32_MAX], [INT64_MIN, INT64_MAX]的那个取值范围来动态确定整个数组的类型. 例如set一开始是 int16_t 类型,当一个取值范围在 [INT32_MIN, INT32_MAX]的值加入到 set 时,则将保存 set 的数组升级成 int32_t 的数组.</li>
<li>length: 表示 set 中值的个数</li>
<li>contents: 指向整数数组的指针</li>
</ul>
<p align="left"><strong>1.7 zset </strong><strong>结构</strong><strong></strong></p>
<p align="left">首先，介绍一下 skip list 的概念，然后再分析 zset 的实现.</p>
<p align="left"><strong>1.7.1 Skip List </strong><strong>介绍</strong><strong></strong></p>
<p align="left"><strong>1.7.1.1 </strong><strong>有序链表</strong><strong></strong></p>
<p align="left">1) Searching a key in a Sorted linked list</p>
<p align="left">//Searching an element &lt;em&gt;x&lt;/em&gt;</p>
<p align="left">cell *p =head ;</p>
<p align="left">while (p-&gt;next-&gt;key &lt; x )  p=p-&gt;next ;</p>
<p align="left">return p ;</p>
<p align="left">Note: we return the element proceeding either the element containing <em>x</em>, or the smallest element with a key larger than <em>x</em> (if <em>x</em> does not exists)</p>
<p align="left">2) inserting a key into a Sorted linked list</p>
<p align="left">//To insert 35 -</p>
<p align="left">p=find(35);</p>
<p align="left">CELL *p1 = (CELL *) malloc(sizeof(CELL));</p>
<p align="left">p1-&gt;key=35;</p>
<p align="left">p1-&gt;next = p-&gt;next ;</p>
<p align="left">p-&gt;next  = p1 ;</p>
<p align="left">3) deleteing a key from a sorted list</p>
<p align="left">//To delete 37 -</p>
<p align="left">p=find(37);</p>
<p align="left">CELL *p1 =p-&gt;next;</p>
<p align="left">p-&gt;next = p1-&gt;next ;</p>
<p align="left">free(p1);</p>
<p align="left"><strong>1.7.1.2 SkipList(</strong><strong>跳跃表</strong><strong>)</strong><strong>定义</strong><strong></strong></p>
<p align="left">SKIP LIST : A data structure for maintaing a set of keys in a sorted order.</p>
<p align="left">Consists of several <strong>levels.</strong></p>
<p align="left">All keys appear in level 1</p>
<p align="left">Each level is a sorted list.</p>
<p align="left">If key <em>x</em> appears in level <em>i</em>, then it also appears in all levels below <em>i</em></p>
<p align="left">An element in level<em> i </em>points (via down pointer) to the element with the same key in the level below.</p>
<p align="left">In each level the keys and appear. (In our implementation, INT_MIN and INT_MAX</p>
<p align="left">Top points to the smallest element in the highest level.</p>
<p align="left"><strong>1.7.1.3 SkipList(</strong><strong>跳跃表</strong><strong>)</strong><strong>操作</strong><strong></strong></p>
<p align="left">1) An empty SkipList</p>
<p align="left">2) Finding an element with key <em>x</em></p>
<p align="left">p=top</p>
<p align="left">While(1)</p>
<p align="left">{</p>
<p align="left">while (p-&gt;next-&gt;key &lt; x ) p=p-&gt;next;</p>
<p align="left">If (p-&gt;down == NULL ) return p-&gt;next</p>
<p align="left">p=p-&gt;down ;</p>
<p align="left">}</p>
<p align="left">Observe that we return <em>x</em>, if exists, or <em>succ(x)</em> if <em>x</em> is not in the SkipList</p>
<p align="left">3) Inserting new element <em>X</em></p>
<p align="left">Determine <strong><em>k</em></strong> the number of levels in which <em>x</em> participates (explained later)</p>
<p align="left">Do find(x), and insert x to the appropriate places in the lowest <strong><em>k</em></strong> levels. (after the elements at which the search path turns down or terminates)</p>
<p align="left">Example – inserting 119. <strong><em>k</em></strong>=2</p>
<p align="left">If <strong><em>k </em></strong>is larger than the current number of levels, add new levels (and update top)</p>
<p align="left">Example – inser(119) when k=4</p>
<p align="left">Determining <em>k</em></p>
<p align="left">k – the number of levels at which an element x participate.</p>
<p align="left">Use a random function OurRnd() — returns 1 or 0 (True/False) with equal probability.</p>
<p align="left">k=1 ;</p>
<p align="left">While( OurRnd() ) k++ ;</p>
<p align="left">Deleteing a key <em>x</em></p>
<p align="left">Find x in all the levels it participates, and delete it using the standard ‘delete from a linked list’ method.</p>
<p align="left">If one or more of the upper levels are empty, remove them (and update top)</p>
<p align="left">Facts about SkipList</p>
<p align="left">The expected number of levels is O( log <em>n</em> )</p>
<p align="left">(here <em>n</em> is the numer of elements)</p>
<p align="left">The expected time for insert/delete/find is O( log <em>n</em> )</p>
<p align="left">The expected size (number of cells) is O(<em>n </em>)</p>
<p align="left"><strong>1.7.2 redis SkipList </strong><strong>实现</strong><strong></strong></p>
<p align="left">/* ZSETs use a specialized version of Skiplists */</p>
<p align="left">typedef struct zskiplistNode</p>
<p align="left">{</p>
<p align="left">robj *obj;</p>
<p align="left">double score;</p>
<p align="left">struct zskiplistNode *backward;</p>
<p align="left">struct zskiplistLevel</p>
<p align="left">{</p>
<p align="left">struct zskiplistNode *forward;</p>
<p align="left">unsigned int span;</p>
<p align="left">} level[];</p>
<p align="left">} zskiplistNode;</p>
<p align="left">typedef struct zskiplist</p>
<p align="left">{</p>
<p align="left">struct zskiplistNode *header, *tail;</p>
<p align="left">unsigned long length;</p>
<p align="left">int level;</p>
<p align="left">} zskiplist;</p>
<p align="left">typedef struct zset</p>
<p align="left">{</p>
<p align="left">dict *dict;</p>
<p align="left">zskiplist *zsl;</p>
<p align="left">} zset;</p>
<p align="left">zset 的实现用到了2个数据结构: hash_table 和 skip list (跳跃表),其中 hash table 是使用 redis 的 dict 来实现的,主要是为了保证查询效率为 O(1),而 skip list (跳跃表) 是用来保证元素有序并能够保证 INSERT 和 REMOVE 操作是 O(logn)的复杂度。</p>
<p align="left">1) zset初始化状态</p>
<p align="left">createZsetObject函数来创建并初始化一个 zset</p>
<p align="left">robj *createZsetObject(void)</p>
<p align="left">{</p>
<p align="left">zset *zs = zmalloc(sizeof(*zs));</p>
<p align="left">robj *o;</p>
<p align="left">zs-&gt;dict = dictCreate(&amp;zsetDictType,NULL);</p>
<p align="left">zs-&gt;zsl = zslCreate();</p>
<p align="left">o = createObject(REDIS_ZSET,zs);</p>
<p align="left">o-&gt;encoding = REDIS_ENCODING_SKIPLIST;</p>
<p align="left">return o;</p>
<p align="left">}</p>
<p align="left">zslCreate()函数用来创建并初如化一个 skiplist。 其中，skiplist 的 level 最大值为 ZSKIPLIST_MAXLEVEL=32 层。</p>
<p align="left">zskiplist *zslCreate(void)</p>
<p align="left">{</p>
<p align="left">int j;</p>
<p align="left">zskiplist *zsl;</p>
<p align="left">zsl = zmalloc(sizeof(*zsl));</p>
<p align="left">zsl-&gt;level = 1;</p>
<p align="left">zsl-&gt;length = 0;</p>
<p align="left">zsl-&gt;header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);</p>
<p align="left">for (j = 0; j &lt; ZSKIPLIST_MAXLEVEL; j++) {</p>
<p align="left">zsl-&gt;header-&gt;level[j].forward = NULL;</p>
<p align="left">zsl-&gt;header-&gt;level[j].span = 0;</p>
<p align="left">}</p>
<p align="left">zsl-&gt;header-&gt;backward = NULL;</p>
<p align="left">zsl-&gt;tail = NULL;</p>
<p align="left">return zsl;</p>
<p align="left">}</p>
<p align="left">2) ZADD myzset 1 “one”</p>
<p align="left">ZADD 命令格式：</p>
<p align="left">ZADD key score member</p>
<ol start="1">
<li>根据 key 从 redisDb 进行查询，返回 zset 对象。</li>
<li>以 member 作为 key,score 作为 value ，向 zset的 dict 进行中插入;</li>
<li>如果返回成功，表明 member 没有在 dict 中出现过，直接向 skiplist 进行插入。</li>
<li>如果步骤返回失败，表明以 member 已经在 dict中出现过，则需要先从 skiplist 中删除，然后以现在的 score 值重新插入。</li>
</ol>
<p align="left">3) ZADD myzset 3 “three”</p>
<p align="left">4) ZADD myzset 2 “two”</p>
<p>&nbsp;</p>

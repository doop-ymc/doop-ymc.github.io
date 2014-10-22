---
title: memcached系列二——内存模型
layout: post
category: memcached
tag:
    - memcached
    - slab
    - 内存模型
---

个人看来对于任意的守护程序，服务类程序，较为重要的几个部分有内存模型、连接模型、线程模型、事件模型等，这里只提到
支持程序服务的底层的几个模型，对于较上层的数据结构实现，协议等则没有考虑。memcached作为一款缓存系统，其内存模型的
设计是有着较多优点的，下面详细介绍。

### memcached内存的整体结构

memcached内存模型中有以下几个概念：

    1. chunk：是用于存放数据的内存块
    2. slab：是用于切分为chunk的内存块，一个slabclass可挂载多个slabs
    3. slabclass：是用于管理相同chunk大小的内存的结构
    4. item：用于管理在于的key/value数据的结构，一个item放于一个chunk中。 
    5. LRU list: 用于管理各个slabclass的最近访问过的item, 以用于item踢出时的选择，list头部是最近访问过的item.
    6. hashtable: 用于查找数据时item寻址，对key计算hash值，再在hashtable中找到对应的item, 取出数据。
    7. slots list: 是slabclass用于管理当前class的空闲item的list, 由slabclass结构中的slots指针指向。

上面的7个概念彼此间的关系如下图示：

<pre class='slab'>
        +-------------------- <ins>slots 回收的空闲item</ins>   ----------------+ +--------------+                
        |     +------+------+    +------+------+     +------+------+ | | <ins>struct item</ins>  |                                                                
 +------|---->| prev | next |<-->| prev | next |<--> | prev | next | | |==============|                                                    
 |      |     +------+------+    +------+------+     +------+------+ | | *next        |                   +-------------+                                
 |      +------------------------------------------------------------- | *prev        |                   |  <ins>hashtable</ins>  |                                
 |                                             +-----------------+     | *h_next -----|------------------>|-------------|                                
 |                                             | chunk(96 bytes) |     |  time        |                   |-------------|                                
 |                                             +-----------------+     |  exptime     |                   |-------------|                                
 |                                             | chunk(96 bytes) |---->|  nbytes      |                   |-------------|                                
 |                                             +-----------------+     |  refcount    |                   |-------------|                                
 |                                        +--->| chunk(96 bytes) |     |  nsuffix     |                   |-------------|                                
 |                                        |    +-----------------+     |  it_flags    |                   |-------------|                                
 |  +--------------+            <ins>slab</ins>      |    |      、、       |     |  slabs_clsid |                   |-------------|                
 |  | <ins>slabclass[0]</ins> |         +--------+   |    +-----------------+     |  nkey        |                   |-------------|                
 |  |==============|         |  slabs |---+    | chunk(96 bytes) |     |   、、       |                   |-------------|                
 |  |  size        |         +--------+        +-----------------+     +--------------+                   |-------------|                
 |  |  perslab     |         |        |              <ins>chunk</ins>                    |                           |-------------|                
 +--|-*slots       |   +---> +--------+                                       |                           |-------------|                
    |  sl_curr     |   |     |        |      +--------------------------------+                           |-------------|                                           
    |  slabs       |   |     +--------+      |                                  *h_next------------------>|-------------|                
    |**slab_list --|---+         、          v                                     |                      |-------------|                
    |  list_size   |             、   +------+------+    +------+------+    +------+------+               |------开-----|                
    |  killing     | ================>| prev | next |<-->| prev | next |<-->| prev | next |               |-------------|                
    |  requested   |          |------>+------+------+    +------+------+    +------+------+<----|         |------链-----|                
    +--------------+       heads[0]                -----<ins>LRU list</ins>-----                         tails[0]    |-------------|                
                                                                                                          |------哈-----|                
    |   、 、 、   |                                                                                      |-------------|
        、 、 、                                                                                          |------希-----|                
    |              |                                       、                                             |-------------|                 
    +--------------+                                       、 *h_next------------------------------------>|------表-----|                 
    | slabclass[1] |                                            |                                         |-------------|                
    +--------------+                                            |               *h_next------------------>|-------------|                
    | slabclass[2] |                                            |                  |                      |-------------|                
    +--------------+                  +------+------+    +------+------+    +------+------+               |-------------|                
    | slabclass[3] | ================>| prev | next |<-->| prev | next |<-->| prev | next |               |-------------|                                                   
    +--------------+          |------>+------+------+    +------+------+    +------+------+<----|         |-------------|                
    | slabclass[4] |       heads[3]               -----<ins>LRU list</ins>-----                          tails[3]    |-------------|                
    +--------------+                                                                                      |-------------|                
    |    -----     |                                       、 *h_next------------------------------------>|-------------|                 
    +--------------+                                       、   |               *h_next------------------>|-------------|                 
    |slabclass[i-1]|                                            |                  |                      |-------------|                
    +--------------+                  +------+------+    +------+------+    +------+------+               |-------------|                
    | slabclass[i] | ================>| prev | next |<-->| prev | next |<-->| prev | next |               |-------------|                
    +--------------+          |------>+------+------+    +------+------+    +------+------+<----|         |-------------|                
                           heads[i]               -----<ins>LRU list</ins>-----                          tails[i]    |-------------|                
                                                                                                          +-------------+
                                                                                                          
                                            <strong>memcached内存管理模型结构图</strong>
</pre>

上图中的chunk, slab实际都是相对于内存上的概念，而不是具体的数据结构，chunk与slab, chunk与item的关系可用下图表示：

<pre class='prepic'>
                                +------------------------------+
                                |   data item    | empty space |<-----Chunk
                                +------------------------------+
                                          <strong>chunk item示意</strong>
                            Chunk
                               ^                                                         
            +------------------|------------------------------------------------------------+
            |   Memory         |                                                            | 
            |  +---------------|---------------------------------------------------------+  |
            |  |      +--------|---------------------+  +------------------------------+ |  |
            |  |      |Page1 +-|---+ +-----+ +-----+ |  |Page2 +-----+ +-----+ +-----+ | |  |
            |  | Slab |(1M)  | 96B | | 96B | | 96B | |  |(1M)  |100B | |100B | |100B | | |  | 
            |  |  1   |      +-----+ +-----+ +-----+ |  |      +-----+ +-----+ +-----+ | |  |
            |  |      +------------------------------+  +------------------------------+ |  |
            |  +-------------------------------------------------------------------------+  |
            |                                                                               |
            |  +-------------------------------------------------------------------------+  |
            |  |      +------------------------------+  +------------------------------+ |  |
            |  |      |Page1 +------+    +------+    |  |Page2 +------+    +-------+   | |  |
            |  | Slab | (1M) | 128B |    | 128B |    |  |(1M)  | 128B |    | 128B  |   | |  |
            |  |   2  |      +------+    +------+    |  |      +------+    +-------+   | |  |
            |  |      +------------------------------+  +------------------------------+ |  |
            |  +-------------------------------------------------------------------------+  |
            +-------------------------------------------------------------------------------+
                                          <strong>slab chunk示意</strong>
</pre>
### 内存模型各结构说明

有了上面的对memcached内存模型中各概念关系的详细示意，实际上对于其内存模型的结构已经有了大致的了解，为更进步深入分析其内存模型内部的机制，
未被需要对下面的几个结构进行了解。

#### slabclass
```c
    typedef struct {
        unsigned int size;      /* class中存放的item的大小*/
        unsigned int perslab;   /* 单个slab可存放的item的个数*/

        void *slots;            /* 可用的item */
        unsigned int sl_curr;   /* 当前使用到的位置*/

        unsigned int slabs;     /* 表示已经使用的slab_list的大小*/

        void **slab_list;       /* 分配的slab位置 */
        unsigned int list_size; /* 表示分配的slab_list的大小*/

        unsigned int killing;   /* 在进行slab reassign时使用*/
        size_t requested;       /* 此slabclass所占用的实际空间的大小*/
    } slabclass_t;
```

#### item
```c
    typedef struct _stritem {
        struct _stritem *next;      /* 指向此item在LRU list中的下一个item */
        struct _stritem *prev;      /* 指向此item在LRU list中的上一个*/
        struct _stritem *h_next;    /* 指向些item在Hashtable中的下一个*/
        rel_time_t      time;       /* 最近的访问时间*/
        rel_time_t      exptime;    /* 过期时间*/
        int             nbytes;     /* 数据的长度*/
        unsigned short  refcount;   /* 引用次数*/
        uint8_t         nsuffix;    /* 标志及长度的后辍串的长度*/
        uint8_t         it_flags;   /* ITEM的状态 */
        uint8_t         slabs_clsid;/* 位于的slabclass的id */
        uint8_t         nkey;       /* key的长度*/
        union {
        |   uint64_t cas;
        |   char end;
        } data[];
        /* 如果启用cas则在结构尾部接8byte的cas数，即uint64_t*/
        /* 然后是1byte的终止符\0 */
        /* then " flags length\r\n" (no terminating null) */
        /* then data with terminating \r\n (no terminating null; it's binary!) */
    } item;
```

#### hashtable
#### LRU list
这两个概念无具体的的结构定义，都是基于item结构内建的指针的，只是分别声明了全局的容器，如下：

```c
    static item** primary_hashtable = 0;        //main hashtable
    //LRU list
    static item *heads[LARGEST_ID];             //指向各slabclass的LRU链表的head
    static item *tails[LARGEST_ID];             //指向各slabclass的LRU链表的tail
```

### slab初始化

slab初始化的过程在不进行预分配的情况下，基本就是计算各slabclass的chunk的大小，及完成各slabclass的结构
的初始化。大体的过程如下：

    1. 依据设置chunk_size，计算最小的chunk的大小，初始化内存限制
    2. 若需要进行预分配，则一次性预申请限制大小的内存（预申请需要在系统支持large_page的情况下，才可使用）
    3. 依据设置的factor, item_size_max, 依次计算各slabclass的chunk大小，及单个page所能分配的对应chunk的个数。
    4. 若预分配标志为真，则为每个slabclass预分配一个slab来划分item(slab的大小为item_size_max)。

具体如下面的源码示：

```c
    void slabs_init(const size_t limit, const double factor, const bool prealloc) {
        int i = POWER_SMALLEST - 1;
        // a. 根据设置的chunk_size计算chunk的最小值. 
        // 默认的chunk_size 为48， item结构大小48
        unsigned int size = sizeof(item) + settings.chunk_size;

        mem_limit = limit;

        //b. 根据设置的预分配标志，完成内存的预申请
        if (prealloc) {
        |   /* Allocate everything in a big chunk with malloc */
        |   mem_base = malloc(mem_limit);
        |   if (mem_base != NULL) {
        |   |   mem_current = mem_base;
        |   |   mem_avail = mem_limit;
        |   } else {
        |   |   fprintf(stderr, "Warning: Failed to allocate requested memory in"
        |   |   |   |   " one large chunk.\nWill allocate in smaller chunks\n");
        |   }
        }

        memset(slabclass, 0, sizeof(slabclass));

        //c. 依据设置的factor及计算chunk的最小值，完成对各级slabclass，即slab的结构的初始化
        while (++i < POWER_LARGEST && size <= settings.item_size_max / factor) {
        |   /* Make sure items are always n-byte aligned */
        |   /*
        |   |* size = (size + (CHUNK_ALIGN_BYTES - 1)) & ~(CHUNK_ALIGN_BYTES - 1);
        |   |*/                   
        |   if (size % CHUNK_ALIGN_BYTES)  
        |   |   size += CHUNK_ALIGN_BYTES - (size % CHUNK_ALIGN_BYTES);

        |   slabclass[i].size = size;  
        |   slabclass[i].perslab = settings.item_size_max / slabclass[i].size;
        |   size *= factor;       
        |   if (settings.verbose > 1) {
        |   |   fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
        |   |   |   |   i, slabclass[i].size, slabclass[i].perslab);
        |   }                     
        }

        power_largest = i;        
        slabclass[power_largest].size = settings.item_size_max;         //item_size_max 默认大小为1M   
        slabclass[power_largest].perslab = 1;
        if (settings.verbose > 1) {    
        |   fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
        |   |   |   i, slabclass[i].size, slabclass[i].perslab);
        }

        /* for the test suite:  faking of how much we've already malloc'd */
        {
        |   char *t_initial_malloc = getenv("T_MEMD_INITIAL_MALLOC");
        |   if (t_initial_malloc) {    
        |   |   mem_malloced = (size_t)atol(t_initial_malloc);
        |   }

        }

        //d. 根据预分配标志，处理各级slabclass的初始化，若需要预分配，则进行各级slab的初始化
        if (prealloc) {
        |   slabs_preallocate(power_largest);
        }
    } 
```
### slab分配

slab的分配，实际上涉及三个主要的过程：   

    1. 新的slab的申请    
    2. 空白的slab的chunk划分   
    3. 空白的chunk的item化

在slab的分配过程，基本涉及了memcached主体的内存模型的构建（除去LRU list, hashtable后期的管理结构外), 下面的流程图，基本对这个过程进行了
概括，如下示：

<pre class='prepic'>                     
                                           +--------------------------------------------------------------+
                                           |1. 内存限制超出，且已经为此slab分配过slabclass时,直接返回失败 |
                  |                +------>|2. 1不满足时，若尝试增长slab链表长度失败时，直接返回失败      |
                  |                |       |2. 分配slab的内存失败时                                       |
                  v                |       +--------------------------------------------------------------+
        +------------------+     失败     +------+
        | 尝试申请新的slab |------------->| 返回 |
        +------------------+              +------+
                  | 成功                 
                  v                      +-----------------------+
  +-------------------------------+      |1. 内存切分为chunk     | 
  | split_slab_page_into_freelist |----->|2. 对切分后的chunk进行 |
  |===============================|      |   item的list化  ------|--+
  | 对新的slab进行划分            |      +-----------------------+  | 
  +-------------------------------+                                 v 
                  |                                      +-----------------------+
                  |                                      |     do_slabs_free     |
                  v                                      |=======================|     
  +-------------------------------+                      | chunk填充item结构，链 |
  |   接至slabclass的slab_list    |                      | 式处理后至slabclass的 |
  | 上，修改已使用的内存大小      |                      | slots上               |
  +-------------------------------+                      +-----------------------+

                                           <strong>slab分配流程示意</strong>
</pre>

下面详细看下这三个函数的源码分析：
#### do_slabs_newslab
```c
    static int do_slabs_newslab(const unsigned int id) {
        slabclass_t *p = &slabclass[id];
        int len = settings.slab_reassign ? settings.item_size_max
        |   : p->size * p->perslab;
        char *ptr;
        //增长链表，为新增的page划分内存

        /**
        |* 三种情况会返回申请新的item失败
        |* 1. 内存限制超出，且已经为此slab分配过slabclass时,直接返回失败
        |* 2. 1不满足时，若尝试增长slab链表长度失败时，直接返回失败（通常情况，系统内存用尽时)
        |* 3. 分配slabclass的内存失败时
        |*
        |* 另这里分配失败时，就会进行item的envict操作
        |*/
        if ((mem_limit && mem_malloced + len > mem_limit && p->slabs > 0) ||
        |   (grow_slab_list(id) == 0) ||
        |   ((ptr = memory_allocate((size_t)len)) == 0)) {

        |   MEMCACHED_SLABS_SLABCLASS_ALLOCATE_FAILED(id);
        |   return 0;
        }

        memset(ptr, 0, (size_t)len);
        //对新分配来的page进行划分
        split_slab_page_into_freelist(ptr, id);

        //连接到slabclass链表上
        p->slab_list[p->slabs++] = ptr;
        mem_malloced += len;
        MEMCACHED_SLABS_SLABCLASS_ALLOCATE(id);

        return 1;
    }
```
#### split_slab_page_into_freelist
这个函数内容不多，单看下源码

```c
    static void split_slab_page_into_freelist(char *ptr, const unsigned int id) {
        slabclass_t *p = &slabclass[id];
        int x;
        for (x = 0; x < p->perslab; x++) {
        |   do_slabs_free(ptr, 0, id);
        |   ptr += p->size;
        }
    }
```
#### do_slabs_free
```c
    static void do_slabs_free(void *ptr, const size_t size, unsigned int id) {
        slabclass_t *p;
        item *it;

        assert(((item *)ptr)->slabs_clsid == 0);
        assert(id >= POWER_SMALLEST && id <= power_largest);
        if (id < POWER_SMALLEST || id > power_largest)
        |   return;

        MEMCACHED_SLABS_FREE(size, id, ptr);
        p = &slabclass[id];

        it = (item *)ptr;
        /*置已划分的标志*/
        it->it_flags |= ITEM_SLABBED;
        it->prev = 0;
        /*插入slots list的首部*/
        it->next = p->slots;
        if (it->next) it->next->prev = it;
        p->slots = it;

        /*增加可用的item计数*/
        p->sl_curr++;
        p->requested -= size;
        return;
    }
```
### LRU list, HashTable的添加、删除

实际上这部分，内容很多，考虑到后面会单开memcached的LRU机制等更详细的item相关的主题，但又为了内存模型的完整性，这里仅说说
item加入LRU list与hashtable的两个操作（这里讨论无锁版本）。

####加入 do_item_link
```c
    int do_item_link(item *it, const uint32_t hv) {
        MEMCACHED_ITEM_LINK(ITEM_key(it), it->nkey, it->nbytes);
        assert((it->it_flags & (ITEM_LINKED|ITEM_SLABBED)) == 0);
        mutex_lock(&cache_lock);  
        /*更改item状态为已使用*/
        it->it_flags |= ITEM_LINKED;
        /*标记访问时间*/   
        it->time = current_time;  

        STATS_LOCK();             
        stats.curr_bytes += ITEM_ntotal(it);
        stats.curr_items += 1;    
        stats.total_items += 1;   
        STATS_UNLOCK();           
      
        /* Allocate a new CAS ID on link. */
        ITEM_set_cas(it, (settings.use_cas) ? get_cas_id() : 0);
        /*加入hash表*/
        assoc_insert(it, hv);     
        /*加入LRU list*/
        item_link_q(it);          
        /*加入LRU表，默认增加一引用计数*/
        refcount_incr(&it->refcount);  
        mutex_unlock(&cache_lock);

        return 1;                 
    }
```
####删除 do_item_unlink
```c
void do_item_unlink(item *it, const uint32_t hv) {
    MEMCACHED_ITEM_UNLINK(ITEM_key(it), it->nkey, it->nbytes);
    mutex_lock(&cache_lock);  
    if ((it->it_flags & ITEM_LINKED) != 0) {
        /*去掉使用标记*/
        it->it_flags &= ~ITEM_LINKED;  
        STATS_LOCK();
        stats.curr_bytes -= ITEM_ntotal(it);
        stats.curr_items -= 1;
        STATS_UNLOCK();       
        /*从hash表中删除*/
        assoc_delete(ITEM_key(it), it->nkey, hv);
        /*从LRU list中去除*/
        item_unlink_q(it);    
        /*
         * 1. 减引用计数
         * 2. 更改所属的slabclass
         * 3. 重新链接至slots list上
         */
        do_item_remove(it);   
    }
    mutex_unlock(&cache_lock);
}
```
###总结
memcached 的内存模型设计较为精巧，上面的内容介绍有些只是说了大概，后面会陆续有memcached的LRU机制、
memcached的引用计数，memcached的slab平衡等。

# Redis源码 

[TOC]

## 数据结构模块

### 字符串的实现

SDS：元数据 + 字符数组

![image-20230202084222439](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302020842546.png)





**使用SDS而不是C字符串的原因**：

- 某些操作的时间复杂度：SDS可以在O(1)获取长度和定位到末尾
- 二进制安全
- 扩缩容操作少 有预分配和惰性释放
- 内部扩容时 主动检查容量 避免缓冲区溢出 覆盖其他的内容



```c
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

// 创建一个SDS
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh; // 指向SDS结构体起始地址的指针
    sds s; // 指向字符数组起始地址的指针
    char type = sdsReqType(initlen); // 找到合适的长度大小类型
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */

    sh = s_malloc(hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1;
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0'; // 最后也会加上空字符串来兼容字符串操作
    return s;
}

sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);
	
    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    memcpy(s+curlen, t, len);
    sdssetlen(s, curlen+len);
    s[curlen+len] = '\0';
    return s;
}

// 预分配扩容逻辑 < 1MB 扩为原长度+所需长度的两倍 避免频繁扩容 
// >= 1MB  每次扩容+1MB
newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;
```



**节省内存方面的努力**

- 对于不同大小的字符串 使用不同的位数来保存长度信息

  8 16 32 64bit

- 编译优化：取消8字节的内存对齐 采用紧凑的方式分配内存

  `__attribute__ ((__packed__))`

  

### Hash表的设计

两个基本问题：（本质都是由Hash表存储的元素数量超过容量导致的）

- 如何解决哈希冲突 链式法 redis中是头插法

  ```c
  typedef struct dictEntry {
      void *key;
    	// 节省内存 如果值为整数或浮点数 可以节省一份指针的内存(8B)
      union {
          void *val;
          uint64_t u64;
          int64_t s64;
          double d;
      } v;
    	// 同一个bucket中的下一个节点
      struct dictEntry *next;
  } dictEntry;
  ```

- 非阻塞式rehash

  渐进式rehash



**Redis如何实现rehash**

基本思路：

- 准备了两个哈希表 用于rehash时交替保存数据
- 正常情况下 数据库键值对写入都是写到ht[0]
- 当扩容 进行rehash时 键值对会被逐渐迁移到ht[1]中
- 迁移完后 释放ht[0] ht[0]指针赋值为ht[1] 重置ht[1]

```c
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
  	// entry二维数组
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2]; // 一个字典中包含两个哈希表
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```



**实现rehash时 需要解决的问题**

- 触发rehash(扩容)的时机
- rehash扩容扩多大
- rehash如何执行



**触发rehash的时机**：`_dictExpandIfNeeded`(添加键值对时检查)

- 哈希表未初始化

- 负载因子>=1 且 hash表可以进行扩容

  （可以进行扩容指 没有RDB或AOF子进程 因为fork是写时复制 此时父进程进行大量写操作会产生很多内存页申请和写入的负担）

  ```c
  void updateDictResizePolicy(void) {
      if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
          dictEnableResize();
      else
          dictDisableResize();
  }
  ```

- 负载因子>=5 此时已经不管有没有子进程在读取数据了

```c
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```



**rehash扩容扩多大**

扩容到当前hash表 元素数量的2倍以上（2的幂 目的是为了可以使用掩码进行取余）



**如何实现渐进式rehash**

实现渐进式rehash的原因：

redis主线程在执行rehash拷贝数据时 无法执行其他请求 所以需要分次执行



```c
// 对传入的字典 一次尝试去迁移n个bucket 有一个index去遍历
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```



迁移的触发时机：

- `dictRehash(d,1);` 被动触发 

  对字典进行增删改查 每次都尝试迁移1个bucket

- `dictRehash(d,100)`  主动触发 防止服务器一直没有接收到请求的情况

  每次事件循环 给每个数据库的表空间字典 最多分配1ms的时间 去尝试迁移100个bucket

  ```c
  int dictRehashMilliseconds(dict *d, int ms) {
      long long start = timeInMilliseconds();
      int rehashes = 0;
  
      while(dictRehash(d,100)) {
          rehashes += 100;
          if (timeInMilliseconds()-start > ms) break;
      }
      return rehashes;
  }
  ```




迁移时对字典的增删查：

- 增：新增的key-value添加到ht[1]中

  ```c
  ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
  ```

- 删：先在ht[0]中尝试删除 再在ht[1]中尝试删除

  ```c
  for (table = 0; table <= 1; table++) {
          idx = h & d->ht[table].sizemask;
          he = d->ht[table].table[idx];
          // ...
          if (!dictIsRehashing(d)) break;
      }
  ```

- 查：先在ht[0]中尝试查找 再在ht[1]中尝试查找

  ```c
  for (table = 0; table <= 1; table++) {
          idx = h & d->ht[table].sizemask;
          he = d->ht[table].table[idx];
          while(he) {
              if (key==he->key || dictCompareKeys(d, key, he->key))
                  return he;
              he = he->next;
          }
          if (!dictIsRehashing(d)) return NULL;
      }
  ```





### 如何设计内存友好的数据结构

高效使用内存：

- 数据结构的优化设计与使用
- 内存数据按一定规则淘汰



**内存优化的数据结构**

- SDS 简单动态字符串
- ziplist 压缩列表
- intset 整合集合



**SDS的内存友好设计**

- 不同长度的字符串使用不同长度的元数据类型存储
- 嵌入式字符串



```c
// 位域定义方法: 冒号+数值
typedef struct redisObject {
        unsigned type:4; // 数据类型String/List/Hash
        unsigned encoding:4; // 实现数据类型的具体数据结构
        unsigned lru:24; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
        int refcount;
        void *ptr;
    } robj;
```



**嵌入式字符串**

使用场景：保存长度<=44B的字符串时使用

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302040933136.jpg)



```c
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= 44)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```

问题：为什么是44B

因为44 = 64 - 16(redisObject) - 3(sdshdr8)-1(\0)

跟jemalloc申请分配内存有关 每次都会返回2的幂次方的空间 



```C
// 字符串长度>44时 redisObject的指针指向一个SDS
// 弊端：需要执行两次内存分配 一次redisObejct 一次SDS 而且也会造成内存碎片

/* Create a string object with encoding OBJ_ENCODING_RAW, that is a plain
 * string object where o->ptr points to a proper sds string. */
robj *createRawStringObject(const char *ptr, size_t len) {
    return createObject(OBJ_STRING, sdsnewlen(ptr,len));
}

robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}
```





![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302041032123.jpg)

```c
/* Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
 * an object where the sds string is actually an unmodifiable string
 * allocated in the same chunk as the object itself. */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
  	// 申请一块连续的内存空间
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
  	// sh指向SDS首地址
    struct sdshdr8 *sh = (void*)(o+1);

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
  	// o->ptr指向字符串首地址
    o->ptr = sh+1;
    o->refcount = 1;
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }

    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr == SDS_NOINIT)
        sh->buf[len] = '\0';
    else if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}
```



**压缩列表和整数集合的设计**

压缩列表：

- List
- Hash
- Sorted Set



压缩列表的创建

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302041044666.jpg)

```c
/* Create a new empty ziplist. */
unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
  // 创建一块连续的内存空间 刚开始只有元数据
    unsigned char *zl = zmalloc(bytes);
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
  	// 连续空间最后一个字节 
    zl[bytes-1] = ZIP_END;
    return zl;
}
```



压缩列表中列表项的编码方式：

- prevlen：前一项的长度 不同的长度对应不同的字节开销 1B(<254B)或者5B

  ```c
  //判断prevlen的长度是否小于ZIP_BIG_PREVLEN，ZIP_BIG_PREVLEN等于254
  if (len < ZIP_BIG_PREVLEN) {
     //如果小于254字节，那么返回prevlen为1字节
     p[0] = len;
     return 1;
  } else {
     //否则，调用zipStorePrevEntryLengthLarge进行编码
     return zipStorePrevEntryLengthLarge(p,len);
  }
  
  
  if (p != NULL) {
      //将prevlen的第1字节设置为ZIP_BIG_PREVLEN，即254
      p[0] = ZIP_BIG_PREVLEN;
    //将前一个列表项的长度值拷贝至prevlen的第2至第5字节，其中sizeof(len)的值为4
      memcpy(p+1,&len,sizeof(len));
      …
  }
  //返回prevlen的大小，为5字节
  return 1+sizeof(len);
  ```

  

- encoding

- data



整数集合：intset

Set数据类型底层实现之一

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```



**节省内存的数据访问**

对于会经常使用的只读的对象，采用了共享对象的设计

```c

void createSharedObjects(void) {
   …
   //常见回复信息
   shared.ok = createObject(OBJ_STRING,sdsnew("+OK\r\n"));
   shared.err = createObject(OBJ_STRING,sdsnew("-ERR\r\n"));
   …
   //常见报错信息
 shared.nokeyerr = createObject(OBJ_STRING,sdsnew("-ERR no such key\r\n"));
 shared.syntaxerr = createObject(OBJ_STRING,sdsnew("-ERR syntax error\r\n"));
   //0到9999的整数
   for (j = 0; j < OBJ_SHARED_INTEGERS; j++) {
        shared.integers[j] =
          makeObjectShared(createObject(OBJ_STRING,(void*)(long)j));
        …
    }
   …
}
```



### 有序集合如何同时支持点查询和范围查询

有序集合：跳表 + 哈希表(member -> score)

```c
typedef struct zset {
    dict *dict;
  	// 支持单点查询zscore
    zskiplist *zsl;
} zset;
```

问题：

- 两个数据结构各自保存了什么数据
- 数据如何保持一致



**跳表的设计与实现**

> 一种多层的有序列表

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302070859201.jpg)



```c
// 结点 支持两个方向的查找
typedef struct zskiplistNode {
    //Sorted Set中的元素 member
    sds ele;
    //元素权重值
    double score;
    //后向指针
    struct zskiplistNode *backward;
    //节点的level数组，保存每层上的前向指针和跨度
    struct zskiplistLevel {
        struct zskiplistNode *forward;
      	// 跨越了level0的几个结点 可以用于计算结点在跳表中的顺序
        unsigned long span;
    } level[];
} zskiplistNode;
```



``` c

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```



**跳表节点查询**

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302080844704.jpg)

```c
// 获取跳表的表头
x = zsl->header;
// 从最大层数开始逐一遍历
for (i = zsl->level-1; i >= 0; i--) {
   ...
   while (x->level[i].forward && 
          (x->level[i].forward->score < score || 
           (x->level[i].forward->score == score && sdscmp(x->level[i].forward->ele,ele) < 0))) {
      ...
      // 当要查找的score > 后续节点的score || score相等但是member>后续节点的member 往后走
      x = x->level[i].forward;
    }
    // 当要查找的score < 后续节点的score 往下走
    ...
}
// 后续节点刚好就是要找的节点
x = x->level[0].forward;
```



**跳表节点层数设置**

理想状态：每一层的节点数都是下一层的1/2 查找复杂度就可以将到O(logN)

弊端：维持节点数量 需要额外的操作

解决方法：用概率随机去近似，插入新节点时，只需要修改前后节点的指针

```c

#define ZSKIPLIST_MAXLEVEL 64  //最大层数为64
#define ZSKIPLIST_P 0.25       //随机数的值为0.25
int zslRandomLevel(void) {
    //初始化层为1
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```



**哈希表和跳表的组合使用**

当Sorted Set编码方式为skiplist时

```c

 //如果采用ziplist编码方式时，zsetAdd函数的处理逻辑
 if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
   ...
}
//如果采用skiplist编码方式时，zsetAdd函数的处理逻辑
else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplistNode *znode;
        dictEntry *de;
        //从哈希表中查询新增元素
        de = dictFind(zs->dict,ele);
        //如果能查询到该元素
        if (de != NULL) {
            /* NX? Return, same element already exists. */
            if (nx) {
                *flags |= ZADD_NOP;
                return 1;
            }
            //从哈希表中查询元素的权重
            curscore = *(double*)dictGetVal(de);


            //如果要更新元素权重值
            if (incr) {
                //更新权重值
               ...
            }


            //如果权重发生变化了
            if (score != curscore) {
                //更新跳表结点
                znode = zslUpdateScore(zs->zsl,curscore,ele,score);
                //让哈希表元素的值指向跳表结点的权重
                dictGetVal(de) = &znode->score; 
                ...
            }
            return 1;
        }
       //如果新元素不存在
        else if (!xx) {
            ele = sdsdup(ele);
            //新插入跳表结点
            znode = zslInsert(zs->zsl,score,ele);
            //新插入哈希表元素
            serverAssert(dictAdd(zs->dict,ele,&znode->score) == DICT_OK);
            ...
            return 1;
        } 
        ..
```



### ziplist -> quicklist -> listpack

**ziplist的不足:** 出现是为了在元素数量少的情况下 尽量节省内存 

- 查找复杂度高

  不是数组 不能随机访问 无法根据索引查找

  除了头尾元素外 只能遍历查找 

- 连锁更新风险(导致元素更新和插入的时间复杂度是O(n^2))

  


**quicklist设计与实现**

控制ziplist的大小和元素个数，ziplist过大时，元素放在下一个新建的链表结点中 减少连锁更新的影响范围 ziplist作为链表的节点

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;     //前一个quicklistNode
    struct quicklistNode *next;     //后一个quicklistNode
    unsigned char *zl;              //quicklistNode指向的ziplist
    unsigned int sz;                //ziplist的字节大小
    unsigned int count : 16;        //ziplist中的元素个数 
    unsigned int encoding : 2;   //编码格式，原生字节数组或压缩存储
    unsigned int container : 2;  //存储方式
    unsigned int recompress : 1; //数据是否被压缩
    unsigned int attempted_compress : 1; //数据能否被压缩
    unsigned int extra : 10; //预留的bit位
} quicklistNode;



typedef struct quicklist {
    quicklistNode *head;      //quicklist的链表头
    quicklistNode *tail;      //quicklist的链表尾
    unsigned long count;     //所有ziplist中的总元素个数
    unsigned long len;       //quicklistNodes的个数
    ...
} quicklist;
```



![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302111152180.jpg)



**listpack的设计与实现**

关键在于列表项设计：

列表项：

- 编码类型：不同的编码类型 就决定了编码类型+实际数据的所占的长度
- 实际数据
- 编码类型+数据的长度



避免连锁更新的方法：

不记录前一项的长度 只记录自己的长度，这样当前元素变化 其他列表项的长度不会受到影响



如何遍历：子问题就是如何知道下一个列表项的首地址

从前向后：

1. 先取列表项第一个字节 判断出编码类型
2. 根据编码类型 计算出 编码类型+实际数据的总长度
3. 从而也就知道最后一项entry-len的长度
4. 这样整个列表项的长度就知道了



从后向前：

逐字节解析 读取entry-len即可

问题：如何知道entry-len的范围 每个字节 最高位为1 表示entry-len没有结束 最高位为0 表示结束   每个字节剩下的7bit表示真实的数字

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302111234118.jpg)
















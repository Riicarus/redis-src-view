# Redis 数据类型

redis 有 5 种不同的数据类型, 用作 redis 存储数据的主要类型.

- String
- List
- Set
- SortedSet(ZSet)
- Hash

## Redis Object

> See: BookMark-ROBJ

`redisObject` 是 redis 数据的一个抽象结构, redis 的各种对数据的操作也是基于此的, 其定义如下:

```c
#define LRU_BITS 24

struct redisObject {
    unsigned type:4;
    unsigned encoding:4;c
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
};

```

上述代码利用 C 语言的位域(bit-fields)来指定不同字段占用的比特位, 来使数据在内存/磁盘中存储得更紧凑, 提高效率.

`redisObject` 主要定义了数据的类型 `type` 和编码 `encoding`, 以及指向具体数据的指针 `void *ptr`, 使用 `void *` 定义使得指针支持指向多种类型的数据, 并且可以根据 `type` 以及 `encoding` 确定其实际类型.

`type` 字段是 5 种 redis 数据类型的具体映射:

```c
/* The actual redis object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```

`encoding` 字段更详尽地描述了 redis 数据类型是如何存储的, 也就是数据结构的具体实现.

```c
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* No longer used: old hash encoding. */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* No longer used: old list/hash/zset encoding. */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of listpacks */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
#define OBJ_ENCODING_LISTPACK 11 /* Encoded as a listpack */
```

---

`redisObject` 的创建如下:

```c
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;
    o->lru = 0;
    return o;
}
```

可以看到, `redisObject` 的默认编码是 `OBJ_ENCODING_RAW`.

## String

Redis 的字符串类型有 3 中不同的实现:

- `OBJ_ENCODING_INT`: 存储整数类型
- `OBJ_ENCODING_EMBSTR`: 存储长度不超过的 44 bytes 的字符串, 以及浮点数类型
- `OBJ_ENCODING_RAW`: 存储长度超过 44 bytes 的字符串

### INT

> See: BookMark-LL2STR

当 `encoding` 为 `OBJ_ENCODING_INT` 时, `ptr` 指针直接指向具体的整数值.

```c
if ((value >= LONG_MIN && value <= LONG_MAX) && flag != LL2STROBJ_NO_INT_ENC) {
    // robj *o;
    o = createObject(OBJ_STRING, NULL);
    o->encoding = OBJ_ENCODING_INT;
    o->ptr = (void*)((long)value);
}
```

这里对整数的存储有一个优化: 在创建 `redisObject` 之前, 先判断要存储的整数的值, 如果整数值在 `0~10000` 之间, redis 会将其 `redisObject` 指向缓存(共享数组 `shares.integers[]`)中预先创建的对象, 以此避免高频使用对象的重复创建, 节约内存, 提高性能.

```c
#define OBJ_SHARED_INTEGERS 10000

if (value >= 0 && value < OBJ_SHARED_INTEGERS && flag == LL2STROBJ_AUTO) {
    o = shared.integers[value];
}
```

> Java 的静态常量池也使用了类似的优化方式, 对 `Integer` 和 `String` 类型进行缓存优化.

`flag` 字段定义了为 `long long` 类型值创建 `redisObject` 的方式.

```c
/* Create a string object from a long long value according to the specified flag. */
#define LL2STROBJ_AUTO 0       /* automatically create the optimal string object */
#define LL2STROBJ_NO_SHARED 1  /* disallow shared objects */
#define LL2STROBJ_NO_INT_ENC 2 /* disallow integer encoded objects. */
```

需要注意的是: 不是所有的整数类型都使用 `OBJ_ENCODING_INT` 编码. 如果其值太大, 或者 redis 处于 `LL2STROBJ_NO_INT_ENC` 模式, 那么可能会根据其长度决定使用 `OBJ_ENCODING_RAW` 或者 `OBJ_ENCODING_EMBSTR` 编码.

---

可以看出, 使用 `OBJ_ENCODING_EMBSTR` 和 `OBJ_ENCODING_RAW` 唯一不同的条件就是需要保存的值的长度.

> See: BookMark-EMBSTR/ROW

```c
/* Create a string object with EMBSTR encoding if it is smaller than
 * OBJ_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
 * used.
 *
 * The current limit of 44 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```

### EMBSTR

同 `OBJ_ENCODING_RAW` 相比, `OBJ_ENCODING_EMBSTR` 同样使用 `char[]` 进行存储, 但是其长度是不可变的.

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

/* Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
 * an object where the sds string is actually an unmodifiable string
 * allocated in the same chunk as the object itself. */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
    struct sdshdr8 *sh = (void*)(o+1);

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    o->lru = 0;

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

从代码中看出, `o.ptr` 指向字符数组, 这里的字符串没有使用 SDS, 而是直接使用 C 中的字符串, 以 `\0` 结尾, 长度不可变.

### RAW

`OBJ_ENCODING_RAW` 类型使用 SDS(Simple Dynamic String) 作为数据存储结构, SDS 是"动态"的字符串类型, 其长度是可变的, 便于支持对字符串的修改操作.

```c
/* Create a string object with encoding OBJ_ENCODING_RAW, that is a plain
 * string object where o->ptr points to a proper sds string. */
robj *createRawStringObject(const char *ptr, size_t len) {
    return createObject(OBJ_STRING, sdsnewlen(ptr,len));
}
```

### SDS

SDS 的定义很简单, 就是一个 `char` 数组.

```c
typedef char *sds;
```

但它不是 SDS 的完整表示, 而只是 SDS 的字符串数组部分, 完整的 SDS 在 redis 中是通过 `sdshdr` 结构体表示的, 理解为 "SDS with header", 主要存储了 SDS 需要的所有信息, 以 `sdshdr8` 举例:

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    // 已使用的空间
    uint8_t len; /* used */
    // 申请到的空间, 剩余空间 = alloc - len
    uint8_t alloc; /* excluding the header and null terminator */
    // SDS 类型
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    // 字符数组
    char buf[];
};
```

SDS 的创建是通过 `sdsnewlen(const char *ptr, size_t len)` 创建的, 实际会调用 `_sdsnewlen(const char *ptr, size_t len, int trymalloc)` 进行创建.

```c
sds sdsnewlen(const void *init, size_t initlen) {
    return _sdsnewlen(init, initlen, 0);
}
```

redis 会根据 `initlen`, 也就是需要存储的字符串的长度决定使用那一种类型的 SDS, 再根据类型获取 `hdrlen`, 也就是 `sdshdr` 的 header 的长度. 之后, 就会根据 `initlen` 和 `hdrlen` 为 SDS 申请一块内存, 并且依据 `init` 判断是否需要对申请到的内存进行初始化, `init` 是需要存储的字符串指针.

如果指定不初始化 SDS, 也就是 `init == SDS_NOINIT`, 或者无法分配足够的空间, `_sdsnewlen` 函数会直接返回一个空指针 `(void *)0`; 如果字符串为空串, 也就是 `init = (void *)0`, 则直接初始化为 0 字节的串.

```c
/* Create a new sds string with the content specified by the 'init' pointer
 * and 'initlen'.
 * If NULL is used for 'init' the string is initialized with zero bytes.
 * If SDS_NOINIT is used, the buffer is left uninitialized;
 *
 * The string is always null-terminated (all the sds strings are, always) so
 * even if you create an sds string with:
 *
 * my_string = sdsnewlen("abc",3);
 *
 * You can print the string with printf() as there is an implicit \0 at the
 * end of the string. However the string is binary safe and can contain
 * \0 characters in the middle, as the length is stored in the sds header. */
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    // `sh` 的类型就是 sdshdr 结构体
    void *sh;
    sds s;

    // 拿到应该初始化的类型
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;

    // 拿到对应的 sds_header 的长度
    int hdrlen = sdsHdrSize(type);

    // 标识位指针标识 sds 的类型 type
    // 标识位指针在 sds_header 的最后一个字节
    unsigned char *fp; /* flags pointer. */
    // 可用空间大小, sh 大小 -> sds 大小
    size_t usable;

    assert(initlen + hdrlen + 1 > initlen); /* Catch size_t overflow */

    // 为 sh 申请内存空间
    // 需要为字符串设置终结符 '\0', 所以申请内存的大小为 hdrlen + initlen + 1
    sh = trymalloc?
        s_trymalloc_usable(hdrlen+initlen+1, &usable) :
        s_malloc_usable(hdrlen+initlen+1, &usable);
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);

    // sds 的起始位置, sh 的结构: sds_header(hdrlen) | sds(initlen)
    s = (char*)sh+hdrlen;
    // flag 始终在最后一个字节
    fp = ((unsigned char*)s)-1;
    // 更新可用空间为 sds 长度
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);

    // 根据初始化类型初始化 sh
    // 通过 SDS_HDR_VAR(T, s) 创建新的 sh 指针, 指向上面创建的 sh 的地址, 并且更新 sh 的字段
    // 初始化时, alloc 和 len 应该是相等的?
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
    }

    // 复制要存储的字符串值到 sds 中
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';

    // 返回的是 sds 的指针, 而不是 sh, 后续通过 SDS_HDR(T, s) 获取指向 sh 的指针
    return s;
}
```

内存结构如下:

- `sds_header`: `len | alloc | flag(1)`
- `sdshdr`: `sds_header(hdrlen) | buffer(initlen)`

---

我们看一看 redis 是如何通过 SDS 指针获取到整个 SDS 的属性的.

```c
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));

#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))

#define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)

// 使用 inline 在执行时将函数展开, 避免创建过多的栈空间
// 获取 sds 的长度
static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}

// 获取 sds 的可用空间
static inline size_t sdsavail(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5: {
            return 0;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            return sh->alloc - sh->len;
        }
    }
    return 0;
}
```

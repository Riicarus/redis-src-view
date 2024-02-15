# Redis Data Types

redis data types include 5 different types.

- String
- List
- Set
- SortedSet(ZSet)
- Hash

## Redis Object

> See: BookMark-ROBJ

definition:

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

The upper code uses C's bit-fields to designate the number of bits occupied by the field. It mainly defines the `type` and `encoding` of one `redisObject`, and maintains a pointer to the distinct redis data type.

The `type` field has 5 values mapped to 5 kinds of redis data types:

```c
/* The actual redis object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */q
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```

The `encoding` field more basically describes how the redis data type is stored.

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

## String

String in redis has 3 encoding implementations:

- `OBJ_ENCODING_INT`: stores integer value
- `OBJ_ENCODING_RAW`: stores char sequence value which is longer than 44 bytes
- `OBJ_ENCODING_EMBSTR`: stores float value

### INT

> See: BookMark-LL2STR

When the encoding of String is `OBJ_ENCODING_INT`, the field `*ptr` directly stores the integer value.

```c
if ((value >= LONG_MIN && value <= LONG_MAX) && flag != LL2STROBJ_NO_INT_ENC) {
    // robj *o;
    o = createObject(OBJ_STRING, NULL);
    o->encoding = OBJ_ENCODING_INT;
    o->ptr = (void*)((long)value);
}
```

There's an optimization for integer value between `0` and `10000`, redis stores them to a shared redis object array `shared.integers[]` to avoid frequently create the same redis objects.

```c
if (value >= 0 && value < OBJ_SHARED_INTEGERS && flag == LL2STROBJ_AUTO) {
    o = shared.integers[value];
}
```

> Java Static Constant Pool also uses the same optimization for integer values and String values to provide higher performance.

The `flag` shows how to create a string object from a `long long` value.

```c
/* Create a string object from a long long value according to the specified flag. */
#define LL2STROBJ_AUTO 0       /* automatically create the optimal string object */
#define LL2STROBJ_NO_SHARED 1  /* disallow shared objects */
#define LL2STROBJ_NO_INT_ENC 2 /* disallow integer encoded objects. */
```

We should mention that not all integer value will be encoded as `OBJ_ENCODING_INT`, if the value is too long or the server is in `LL2STROBJ_NO_INT_ENC` mode, it will be encoded as `OBJ_ENCODING_RAW` or `OBJ_ENCODING_EMBSTR` according to its length.

---

So the only limits between `OBJ_ENCODING_EMBSTR` and `OBJ_ENCODING_RAW` is the value's length.

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

### RAW

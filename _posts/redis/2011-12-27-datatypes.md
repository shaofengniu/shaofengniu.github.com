---
title: Data Types of Redis
layout: post
tags: ["Redis"]

---

# Strings #
Strings are the basic building blocks of Redis. It's defined in `sds.h`.
{% highlight c %}
//redis.h
typedef char *sds;
struct sdshdr {
    int len;
    int free;
    char buf[];
};
{% endhighlight %}

![Redis string](/assets/images/redis/sds.png)

Redis String is [binary safe](http://en.wikipedia.org/wiki/Binary-safe), since it uses the
`len` and `free` fields in `struct sdshdr` to decide the end of the
string instead of the `\'0'` character. 

Every time a new string object is created, an `sds` variable, which is
a pointer to the real data part of the string object, is returned.

# Lists #
There are two different list implementations in Redis: one is the
usual double linked list defined in `adlist.h`, the other is
the ziplist defined in `ziplist.h`. 

{% highlight c %}
// adlist.h
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned int len;
} list;
{% endhighlight %}

The double linked list version of Redis list supports constant time
insertion and deletion of elements near the head and tail. But when
the payload size is small, the pointer overhead makes it not that much
memory efficient.

So Redis makes use of another version of list, the ziplist, when
payload size is relativly small. The ziplist is basically a dynamic
array, which improves the memory efficiency at the cost of O(whole
data size) `LPUSH`/`LPOP`. And a little more CPU will be used to
decode and encode ziplist entry.
![Ziplist](/assets/images/redis/ziplist.png)

When list contains only integer values or raw-encoded objects no bigger
than `server.list_max_ziplist_value`, the ziplist will be used to
store this values. Otherwise the double linked version will be used
instead.

# Hashes #
There are also two types of hash implementations: a ordinary hash map
defined in `dict.h` and a zipmap defined in `zipmap.h`. 

{% highlight c %}
// dict.h
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;

typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dictEntry {
    void *key;
    void *val;
    struct dictEntry *next;
} dictEntry;
{% endhighlight %}

The hash table defined in `dict.h` is merely a standard
implementation. Collions are handled by chaining and the table will
auto resize if needed.

Obviously, it's not memory efficient if we just need a small map data
structure from string to string. So Redis takes the same approach as the
ziplist, it implements a zipmap.

The zipmap structure is basically the same idea with ziplist. It use a
continuous block of memory to save the key-value pairs.

![zip map](/assets/images/redis/zipmap.png)

The zipmap structure is mainly used by the Hashes commands for a hash
with a few fields (where few means up to one hundred or so). Otherwise
the hash table version will be used.

# Sets #
The Redis set is implemented based on the hash table defined in
`dict.h` except when all the elements of set are integers. In this
case, a int set defined in `intset.h` will be used.

{% highlight c %}
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
{% endhighlight %}

The integer values will be stored in a sorted order so that we can
efficiently find a integer using binary search.

# Sorted Sets #
In order to get O(log(N)) INSERT and REMOVE operations into a sorted
data structure, Redis uses two data structure: one is a hash table,
the other is a [skip list](http://en.wikipedia.org/wiki/Skip_list).

The implementation of a skip list is much simpler than a balanced
tree and the insertion and deletion is faster than the version of a
balanced tree by a constant factor in average.

"The elements are added to an hash table mapping Redis objects to
scores. At the same time the elements are added to a skip list mapping
scores to Redis objects." 

The full implementaion is in `t_zset.c`.

The skip list implementation of Redis is basically the same as in the
original paper "Skip Lists: A Probabilistic* Alternative to Balanced
Trees", except that:

* This implementation allows for repeated scores.
* The comparison is not just by score but also by attached member.
* There is a back pointer on level 1, which can be used to do reverse
traversal.

Just like the list and set, the sorted set implementation takes a
hybrid approach: use ziplist when members are small, otherwise use the
skip list.


# References #
1. [Redis Data types](http://redis.io/topics/data-types)
2. [Skip list on wikipedia](http://en.wikipedia.org/wiki/Skip_list)
3. Skip Lists: A Probabilistic* Alternative to Balanced Trees

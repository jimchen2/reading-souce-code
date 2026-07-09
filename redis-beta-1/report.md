From my perspective this is how Redis works. it is simply a key-value cache, you give it a key, and you give any string to store into it, and you retrieve it very fast. In this blog I am looking at the source code of [Redis-Beta-1](https://download.redis.io/releases/). I am just going to downloaded the smallest version on the website.

```
user@fedora ~/D/r/redis> ls *.c
adlist.c  ae.c  anet.c  dict.c  picol.c  redis.c  sds.c
```

Basically works out of box

```
user@fedora ~/D/r/redis> make
cc -c -O2 -Wall -W -DSDS_ABORT_ON_OOM -g  adlist.c
cc -c -O2 -Wall -W -DSDS_ABORT_ON_OOM -g  ae.c
cc -c -O2 -Wall -W -DSDS_ABORT_ON_OOM -g  anet.c
cc -c -O2 -Wall -W -DSDS_ABORT_ON_OOM -g  dict.c
cc -c -O2 -Wall -W -DSDS_ABORT_ON_OOM -g  redis.c
cc -c -O2 -Wall -W -DSDS_ABORT_ON_OOM -g  sds.c
cc -c -O2 -Wall -W -DSDS_ABORT_ON_OOM -g  picol.c
cc -o redis-server -O2 -Wall -W -DSDS_ABORT_ON_OOM -g adlist.o ae.o anet.o dict.o redis.o sds.o picol.o

Hint: To run the test-redis.tcl script is a good idea.
Launch the redis server with ./redis-server, then in another
terminal window enter this directory and run 'make test'.
```

## `sds.c`

### Struct of SDS

In the first version the struct is basically

```
struct sdshdr {
    long len;  // Length of the string
    long free; // Free space remaining in the buffer
    char buf[0]; // The actual string data
};
```

When you create an SDS string by calling the function `sdsnewlen`, it allocates `sizeof(struct sdshdr)+initlen+1` lens and returns a pointer to the `buf`, while the `len` and `free` and located 8/16 bytes backward. _I think you can also just return a pointer to the start and call `(char *)my_string + 16` for the string? But it's the same_ The `sdslen` returns the length like this `(void*) (s-(sizeof(struct sdshdr)))`

Here are some intuitive basic functions

```
sds sdsnewlen(const void *init, size_t initlen);
size_t sdslen(const sds s);                    // Return the length
size_t sdsavail(sds s);                        // Return the free memory
void sdsfree(sds s);                           // Free the SDS string
```

### Memory Allocation

`SDS_ABORT_ON_OOM` is the thing defined to let the program instantly crash on memory failure. There is no error handling here.

```
static sds sdsMakeRoomFor(sds s, size_t addlen)
sds sdscatlen(sds s, void *t, size_t len);
```

Here you ask for "addlen" bytes in memory,

- If there are more free bytes, it returns
- If there aren't more free bytes, it updates the new len to `sizeof(struct sdshdr)+(len+addlen)*2+1` (the 1 in the end is for `\0`), basically doubling the space in the buffer after allocating.

### Other Functions

```
sds sdscpy(sds s, char *t);            // Safe copy
sds sdstrim(sds s, const char *cset);  // Trim chars from start and end
sds sdsrange(sds s, long start, long end); // Cut the array based on indexes
```

```
sds sdscatprintf(sds s, const char *fmt, ...); // Example: my_string = sdscatprintf(my_string, "Name is %s", "John");
```

The function `sdscatprintf` uses a trick to first put "\0" in "-2" position, call `vsnprintf` to write, and check if it was overwritten. If it was, allocate twice the space and write again.

```
sds *sdssplitlen(char *s, int len, char *sep, 2 seplen, int *count); // Example: sds *pieces = sdssplitlen("apple||bar||cat", 15, "||", 2, &count);
```

Note here it uses raw string matching (not Don Knuth's smart algorithm). I don't know why as I was "taught in classes" to use efficient (O(M+N)) algorithms. Ok, let's drop it.

```
    for (j = 0; j < (len-(seplen-1)); j++) {
...
        /* search the separator */
        if ((seplen == 1 && *(s+j) == sep[0]) || (memcmp(s+j,sep,seplen) == 0))
```

Now, this `if` condition confused me again and I stared at it for a long time as I thought the two things are the same. I was never "taught in classes" to do a direct comparison and avoid memcmp make the 1-char condition faster.

But here's the problem:

- If `seplen` is 1, and it does match, it's fast and good.
- If `seplen` is 1, and it does not match, the first condition evaluates to false, doesn't it end up calling `memcmp` again?

I thought it should be like this

```
if (seplen == 1 ? (*(s+j) == sep[0]) : (memcmp(s+j, sep, seplen) == 0))
```

Then I went to look at the [current Redis source code](https://github.com/redis/redis/blob/b22ce4abb50360bda0e7a7ae9bfa95e1af38853c/src/sds.c) or [Valkey source code](https://github.com/valkey-io/valkey/blob/f3e957cee88ab4f76bc537a8148040152d944e62/src/sds.c), and the line is exactly the same `if ((seplen == 1 && *(s+j) == sep[0]) || (memcmp(s+j,sep,seplen) == 0))`

Alright, so we wrote 2 pieces of source code into [Godbolt](https://godbolt.org/), and added the flag `-O3`

```
#include <string.h>

int count_original(const char *s, int len, const char *sep, int seplen) {
    int matches = 0;
    for (int j = 0; j < (len - (seplen - 1)); j++) {
        if ((seplen == 1 && *(s+j) == sep[0]) || (memcmp(s+j, sep, seplen) == 0)) {
            matches++;
        }
    }
    return matches;
}

int count_ternary(const char *s, int len, const char *sep, int seplen) {
    int matches = 0;
    for (int j = 0; j < (len - (seplen - 1)); j++) {
        if (seplen == 1 ? (*(s+j) == sep[0]) : (memcmp(s+j, sep, seplen) == 0)) {
            matches++;
        }
    }
    return matches;
}
```

See the [Godbolt comparison](https://godbolt.org/z/8ETjnjva6)

Because of the short-circuiting fallback to `memcmp`, GCC evaluates this loop using scalar instructions and branching.

```
.L9:
        cmp     BYTE PTR [rdi], cl
        je      .L7
        movzx   esi, BYTE PTR [rdi]
        cmp     BYTE PTR [rdx], sil
        jne     .L8
```

Because `seplen` is loop-invariant, this strict ternary separation allows modern compilers (GCC -O3) to unswitch the loop. In the `seplen == 1` case, GCC now successfully auto-vectorizes the loop, replacing the byte-by-byte comparison with 128-bit SIMD instructions (`movdqu`, `pcmpeqb`), processing 16 bytes at a time.

```
.L21:
        movdqu  xmm0, XMMWORD PTR [rax]
        add     rax, 16
        pcmpeqb xmm0, xmm6
        pand    xmm0, xmm7
        movdqa  xmm2, xmm0
        punpckhbw       xmm0, xmm5
        punpcklbw       xmm2, xmm5
        movdqa  xmm1, xmm2
        punpckhwd       xmm2, xmm3
        punpcklwd       xmm1, xmm3
        paddd   xmm1, xmm4
        paddd   xmm1, xmm2
        movdqa  xmm2, xmm0
        punpckhwd       xmm0, xmm3
        punpcklwd       xmm2, xmm3
        paddd   xmm1, xmm2
        movdqa  xmm4, xmm1
        paddd   xmm4, xmm0
        cmp     rdx, rax
        jne     .L21
```

So I cloned the Valkey source code and I ran the benchmark `src/valkey-benchmark -q -n 100000 -c 50 -t set,get`

```
user@fedora ~/D/valkey (unstable) [1]> src/valkey-benchmark -q -n 1000000 -c 50 -t set,get
SET: 91827.36 requests per second, p50=0.279 msec
GET: 93196.65 requests per second, p50=0.279 msec

user@fedora ~/D/valkey (unstable)> src/valkey-benchmark -q -n 1000000 -c 50 -t set,get
SET: 92524.06 requests per second, p50=0.279 msec
GET: 92387.28 requests per second, p50=0.279 msec
```

I changed the `sds.c` code

```
user@fedora ~/D/valkey (unstable) [1]> src/valkey-benchmark -q -n 1000000 -c 50 -t set,get
SET: 91617.04 requests per second, p50=0.279 msec
GET: 93344.53 requests per second, p50=0.279 msec

user@fedora ~/D/valkey (unstable)> src/valkey-benchmark -q -n 1000000 -c 50 -t set,get
SET: 92532.62 requests per second, p50=0.279 msec
GET: 93014.60 requests per second, p50=0.279 msec
```

There was basically no differences. I don't know much about the benchmarks anyway and networking might be the bottleneck.. Anyway, I am still submitting a Pull Request to Valkey.

### Submitting a Pull Request

This is my first time submitting a Pull Request to a public open source project. Here is the [contributing guide](https://github.com/valkey-io/valkey/blob/unstable/CONTRIBUTING.md).

1. Fork Valkey on GitHub ([HOWTO](https://docs.github.com/en/github/getting-started-with-github/fork-a-repo))
1. Create a topic branch (`git checkout -b my_branch`)
1. Make the needed changes and commit with a DCO. (`git commit -s`)
1. Push to your branch (`git push origin my_branch`)
1. Initiate a pull request on GitHub ([HOWTO](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/creating-a-pull-request))
1. Done :)

Basically here you don't just do `git add . && git commit -m "Updates" && git push`

```
git add src/sds.c
git add deps/libvalkey/src/sds.c
git commit -s -m "Optimize separator matching in sdssplitlen"
```

Then I tested it locally (just to see how it goes)

```
make -j$(nproc)
make test
```

And I got everything good except for the last error "Executing test client: I/O error reading reply". And I tested it on [Github actions](https://github.com/jimchen2/valkey/actions/runs/23866167253), and there are a lot of errors as well. It's like, you get these errors even by directly cloning from git. Anyway, I am [submitting a PR](https://github.com/valkey-io/valkey/pull/3435).

Ok, turned out that it was ignored. So I guess pull requests to famous projects aren't a big deal after all. Someone in Valkey approved my PR.

Like 2 months passed and I got very distracted by other things. So I deleted my repos.

## `adlist.c`

Again, in the "Data Structures" course in C, I never had to write a function inside a struct, simply because

- We are never taught about memory and in the classes they don't care what you freed
- The struts or lists always contain one type of data (usually a student, names, classes, etc)

So I thought I learned C very well! Everyone went on to do machine learning or developing useless react frontends, and no one gives a damn about C anymore.

```
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
    int len;
} list;
```

We then have these macros to define the functions

```
#define listSetDupMethod(l,m) ((l)->dup = (m))
#define listSetFreeMethod(l,m) ((l)->free = (m))
#define listSetMatchMethod(l,m) ((l)->match = (m))
```

The other functions are not hard to understand

```
list *listCreate(void);
void listRelease(list *list);
list *listAddNodeHead(list *list, void *value);
list *listAddNodeTail(list *list, void *value);
void listDelNode(list *list, listNode *node);
listIter *listGetIterator(list *list, int direction);
listNode *listNextElement(listIter *iter);
void listReleaseIterator(listIter *iter);
list *listDup(list *orig);
listNode *listSearchKey(list *list, void *key);
listNode *listIndex(list *list, int index);
```

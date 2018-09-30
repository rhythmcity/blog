<center><font face="微软雅黑" size="10">AutoReleasePool</font></center>


```objc

void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}

void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}

```

#### Push

Push 操作传入了一个哨兵占位对象，代码看出其实就是一个 __nil__

```objc

#   define POOL_BOUNDARY nil

static inline void *push()
{
    id *dest;
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}

```
autoreleaseFast

```objc

static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}

```

hotPage

``` objc

#   define EMPTY_POOL_PLACEHOLDER ((id*)1)

static inline AutoreleasePoolPage *hotPage()
{
    AutoreleasePoolPage *result = (AutoreleasePoolPage *)
        tls_get_direct(key);
    if ((id *)result == EMPTY_POOL_PLACEHOLDER) return nil;
    if (result) result->fastcheck();
    return result;
}

```


add   入栈操作

``` objc

id *add(id obj)
{
    assert(!full());
    unprotect();
    id *ret = next;  // faster than `return next-1` because of aliasing
    *next++ = obj;
    protect();
    return ret;
}

```

autoreleaseFullPage  遍历child 有没有未满的 如果没有就创建一个

```objc

static __attribute__((noinline))
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
    // The hot page is full.
    // Step to the next non-full page, adding a new page if necessary.
    // Then add the object to that page.
    assert(page == hotPage());
    assert(page->full()  ||  DebugPoolAllocation);

    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}

```

autoreleaseNoPage  创建一个page 并添加 压栈

```objc

static __attribute__((noinline))
id *autoreleaseNoPage(id obj)
{
    // "No page" could mean no pool has been pushed
    // or an empty placeholder pool has been pushed and has no contents yet
    assert(!hotPage());

    bool pushExtraBoundary = false;
    if (haveEmptyPoolPlaceholder()) {
        // We are pushing a second pool over the empty placeholder pool
        // or pushing the first object into the empty placeholder pool.
        // Before doing that, push a pool boundary on behalf of the pool
        // that is currently represented by the empty placeholder.
        pushExtraBoundary = true;
    }
    else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
        // We are pushing an object with no pool in place,
        // and no-pool debugging was requested by environment.
        _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                     "autoreleased with no pool in place - "
                     "just leaking - break on "
                     "objc_autoreleaseNoPool() to debug",
                     pthread_self(), (void*)obj, object_getClassName(obj));
        objc_autoreleaseNoPool(obj);
        return nil;
    }
    else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
        // We are pushing a pool with no pool in place,
        // and alloc-per-pool debugging was not requested.
        // Install and return the empty pool placeholder.
        return setEmptyPoolPlaceholder();
    }

    // We are pushing an object or a non-placeholder'd pool.

    // Install the first page.
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);

    // Push a boundary on behalf of the previously-placeholder'd pool.
    if (pushExtraBoundary) {
        page->add(POOL_BOUNDARY);
    }

    // Push the requested object or pool.
    return page->add(obj);
}

```

#### Pop

如果token 等于 **EMPTY_POOL_PLACEHOLDER** 检查是否有 **hotpage** 如果有重新执行**pop**
如果没有将 **hotpage** 设置为 **nil**

```objc

static inline void pop(void *token)
{
    AutoreleasePoolPage *page;
    id *stop;

    if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
        // Popping the top-level placeholder pool.
        if (hotPage()) {
            // Pool was used. Pop its contents normally.
            // Pool pages remain allocated for re-use as usual.
            pop(coldPage()->begin());
        } else {
            // Pool was never used. Clear the placeholder.
            setHotPage(nil);
        }
        return;
    }

    page = pageForPointer(token);
    stop = (id *)token;


    if (*stop != POOL_BOUNDARY) {
        if (stop == page->begin()  &&  !page->parent) {
            // Start of coldest page may correctly not be POOL_BOUNDARY:
            // 1. top-level pool is popped, leaving the cold page in place
            // 2. an object is autoreleased with no pool
        } else {
            // Error. For bincompat purposes this is not
            // fatal in executables built with old SDKs.
            return badPop(token);
        }
    }

    if (PrintPoolHiwat) printHiwat();

    page->releaseUntil(stop);

    // memory: delete empty children
    if (DebugPoolAllocation  &&  page->empty()) {
        // special case: delete everything during page-per-pool debugging
        AutoreleasePoolPage *parent = page->parent;
        page->kill();
        setHotPage(parent);
    } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
        // special case: delete everything for pop(top)
        // when debugging missing autorelease pools
        page->kill();
        setHotPage(nil);
    }
    else if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}

```
pageForPointer 获取 token 所在的 page

```objc

static AutoreleasePoolPage *pageForPointer(const void *p)
{
    return pageForPointer((uintptr_t)p);
}

static AutoreleasePoolPage *pageForPointer(uintptr_t p)
{
    AutoreleasePoolPage *result;
    uintptr_t offset = p % SIZE;

    assert(offset >= sizeof(AutoreleasePoolPage));

    result = (AutoreleasePoolPage *)(p - offset);
    result->fastcheck();

    return result;
}

```

 此处判断如果token 不是哨兵对象（也就是不是nil）则判断如果 page是第一个的话不做处理属于正常现象相当于我还没有 push  就进行了 pop ， 如果不是的话 则认为是错误操作进入 **badPop** 流程


 ```objc

 if (*stop != POOL_BOUNDARY) {
    if (stop == page->begin()  &&  !page->parent) {
        // Start of coldest page may correctly not be POOL_BOUNDARY:
        // 1. top-level pool is popped, leaving the cold page in place
        // 2. an object is autoreleased with no pool
    } else {
        // Error. For bincompat purposes this is not
        // fatal in executables built with old SDKs.
        return badPop(token);
    }
}

```

 badPop 在新版的SDK (10_12, 10_0, 10_0, 3_0, 2_0) 会进入 fatal

```objc

static void badPop(void *token)
{
    // Error. For bincompat purposes this is not
    // fatal in executables built with old SDKs.

    if (DebugPoolAllocation || sdkIsAtLeast(10_12, 10_0, 10_0, 3_0, 2_0)) {
        // OBJC_DEBUG_POOL_ALLOCATION or new SDK. Bad pop is fatal.
        _objc_fatal
            ("Invalid or prematurely-freed autorelease pool %p.", token);
    }

    // Old SDK. Bad pop is warned once.
    static bool complained = false;
    if (!complained) {
        complained = true;
        _objc_inform_now_and_on_crash
            ("Invalid or prematurely-freed autorelease pool %p. "
             "Set a breakpoint on objc_autoreleasePoolInvalid to debug. "
             "Proceeding anyway because the app is old "
             "(SDK version " SDK_FORMAT "). Memory errors are likely.",
                 token, FORMAT_SDK(sdkVersion()));
    }
    objc_autoreleasePoolInvalid(token);
}

```

上面判断走完 接下来进去正常的pop 流程


```objc

void releaseUntil(id *stop)
{
    // Not recursive: we don't want to blow out the stack
    // if a thread accumulates a stupendous amount of garbage

    while (this->next != stop) {
        // Restart from hotPage() every time, in case -release
        // autoreleased more objects
        AutoreleasePoolPage *page = hotPage();

        // fixme I think this `while` can be `if`, but I can't prove it
        while (page->empty()) {
            page = page->parent;
            setHotPage(page);
        }

        page->unprotect();
        id obj = *--page->next;
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
        page->protect();

        if (obj != POOL_BOUNDARY) {
            objc_release(obj);
        }
    }

    setHotPage(this);

#if DEBUG
    // we expect any children to be completely empty
    for (AutoreleasePoolPage *page = child; page; page = page->child) {
        assert(page->empty());
    }
#endif
}

```

可以看出来只要的逻辑是 遍历当前的page的对象 如果当前的对象不是哨兵对象 **POOL_BOUNDARY** 执行 **objc_release();** 操作  如果是 则停止

接下来的操作是 删除那些为空的page


```objc
// memory: delete empty children
       if (DebugPoolAllocation  &&  page->empty()) {
           // special case: delete everything during page-per-pool debugging
           AutoreleasePoolPage *parent = page->parent;
           page->kill();
           setHotPage(parent);
       } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
           // special case: delete everything for pop(top)
           // when debugging missing autorelease pools
           page->kill();
           setHotPage(nil);
       }
       else if (page->child) {
           // hysteresis: keep one empty child if page is more than half full
           if (page->lessThanHalfFull()) {
               page->child->kill();
           }
           else if (page->child->child) {
               page->child->child->kill();
           }
       }
```

kill 方法

```objc

void kill()
 {
     // Not recursive: we don't want to blow out the stack
     // if a thread accumulates a stupendous amount of garbage
     AutoreleasePoolPage *page = this;
     while (page->child) page = page->child;

     AutoreleasePoolPage *deathptr;
     do {
         deathptr = page;
         page = page->parent;
         if (page) {
             page->unprotect();
             page->child = nil;
             page->protect();
         }
         delete deathptr;
     } while (deathptr != this);
 }

```

#### 总结


>AutoreleasePool是 AutoreleasePoolPage 的组成的双向链表维护一个栈， autorelease 的对象记录在这个栈中；
>使用 POOL_BOUNDARY 也就是 nil  哨兵对象来对自动释放池进行分隔 每次插入POOL_BOUNDARY进行标记，pop操作时释放最近POOL_BOUNDARY上的所有对象。

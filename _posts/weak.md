<center><font face="微软雅黑" size="10">weak</font></center>


``` objc

{
    id obj1 = [[NSObject alloc] init];
    id __weak obj2 = obj1;
}

```
上述代码 在编译期会变成如下：

```objc

id obj2;
objc_initWeak(&obj2, obj1);
objc_destroyWeak(&obj2);

```

objc_initWeak

```objc

// 定义三个布尔值 传入storeweak
ontHaveOld = false
DoHaveNew = true
DoCrashIfDeallocating = true

id
objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}

```

objc_destroyWeak

```objc

DoHaveOld = true
DontHaveNew = false
DontCrashIfDeallocating = false

void
objc_destroyWeak(id *location)
{
    (void)storeWeak<DoHaveOld, DontHaveNew, DontCrashIfDeallocating>
        (location, nil);
}

```

上面的两段代码都调用了  **storeWeak** 代码

storeWeak

```objc

// 更新一个弱引用变量
// 如果 HaveOld 是 true, 变量是个有效值，需要被及时清理。变量可以为 nil。
// 如果 HaveNew 是 true, 需要一个新的 value 来替换变量。变量可以为 nil
// 如果crashifdeallocation 是 ture ，那么如果 newObj 是 deallocating，或者 newObj 的类不支持弱引用，则该进程就会停止。
// 如果crashifdeallocation 是 false，那么 nil 会被存储。

template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating>
static id
storeWeak(id *location, objc_object *newObj)
{
    assert(haveOld  ||  haveNew);
    if (!haveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // 为旧的和新的值获取锁。
    // Order by lock address to prevent lock ordering problems.
    //
    // Retry if the old value changes underneath us.
 retry:
    if (haveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no
    // weakly-referenced object has an un-+initialized isa.
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&
            !((objc_class *)cls)->isInitialized())
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // Clean up old value, if any.
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any.
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location,
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }

    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}

```


***

#### dealloc


```objc

- (void)dealloc {
    _objc_rootDealloc(self);
}


void
_objc_rootDealloc(id obj)
{
    assert(obj);

    obj->rootDealloc();
}


inline void
objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;  // fixme necessary?

    if (fastpath(isa.nonpointer  &&
                 !isa.weakly_referenced  &&
                 !isa.has_assoc  &&
                 !isa.has_cxx_dtor  &&
                 !isa.has_sidetable_rc))
    {
        assert(!sidetable_present());
        free(this);
    }
    else {
        object_dispose((id)this);
    }
}


id
object_dispose(id obj)
{
    if (!obj) return nil;

    objc_destructInstance(obj);
    free(obj);

    return nil;
}


/***********************************************************************
* objc_destructInstance
* Destroys an instance without freeing memory.
* Calls C++ destructors.
* Calls ARC ivar cleanup.
* Removes associative references.
* Returns `obj`. Does nothing if `obj` is nil.
**********************************************************************/
void *objc_destructInstance(id obj)
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj);
        obj->clearDeallocating();
    }

    return obj;
}


inline void
objc_object::clearDeallocating()
{
    if (slowpath(!isa.nonpointer)) {
        // Slow path for raw pointer isa.
        sidetable_clearDeallocating();
    }
    else if (slowpath(isa.weakly_referenced  ||  isa.has_sidetable_rc)) {
        // Slow path for non-pointer isa with weak refs and/or side table data.
        clearDeallocating_slow();
    }

    assert(!sidetable_present());
}


NEVER_INLINE void
objc_object::clearDeallocating_slow()
{
    assert(isa.nonpointer  &&  (isa.weakly_referenced || isa.has_sidetable_rc));

    SideTable& table = SideTables()[this];
    table.lock();
    if (isa.weakly_referenced) {
        weak_clear_no_lock(&table.weak_table, (id)this);
    }
    if (isa.has_sidetable_rc) {
        table.refcnts.erase(this);
    }
    table.unlock();
}


void
objc_object::sidetable_clearDeallocating()
{
    SideTable& table = SideTables()[this];

    // clear any weak table items
    // clear extra retain count and deallocating bit
    // (fixme warn or abort if extra retain count == 0 ?)
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        if (it->second & SIDE_TABLE_WEAKLY_REFERENCED) {
            weak_clear_no_lock(&table.weak_table, (id)this);
        }
        table.refcnts.erase(it);
    }
    table.unlock();
}




```
最终会调用到如下方法 会遍历把置为nil并且删除

```objc

/**
 * Called by dealloc; nils out all weak pointers that point to the
 * provided object so that they can no longer be used.
 *
 * @param weak_table
 * @param referent The object being deallocated.
 */
void
weak_clear_no_lock(weak_table_t *weak_table, id referent_id)
{
    objc_object *referent = (objc_object *)referent_id;

    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;

    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    }
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }

    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n",
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }

    weak_entry_remove(weak_table, entry);
}

```

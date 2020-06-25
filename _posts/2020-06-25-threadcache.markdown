---
layout:     post
title:      "TCMalloc中如何做thread-cache"
subtitle:   "如何实现thread cache"
date:       2020-06-25 16:32:00
author:     "Jfchen"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - tcmalloc
---

### 涉及到的API
- pthread_key_t
- pthread_key_create
- pthread_setspecific
- pthread_getspecific
- pthread_key_delete

### 实现
内部有一个数据结构：ThreadCache, 在初始化的时候会创建pthread_key_t：
```
void ThreadCache::InitTSD() {
  ASSERT(!tsd_inited_);
  pthread_key_create(&heap_key_, DestroyThreadCache);
  tsd_inited_ = true;
}
DestroyThreadCache为ThreadCache清理函数
```

在调用malloc时，查找当前线程是否有相应的ThreadCache对象，每一个线程对应一个ThreadCache对象，该对象中缓存了可以分配出去的空闲内存，对于小内存分配，直接从该对象中取内存，因此，小内存分配没有加解锁的操作，这也是TCMalloc高效的原因之一：
```
inline ThreadCache* ABSL_ATTRIBUTE_ALWAYS_INLINE
ThreadCache::GetCacheIfPresent() {
#ifdef ABSL_HAVE_TLS
  // __thread is faster
  return thread_local_data_;
#else
  return tsd_inited_
             ? reinterpret_cast<ThreadCache*>(pthread_getspecific(heap_key_))
             : nullptr;
#endif
}
```

如果当前线程没有对应的ThreadCache对象，则需要创建该对象：
```
ThreadCache* ThreadCache::CreateCacheIfNecessary() {
    ...
      {
    absl::base_internal::SpinLockHolder h(&pageheap_lock);
    const pthread_t me = pthread_self();

    // This may be a recursive malloc call from pthread_setspecific()
    // In that case, the heap for this thread has already been created
    // and added to the linked list.  So we search for that first.
    if (maybe_reentrant) {
      for (ThreadCache* h = thread_heaps_; h != nullptr; h = h->next_) {
        if (h->tid_ == me) {
          heap = h;
          break;
        }
      }
    }

    if (heap == nullptr) {
      heap = NewHeap(me); // 创建ThreadCache对象
    }
  }
   ...
   pthread_setspecific(heap_key_, heap); // 进行thread-local存储，下次可以获取ThreadCache对象
   ...
}

```

### Notes
在ThreadCache中，定义了一个static pthread_key_t变量，所有的线程共享该变量，尽管该变量是同一个值，但是不同的线程中存储的值不一样，因此通过pthread_getspecific获取的值也是各自线程中存储的值。

与c++11中的thread_local关键字同义。
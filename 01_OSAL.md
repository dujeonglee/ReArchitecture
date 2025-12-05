# Layer 1: OS Abstraction Layer (OSAL)

## 1. 개요

### 1.1 역할

OS 종속적인 API를 추상화하여 드라이버 Core 로직이 플랫폼 독립적으로 동작하도록 한다.

### 1.2 설계 목표

- Linux/Windows 간 코드 공유 극대화
- OS API 변경 시 영향 범위 최소화
- 컴파일 타임 플랫폼 선택

---

## 2. 모듈 구성

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: OS Abstraction Layer (OSAL)                                          │
│                                                                                 │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │ osal_memory │ │ osal_sync   │ │ osal_time   │ │ osal_thread │               │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘               │
│  ┌─────────────┐ ┌─────────────┐                                               │
│  │ osal_atomic │ │ osal_debug  │                                               │
│  └─────────────┘ └─────────────┘                                               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 모듈 상세

### 3.1 osal_memory

| 항목 | 내용 |
|------|------|
| **역할** | 메모리 할당/해제 추상화 |
| **책임** | 동적 메모리 관리, DMA 메모리 할당, 메모리 풀 관리 |
| **의존성** | 없음 (OS API 직접 사용) |

#### 인터페이스

```c
/* osal_memory.h */

/**
 * 일반 메모리 할당
 */
void *osal_mem_alloc(size_t size);
void *osal_mem_zalloc(size_t size);  /* zero-initialized */
void osal_mem_free(void *ptr);

/**
 * DMA 메모리 (버스 통신용)
 */
void *osal_dma_alloc(size_t size, dma_addr_t *dma_handle);
void osal_dma_free(void *ptr, size_t size, dma_addr_t dma_handle);

/**
 * 메모리 조작
 */
void *osal_memcpy(void *dest, const void *src, size_t n);
void *osal_memset(void *s, int c, size_t n);
int osal_memcmp(const void *s1, const void *s2, size_t n);
```

#### Linux 구현 예시

```c
/* osal_memory_linux.c */

#include <linux/slab.h>
#include <linux/dma-mapping.h>

void *osal_mem_alloc(size_t size)
{
    return kmalloc(size, GFP_KERNEL);
}

void *osal_mem_zalloc(size_t size)
{
    return kzalloc(size, GFP_KERNEL);
}

void osal_mem_free(void *ptr)
{
    kfree(ptr);
}

void *osal_dma_alloc(size_t size, dma_addr_t *dma_handle)
{
    /* dev는 전역 또는 파라미터로 전달 */
    return dma_alloc_coherent(g_dev, size, dma_handle, GFP_KERNEL);
}

void osal_dma_free(void *ptr, size_t size, dma_addr_t dma_handle)
{
    dma_free_coherent(g_dev, size, ptr, dma_handle);
}
```

#### Windows 구현 예시

```c
/* osal_memory_windows.c */

#include <wdm.h>

void *osal_mem_alloc(size_t size)
{
    return ExAllocatePoolWithTag(NonPagedPool, size, 'WIFI');
}

void *osal_mem_zalloc(size_t size)
{
    void *ptr = ExAllocatePoolWithTag(NonPagedPool, size, 'WIFI');
    if (ptr)
        RtlZeroMemory(ptr, size);
    return ptr;
}

void osal_mem_free(void *ptr)
{
    if (ptr)
        ExFreePoolWithTag(ptr, 'WIFI');
}
```

---

### 3.2 osal_sync

| 항목 | 내용 |
|------|------|
| **역할** | 동기화 프리미티브 추상화 |
| **책임** | Mutex, Spinlock, Semaphore, Completion 관리 |
| **의존성** | 없음 |

#### 인터페이스

```c
/* osal_sync.h */

/**
 * Mutex - sleeping lock
 */
typedef struct osal_mutex osal_mutex_t;

int osal_mutex_init(osal_mutex_t *mutex);
void osal_mutex_destroy(osal_mutex_t *mutex);
void osal_mutex_lock(osal_mutex_t *mutex);
void osal_mutex_unlock(osal_mutex_t *mutex);
int osal_mutex_trylock(osal_mutex_t *mutex);  /* 0: 성공, -1: 실패 */

/**
 * Spinlock - non-sleeping lock
 */
typedef struct osal_spinlock osal_spinlock_t;

void osal_spin_lock_init(osal_spinlock_t *lock);
void osal_spin_lock(osal_spinlock_t *lock);
void osal_spin_unlock(osal_spinlock_t *lock);
void osal_spin_lock_bh(osal_spinlock_t *lock);      /* bottom-half disable */
void osal_spin_unlock_bh(osal_spinlock_t *lock);
void osal_spin_lock_irqsave(osal_spinlock_t *lock, unsigned long *flags);
void osal_spin_unlock_irqrestore(osal_spinlock_t *lock, unsigned long flags);

/**
 * Completion - 이벤트 대기
 */
typedef struct osal_completion osal_completion_t;

void osal_completion_init(osal_completion_t *comp);
void osal_completion_wait(osal_completion_t *comp);
int osal_completion_wait_timeout(osal_completion_t *comp, unsigned long timeout_ms);
void osal_completion_complete(osal_completion_t *comp);
void osal_completion_complete_all(osal_completion_t *comp);
void osal_completion_reinit(osal_completion_t *comp);

/**
 * Semaphore
 */
typedef struct osal_semaphore osal_semaphore_t;

void osal_sema_init(osal_semaphore_t *sem, int val);
void osal_sema_down(osal_semaphore_t *sem);
int osal_sema_down_timeout(osal_semaphore_t *sem, unsigned long timeout_ms);
void osal_sema_up(osal_semaphore_t *sem);
```

#### Linux 구현 예시

```c
/* osal_sync_linux.c */

#include <linux/mutex.h>
#include <linux/spinlock.h>
#include <linux/completion.h>

struct osal_mutex {
    struct mutex m;
};

int osal_mutex_init(osal_mutex_t *mutex)
{
    mutex_init(&mutex->m);
    return 0;
}

void osal_mutex_lock(osal_mutex_t *mutex)
{
    mutex_lock(&mutex->m);
}

void osal_mutex_unlock(osal_mutex_t *mutex)
{
    mutex_unlock(&mutex->m);
}

/* Spinlock */
struct osal_spinlock {
    spinlock_t lock;
};

void osal_spin_lock_init(osal_spinlock_t *lock)
{
    spin_lock_init(&lock->lock);
}

void osal_spin_lock_irqsave(osal_spinlock_t *lock, unsigned long *flags)
{
    spin_lock_irqsave(&lock->lock, *flags);
}

void osal_spin_unlock_irqrestore(osal_spinlock_t *lock, unsigned long flags)
{
    spin_unlock_irqrestore(&lock->lock, flags);
}

/* Completion */
struct osal_completion {
    struct completion c;
};

void osal_completion_init(osal_completion_t *comp)
{
    init_completion(&comp->c);
}

int osal_completion_wait_timeout(osal_completion_t *comp, unsigned long timeout_ms)
{
    unsigned long jiffies_timeout = msecs_to_jiffies(timeout_ms);
    long ret = wait_for_completion_timeout(&comp->c, jiffies_timeout);
    return ret ? 0 : -ETIMEDOUT;
}

void osal_completion_complete(osal_completion_t *comp)
{
    complete(&comp->c);
}
```

#### Windows 구현 예시

```c
/* osal_sync_windows.c */

#include <wdm.h>

struct osal_mutex {
    KMUTEX m;
};

int osal_mutex_init(osal_mutex_t *mutex)
{
    KeInitializeMutex(&mutex->m, 0);
    return 0;
}

void osal_mutex_lock(osal_mutex_t *mutex)
{
    KeWaitForSingleObject(&mutex->m, Executive, KernelMode, FALSE, NULL);
}

void osal_mutex_unlock(osal_mutex_t *mutex)
{
    KeReleaseMutex(&mutex->m, FALSE);
}

/* Spinlock */
struct osal_spinlock {
    KSPIN_LOCK lock;
    KIRQL old_irql;
};

void osal_spin_lock_init(osal_spinlock_t *lock)
{
    KeInitializeSpinLock(&lock->lock);
}

void osal_spin_lock_irqsave(osal_spinlock_t *lock, unsigned long *flags)
{
    KeAcquireSpinLock(&lock->lock, &lock->old_irql);
    *flags = (unsigned long)lock->old_irql;
}

void osal_spin_unlock_irqrestore(osal_spinlock_t *lock, unsigned long flags)
{
    KeReleaseSpinLock(&lock->lock, (KIRQL)flags);
}

/* Completion using KEVENT */
struct osal_completion {
    KEVENT event;
};

void osal_completion_init(osal_completion_t *comp)
{
    KeInitializeEvent(&comp->event, NotificationEvent, FALSE);
}

int osal_completion_wait_timeout(osal_completion_t *comp, unsigned long timeout_ms)
{
    LARGE_INTEGER timeout;
    timeout.QuadPart = -10000LL * timeout_ms;  /* 100ns 단위, 음수는 상대 시간 */
    
    NTSTATUS status = KeWaitForSingleObject(&comp->event, Executive, 
                                             KernelMode, FALSE, &timeout);
    return (status == STATUS_SUCCESS) ? 0 : -ETIMEDOUT;
}

void osal_completion_complete(osal_completion_t *comp)
{
    KeSetEvent(&comp->event, IO_NO_INCREMENT, FALSE);
}
```

---

### 3.3 osal_time

| 항목 | 내용 |
|------|------|
| **역할** | 시간 관련 기능 추상화 |
| **책임** | 현재 시간, 지연, 타이머 관리 |
| **의존성** | 없음 |

#### 인터페이스

```c
/* osal_time.h */

/**
 * 현재 시간 조회
 */
uint64_t osal_get_time_ms(void);      /* milliseconds since boot */
uint64_t osal_get_time_us(void);      /* microseconds since boot */
uint64_t osal_get_time_ns(void);      /* nanoseconds since boot */

/**
 * 지연 (sleep 가능, 컨텍스트 스위칭 발생)
 */
void osal_msleep(unsigned int ms);
void osal_usleep_range(unsigned long min_us, unsigned long max_us);

/**
 * 지연 (busy wait, 인터럽트 컨텍스트에서 사용)
 */
void osal_mdelay(unsigned int ms);
void osal_udelay(unsigned int us);

/**
 * 타이머
 */
typedef struct osal_timer osal_timer_t;
typedef void (*osal_timer_callback_t)(void *context);

int osal_timer_init(osal_timer_t *timer, osal_timer_callback_t cb, void *context);
void osal_timer_destroy(osal_timer_t *timer);
int osal_timer_start(osal_timer_t *timer, unsigned long interval_ms, bool periodic);
void osal_timer_stop(osal_timer_t *timer);
bool osal_timer_is_active(osal_timer_t *timer);
int osal_timer_modify(osal_timer_t *timer, unsigned long interval_ms);
```

#### Linux 구현 예시

```c
/* osal_time_linux.c */

#include <linux/jiffies.h>
#include <linux/delay.h>
#include <linux/timer.h>
#include <linux/ktime.h>

uint64_t osal_get_time_ms(void)
{
    return ktime_to_ms(ktime_get_boottime());
}

uint64_t osal_get_time_us(void)
{
    return ktime_to_us(ktime_get_boottime());
}

void osal_msleep(unsigned int ms)
{
    msleep(ms);
}

void osal_usleep_range(unsigned long min_us, unsigned long max_us)
{
    usleep_range(min_us, max_us);
}

void osal_mdelay(unsigned int ms)
{
    mdelay(ms);
}

void osal_udelay(unsigned int us)
{
    udelay(us);
}

/* Timer */
struct osal_timer {
    struct timer_list timer;
    osal_timer_callback_t callback;
    void *context;
    unsigned long interval_ms;
    bool periodic;
};

static void timer_callback_wrapper(struct timer_list *t)
{
    osal_timer_t *timer = from_timer(timer, t, timer);
    
    if (timer->callback)
        timer->callback(timer->context);
    
    if (timer->periodic)
        mod_timer(&timer->timer, jiffies + msecs_to_jiffies(timer->interval_ms));
}

int osal_timer_init(osal_timer_t *timer, osal_timer_callback_t cb, void *context)
{
    timer->callback = cb;
    timer->context = context;
    timer->periodic = false;
    timer_setup(&timer->timer, timer_callback_wrapper, 0);
    return 0;
}

int osal_timer_start(osal_timer_t *timer, unsigned long interval_ms, bool periodic)
{
    timer->interval_ms = interval_ms;
    timer->periodic = periodic;
    mod_timer(&timer->timer, jiffies + msecs_to_jiffies(interval_ms));
    return 0;
}

void osal_timer_stop(osal_timer_t *timer)
{
    del_timer_sync(&timer->timer);
}

bool osal_timer_is_active(osal_timer_t *timer)
{
    return timer_pending(&timer->timer);
}
```

---

### 3.4 osal_thread

| 항목 | 내용 |
|------|------|
| **역할** | 스레드/워크 관리 추상화 |
| **책임** | 커널 스레드, Workqueue 관리 |
| **의존성** | osal_sync (내부에서 사용) |

#### 인터페이스

```c
/* osal_thread.h */

/**
 * Workqueue
 */
typedef struct osal_workqueue osal_workqueue_t;
typedef struct osal_work osal_work_t;
typedef void (*osal_work_func_t)(osal_work_t *work);

/* container_of 매크로 */
#define osal_work_to_context(work_ptr, type, member) \
    container_of(work_ptr, type, member)

osal_workqueue_t *osal_workqueue_create(const char *name);
osal_workqueue_t *osal_workqueue_create_singlethread(const char *name);
void osal_workqueue_destroy(osal_workqueue_t *wq);
void osal_workqueue_flush(osal_workqueue_t *wq);

void osal_work_init(osal_work_t *work, osal_work_func_t func);
int osal_work_schedule(osal_workqueue_t *wq, osal_work_t *work);
int osal_work_schedule_delayed(osal_workqueue_t *wq, osal_work_t *work, 
                               unsigned long delay_ms);
bool osal_work_cancel(osal_work_t *work);
bool osal_work_cancel_sync(osal_work_t *work);
void osal_work_flush(osal_work_t *work);
bool osal_work_pending(osal_work_t *work);

/**
 * Kernel Thread
 */
typedef struct osal_thread osal_thread_t;
typedef int (*osal_thread_func_t)(void *context);

osal_thread_t *osal_thread_create(const char *name, osal_thread_func_t func, 
                                   void *context);
void osal_thread_stop(osal_thread_t *thread);
bool osal_thread_should_stop(void);
void osal_thread_set_priority(osal_thread_t *thread, int priority);
```

#### Linux 구현 예시

```c
/* osal_thread_linux.c */

#include <linux/workqueue.h>
#include <linux/kthread.h>

struct osal_workqueue {
    struct workqueue_struct *wq;
};

struct osal_work {
    struct work_struct work;
    struct delayed_work dwork;
    osal_work_func_t func;
    bool is_delayed;
};

static void work_callback_wrapper(struct work_struct *w)
{
    osal_work_t *work = container_of(w, osal_work_t, work);
    if (work->func)
        work->func(work);
}

static void delayed_work_callback_wrapper(struct work_struct *w)
{
    osal_work_t *work = container_of(w, osal_work_t, dwork.work);
    if (work->func)
        work->func(work);
}

osal_workqueue_t *osal_workqueue_create(const char *name)
{
    osal_workqueue_t *wq = osal_mem_zalloc(sizeof(*wq));
    if (!wq)
        return NULL;
    
    wq->wq = alloc_workqueue(name, WQ_MEM_RECLAIM | WQ_HIGHPRI, 0);
    if (!wq->wq) {
        osal_mem_free(wq);
        return NULL;
    }
    
    return wq;
}

void osal_work_init(osal_work_t *work, osal_work_func_t func)
{
    work->func = func;
    work->is_delayed = false;
    INIT_WORK(&work->work, work_callback_wrapper);
    INIT_DELAYED_WORK(&work->dwork, delayed_work_callback_wrapper);
}

int osal_work_schedule(osal_workqueue_t *wq, osal_work_t *work)
{
    work->is_delayed = false;
    return queue_work(wq->wq, &work->work) ? 0 : -EBUSY;
}

int osal_work_schedule_delayed(osal_workqueue_t *wq, osal_work_t *work, 
                               unsigned long delay_ms)
{
    work->is_delayed = true;
    return queue_delayed_work(wq->wq, &work->dwork, 
                              msecs_to_jiffies(delay_ms)) ? 0 : -EBUSY;
}

/* Kernel Thread */
struct osal_thread {
    struct task_struct *task;
    osal_thread_func_t func;
    void *context;
};

osal_thread_t *osal_thread_create(const char *name, osal_thread_func_t func, 
                                   void *context)
{
    osal_thread_t *thread = osal_mem_zalloc(sizeof(*thread));
    if (!thread)
        return NULL;
    
    thread->func = func;
    thread->context = context;
    thread->task = kthread_run(func, context, "%s", name);
    
    if (IS_ERR(thread->task)) {
        osal_mem_free(thread);
        return NULL;
    }
    
    return thread;
}

void osal_thread_stop(osal_thread_t *thread)
{
    if (thread && thread->task) {
        kthread_stop(thread->task);
        osal_mem_free(thread);
    }
}

bool osal_thread_should_stop(void)
{
    return kthread_should_stop();
}
```

---

### 3.5 osal_atomic

| 항목 | 내용 |
|------|------|
| **역할** | 원자적 연산 추상화 |
| **책임** | Atomic 변수 연산 |
| **의존성** | 없음 |

#### 인터페이스

```c
/* osal_atomic.h */

typedef struct osal_atomic osal_atomic_t;

void osal_atomic_set(osal_atomic_t *v, int i);
int osal_atomic_read(osal_atomic_t *v);
void osal_atomic_inc(osal_atomic_t *v);
void osal_atomic_dec(osal_atomic_t *v);
int osal_atomic_inc_return(osal_atomic_t *v);
int osal_atomic_dec_return(osal_atomic_t *v);
bool osal_atomic_dec_and_test(osal_atomic_t *v);  /* returns true if result is 0 */
int osal_atomic_add_return(int i, osal_atomic_t *v);
int osal_atomic_sub_return(int i, osal_atomic_t *v);
int osal_atomic_cmpxchg(osal_atomic_t *v, int old_val, int new_val);

/* 64-bit atomic */
typedef struct osal_atomic64 osal_atomic64_t;

void osal_atomic64_set(osal_atomic64_t *v, int64_t i);
int64_t osal_atomic64_read(osal_atomic64_t *v);
void osal_atomic64_inc(osal_atomic64_t *v);
int64_t osal_atomic64_add_return(int64_t i, osal_atomic64_t *v);
```

#### Linux 구현 예시

```c
/* osal_atomic_linux.c */

#include <linux/atomic.h>

struct osal_atomic {
    atomic_t val;
};

void osal_atomic_set(osal_atomic_t *v, int i)
{
    atomic_set(&v->val, i);
}

int osal_atomic_read(osal_atomic_t *v)
{
    return atomic_read(&v->val);
}

void osal_atomic_inc(osal_atomic_t *v)
{
    atomic_inc(&v->val);
}

int osal_atomic_inc_return(osal_atomic_t *v)
{
    return atomic_inc_return(&v->val);
}

bool osal_atomic_dec_and_test(osal_atomic_t *v)
{
    return atomic_dec_and_test(&v->val);
}

int osal_atomic_cmpxchg(osal_atomic_t *v, int old_val, int new_val)
{
    return atomic_cmpxchg(&v->val, old_val, new_val);
}
```

---

### 3.6 osal_debug

| 항목 | 내용 |
|------|------|
| **역할** | 디버그/로깅 추상화 |
| **책임** | 로그 출력, Assert, 덤프 |
| **의존성** | 없음 |

#### 인터페이스

```c
/* osal_debug.h */

/**
 * Log levels
 */
enum osal_log_level {
    OSAL_LOG_ERROR = 0,
    OSAL_LOG_WARN,
    OSAL_LOG_INFO,
    OSAL_LOG_DEBUG,
    OSAL_LOG_VERBOSE,
};

/**
 * 런타임 로그 레벨 설정
 */
void osal_set_log_level(enum osal_log_level level);
enum osal_log_level osal_get_log_level(void);

/**
 * 로그 출력
 */
void osal_log(enum osal_log_level level, const char *fmt, ...);
void osal_log_ratelimited(enum osal_log_level level, const char *fmt, ...);

/**
 * Convenience macros
 */
#define OSAL_ERR(fmt, ...)   osal_log(OSAL_LOG_ERROR, "[ERR] " fmt, ##__VA_ARGS__)
#define OSAL_WARN(fmt, ...)  osal_log(OSAL_LOG_WARN, "[WARN] " fmt, ##__VA_ARGS__)
#define OSAL_INFO(fmt, ...)  osal_log(OSAL_LOG_INFO, "[INFO] " fmt, ##__VA_ARGS__)
#define OSAL_DBG(fmt, ...)   osal_log(OSAL_LOG_DEBUG, "[DBG] " fmt, ##__VA_ARGS__)
#define OSAL_VERB(fmt, ...)  osal_log(OSAL_LOG_VERBOSE, "[VERB] " fmt, ##__VA_ARGS__)

/**
 * Function enter/exit tracing
 */
#define OSAL_ENTER() OSAL_VERB("%s: enter\n", __func__)
#define OSAL_EXIT()  OSAL_VERB("%s: exit\n", __func__)

/**
 * Assert
 */
#define OSAL_ASSERT(cond) \
    do { \
        if (unlikely(!(cond))) \
            osal_assert_fail(__FILE__, __LINE__, __func__, #cond); \
    } while(0)

#define OSAL_BUG_ON(cond) \
    do { \
        if (unlikely(cond)) \
            osal_bug(__FILE__, __LINE__, __func__, #cond); \
    } while(0)

void osal_assert_fail(const char *file, int line, const char *func, 
                      const char *cond);
void osal_bug(const char *file, int line, const char *func, const char *cond);

/**
 * Hex dump
 */
void osal_hex_dump(const char *prefix, const void *buf, size_t len);
void osal_hex_dump_debug(const char *prefix, const void *buf, size_t len);
```

#### Linux 구현 예시

```c
/* osal_debug_linux.c */

#include <linux/printk.h>
#include <linux/ratelimit.h>

static enum osal_log_level g_log_level = OSAL_LOG_INFO;

void osal_set_log_level(enum osal_log_level level)
{
    g_log_level = level;
}

void osal_log(enum osal_log_level level, const char *fmt, ...)
{
    va_list args;
    
    if (level > g_log_level)
        return;
    
    va_start(args, fmt);
    
    switch (level) {
    case OSAL_LOG_ERROR:
        vprintk(KERN_ERR "wifi: " fmt, args);
        break;
    case OSAL_LOG_WARN:
        vprintk(KERN_WARNING "wifi: " fmt, args);
        break;
    case OSAL_LOG_INFO:
        vprintk(KERN_INFO "wifi: " fmt, args);
        break;
    case OSAL_LOG_DEBUG:
    case OSAL_LOG_VERBOSE:
        vprintk(KERN_DEBUG "wifi: " fmt, args);
        break;
    }
    
    va_end(args);
}

void osal_assert_fail(const char *file, int line, const char *func,
                      const char *cond)
{
    pr_err("wifi: ASSERT FAILED: %s at %s:%d (%s)\n", cond, file, line, func);
    WARN_ON(1);
}

void osal_bug(const char *file, int line, const char *func, const char *cond)
{
    pr_err("wifi: BUG: %s at %s:%d (%s)\n", cond, file, line, func);
    BUG();
}

void osal_hex_dump(const char *prefix, const void *buf, size_t len)
{
    print_hex_dump(KERN_INFO, prefix, DUMP_PREFIX_OFFSET, 16, 1, buf, len, true);
}
```

---

## 4. 디렉토리 구조

```
osal/
├── include/
│   ├── osal.h              # 모든 OSAL 헤더를 include하는 통합 헤더
│   ├── osal_memory.h
│   ├── osal_sync.h
│   ├── osal_time.h
│   ├── osal_thread.h
│   ├── osal_atomic.h
│   └── osal_debug.h
│
├── linux/
│   ├── osal_memory_linux.c
│   ├── osal_sync_linux.c
│   ├── osal_time_linux.c
│   ├── osal_thread_linux.c
│   ├── osal_atomic_linux.c
│   └── osal_debug_linux.c
│
└── windows/
    ├── osal_memory_windows.c
    ├── osal_sync_windows.c
    ├── osal_time_windows.c
    ├── osal_thread_windows.c
    ├── osal_atomic_windows.c
    └── osal_debug_windows.c
```

---

## 5. 주목할 특성

### 5.1 컴파일 타임 플랫폼 선택

**Makefile (Linux):**

```makefile
OSAL_SRCS := osal/linux/osal_memory_linux.c \
             osal/linux/osal_sync_linux.c \
             osal/linux/osal_time_linux.c \
             osal/linux/osal_thread_linux.c \
             osal/linux/osal_atomic_linux.c \
             osal/linux/osal_debug_linux.c

obj-m += wifi_driver.o
wifi_driver-y += $(OSAL_SRCS:.c=.o)
```

### 5.2 의존성

```
┌─────────────────────────────────────────┐
│          상위 모든 레이어               │
│  (Kernel IF, Service, Core, FW Msg)     │
└────────────────┬────────────────────────┘
                 │ 사용
                 ▼
┌─────────────────────────────────────────┐
│              OSAL                        │
│  (osal_memory, osal_sync, osal_time,    │
│   osal_thread, osal_atomic, osal_debug) │
└────────────────┬────────────────────────┘
                 │ 사용
                 ▼
┌─────────────────────────────────────────┐
│           OS Kernel API                  │
│     (Linux: kmalloc, mutex, ...)        │
│     (Windows: ExAllocatePool, ...)      │
└─────────────────────────────────────────┘
```

- **외부 의존성**: OS Kernel API만 사용
- **내부 의존성**: 없음 (최하위 레이어)
- **의존하는 모듈**: 모든 상위 레이어

### 5.3 Thread-Safety

모든 OSAL 함수는 thread-safe 해야 함:

| 모듈 | Thread-Safety |
|------|---------------|
| osal_memory | ✅ OS가 보장 |
| osal_sync | ✅ 동기화 프리미티브 |
| osal_time | ✅ 읽기 전용 또는 OS 보장 |
| osal_thread | ✅ OS가 보장 |
| osal_atomic | ✅ 원자적 연산 |
| osal_debug | ✅ 로그 레벨만 전역 (race 무해) |

### 5.4 Error Handling 규칙

```c
/* 메모리 할당 실패 */
void *ptr = osal_mem_alloc(size);
if (!ptr)
    return -ENOMEM;  /* 또는 NULL */

/* 타임아웃 */
ret = osal_completion_wait_timeout(&comp, 1000);
if (ret == -ETIMEDOUT)
    /* 타임아웃 처리 */

/* 일반 에러 */
ret = osal_timer_init(&timer, callback, ctx);
if (ret < 0)
    return ret;
```
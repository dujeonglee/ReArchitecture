# Layer 0: Driver Lifecycle Manager - Overview

## 1. 개요

### 1.1 역할

드라이버 전체 생명주기를 관리하는 Orchestration Layer. 초기화/종료 시퀀스, Recovery, Power Management를 담당한다.

### 1.2 설계 목표

- 레이어별 초기화/종료 순서 명확화
- FW 장애 시 자동 복구 (Recovery)
- System suspend/resume 조율
- 에러 상황에서의 graceful degradation

### 1.3 Layer 0 위치

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 0: Driver Lifecycle Manager (Orchestration Layer)                        │
│                                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐              │
│  │    drv_init      │  │   drv_recovery   │  │    drv_power     │              │
│  │    drv_deinit    │  │                  │  │                  │              │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘              │
│            │                    │                    │                          │
│            └────────────────────┼────────────────────┘                          │
│                                 │                                               │
│                                 ▼                                               │
│                    ┌──────────────────────┐                                     │
│                    │   lifecycle_core     │                                     │
│                    │  (상태 머신, 조율)    │                                     │
│                    └──────────────────────┘                                     │
└─────────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  │ 각 레이어 제어
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│    Layer 1      Layer 2      Layer 3      Layer 4      Layer 5      Layer 6    │
│    (OSAL)    (Kernel IF)  (Service)    (Core)     (FW Msg)      (HIP)         │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│    Layer 7: Customer Extensions (Side Layer - Hook으로 연결)                    │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.4 Customer Extension과의 차이

| 측면 | Layer 0 (Lifecycle) | Layer 7 (Customer) |
|------|---------------------|---------------------|
| **역할** | 시스템 제어/조율 | 기능 확장/커스터마이징 |
| **위치** | Core 위 (Orchestrator) | Core 옆 (Side Layer) |
| **방향** | 각 레이어를 호출 (능동) | Hook으로 호출됨 (수동) |
| **호출 시점** | 시스템 이벤트 (probe, remove, suspend) | 런타임 이벤트 (connect, scan) |
| **필수 여부** | 필수 | 선택적 |

---

## 2. 모듈 구성

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 0: Driver Lifecycle Manager                                              │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         lifecycle_core                                   │   │
│  │                    (상태 머신, 레이어 조율)                                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                     │                                           │
│       ┌─────────────────────────────┼─────────────────────────────┐            │
│       ▼                             ▼                             ▼            │
│  ┌─────────────┐             ┌─────────────┐             ┌─────────────┐       │
│  │  drv_init   │             │drv_recovery │             │  drv_power  │       │
│  │  drv_deinit │             │             │             │             │       │
│  └─────────────┘             └─────────────┘             └─────────────┘       │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         layer_registry                                   │   │
│  │                   (레이어 등록, 의존성 관리)                               │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

| 모듈 | 역할 | 문서 |
|------|------|------|
| **lifecycle_core** | 상태 머신, 전체 조율 | 본 문서 |
| **drv_init / drv_deinit** | 초기화/종료 시퀀스 | 08-1_DRV_INIT.md |
| **drv_recovery** | FW 장애 복구 | 08-2_DRV_RECOVERY.md |
| **drv_power** | Suspend/Resume 관리 | 08-3_DRV_POWER.md |
| **layer_registry** | 레이어 등록 및 의존성 | 본 문서 |

---

## 3. 드라이버 상태 머신

### 3.1 상태 정의

```c
/* lifecycle_core.h */

enum drv_state {
    DRV_STATE_UNLOADED = 0,      /* 드라이버 미로드 */
    DRV_STATE_PROBING,           /* probe 진행 중 */
    DRV_STATE_INITIALIZED,       /* 초기화 완료, FW 미로드 */
    DRV_STATE_FW_LOADING,        /* FW 다운로드 중 */
    DRV_STATE_FW_BOOTING,        /* FW 부팅 중 */
    DRV_STATE_RUNNING,           /* 정상 동작 */
    DRV_STATE_SUSPENDING,        /* Suspend 진행 중 */
    DRV_STATE_SUSPENDED,         /* Suspended 상태 */
    DRV_STATE_RESUMING,          /* Resume 진행 중 */
    DRV_STATE_RECOVERING,        /* Recovery 진행 중 */
    DRV_STATE_STOPPING,          /* 종료 진행 중 */
    DRV_STATE_ERROR,             /* 에러 상태 */
};
```

### 3.2 상태 전이 다이어그램

```
                            ┌──────────────┐
                            │  UNLOADED    │
                            └──────┬───────┘
                                   │ probe()
                                   ▼
                            ┌──────────────┐
                            │   PROBING    │
                            └──────┬───────┘
                                   │ init 완료
                                   ▼
                            ┌──────────────┐
                            │ INITIALIZED  │
                            └──────┬───────┘
                                   │ FW 다운로드 시작
                                   ▼
                            ┌──────────────┐
                            │  FW_LOADING  │
                            └──────┬───────┘
                                   │ 다운로드 완료
                                   ▼
                            ┌──────────────┐
                            │  FW_BOOTING  │
                            └──────┬───────┘
                                   │ FW Ready
                                   ▼
    ┌───────────────────────┬──────────────┬───────────────────────┐
    │                       │   RUNNING    │                       │
    │                       └──────┬───────┘                       │
    │                              │                               │
    │ suspend()                    │ FW error                      │ remove()
    ▼                              ▼                               ▼
┌──────────────┐            ┌──────────────┐              ┌──────────────┐
│  SUSPENDING  │            │  RECOVERING  │              │   STOPPING   │
└──────┬───────┘            └──────┬───────┘              └──────┬───────┘
       │                           │                             │
       ▼                           │ 복구 완료                    ▼
┌──────────────┐                   │                      ┌──────────────┐
│  SUSPENDED   │                   └──────────────────────│  UNLOADED    │
└──────┬───────┘                          ▲               └──────────────┘
       │ resume()                         │
       ▼                                  │
┌──────────────┐                          │
│   RESUMING   │──────────────────────────┘
└──────────────┘      복구 완료

                      에러 발생 시
                           │
                           ▼
                    ┌──────────────┐
                    │    ERROR     │
                    └──────────────┘
```

### 3.3 상태 전이 규칙

```c
/* lifecycle_core.c */

/**
 * 유효한 상태 전이 테이블
 */
static const bool valid_transitions[DRV_STATE_MAX][DRV_STATE_MAX] = {
    /* From UNLOADED */
    [DRV_STATE_UNLOADED][DRV_STATE_PROBING] = true,
    
    /* From PROBING */
    [DRV_STATE_PROBING][DRV_STATE_INITIALIZED] = true,
    [DRV_STATE_PROBING][DRV_STATE_ERROR] = true,
    
    /* From INITIALIZED */
    [DRV_STATE_INITIALIZED][DRV_STATE_FW_LOADING] = true,
    [DRV_STATE_INITIALIZED][DRV_STATE_STOPPING] = true,
    
    /* From FW_LOADING */
    [DRV_STATE_FW_LOADING][DRV_STATE_FW_BOOTING] = true,
    [DRV_STATE_FW_LOADING][DRV_STATE_ERROR] = true,
    
    /* From FW_BOOTING */
    [DRV_STATE_FW_BOOTING][DRV_STATE_RUNNING] = true,
    [DRV_STATE_FW_BOOTING][DRV_STATE_ERROR] = true,
    
    /* From RUNNING */
    [DRV_STATE_RUNNING][DRV_STATE_SUSPENDING] = true,
    [DRV_STATE_RUNNING][DRV_STATE_RECOVERING] = true,
    [DRV_STATE_RUNNING][DRV_STATE_STOPPING] = true,
    
    /* From SUSPENDING */
    [DRV_STATE_SUSPENDING][DRV_STATE_SUSPENDED] = true,
    [DRV_STATE_SUSPENDING][DRV_STATE_ERROR] = true,
    
    /* From SUSPENDED */
    [DRV_STATE_SUSPENDED][DRV_STATE_RESUMING] = true,
    [DRV_STATE_SUSPENDED][DRV_STATE_STOPPING] = true,
    
    /* From RESUMING */
    [DRV_STATE_RESUMING][DRV_STATE_RUNNING] = true,
    [DRV_STATE_RESUMING][DRV_STATE_RECOVERING] = true,
    [DRV_STATE_RESUMING][DRV_STATE_ERROR] = true,
    
    /* From RECOVERING */
    [DRV_STATE_RECOVERING][DRV_STATE_RUNNING] = true,
    [DRV_STATE_RECOVERING][DRV_STATE_ERROR] = true,
    [DRV_STATE_RECOVERING][DRV_STATE_STOPPING] = true,
    
    /* From STOPPING */
    [DRV_STATE_STOPPING][DRV_STATE_UNLOADED] = true,
    
    /* From ERROR */
    [DRV_STATE_ERROR][DRV_STATE_RECOVERING] = true,
    [DRV_STATE_ERROR][DRV_STATE_STOPPING] = true,
};
```

---

## 4. Layer Registry

### 4.1 레이어 등록 인터페이스

```c
/* layer_registry.h */

/**
 * 레이어 ID
 */
enum layer_id {
    LAYER_OSAL = 1,              /* Layer 1 */
    LAYER_KERNEL_IF,             /* Layer 2 */
    LAYER_SERVICE,               /* Layer 3 */
    LAYER_CORE,                  /* Layer 4 */
    LAYER_FW_MSG,                /* Layer 5 */
    LAYER_HIP,                   /* Layer 6 */
    LAYER_CUSTOMER,              /* Layer 7 */
    LAYER_MAX,
};

/**
 * 레이어 Ops (각 레이어가 구현)
 */
struct layer_ops {
    const char *name;
    
    /* 초기화/종료 */
    int (*init)(void *platform_data);
    void (*deinit)(void);
    
    /* 시작/중지 (FW 로드 후) */
    int (*start)(void);
    void (*stop)(void);
    
    /* Power */
    int (*suspend)(void);
    int (*resume)(void);
    
    /* Recovery */
    int (*pre_recovery)(void);   /* Recovery 전 정리 */
    int (*post_recovery)(void);  /* Recovery 후 복원 */
    
    /* 상태 조회 */
    bool (*is_ready)(void);
};

/**
 * 레이어 등록
 */
int layer_registry_register(enum layer_id id, const struct layer_ops *ops);
void layer_registry_unregister(enum layer_id id);

/**
 * 레이어 조회
 */
const struct layer_ops *layer_registry_get(enum layer_id id);
```

### 4.2 레이어 의존성

```c
/* layer_registry.c */

/**
 * 레이어 의존성 정의
 * 초기화 순서: 의존하는 레이어가 먼저 초기화되어야 함
 */
static const enum layer_id layer_dependencies[LAYER_MAX][LAYER_MAX] = {
    /* Layer 1 (OSAL): 의존성 없음 */
    [LAYER_OSAL] = { 0 },
    
    /* Layer 2 (Kernel IF): OSAL 의존 */
    [LAYER_KERNEL_IF] = { LAYER_OSAL, 0 },
    
    /* Layer 3 (Service): OSAL, Kernel IF 의존 */
    [LAYER_SERVICE] = { LAYER_OSAL, LAYER_KERNEL_IF, 0 },
    
    /* Layer 4 (Core): OSAL, Service 의존 */
    [LAYER_CORE] = { LAYER_OSAL, LAYER_SERVICE, 0 },
    
    /* Layer 5 (FW Msg): OSAL, HIP 의존 */
    [LAYER_FW_MSG] = { LAYER_OSAL, LAYER_HIP, 0 },
    
    /* Layer 6 (HIP): OSAL 의존 */
    [LAYER_HIP] = { LAYER_OSAL, 0 },
    
    /* Layer 7 (Customer): OSAL 의존, 다른 레이어 init 후 */
    [LAYER_CUSTOMER] = { LAYER_OSAL, 0 },
};

/**
 * 초기화 순서 (위상 정렬 결과)
 */
static const enum layer_id init_order[] = {
    LAYER_OSAL,          /* 1. OS Abstraction */
    LAYER_HIP,           /* 2. Bus Interface */
    LAYER_FW_MSG,        /* 3. FW Message */
    LAYER_KERNEL_IF,     /* 4. Kernel Interface */
    LAYER_SERVICE,       /* 5. Service Manager */
    LAYER_CORE,          /* 6. Core Protocol */
    LAYER_CUSTOMER,      /* 7. Customer Extensions */
};

/**
 * 종료 순서 (초기화 역순)
 */
static const enum layer_id deinit_order[] = {
    LAYER_CUSTOMER,
    LAYER_CORE,
    LAYER_SERVICE,
    LAYER_KERNEL_IF,
    LAYER_FW_MSG,
    LAYER_HIP,
    LAYER_OSAL,
};
```

### 4.3 레이어 등록 예시

```c
/* 각 레이어에서 등록 */

/* osal.c */
static const struct layer_ops osal_layer_ops = {
    .name = "OSAL",
    .init = osal_init,
    .deinit = osal_deinit,
    .start = NULL,              /* OSAL은 start 불필요 */
    .stop = NULL,
    .suspend = NULL,
    .resume = NULL,
    .is_ready = osal_is_ready,
};

int osal_register(void)
{
    return layer_registry_register(LAYER_OSAL, &osal_layer_ops);
}

/* hip_core.c */
static const struct layer_ops hip_layer_ops = {
    .name = "HIP",
    .init = hip_init_internal,
    .deinit = hip_deinit,
    .start = hip_start,
    .stop = hip_stop,
    .suspend = hip_suspend,
    .resume = hip_resume,
    .pre_recovery = hip_pre_recovery,
    .post_recovery = hip_post_recovery,
    .is_ready = hip_is_ready,
};

int hip_register(void)
{
    return layer_registry_register(LAYER_HIP, &hip_layer_ops);
}

/* vif_manager.c */
static const struct layer_ops service_layer_ops = {
    .name = "Service",
    .init = service_init,
    .deinit = service_deinit,
    .start = service_start,
    .stop = service_stop,
    .suspend = service_suspend,
    .resume = service_resume,
    .pre_recovery = service_pre_recovery,
    .post_recovery = service_post_recovery,
    .is_ready = service_is_ready,
};
```

---

## 5. Lifecycle Core 인터페이스

### 5.1 주요 인터페이스

```c
/* lifecycle_core.h */

/**
 * 초기화 (드라이버 probe에서 호출)
 */
int lifecycle_probe(void *platform_data);

/**
 * 종료 (드라이버 remove에서 호출)
 */
void lifecycle_remove(void);

/**
 * Power Management
 */
int lifecycle_suspend(void);
int lifecycle_resume(void);

/**
 * Recovery 트리거
 */
int lifecycle_trigger_recovery(enum recovery_reason reason);

/**
 * 상태 조회
 */
enum drv_state lifecycle_get_state(void);
bool lifecycle_is_running(void);

/**
 * 상태 변경 대기
 */
int lifecycle_wait_for_state(enum drv_state state, unsigned long timeout_ms);

/**
 * 상태 변경 알림 콜백
 */
typedef void (*lifecycle_state_callback_t)(enum drv_state old_state,
                                            enum drv_state new_state,
                                            void *ctx);
int lifecycle_register_callback(lifecycle_state_callback_t cb, void *ctx);
void lifecycle_unregister_callback(lifecycle_state_callback_t cb);
```

### 5.2 구현 개요

```c
/* lifecycle_core.c */

struct lifecycle_context {
    enum drv_state state;
    osal_mutex_t state_lock;
    osal_completion_t state_change;
    
    /* Platform data */
    void *platform_data;
    
    /* 콜백 */
    struct {
        lifecycle_state_callback_t fn;
        void *ctx;
    } callbacks[8];
    int callback_count;
    
    /* 통계 */
    struct {
        u32 recovery_count;
        u32 suspend_count;
        u64 uptime_ms;
        u64 last_recovery_time;
    } stats;
    
    /* Recovery 관련 */
    struct work_struct recovery_work;
    enum recovery_reason recovery_reason;
};

static struct lifecycle_context g_lifecycle;

/**
 * 상태 전이
 */
static int set_state(enum drv_state new_state)
{
    enum drv_state old_state;
    int i;
    
    osal_mutex_lock(&g_lifecycle.state_lock);
    
    old_state = g_lifecycle.state;
    
    /* 유효성 검사 */
    if (!valid_transitions[old_state][new_state]) {
        OSAL_ERR("Invalid state transition: %s → %s\n",
                 state_to_str(old_state), state_to_str(new_state));
        osal_mutex_unlock(&g_lifecycle.state_lock);
        return -EINVAL;
    }
    
    g_lifecycle.state = new_state;
    
    OSAL_INFO("State: %s → %s\n", 
              state_to_str(old_state), state_to_str(new_state));
    
    osal_mutex_unlock(&g_lifecycle.state_lock);
    
    /* 콜백 호출 */
    for (i = 0; i < g_lifecycle.callback_count; i++) {
        if (g_lifecycle.callbacks[i].fn) {
            g_lifecycle.callbacks[i].fn(old_state, new_state,
                                        g_lifecycle.callbacks[i].ctx);
        }
    }
    
    /* 대기자 깨우기 */
    osal_completion_complete_all(&g_lifecycle.state_change);
    
    return 0;
}

/**
 * 상태 대기
 */
int lifecycle_wait_for_state(enum drv_state state, unsigned long timeout_ms)
{
    int ret;
    
    while (g_lifecycle.state != state) {
        ret = osal_completion_wait_timeout(&g_lifecycle.state_change,
                                            timeout_ms);
        if (ret == -ETIMEDOUT)
            return -ETIMEDOUT;
        
        osal_completion_reinit(&g_lifecycle.state_change);
    }
    
    return 0;
}
```

---

## 6. 전체 흐름 요약

### 6.1 드라이버 로딩

```
Platform Driver probe()
        │
        ▼
lifecycle_probe()
        │
        ├──▶ drv_init_sequence()
        │         │
        │         ├──▶ Layer 1 init (OSAL)
        │         ├──▶ Layer 6 init (HIP)
        │         ├──▶ Layer 5 init (FW Msg)
        │         ├──▶ Layer 2 init (Kernel IF)
        │         ├──▶ Layer 3 init (Service)
        │         ├──▶ Layer 4 init (Core)
        │         └──▶ Layer 7 init (Customer)
        │
        ├──▶ drv_fw_load()
        │         │
        │         ├──▶ FW 다운로드
        │         └──▶ FW 부팅 & Ready 대기
        │
        └──▶ drv_start_sequence()
                  │
                  ├──▶ Layer 6 start (HIP)
                  ├──▶ Layer 5 start (FW Msg)
                  ├──▶ Layer 3 start (Service)
                  └──▶ State → RUNNING
```

### 6.2 드라이버 언로딩

```
Platform Driver remove()
        │
        ▼
lifecycle_remove()
        │
        ├──▶ drv_stop_sequence()
        │         │
        │         ├──▶ Layer 3 stop (Service)
        │         ├──▶ Layer 5 stop (FW Msg)
        │         └──▶ Layer 6 stop (HIP)
        │
        └──▶ drv_deinit_sequence()
                  │
                  ├──▶ Layer 7 deinit (Customer)
                  ├──▶ Layer 4 deinit (Core)
                  ├──▶ Layer 3 deinit (Service)
                  ├──▶ Layer 2 deinit (Kernel IF)
                  ├──▶ Layer 5 deinit (FW Msg)
                  ├──▶ Layer 6 deinit (HIP)
                  └──▶ Layer 1 deinit (OSAL)
```

### 6.3 Recovery

```
FW Error 감지 (HIP에서)
        │
        ▼
lifecycle_trigger_recovery()
        │
        ├──▶ State → RECOVERING
        │
        ├──▶ drv_recovery_sequence()
        │         │
        │         ├──▶ 각 레이어 pre_recovery()
        │         ├──▶ FW Reset & Reload
        │         └──▶ 각 레이어 post_recovery()
        │
        └──▶ State → RUNNING
```

---

## 7. 디렉토리 구조

```
lifecycle/
├── lifecycle_core.c         # 상태 머신, 전체 조율
├── lifecycle_core.h
├── layer_registry.c         # 레이어 등록/의존성
├── layer_registry.h
│
├── drv_init.c               # 초기화 시퀀스
├── drv_init.h
├── drv_deinit.c             # 종료 시퀀스
│
├── drv_recovery.c           # Recovery 로직
├── drv_recovery.h
│
├── drv_power.c              # Suspend/Resume
└── drv_power.h
```

---

## 8. 다음 문서

| 문서 | 내용 |
|------|------|
| **08-1_DRV_INIT.md** | 초기화/종료 시퀀스 상세 |
| **08-2_DRV_RECOVERY.md** | Recovery 메커니즘 상세 |
| **08-3_DRV_POWER.md** | Power Management 상세 |

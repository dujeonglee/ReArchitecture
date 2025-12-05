# Layer 7: Customer Extensions (Side Layer)

## 1. 개요

### 1.1 역할

고객(벤더)별 커스터마이징을 Core 코드와 분리하여 관리한다. Hook 기반의 확장 메커니즘을 통해 Core 수정 없이 기능을 추가하거나 동작을 변경할 수 있다.

### 1.2 설계 목표

- Core 코드 무수정: 고객 요구사항이 Core에 영향을 주지 않음
- 고객별 격리: 벤더 A의 코드가 벤더 B에 영향을 주지 않음
- 선택적 빌드: 고객별로 필요한 확장만 포함
- 유지보수 용이: Core 업그레이드 시 고객 코드 영향 최소화

### 1.3 Side Layer 개념

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Main Layers (Core Path)                                                        │
│                                                                                 │
│  Layer 2 ──▶ Layer 3 ──▶ Layer 4 ──▶ Layer 5 ──▶ Layer 6                       │
│     │           │           │           │                                       │
│     │           │           │           │                                       │
│     ▼           ▼           ▼           ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    Customer Hook Registry                                │   │
│  │                         (Layer 7)                                        │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │ cust_common │  │cust_vendor_A│  │cust_vendor_B│  │cust_vendor_C│     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘

특징:
- Customer Layer는 Main Path에 "끼어드는" Side Layer
- Core가 Customer를 직접 호출하지 않음 (Hook으로만 연결)
- Customer가 Core를 호출할 수 있음 (단방향)
```

---

## 2. 모듈 구성

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 7: Customer Extensions                                                   │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                     customer_hook_registry                               │   │
│  │                  (Hook 등록/해제, 디스패치 관리)                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                     │                                           │
│       ┌─────────────────────────────┼─────────────────────────────┐            │
│       ▼                             ▼                             ▼            │
│  ┌─────────────┐             ┌─────────────┐             ┌─────────────┐       │
│  │cust_common  │             │cust_vendor_A│             │cust_vendor_B│       │
│  │ (공통 확장)  │             │ (벤더 A 전용)│             │ (벤더 B 전용)│       │
│  └─────────────┘             └─────────────┘             └─────────────┘       │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                      customer_vendor_cmd                                 │   │
│  │                   (Vendor nl80211 확장 명령)                              │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Hook 시스템

### 3.1 Hook Point 정의

```c
/* customer_hook.h */

/**
 * Hook Point ID
 * Core의 주요 지점에서 호출되는 Hook
 */
enum customer_hook_id {
    /* VIF 관련 */
    CUST_HOOK_VIF_CREATE_PRE,        /* VIF 생성 전 */
    CUST_HOOK_VIF_CREATE_POST,       /* VIF 생성 후 */
    CUST_HOOK_VIF_DESTROY_PRE,       /* VIF 삭제 전 */
    CUST_HOOK_VIF_DESTROY_POST,      /* VIF 삭제 후 */
    
    /* Scan 관련 */
    CUST_HOOK_SCAN_START,            /* 스캔 시작 */
    CUST_HOOK_SCAN_RESULT,           /* 스캔 결과 수신 */
    CUST_HOOK_SCAN_DONE,             /* 스캔 완료 */
    
    /* Connect 관련 */
    CUST_HOOK_CONNECT_PRE,           /* 연결 시작 전 */
    CUST_HOOK_CONNECT_POST,          /* 연결 완료 후 */
    CUST_HOOK_DISCONNECT_PRE,        /* 연결 해제 전 */
    CUST_HOOK_DISCONNECT_POST,       /* 연결 해제 후 */
    
    /* Roaming 관련 */
    CUST_HOOK_ROAM_CANDIDATE,        /* 로밍 후보 평가 */
    CUST_HOOK_ROAM_START,            /* 로밍 시작 */
    CUST_HOOK_ROAM_COMPLETE,         /* 로밍 완료 */
    
    /* AP 관련 */
    CUST_HOOK_AP_START_PRE,          /* AP 시작 전 */
    CUST_HOOK_AP_START_POST,         /* AP 시작 후 */
    CUST_HOOK_AP_STOP_PRE,           /* AP 중지 전 */
    CUST_HOOK_AP_STA_CONNECT,        /* STA 연결 (AP 모드) */
    CUST_HOOK_AP_STA_DISCONNECT,     /* STA 연결 해제 (AP 모드) */
    
    /* Data Path 관련 */
    CUST_HOOK_TX_PRE,                /* TX 전 */
    CUST_HOOK_TX_POST,               /* TX 후 */
    CUST_HOOK_RX_PRE,                /* RX 전 */
    CUST_HOOK_RX_POST,               /* RX 후 */
    
    /* Power 관련 */
    CUST_HOOK_SUSPEND_PRE,           /* Suspend 전 */
    CUST_HOOK_SUSPEND_POST,          /* Suspend 후 */
    CUST_HOOK_RESUME_PRE,            /* Resume 전 */
    CUST_HOOK_RESUME_POST,           /* Resume 후 */
    
    /* FW 관련 */
    CUST_HOOK_FW_READY,              /* FW Ready */
    CUST_HOOK_FW_ERROR,              /* FW Error */
    
    CUST_HOOK_MAX,
};

/**
 * Hook 반환 값
 */
enum customer_hook_result {
    CUST_HOOK_CONTINUE = 0,          /* 계속 진행 */
    CUST_HOOK_STOP,                  /* 처리 중단 (성공) */
    CUST_HOOK_ERROR,                 /* 에러로 중단 */
    CUST_HOOK_MODIFIED,              /* 데이터 수정됨, 계속 진행 */
};

/**
 * Hook 콜백 타입
 */
typedef enum customer_hook_result (*customer_hook_fn)(
    enum customer_hook_id hook_id,
    void *data,
    void *ctx
);
```

### 3.2 Hook 데이터 구조체

```c
/* customer_hook_data.h */

/**
 * VIF Hook 데이터
 */
struct cust_hook_vif_data {
    struct vif_context *vif;
    enum vif_type type;
    const char *name;
    int result;                      /* POST hook에서 결과 */
};

/**
 * Scan Hook 데이터
 */
struct cust_hook_scan_data {
    struct vif_context *vif;
    struct cfg80211_scan_request *request;  /* START */
    struct cfg80211_bss *bss;               /* RESULT */
    bool aborted;                           /* DONE */
};

/**
 * Connect Hook 데이터
 */
struct cust_hook_connect_data {
    struct vif_context *vif;
    struct cfg80211_connect_params *params; /* PRE */
    const u8 *bssid;                        /* POST */
    u16 status;                             /* POST */
    int rssi;
};

/**
 * Roaming Hook 데이터
 */
struct cust_hook_roam_data {
    struct vif_context *vif;
    const u8 *current_bssid;
    const u8 *target_bssid;
    int current_rssi;
    int target_rssi;
    int score;                       /* CANDIDATE: 점수 조정 가능 */
    bool allow;                      /* CANDIDATE: 로밍 허용 여부 */
};

/**
 * AP Hook 데이터
 */
struct cust_hook_ap_data {
    struct vif_context *vif;
    struct cfg80211_ap_settings *settings;  /* START */
    const u8 *sta_mac;                      /* STA_CONNECT/DISCONNECT */
    u16 reason;                             /* STA_DISCONNECT */
};

/**
 * TX/RX Hook 데이터
 */
struct cust_hook_data_path {
    struct vif_context *vif;
    struct sk_buff *skb;
    bool drop;                       /* true면 드롭 */
};

/**
 * Power Hook 데이터
 */
struct cust_hook_power_data {
    bool is_suspend;                 /* true: suspend, false: resume */
    int result;
};
```

---

## 4. 모듈 상세

### 4.1 customer_hook_registry

| 항목 | 내용 |
|------|------|
| **역할** | Hook 등록/해제 및 디스패치 관리 |
| **책임** | Hook 콜백 관리, 우선순위 처리, 체이닝 |
| **의존성** | OSAL |

#### 인터페이스

```c
/* customer_hook_registry.h */

/**
 * Hook 우선순위
 */
enum customer_hook_priority {
    CUST_HOOK_PRIO_FIRST = 0,        /* 가장 먼저 실행 */
    CUST_HOOK_PRIO_HIGH = 25,
    CUST_HOOK_PRIO_NORMAL = 50,
    CUST_HOOK_PRIO_LOW = 75,
    CUST_HOOK_PRIO_LAST = 100,       /* 가장 나중에 실행 */
};

/**
 * Hook 등록
 */
int customer_hook_register(enum customer_hook_id hook_id,
                           customer_hook_fn fn,
                           void *ctx,
                           enum customer_hook_priority priority);

/**
 * Hook 해제
 */
void customer_hook_unregister(enum customer_hook_id hook_id,
                              customer_hook_fn fn);

/**
 * Hook 호출 (Core에서 호출)
 */
enum customer_hook_result customer_hook_call(enum customer_hook_id hook_id,
                                              void *data);

/**
 * 모든 Hook 해제 (특정 컨텍스트)
 */
void customer_hook_unregister_all(void *ctx);

/**
 * Hook 활성화/비활성화
 */
void customer_hook_enable(enum customer_hook_id hook_id, bool enable);
bool customer_hook_is_enabled(enum customer_hook_id hook_id);

/**
 * 디버그: 등록된 Hook 출력
 */
void customer_hook_dump(void);
```

#### 구현 예시

```c
/* customer_hook_registry.c */

#define MAX_HOOKS_PER_POINT  8

struct hook_entry {
    customer_hook_fn fn;
    void *ctx;
    enum customer_hook_priority priority;
    bool enabled;
};

struct hook_point {
    struct hook_entry entries[MAX_HOOKS_PER_POINT];
    int count;
    bool enabled;
    osal_spinlock_t lock;
};

static struct hook_point g_hooks[CUST_HOOK_MAX];

/**
 * 초기화
 */
int customer_hook_registry_init(void)
{
    int i;
    
    for (i = 0; i < CUST_HOOK_MAX; i++) {
        memset(&g_hooks[i], 0, sizeof(struct hook_point));
        g_hooks[i].enabled = true;
        osal_spin_lock_init(&g_hooks[i].lock);
    }
    
    return 0;
}

/**
 * Hook 등록
 */
int customer_hook_register(enum customer_hook_id hook_id,
                           customer_hook_fn fn,
                           void *ctx,
                           enum customer_hook_priority priority)
{
    struct hook_point *hp;
    struct hook_entry *entry;
    unsigned long flags;
    int i, insert_idx;
    
    if (hook_id >= CUST_HOOK_MAX || !fn)
        return -EINVAL;
    
    hp = &g_hooks[hook_id];
    
    osal_spin_lock_irqsave(&hp->lock, &flags);
    
    /* 중복 검사 */
    for (i = 0; i < hp->count; i++) {
        if (hp->entries[i].fn == fn && hp->entries[i].ctx == ctx) {
            osal_spin_unlock_irqrestore(&hp->lock, flags);
            return -EEXIST;
        }
    }
    
    /* 공간 검사 */
    if (hp->count >= MAX_HOOKS_PER_POINT) {
        osal_spin_unlock_irqrestore(&hp->lock, flags);
        return -ENOSPC;
    }
    
    /* 우선순위에 따른 삽입 위치 찾기 */
    insert_idx = hp->count;
    for (i = 0; i < hp->count; i++) {
        if (priority < hp->entries[i].priority) {
            insert_idx = i;
            break;
        }
    }
    
    /* 뒤로 밀기 */
    for (i = hp->count; i > insert_idx; i--)
        hp->entries[i] = hp->entries[i - 1];
    
    /* 삽입 */
    entry = &hp->entries[insert_idx];
    entry->fn = fn;
    entry->ctx = ctx;
    entry->priority = priority;
    entry->enabled = true;
    hp->count++;
    
    osal_spin_unlock_irqrestore(&hp->lock, flags);
    
    OSAL_DBG("Hook registered: id=%d, fn=%p, prio=%d\n", 
             hook_id, fn, priority);
    
    return 0;
}

/**
 * Hook 호출
 */
enum customer_hook_result customer_hook_call(enum customer_hook_id hook_id,
                                              void *data)
{
    struct hook_point *hp;
    enum customer_hook_result result = CUST_HOOK_CONTINUE;
    unsigned long flags;
    int i;
    
    if (hook_id >= CUST_HOOK_MAX)
        return CUST_HOOK_CONTINUE;
    
    hp = &g_hooks[hook_id];
    
    /* Hook point 비활성화 검사 */
    if (!hp->enabled)
        return CUST_HOOK_CONTINUE;
    
    osal_spin_lock_irqsave(&hp->lock, &flags);
    
    for (i = 0; i < hp->count; i++) {
        struct hook_entry *entry = &hp->entries[i];
        
        if (!entry->enabled)
            continue;
        
        /* Hook 함수 호출 */
        result = entry->fn(hook_id, data, entry->ctx);
        
        /* STOP이나 ERROR면 체인 중단 */
        if (result == CUST_HOOK_STOP || result == CUST_HOOK_ERROR)
            break;
    }
    
    osal_spin_unlock_irqrestore(&hp->lock, flags);
    
    return result;
}
```

### 4.2 Core에서 Hook 호출 예시

```c
/* vif_manager.c */

struct vif_context *vif_manager_create_vif(enum vif_type type, const char *name)
{
    struct cust_hook_vif_data hook_data;
    enum customer_hook_result hook_result;
    struct vif_context *vif;
    
    /* PRE Hook */
    hook_data.type = type;
    hook_data.name = name;
    hook_data.vif = NULL;
    
    hook_result = customer_hook_call(CUST_HOOK_VIF_CREATE_PRE, &hook_data);
    if (hook_result == CUST_HOOK_ERROR) {
        OSAL_WARN("VIF create blocked by customer hook\n");
        return NULL;
    }
    
    /* 실제 VIF 생성 로직 */
    vif = create_vif_internal(type, name);
    if (!vif)
        return NULL;
    
    /* POST Hook */
    hook_data.vif = vif;
    hook_data.result = 0;
    
    customer_hook_call(CUST_HOOK_VIF_CREATE_POST, &hook_data);
    
    return vif;
}

/* svc_sta.c */

static void handle_roam_candidate(struct svc_sta_context *sta,
                                   struct roam_candidate *candidate)
{
    struct cust_hook_roam_data hook_data;
    
    /* 로밍 후보 평가 Hook */
    hook_data.vif = sta->vif;
    hook_data.current_bssid = sta->bssid;
    hook_data.target_bssid = candidate->bssid;
    hook_data.current_rssi = sta->rssi;
    hook_data.target_rssi = candidate->rssi;
    hook_data.score = candidate->score;
    hook_data.allow = true;
    
    customer_hook_call(CUST_HOOK_ROAM_CANDIDATE, &hook_data);
    
    /* Hook에서 수정한 값 적용 */
    candidate->score = hook_data.score;
    
    if (!hook_data.allow) {
        OSAL_DBG("Roaming to %pM blocked by customer\n", candidate->bssid);
        candidate->blocked = true;
    }
}

/* ma_handler.c */

int ma_tx_submit(struct vif_context *vif, struct sk_buff *skb)
{
    struct cust_hook_data_path hook_data;
    enum customer_hook_result hook_result;
    
    /* TX PRE Hook */
    hook_data.vif = vif;
    hook_data.skb = skb;
    hook_data.drop = false;
    
    hook_result = customer_hook_call(CUST_HOOK_TX_PRE, &hook_data);
    
    if (hook_data.drop || hook_result == CUST_HOOK_STOP) {
        dev_kfree_skb_any(skb);
        return 0;  /* 성공으로 처리 (드롭) */
    }
    
    /* 실제 TX 로직 */
    return tx_submit_internal(vif, skb);
}
```

---

### 4.3 cust_common (공통 확장)

| 항목 | 내용 |
|------|------|
| **역할** | 모든 고객에게 공통으로 적용되는 확장 |
| **책임** | 공통 로깅, 통계, 디버그 기능 |
| **의존성** | customer_hook_registry |

#### 구현 예시

```c
/* cust_common.c */

/**
 * 공통 이벤트 로깅
 */
static enum customer_hook_result common_connect_logger(
    enum customer_hook_id hook_id,
    void *data,
    void *ctx)
{
    struct cust_hook_connect_data *conn = data;
    
    if (hook_id == CUST_HOOK_CONNECT_PRE) {
        OSAL_INFO("[CUST] Connect attempt: SSID=%.*s\n",
                  (int)conn->params->ssid_len, conn->params->ssid);
    } else if (hook_id == CUST_HOOK_CONNECT_POST) {
        if (conn->status == 0)
            OSAL_INFO("[CUST] Connected to %pM, RSSI=%d\n",
                      conn->bssid, conn->rssi);
        else
            OSAL_INFO("[CUST] Connect failed: status=%d\n", conn->status);
    }
    
    return CUST_HOOK_CONTINUE;
}

/**
 * 공통 로밍 로깅
 */
static enum customer_hook_result common_roam_logger(
    enum customer_hook_id hook_id,
    void *data,
    void *ctx)
{
    struct cust_hook_roam_data *roam = data;
    
    OSAL_INFO("[CUST] Roam %s: %pM → %pM, RSSI: %d → %d\n",
              hook_id == CUST_HOOK_ROAM_START ? "start" : "complete",
              roam->current_bssid, roam->target_bssid,
              roam->current_rssi, roam->target_rssi);
    
    return CUST_HOOK_CONTINUE;
}

/**
 * 초기화
 */
int cust_common_init(void)
{
    int ret;
    
    /* 이벤트 로깅 Hook 등록 */
    ret = customer_hook_register(CUST_HOOK_CONNECT_PRE,
                                  common_connect_logger, NULL,
                                  CUST_HOOK_PRIO_LAST);
    if (ret < 0)
        return ret;
    
    ret = customer_hook_register(CUST_HOOK_CONNECT_POST,
                                  common_connect_logger, NULL,
                                  CUST_HOOK_PRIO_LAST);
    if (ret < 0)
        return ret;
    
    ret = customer_hook_register(CUST_HOOK_ROAM_START,
                                  common_roam_logger, NULL,
                                  CUST_HOOK_PRIO_LAST);
    if (ret < 0)
        return ret;
    
    ret = customer_hook_register(CUST_HOOK_ROAM_COMPLETE,
                                  common_roam_logger, NULL,
                                  CUST_HOOK_PRIO_LAST);
    
    OSAL_INFO("cust_common initialized\n");
    
    return ret;
}

void cust_common_deinit(void)
{
    customer_hook_unregister(CUST_HOOK_CONNECT_PRE, common_connect_logger);
    customer_hook_unregister(CUST_HOOK_CONNECT_POST, common_connect_logger);
    customer_hook_unregister(CUST_HOOK_ROAM_START, common_roam_logger);
    customer_hook_unregister(CUST_HOOK_ROAM_COMPLETE, common_roam_logger);
}
```

---

### 4.4 cust_vendor_A (벤더 A 전용)

| 항목 | 내용 |
|------|------|
| **역할** | 벤더 A 전용 커스터마이징 |
| **예시** | 커스텀 로밍 알고리즘, 특정 AP 블랙리스트, Private 기능 |
| **의존성** | customer_hook_registry |

#### 구현 예시

```c
/* cust_vendor_a.c */

/**
 * 벤더 A 컨텍스트
 */
struct vendor_a_context {
    /* 로밍 블랙리스트 */
    u8 blacklist[10][ETH_ALEN];
    int blacklist_count;
    
    /* 커스텀 로밍 파라미터 */
    int roam_rssi_boost;         /* 특정 조건에서 RSSI 보정 */
    bool prefer_5g;              /* 5GHz 선호 */
    
    /* 통계 */
    u32 blocked_roams;
    u32 modified_scores;
};

static struct vendor_a_context g_vendor_a;

/**
 * 커스텀 로밍 후보 평가
 */
static enum customer_hook_result vendor_a_roam_candidate(
    enum customer_hook_id hook_id,
    void *data,
    void *ctx)
{
    struct cust_hook_roam_data *roam = data;
    struct vendor_a_context *va = ctx;
    int i;
    
    /* 블랙리스트 검사 */
    for (i = 0; i < va->blacklist_count; i++) {
        if (memcmp(roam->target_bssid, va->blacklist[i], ETH_ALEN) == 0) {
            OSAL_DBG("[VendorA] BSSID %pM is blacklisted\n", roam->target_bssid);
            roam->allow = false;
            va->blocked_roams++;
            return CUST_HOOK_MODIFIED;
        }
    }
    
    /* 5GHz 선호 로직 */
    if (va->prefer_5g) {
        int target_channel = bssid_to_channel(roam->target_bssid);
        
        if (target_channel >= 36 && target_channel <= 165) {
            /* 5GHz면 점수 부스트 */
            roam->score += 20;
            va->modified_scores++;
            OSAL_DBG("[VendorA] 5GHz boost: score=%d\n", roam->score);
        }
    }
    
    /* RSSI 보정 */
    if (va->roam_rssi_boost != 0) {
        roam->score += va->roam_rssi_boost;
    }
    
    return CUST_HOOK_MODIFIED;
}

/**
 * 커스텀 연결 전 처리
 */
static enum customer_hook_result vendor_a_connect_pre(
    enum customer_hook_id hook_id,
    void *data,
    void *ctx)
{
    struct cust_hook_connect_data *conn = data;
    
    /* 벤더 A 특정 로직: 특정 SSID 패턴 차단 */
    if (conn->params->ssid_len >= 5 &&
        memcmp(conn->params->ssid, "BLOCK", 5) == 0) {
        OSAL_WARN("[VendorA] Blocked SSID pattern\n");
        return CUST_HOOK_ERROR;
    }
    
    return CUST_HOOK_CONTINUE;
}

/**
 * 초기화
 */
int cust_vendor_a_init(void)
{
    int ret;
    
    memset(&g_vendor_a, 0, sizeof(g_vendor_a));
    
    /* 기본 설정 */
    g_vendor_a.prefer_5g = true;
    g_vendor_a.roam_rssi_boost = 5;
    
    /* Hook 등록 */
    ret = customer_hook_register(CUST_HOOK_ROAM_CANDIDATE,
                                  vendor_a_roam_candidate,
                                  &g_vendor_a,
                                  CUST_HOOK_PRIO_NORMAL);
    if (ret < 0)
        return ret;
    
    ret = customer_hook_register(CUST_HOOK_CONNECT_PRE,
                                  vendor_a_connect_pre,
                                  &g_vendor_a,
                                  CUST_HOOK_PRIO_HIGH);
    if (ret < 0)
        goto err_unregister;
    
    OSAL_INFO("[VendorA] Customer extension initialized\n");
    return 0;
    
err_unregister:
    customer_hook_unregister(CUST_HOOK_ROAM_CANDIDATE, vendor_a_roam_candidate);
    return ret;
}

/**
 * 블랙리스트 관리 API
 */
int cust_vendor_a_add_blacklist(const u8 *bssid)
{
    if (g_vendor_a.blacklist_count >= 10)
        return -ENOSPC;
    
    memcpy(g_vendor_a.blacklist[g_vendor_a.blacklist_count], bssid, ETH_ALEN);
    g_vendor_a.blacklist_count++;
    
    return 0;
}

void cust_vendor_a_clear_blacklist(void)
{
    g_vendor_a.blacklist_count = 0;
}
```

---

### 4.5 customer_vendor_cmd (Vendor Command 확장)

| 항목 | 내용 |
|------|------|
| **역할** | 고객별 nl80211 vendor command 확장 |
| **책임** | 고객 전용 명령어 처리 |
| **의존성** | vendor_nl80211 (Layer 2) |

#### 구현 예시

```c
/* customer_vendor_cmd.c */

/**
 * 벤더 A 전용 명령어 ID (WIFI_VENDOR_CMD_CUSTOMER_START 이후)
 */
enum vendor_a_cmd_id {
    VENDOR_A_CMD_SET_ROAM_PARAMS = WIFI_VENDOR_CMD_CUSTOMER_START,
    VENDOR_A_CMD_GET_ROAM_STATS,
    VENDOR_A_CMD_ADD_BLACKLIST,
    VENDOR_A_CMD_CLEAR_BLACKLIST,
    VENDOR_A_CMD_SET_PREFER_5G,
};

/**
 * 벤더 A 전용 attribute
 */
enum vendor_a_attrs {
    VENDOR_A_ATTR_INVALID = 0,
    VENDOR_A_ATTR_RSSI_BOOST,
    VENDOR_A_ATTR_PREFER_5G,
    VENDOR_A_ATTR_BSSID,
    VENDOR_A_ATTR_BLOCKED_COUNT,
    VENDOR_A_ATTR_MODIFIED_COUNT,
    VENDOR_A_ATTR_MAX,
};

static const struct nla_policy vendor_a_policy[VENDOR_A_ATTR_MAX] = {
    [VENDOR_A_ATTR_RSSI_BOOST] = { .type = NLA_S32 },
    [VENDOR_A_ATTR_PREFER_5G] = { .type = NLA_U8 },
    [VENDOR_A_ATTR_BSSID] = { .type = NLA_BINARY, .len = ETH_ALEN },
};

/**
 * SET_ROAM_PARAMS 명령 처리
 */
static int vendor_a_set_roam_params(struct wiphy *wiphy,
                                    struct wireless_dev *wdev,
                                    const void *data, int data_len)
{
    struct nlattr *tb[VENDOR_A_ATTR_MAX];
    int ret;
    
    ret = nla_parse(tb, VENDOR_A_ATTR_MAX - 1, data, data_len,
                    vendor_a_policy, NULL);
    if (ret)
        return ret;
    
    if (tb[VENDOR_A_ATTR_RSSI_BOOST])
        g_vendor_a.roam_rssi_boost = nla_get_s32(tb[VENDOR_A_ATTR_RSSI_BOOST]);
    
    if (tb[VENDOR_A_ATTR_PREFER_5G])
        g_vendor_a.prefer_5g = nla_get_u8(tb[VENDOR_A_ATTR_PREFER_5G]);
    
    return 0;
}

/**
 * GET_ROAM_STATS 명령 처리
 */
static int vendor_a_get_roam_stats(struct wiphy *wiphy,
                                   struct wireless_dev *wdev,
                                   const void *data, int data_len)
{
    struct sk_buff *reply;
    
    reply = cfg80211_vendor_cmd_alloc_reply_skb(wiphy, 64);
    if (!reply)
        return -ENOMEM;
    
    nla_put_u32(reply, VENDOR_A_ATTR_BLOCKED_COUNT, g_vendor_a.blocked_roams);
    nla_put_u32(reply, VENDOR_A_ATTR_MODIFIED_COUNT, g_vendor_a.modified_scores);
    
    return cfg80211_vendor_cmd_reply(reply);
}

/**
 * 고객 명령어 등록
 */
int customer_vendor_cmd_init(void)
{
    int ret;
    
    /* vendor_nl80211에 고객 명령어 등록 */
    ret = vendor_nl80211_register_customer_cmd(
        VENDOR_A_CMD_SET_ROAM_PARAMS,
        vendor_a_set_roam_params,
        vendor_a_policy,
        VENDOR_A_ATTR_MAX - 1);
    if (ret < 0)
        return ret;
    
    ret = vendor_nl80211_register_customer_cmd(
        VENDOR_A_CMD_GET_ROAM_STATS,
        vendor_a_get_roam_stats,
        NULL, 0);
    if (ret < 0)
        return ret;
    
    /* 추가 명령어 등록... */
    
    return 0;
}
```

---

## 5. 디렉토리 구조

```
customer/
├── customer_hook.h              # Hook 정의
├── customer_hook_data.h         # Hook 데이터 구조체
├── customer_hook_registry.c     # Hook 레지스트리
├── customer_hook_registry.h
│
├── cust_common.c                # 공통 확장
├── cust_common.h
│
├── vendors/
│   ├── cust_vendor_a.c          # 벤더 A 전용
│   ├── cust_vendor_a.h
│   ├── cust_vendor_b.c          # 벤더 B 전용
│   ├── cust_vendor_b.h
│   └── ...
│
└── customer_vendor_cmd.c        # Vendor command 확장
```

---

## 6. 빌드 설정

### 6.1 Kconfig

```kconfig
# customer/Kconfig

config WIFI_CUSTOMER_EXTENSIONS
    bool "Enable customer extensions"
    default y
    help
      Enable customer-specific extensions through hook system.

config WIFI_CUST_COMMON
    bool "Common customer extensions"
    depends on WIFI_CUSTOMER_EXTENSIONS
    default y

config WIFI_CUST_VENDOR_A
    bool "Vendor A extensions"
    depends on WIFI_CUSTOMER_EXTENSIONS
    default n

config WIFI_CUST_VENDOR_B
    bool "Vendor B extensions"
    depends on WIFI_CUSTOMER_EXTENSIONS
    default n
```

### 6.2 Makefile

```makefile
# customer/Makefile

obj-$(CONFIG_WIFI_CUSTOMER_EXTENSIONS) += customer_hook_registry.o

obj-$(CONFIG_WIFI_CUST_COMMON) += cust_common.o

obj-$(CONFIG_WIFI_CUST_VENDOR_A) += vendors/cust_vendor_a.o
obj-$(CONFIG_WIFI_CUST_VENDOR_B) += vendors/cust_vendor_b.o
```

---

## 7. 주목할 특성

### 7.1 Hook 체이닝

```
customer_hook_call(CUST_HOOK_ROAM_CANDIDATE, &data)
    │
    ├──▶ [PRIO_FIRST] security_check()    → CONTINUE
    │
    ├──▶ [PRIO_HIGH]  vendor_a_filter()   → MODIFIED (score 변경)
    │
    ├──▶ [PRIO_NORMAL] vendor_b_check()   → CONTINUE
    │
    └──▶ [PRIO_LAST]  common_logger()     → CONTINUE

결과: MODIFIED (최종 반환값)
```

### 7.2 Core와 Customer 분리

```
┌────────────────────────────────────────────────────────────────┐
│  Core Code (수정 금지)                                         │
│                                                                │
│  vif_manager.c:                                                │
│      hook_result = customer_hook_call(HOOK_ID, &data);        │
│      if (hook_result == CUST_HOOK_ERROR)                      │
│          return error;                                         │
│      /* 일반 로직 계속 */                                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
                          │
                          │ Hook 호출
                          ▼
┌────────────────────────────────────────────────────────────────┐
│  Customer Code (자유롭게 수정 가능)                             │
│                                                                │
│  cust_vendor_a.c:                                              │
│      static hook_result my_hook(hook_id, data, ctx) {         │
│          /* 벤더 A 전용 로직 */                                 │
│          return CUST_HOOK_CONTINUE;                           │
│      }                                                         │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 7.3 의존성 규칙

```
Core Layers ──────▶ customer_hook_registry ◀────── Customer Modules
   (호출)                    (관리)                   (등록)

규칙:
- Core는 customer_hook_call()만 호출
- Core는 개별 Customer 모듈을 직접 참조하지 않음
- Customer는 Core API를 호출할 수 있음 (역방향)
- Customer 모듈 간 직접 의존 금지
```

### 7.4 런타임 활성화/비활성화

```c
/* debugfs 또는 procfs를 통한 제어 */

/* 특정 Hook 비활성화 */
echo "disable CUST_HOOK_ROAM_CANDIDATE" > /sys/kernel/debug/wifi/customer_hooks

/* 특정 벤더 모듈 비활성화 */
echo "disable vendor_a" > /sys/kernel/debug/wifi/customer_modules

/* API로도 가능 */
customer_hook_enable(CUST_HOOK_TX_PRE, false);  /* TX Hook 비활성화 */
```

### 7.5 디버깅

```c
/* Hook 덤프 출력 */
customer_hook_dump();

/* 출력 예:
[CUST] Hook dump:
  CUST_HOOK_VIF_CREATE_PRE:
    [0] fn=0xffff1234, ctx=0x0, prio=50, enabled=1
  CUST_HOOK_CONNECT_PRE:
    [0] fn=0xffff2345, ctx=0xffff8000, prio=25, enabled=1
    [1] fn=0xffff3456, ctx=0x0, prio=100, enabled=1
  CUST_HOOK_ROAM_CANDIDATE:
    [0] fn=0xffff4567, ctx=0xffff8000, prio=50, enabled=1
  ...
*/
```

### 7.6 Customer 모듈 추가 가이드

1. `customer/vendors/` 아래에 새 파일 생성
2. Hook 콜백 함수 구현
3. Init 함수에서 Hook 등록
4. Kconfig/Makefile에 추가
5. 필요시 vendor command 추가

```c
/* 새 벤더 모듈 템플릿 */

/* cust_vendor_new.c */

#include "customer_hook.h"

struct vendor_new_context {
    /* 벤더 전용 데이터 */
};

static struct vendor_new_context g_vendor_new;

static enum customer_hook_result vendor_new_hook(
    enum customer_hook_id hook_id,
    void *data,
    void *ctx)
{
    /* 구현 */
    return CUST_HOOK_CONTINUE;
}

int cust_vendor_new_init(void)
{
    return customer_hook_register(CUST_HOOK_XXX,
                                   vendor_new_hook,
                                   &g_vendor_new,
                                   CUST_HOOK_PRIO_NORMAL);
}

void cust_vendor_new_deinit(void)
{
    customer_hook_unregister(CUST_HOOK_XXX, vendor_new_hook);
}
```

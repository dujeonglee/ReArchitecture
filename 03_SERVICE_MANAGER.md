# Layer 3: Service Manager

## 1. 개요

### 1.1 역할

VIF(Virtual Interface) 라이프사이클을 관리하고, 서비스 타입별(STA, MHS, P2P, NAN) 모듈로 요청을 라우팅한다.

### 1.2 설계 목표

- God Module 방지: vif_manager는 라우팅만, 비즈니스 로직은 각 서비스 모듈에
- 다형성: service_ops를 통한 서비스 타입 추상화
- 멀티 VIF 지원: 최대 3개 VIF 동시 운용

---

## 2. 모듈 구성

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 3: Service Manager                                                       │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                           vif_manager                                    │   │
│  │         (VIF lifecycle, 최대 3개 동시, 타입별 라우팅)                      │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                     │                                           │
│       ┌─────────────┬───────────────┼───────────────┬─────────────┐            │
│       ▼             ▼               ▼               ▼             ▼            │
│  ┌─────────┐  ┌─────────┐     ┌─────────┐     ┌─────────┐  ┌─────────┐        │
│  │ svc_sta │  │ svc_mhs │     │ svc_p2p │     │ svc_nan │  │ svc_test│        │
│  └─────────┘  └─────────┘     └─────────┘     └─────────┘  └─────────┘        │
│       │             │               │               │             │            │
│       └─────────────┴───────────────┴───────────────┴─────────────┘            │
│                                     │                                           │
│                                     ▼                                           │
│                        ┌───────────────────────┐                               │
│                        │    service_common     │                               │
│                        └───────────────────────┘                               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 모듈 상세

### 3.1 vif_manager

| 항목 | 내용 |
|------|------|
| **역할** | VIF 라이프사이클 관리 및 서비스 라우팅 |
| **책임** | VIF 생성/삭제, 타입별 서비스 디스패치, 동시 운용 정책 관리, 리소스 충돌 조정 |
| **의존성** | OSAL, 각 svc_* 모듈 |
| **권장 LOC** | < 500 |

#### 핵심 자료 구조

```c
/* vif_manager.h */

#define MAX_VIF 3

/**
 * VIF 타입
 */
enum vif_type {
    VIF_TYPE_STA,
    VIF_TYPE_AP,          /* MHS (Mobile Hotspot) */
    VIF_TYPE_P2P_DEVICE,
    VIF_TYPE_P2P_GO,
    VIF_TYPE_P2P_GC,
    VIF_TYPE_NAN,
    VIF_TYPE_TEST,
};

/**
 * VIF 컨텍스트
 */
struct vif_context {
    int vif_id;
    enum vif_type type;
    struct net_device *ndev;
    struct wireless_dev *wdev;
    
    /* 연결된 서비스 모듈 */
    void *service;                    /* svc_sta_context, svc_mhs_context 등 */
    const struct service_ops *svc_ops;
    
    /* 상태 */
    bool active;
    bool connected;
    int channel;
    int rssi;
    
    /* 통계 */
    struct vif_stats *stats;
    
    /* debugfs */
    struct dentry *debugfs_dir;
};

/**
 * VIF 매니저
 */
struct vif_manager {
    struct vif_context *vif_table[MAX_VIF];
    int vif_count;
    osal_mutex_t lock;
    
    /* 동시 운용 정책 */
    const struct vif_combination_policy *policy;
    
    /* 전역 상태 */
    bool initialized;
    struct wiphy *wiphy;
};
```

#### 인터페이스

```c
/* vif_manager.h */

/**
 * 초기화/해제
 */
int vif_manager_init(struct wiphy *wiphy);
void vif_manager_deinit(void);

/**
 * VIF 라이프사이클
 */
struct vif_context *vif_manager_create_vif(enum vif_type type, const char *name);
int vif_manager_destroy_vif(struct vif_context *vif);
int vif_manager_change_vif_type(struct vif_context *vif, enum vif_type new_type);

/**
 * 조회
 */
struct vif_context *vif_manager_get_vif(int vif_id);
struct vif_context *vif_manager_find_by_ndev(struct net_device *ndev);
struct vif_context *vif_manager_find_by_wdev(struct wireless_dev *wdev);
struct vif_context *vif_manager_find_by_type(enum vif_type type);
int vif_manager_get_vif_count(void);

/**
 * 조합 검사
 */
bool vif_manager_can_add_vif(enum vif_type type);
int vif_manager_check_channel_conflict(struct vif_context *vif, int channel);
int vif_manager_resolve_channel_conflict(struct vif_context *vif_a, 
                                          struct vif_context *vif_b);

/**
 * 디스패치 (Layer 2에서 호출)
 */
int vif_manager_dispatch_scan(struct vif_context *vif, 
                               struct cfg80211_scan_request *request);
int vif_manager_dispatch_connect(struct vif_context *vif,
                                  struct cfg80211_connect_params *params);
int vif_manager_dispatch_disconnect(struct vif_context *vif, u16 reason_code);
int vif_manager_dispatch_start_ap(struct vif_context *vif,
                                   struct cfg80211_ap_settings *settings);
int vif_manager_dispatch_stop_ap(struct vif_context *vif);

/**
 * FW 이벤트 디스패치 (FW Message Layer에서 호출)
 */
void vif_manager_dispatch_fw_event(int vif_id, struct fw_event *event);

/**
 * 유틸리티
 */
int vif_manager_get_rssi(struct vif_context *vif);
const char *vif_type_to_str(enum vif_type type);
const char *vif_state_to_str(int state);
```

#### 구현 예시

```c
/* vif_manager.c */

static struct vif_manager g_vif_mgr;

/**
 * 초기화
 */
int vif_manager_init(struct wiphy *wiphy)
{
    int i;
    
    memset(&g_vif_mgr, 0, sizeof(g_vif_mgr));
    
    osal_mutex_init(&g_vif_mgr.lock);
    g_vif_mgr.wiphy = wiphy;
    g_vif_mgr.policy = get_default_vif_policy();
    
    for (i = 0; i < MAX_VIF; i++)
        g_vif_mgr.vif_table[i] = NULL;
    
    g_vif_mgr.initialized = true;
    
    OSAL_INFO("vif_manager initialized\n");
    return 0;
}

/**
 * VIF 생성
 */
struct vif_context *vif_manager_create_vif(enum vif_type type, const char *name)
{
    struct vif_context *vif;
    int vif_id;
    int ret;
    
    osal_mutex_lock(&g_vif_mgr.lock);
    
    /* 조합 가능 여부 검사 */
    if (!vif_manager_can_add_vif_locked(type)) {
        OSAL_ERR("Cannot add VIF type %d: combination not allowed\n", type);
        osal_mutex_unlock(&g_vif_mgr.lock);
        return NULL;
    }
    
    /* 빈 슬롯 찾기 */
    vif_id = find_free_vif_slot_locked();
    if (vif_id < 0) {
        OSAL_ERR("No free VIF slot\n");
        osal_mutex_unlock(&g_vif_mgr.lock);
        return NULL;
    }
    
    /* VIF 컨텍스트 생성 */
    vif = osal_mem_zalloc(sizeof(*vif));
    if (!vif) {
        osal_mutex_unlock(&g_vif_mgr.lock);
        return NULL;
    }
    
    vif->vif_id = vif_id;
    vif->type = type;
    
    /* 타입별 서비스 연결 */
    ret = attach_service_locked(vif, type);
    if (ret < 0) {
        osal_mem_free(vif);
        osal_mutex_unlock(&g_vif_mgr.lock);
        return NULL;
    }
    
    /* netdev 생성 */
    vif->ndev = netdev_if_create(vif, vif_type_to_nl80211(type), name);
    if (!vif->ndev) {
        detach_service_locked(vif);
        osal_mem_free(vif);
        osal_mutex_unlock(&g_vif_mgr.lock);
        return NULL;
    }
    
    g_vif_mgr.vif_table[vif_id] = vif;
    g_vif_mgr.vif_count++;
    
    osal_mutex_unlock(&g_vif_mgr.lock);
    
    OSAL_INFO("VIF created: id=%d, type=%s, name=%s\n",
              vif_id, vif_type_to_str(type), name);
    
    return vif;
}

/**
 * 타입별 서비스 연결 (내부 함수)
 */
static int attach_service_locked(struct vif_context *vif, enum vif_type type)
{
    switch (type) {
    case VIF_TYPE_STA:
        vif->service = svc_sta_create(vif);
        vif->svc_ops = svc_sta_get_ops();
        break;
        
    case VIF_TYPE_AP:
        vif->service = svc_mhs_create(vif);
        vif->svc_ops = svc_mhs_get_ops();
        break;
        
    case VIF_TYPE_P2P_DEVICE:
    case VIF_TYPE_P2P_GO:
    case VIF_TYPE_P2P_GC:
        vif->service = svc_p2p_create(vif, type);
        vif->svc_ops = svc_p2p_get_ops();
        break;
        
    case VIF_TYPE_NAN:
        vif->service = svc_nan_create(vif);
        vif->svc_ops = svc_nan_get_ops();
        break;
        
    case VIF_TYPE_TEST:
        vif->service = svc_test_create(vif);
        vif->svc_ops = svc_test_get_ops();
        break;
        
    default:
        return -EINVAL;
    }
    
    if (!vif->service)
        return -ENOMEM;
    
    return 0;
}

/**
 * 디스패치 (다형성 적용)
 */
int vif_manager_dispatch_connect(struct vif_context *vif,
                                  struct cfg80211_connect_params *params)
{
    if (!vif || !vif->svc_ops)
        return -EINVAL;
    
    if (!vif->svc_ops->connect) {
        OSAL_ERR("VIF type %d does not support connect\n", vif->type);
        return -EOPNOTSUPP;
    }
    
    return vif->svc_ops->connect(vif->service, params);
}

int vif_manager_dispatch_start_ap(struct vif_context *vif,
                                   struct cfg80211_ap_settings *settings)
{
    if (!vif || !vif->svc_ops)
        return -EINVAL;
    
    if (!vif->svc_ops->start_ap) {
        OSAL_ERR("VIF type %d does not support start_ap\n", vif->type);
        return -EOPNOTSUPP;
    }
    
    return vif->svc_ops->start_ap(vif->service, settings);
}

/**
 * FW 이벤트 디스패치
 */
void vif_manager_dispatch_fw_event(int vif_id, struct fw_event *event)
{
    struct vif_context *vif;
    
    osal_mutex_lock(&g_vif_mgr.lock);
    
    vif = g_vif_mgr.vif_table[vif_id];
    if (!vif || !vif->svc_ops || !vif->svc_ops->handle_fw_event) {
        osal_mutex_unlock(&g_vif_mgr.lock);
        return;
    }
    
    osal_mutex_unlock(&g_vif_mgr.lock);
    
    vif->svc_ops->handle_fw_event(vif->service, event);
}
```

#### VIF 조합 정책

```c
/* vif_combination_policy.c */

/**
 * 지원하는 VIF 조합:
 * - STA only
 * - AP only  
 * - STA + AP
 * - STA + P2P_GO
 * - STA + P2P_GC
 * - STA + AP + P2P (triple)
 * - NAN (단독 또는 STA와만)
 * - TEST (단독)
 */

struct vif_combination {
    uint32_t types;      /* 허용되는 VIF 타입 비트맵 */
    int max_num;         /* 최대 VIF 개수 */
    bool mcc_allowed;    /* Multi-Channel Concurrency 허용 */
};

static const struct vif_combination allowed_combinations[] = {
    /* Single VIF */
    { 
        .types = BIT(VIF_TYPE_STA), 
        .max_num = 1,
        .mcc_allowed = false,
    },
    { 
        .types = BIT(VIF_TYPE_AP), 
        .max_num = 1,
        .mcc_allowed = false,
    },
    
    /* Dual VIF */
    { 
        .types = BIT(VIF_TYPE_STA) | BIT(VIF_TYPE_AP), 
        .max_num = 2,
        .mcc_allowed = true,
    },
    { 
        .types = BIT(VIF_TYPE_STA) | BIT(VIF_TYPE_P2P_GO), 
        .max_num = 2,
        .mcc_allowed = true,
    },
    { 
        .types = BIT(VIF_TYPE_STA) | BIT(VIF_TYPE_P2P_GC), 
        .max_num = 2,
        .mcc_allowed = true,
    },
    { 
        .types = BIT(VIF_TYPE_STA) | BIT(VIF_TYPE_NAN), 
        .max_num = 2,
        .mcc_allowed = false,
    },
    
    /* Triple VIF */
    { 
        .types = BIT(VIF_TYPE_STA) | BIT(VIF_TYPE_AP) | BIT(VIF_TYPE_P2P_GC), 
        .max_num = 3,
        .mcc_allowed = true,
    },
    
    /* Test mode (exclusive) */
    { 
        .types = BIT(VIF_TYPE_TEST), 
        .max_num = 1,
        .mcc_allowed = false,
    },
};

bool vif_manager_can_add_vif_locked(enum vif_type new_type)
{
    uint32_t current_types = 0;
    uint32_t proposed;
    int i, j;
    
    /* 현재 활성 VIF 타입 수집 */
    for (i = 0; i < MAX_VIF; i++) {
        if (g_vif_mgr.vif_table[i])
            current_types |= BIT(g_vif_mgr.vif_table[i]->type);
    }
    
    proposed = current_types | BIT(new_type);
    
    /* 허용된 조합인지 검사 */
    for (j = 0; j < ARRAY_SIZE(allowed_combinations); j++) {
        const struct vif_combination *comb = &allowed_combinations[j];
        
        if ((proposed & ~comb->types) == 0 &&  /* 허용된 타입만 포함 */
            (g_vif_mgr.vif_count + 1) <= comb->max_num) {
            return true;
        }
    }
    
    return false;
}
```

---

### 3.2 service_ops (서비스 인터페이스)

| 항목 | 내용 |
|------|------|
| **역할** | 서비스 모듈의 공통 인터페이스 정의 |
| **책임** | 다형성 지원, vif_manager에서 서비스 호출 통일 |

#### 인터페이스 정의

```c
/* service_ops.h */

/**
 * 서비스 공통 인터페이스
 * 
 * 각 서비스 모듈은 이 인터페이스를 구현한다.
 * vif_manager는 이 인터페이스를 통해 서비스를 호출한다.
 * 지원하지 않는 operation은 NULL로 설정한다.
 */
struct service_ops {
    /* 공통 라이프사이클 */
    int (*init)(void *service);
    void (*deinit)(void *service);
    int (*start)(void *service);
    void (*stop)(void *service);
    
    /* STA/P2P GC 용 */
    int (*scan)(void *service, struct cfg80211_scan_request *request);
    void (*abort_scan)(void *service);
    int (*connect)(void *service, struct cfg80211_connect_params *params);
    int (*disconnect)(void *service, u16 reason_code);
    int (*set_pmksa)(void *service, struct cfg80211_pmksa *pmksa);
    int (*del_pmksa)(void *service, struct cfg80211_pmksa *pmksa);
    
    /* AP/P2P GO 용 */
    int (*start_ap)(void *service, struct cfg80211_ap_settings *settings);
    int (*stop_ap)(void *service);
    int (*change_beacon)(void *service, struct cfg80211_beacon_data *beacon);
    int (*del_station)(void *service, const u8 *mac, u16 reason);
    int (*change_station)(void *service, const u8 *mac, 
                          struct station_parameters *params);
    
    /* P2P specific */
    int (*remain_on_channel)(void *service, struct ieee80211_channel *chan,
                             unsigned int duration, u64 *cookie);
    int (*cancel_remain_on_channel)(void *service, u64 cookie);
    int (*mgmt_tx)(void *service, struct cfg80211_mgmt_tx_params *params,
                   u64 *cookie);
    
    /* Key 관리 */
    int (*add_key)(void *service, u8 key_idx, bool pairwise,
                   const u8 *mac_addr, struct key_params *params);
    int (*del_key)(void *service, u8 key_idx, bool pairwise, const u8 *mac_addr);
    int (*set_default_key)(void *service, u8 key_idx, bool unicast, bool multicast);
    
    /* Power 관리 */
    int (*set_power_mgmt)(void *service, bool enabled, int timeout);
    
    /* FW 이벤트 수신 */
    void (*handle_fw_event)(void *service, struct fw_event *event);
    
    /* 상태 조회 */
    int (*get_rssi)(void *service);
    int (*get_station_info)(void *service, const u8 *mac, 
                            struct station_info *sinfo);
};
```

---

### 3.3 svc_sta (Station Service)

| 항목 | 내용 |
|------|------|
| **역할** | STA (Station) 모드 서비스 |
| **책임** | 인프라 모드 연결, 스캔, 로밍, 전력 관리 정책 결정 |
| **상태 머신** | DISCONNECTED → SCANNING → CONNECTING → CONNECTED → ROAMING |
| **Core 의존** | sme_sta |
| **권장 LOC** | < 1000 |

#### 상태 머신

```
┌─────────────────────────────────────────────────────────────────┐
│  svc_sta State Machine                                          │
│                                                                 │
│  ┌──────────────┐                                               │
│  │ DISCONNECTED │◀─────────────────────────────────┐            │
│  └──────┬───────┘                                  │            │
│         │ scan_request                             │            │
│         ▼                                          │            │
│  ┌──────────────┐                                  │            │
│  │   SCANNING   │──scan_done──┐                    │            │
│  └──────┬───────┘             │                    │            │
│         │ connect             │ (no connect)       │            │
│         ▼                     ▼                    │            │
│  ┌──────────────┐      ┌──────────────┐           │            │
│  │  CONNECTING  │      │ DISCONNECTED │           │            │
│  └──────┬───────┘      └──────────────┘           │            │
│         │ success                                  │            │
│         ▼                                          │ disconnect │
│  ┌──────────────┐                                  │            │
│  │  CONNECTED   │──────────────────────────────────┘            │
│  └──────┬───────┘                                               │
│         │ rssi_low / beacon_loss                                │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │   ROAMING    │───roam_success──▶ CONNECTED                  │
│  └──────────────┘───roam_fail─────▶ DISCONNECTED               │
└─────────────────────────────────────────────────────────────────┘
```

#### 자료 구조

```c
/* svc_sta.h */

enum svc_sta_state {
    SVC_STA_STATE_DISCONNECTED,
    SVC_STA_STATE_SCANNING,
    SVC_STA_STATE_CONNECTING,
    SVC_STA_STATE_CONNECTED,
    SVC_STA_STATE_ROAMING,
    SVC_STA_STATE_DISCONNECTING,
};

struct svc_sta_context {
    struct vif_context *vif;
    enum svc_sta_state state;
    
    /* 연결 정보 */
    u8 bssid[ETH_ALEN];
    u8 ssid[IEEE80211_MAX_SSID_LEN];
    int ssid_len;
    int channel;
    int rssi;
    
    /* Security */
    enum nl80211_auth_type auth_type;
    u32 cipher_pairwise;
    u32 cipher_group;
    u32 wpa_versions;
    
    /* 로밍 설정 */
    bool roam_enabled;
    int roam_rssi_threshold;
    int roam_rssi_delta;
    
    /* 전력 관리 */
    bool ps_enabled;
    int ps_timeout;
    
    /* 타이머 */
    osal_timer_t connect_timer;
    osal_timer_t roam_scan_timer;
    osal_timer_t keepalive_timer;
    
    /* 스캔 요청 저장 */
    struct cfg80211_scan_request *scan_request;
    
    /* 통계 */
    struct svc_sta_stats stats;
};
```

#### 인터페이스

```c
/* svc_sta.h */

/**
 * 생성/삭제
 */
void *svc_sta_create(struct vif_context *vif);
void svc_sta_destroy(void *service);
const struct service_ops *svc_sta_get_ops(void);

/**
 * 상태 조회
 */
enum svc_sta_state svc_sta_get_state(void *service);
bool svc_sta_is_connected(void *service);
```

#### 구현 예시

```c
/* svc_sta.c */

/**
 * 생성
 */
void *svc_sta_create(struct vif_context *vif)
{
    struct svc_sta_context *sta;
    
    sta = osal_mem_zalloc(sizeof(*sta));
    if (!sta)
        return NULL;
    
    sta->vif = vif;
    sta->state = SVC_STA_STATE_DISCONNECTED;
    
    /* 기본 설정 */
    sta->roam_enabled = true;
    sta->roam_rssi_threshold = -70;  /* dBm */
    sta->roam_rssi_delta = 10;
    sta->ps_enabled = true;
    
    /* 타이머 초기화 */
    osal_timer_init(&sta->connect_timer, connect_timeout_handler, sta);
    osal_timer_init(&sta->roam_scan_timer, roam_scan_handler, sta);
    osal_timer_init(&sta->keepalive_timer, keepalive_handler, sta);
    
    OSAL_INFO("svc_sta created for vif %d\n", vif->vif_id);
    
    return sta;
}

/**
 * Scan
 */
static int svc_sta_scan(void *service, struct cfg80211_scan_request *request)
{
    struct svc_sta_context *sta = service;
    int ret;
    
    OSAL_DBG("svc_sta_scan: state=%d, n_channels=%d\n", 
             sta->state, request->n_channels);
    
    /* 상태 검사 */
    if (sta->state == SVC_STA_STATE_CONNECTING ||
        sta->state == SVC_STA_STATE_DISCONNECTING) {
        OSAL_WARN("Cannot scan in state %d\n", sta->state);
        return -EBUSY;
    }
    
    /* 스캔 요청 저장 */
    sta->scan_request = request;
    
    /* 이전 상태 저장하고 SCANNING으로 전이 */
    if (sta->state == SVC_STA_STATE_DISCONNECTED)
        sta->state = SVC_STA_STATE_SCANNING;
    
    /* Core SME로 위임 */
    ret = sme_sta_scan_request(sta->vif, request);
    if (ret < 0) {
        sta->scan_request = NULL;
        if (sta->state == SVC_STA_STATE_SCANNING)
            sta->state = SVC_STA_STATE_DISCONNECTED;
    }
    
    return ret;
}

/**
 * Connect
 */
static int svc_sta_connect(void *service, struct cfg80211_connect_params *params)
{
    struct svc_sta_context *sta = service;
    int ret;
    
    OSAL_INFO("svc_sta_connect: ssid=%.*s\n", 
              (int)params->ssid_len, params->ssid);
    
    /* 상태 검사 */
    if (sta->state != SVC_STA_STATE_DISCONNECTED &&
        sta->state != SVC_STA_STATE_SCANNING) {
        OSAL_WARN("Cannot connect in state %d\n", sta->state);
        return -EALREADY;
    }
    
    /* 상태 전이 */
    sta->state = SVC_STA_STATE_CONNECTING;
    
    /* 연결 정보 저장 */
    memcpy(sta->ssid, params->ssid, params->ssid_len);
    sta->ssid_len = params->ssid_len;
    if (params->bssid)
        memcpy(sta->bssid, params->bssid, ETH_ALEN);
    
    /* Security 정보 저장 */
    sta->auth_type = params->auth_type;
    sta->wpa_versions = params->crypto.wpa_versions;
    if (params->crypto.n_ciphers_pairwise > 0)
        sta->cipher_pairwise = params->crypto.ciphers_pairwise[0];
    sta->cipher_group = params->crypto.cipher_group;
    
    /* 타임아웃 타이머 시작 */
    osal_timer_start(&sta->connect_timer, CONNECT_TIMEOUT_MS, false);
    
    /* Core SME로 위임 */
    ret = sme_sta_connect(sta->vif, params);
    if (ret < 0) {
        sta->state = SVC_STA_STATE_DISCONNECTED;
        osal_timer_stop(&sta->connect_timer);
    }
    
    return ret;
}

/**
 * FW 이벤트 핸들러
 */
static void svc_sta_handle_fw_event(void *service, struct fw_event *event)
{
    struct svc_sta_context *sta = service;
    
    OSAL_DBG("svc_sta_handle_fw_event: id=%d, state=%d\n", 
             event->id, sta->state);
    
    switch (event->id) {
    case MLME_SCAN_DONE_IND:
        handle_scan_done(sta, event);
        break;
        
    case MLME_CONNECT_IND:
        handle_connect_ind(sta, event);
        break;
        
    case MLME_DISCONNECT_IND:
        handle_disconnect_ind(sta, event);
        break;
        
    case MLME_ROAM_START_IND:
        handle_roam_start_ind(sta, event);
        break;
        
    case MLME_ROAM_COMPLETE_IND:
        handle_roam_complete_ind(sta, event);
        break;
        
    case MLME_RSSI_IND:
        handle_rssi_ind(sta, event);
        break;
        
    case MLME_BEACON_LOSS_IND:
        handle_beacon_loss_ind(sta, event);
        break;
        
    default:
        OSAL_DBG("Unhandled event: %d\n", event->id);
        break;
    }
}

/**
 * 연결 완료 처리
 */
static void handle_connect_ind(struct svc_sta_context *sta,
                               struct fw_event *event)
{
    struct mlme_connect_ind *ind = event->data;
    
    /* 타이머 중지 */
    osal_timer_stop(&sta->connect_timer);
    
    if (ind->result_code == 0) {
        /* 연결 성공 */
        sta->state = SVC_STA_STATE_CONNECTED;
        memcpy(sta->bssid, ind->bssid, ETH_ALEN);
        sta->channel = ind->channel;
        sta->vif->connected = true;
        sta->vif->channel = ind->channel;
        
        /* cfg80211에 알림 */
        cfg80211_if_connect_result(sta->vif->ndev, ind->bssid,
                                   ind->req_ie, ind->req_ie_len,
                                   ind->resp_ie, ind->resp_ie_len,
                                   WLAN_STATUS_SUCCESS);
        
        /* netdev carrier on */
        netdev_if_carrier_on(sta->vif->ndev);
        
        /* Keepalive 타이머 시작 */
        if (sta->ps_enabled)
            osal_timer_start(&sta->keepalive_timer, KEEPALIVE_INTERVAL_MS, true);
        
        OSAL_INFO("Connected to %pM on channel %d\n", sta->bssid, sta->channel);
        
    } else {
        /* 연결 실패 */
        sta->state = SVC_STA_STATE_DISCONNECTED;
        sta->vif->connected = false;
        
        cfg80211_if_connect_result(sta->vif->ndev, NULL,
                                   NULL, 0, NULL, 0,
                                   ind->result_code);
        
        OSAL_INFO("Connect failed: reason=%d\n", ind->result_code);
    }
}

/**
 * 로밍 결정 (정책)
 */
static void handle_rssi_ind(struct svc_sta_context *sta, struct fw_event *event)
{
    struct mlme_rssi_ind *ind = event->data;
    
    sta->rssi = ind->rssi;
    sta->vif->rssi = ind->rssi;
    
    /* 로밍 트리거 조건 검사 (정책 결정) */
    if (sta->roam_enabled && 
        sta->state == SVC_STA_STATE_CONNECTED &&
        ind->rssi < sta->roam_rssi_threshold) {
        
        OSAL_INFO("RSSI low (%d dBm), triggering roam scan\n", ind->rssi);
        
        /* 로밍 스캔 타이머 시작 (곧바로 하지 않고 디바운스) */
        if (!osal_timer_is_active(&sta->roam_scan_timer))
            osal_timer_start(&sta->roam_scan_timer, ROAM_SCAN_DELAY_MS, false);
    }
}

/**
 * service_ops 등록
 */
static const struct service_ops sta_ops = {
    .init = svc_sta_init,
    .deinit = svc_sta_deinit,
    .scan = svc_sta_scan,
    .abort_scan = svc_sta_abort_scan,
    .connect = svc_sta_connect,
    .disconnect = svc_sta_disconnect,
    .set_pmksa = svc_sta_set_pmksa,
    .del_pmksa = svc_sta_del_pmksa,
    .add_key = svc_sta_add_key,
    .del_key = svc_sta_del_key,
    .set_default_key = svc_sta_set_default_key,
    .set_power_mgmt = svc_sta_set_power_mgmt,
    .handle_fw_event = svc_sta_handle_fw_event,
    .get_rssi = svc_sta_get_rssi,
    .get_station_info = svc_sta_get_station_info,
};

const struct service_ops *svc_sta_get_ops(void)
{
    return &sta_ops;
}
```

---

### 3.4 svc_mhs (Mobile Hotspot Service)

| 항목 | 내용 |
|------|------|
| **역할** | MHS (Mobile Hotspot / SoftAP) 모드 서비스 |
| **책임** | AP 시작/중지, 연결된 STA 관리, Beacon 관리, ACS |
| **상태 머신** | STOPPED → STARTING → STARTED → STOPPING |
| **Core 의존** | sme_ap |
| **권장 LOC** | < 800 |

#### 상태 머신

```
┌─────────────────────────────────────────────────────────────────┐
│  svc_mhs State Machine                                          │
│                                                                 │
│  ┌──────────────┐                                               │
│  │   STOPPED    │◀────────────────────────────────┐             │
│  └──────┬───────┘                                 │             │
│         │ start_ap                                │             │
│         ▼                                         │             │
│  ┌──────────────┐                                 │             │
│  │   STARTING   │                                 │             │
│  └──────┬───────┘                                 │             │
│         │ start_cfm (success)                     │ stop_ap     │
│         ▼                                         │             │
│  ┌──────────────┐                                 │             │
│  │   STARTED    │─────────────────────────────────┤             │
│  └──────┬───────┘                                 │             │
│         │ stop_ap                                 │             │
│         ▼                                         │             │
│  ┌──────────────┐                                 │             │
│  │   STOPPING   │─────────────────────────────────┘             │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
```

#### 자료 구조

```c
/* svc_mhs.h */

#define MAX_AP_STATIONS 10

enum svc_mhs_state {
    SVC_MHS_STATE_STOPPED,
    SVC_MHS_STATE_STARTING,
    SVC_MHS_STATE_STARTED,
    SVC_MHS_STATE_STOPPING,
};

struct svc_mhs_station {
    u8 mac[ETH_ALEN];
    u16 aid;
    bool authorized;
    int rssi;
    uint64_t rx_bytes;
    uint64_t tx_bytes;
    uint64_t connected_time;
};

struct svc_mhs_context {
    struct vif_context *vif;
    enum svc_mhs_state state;
    
    /* AP 설정 */
    u8 ssid[IEEE80211_MAX_SSID_LEN];
    int ssid_len;
    int channel;
    int bandwidth;
    bool hidden_ssid;
    int max_stations;
    
    /* Security */
    enum nl80211_auth_type auth_type;
    u32 cipher_pairwise;
    u32 cipher_group;
    
    /* 연결된 STA 목록 */
    struct svc_mhs_station stations[MAX_AP_STATIONS];
    int station_count;
    osal_mutex_t sta_lock;
    
    /* Beacon */
    u8 *beacon_head;
    int beacon_head_len;
    u8 *beacon_tail;
    int beacon_tail_len;
    int beacon_interval;
    int dtim_period;
    
    /* ACS (Auto Channel Selection) */
    bool acs_enabled;
    int acs_result_channel;
    
    /* 타이머 */
    osal_timer_t inactivity_timer;
};
```

---

### 3.5 svc_p2p (Wi-Fi Direct Service)

| 항목 | 내용 |
|------|------|
| **역할** | Wi-Fi Direct (P2P) 서비스 |
| **책임** | P2P Discovery, GO Negotiation, Group 관리 |
| **특징** | GO 모드에서 sme_ap 재사용, GC 모드에서 sme_sta 재사용 |
| **Core 의존** | p2p_mgmt, sme_ap (GO), sme_sta (GC) |
| **권장 LOC** | < 1000 |

#### 상태 머신

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  svc_p2p State Machine                                                      │
│                                                                             │
│  ┌────────────┐                                                             │
│  │   IDLE     │◀──────────────────────────────────────────────┐             │
│  └─────┬──────┘                                               │             │
│        │ find/listen                                          │             │
│        ▼                                                      │             │
│  ┌────────────┐                                               │             │
│  │ DISCOVERING│──────────────────────────────┐                │             │
│  └─────┬──────┘                              │                │             │
│        │ prov_disc / go_nego_req             │ cancel         │             │
│        ▼                                     ▼                │             │
│  ┌────────────┐                        ┌────────────┐         │             │
│  │ NEGOTIATING│                        │   IDLE     │         │             │
│  └─────┬──────┘                        └────────────┘         │             │
│        │                                                      │             │
│        ├─────(I'm GO)────▶┌────────────┐                      │             │
│        │                  │  GROUP_GO  │──stop_group──────────┤             │
│        │                  └────────────┘                      │             │
│        │                                                      │             │
│        └─────(I'm GC)────▶┌────────────┐                      │             │
│                           │  GROUP_GC  │──leave_group─────────┘             │
│                           └────────────┘                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 역할 위임 (상속/위임 패턴)

```c
/* svc_p2p.c */

/**
 * P2P는 역할(GO/GC)에 따라 다른 Core 모듈로 위임한다.
 * 
 * GO 역할: sme_ap의 기능을 재사용
 * GC 역할: sme_sta의 기능을 재사용
 * P2P specific: p2p_mgmt에서 처리 (discovery, negotiation, NOA 등)
 */

enum p2p_role {
    P2P_ROLE_DEVICE,
    P2P_ROLE_GO,
    P2P_ROLE_GC,
};

struct svc_p2p_context {
    struct vif_context *vif;
    enum svc_p2p_state state;
    enum p2p_role role;
    
    /* P2P Device Info */
    u8 device_addr[ETH_ALEN];
    char device_name[32];
    u16 config_methods;
    u8 primary_dev_type[8];
    
    /* Discovery */
    int listen_channel;
    unsigned int listen_duration;
    bool in_listen;
    
    /* GO Negotiation */
    u8 go_intent;
    bool persistent;
    
    /* Group */
    u8 group_bssid[ETH_ALEN];
    u8 go_dev_addr[ETH_ALEN];
    
    /* 역할별 위임 */
    union {
        struct {
            /* GO 모드일 때 sme_ap context */
            void *ap_ctx;
        } go;
        struct {
            /* GC 모드일 때 sme_sta context */
            void *sta_ctx;
        } gc;
    } role_ctx;
    
    /* ROC (Remain on Channel) */
    struct ieee80211_channel *roc_channel;
    unsigned int roc_duration;
    u64 roc_cookie;
    osal_timer_t roc_timer;
};

/**
 * Connect (GC 역할일 때)
 */
static int svc_p2p_connect(void *service, struct cfg80211_connect_params *params)
{
    struct svc_p2p_context *p2p = service;
    
    /* GC 역할일 때만 유효 */
    if (p2p->role != P2P_ROLE_GC) {
        OSAL_ERR("Connect not allowed in role %d\n", p2p->role);
        return -EINVAL;
    }
    
    /* sme_sta로 위임 */
    return sme_sta_connect(p2p->vif, params);
}

/**
 * Start AP (GO 역할일 때)
 */
static int svc_p2p_start_ap(void *service, struct cfg80211_ap_settings *settings)
{
    struct svc_p2p_context *p2p = service;
    
    /* GO 역할일 때만 유효 */
    if (p2p->role != P2P_ROLE_GO) {
        OSAL_ERR("Start AP not allowed in role %d\n", p2p->role);
        return -EINVAL;
    }
    
    /* sme_ap로 위임 */
    return sme_ap_start(p2p->vif, settings);
}

/**
 * GO Negotiation 결과 처리
 */
static void handle_go_nego_result(struct svc_p2p_context *p2p,
                                  struct p2p_go_nego_result *result)
{
    if (result->role_go) {
        /* 내가 GO */
        p2p->role = P2P_ROLE_GO;
        p2p->state = SVC_P2P_STATE_GROUP_GO;
        
        memcpy(p2p->group_bssid, result->bssid, ETH_ALEN);
        
        OSAL_INFO("P2P: I am GO, BSSID=%pM\n", p2p->group_bssid);
        
        /* AP 시작 준비 - sme_ap 활용 */
        /* 실제 AP start는 cfg80211 start_ap callback에서 */
        
    } else {
        /* 내가 GC */
        p2p->role = P2P_ROLE_GC;
        p2p->state = SVC_P2P_STATE_GROUP_GC;
        
        memcpy(p2p->go_dev_addr, result->go_dev_addr, ETH_ALEN);
        memcpy(p2p->group_bssid, result->bssid, ETH_ALEN);
        
        OSAL_INFO("P2P: I am GC, GO=%pM\n", p2p->go_dev_addr);
        
        /* 연결 준비 - sme_sta 활용 */
        /* 실제 connect는 cfg80211 connect callback에서 */
    }
}
```

---

### 3.6 service_common

| 항목 | 내용 |
|------|------|
| **역할** | 서비스 모듈들의 공통 유틸리티 |
| **책임** | BSS 관리, IE 처리, 이벤트 노티피케이션, 통계 |
| **특징** | Stateless, 순수 헬퍼 함수 |

#### 인터페이스

```c
/* service_common.h */

/**
 * BSS 관리
 */
struct bss_info {
    struct list_head list;
    u8 bssid[ETH_ALEN];
    u8 ssid[IEEE80211_MAX_SSID_LEN];
    int ssid_len;
    int channel;
    int rssi;
    u64 timestamp;
    u16 beacon_interval;
    u16 capability;
    u8 *ies;
    int ies_len;
};

struct bss_info *svc_common_bss_alloc(void);
void svc_common_bss_free(struct bss_info *bss);
struct bss_info *svc_common_bss_find_by_bssid(struct list_head *list, 
                                               const u8 *bssid);
struct bss_info *svc_common_bss_find_by_ssid(struct list_head *list,
                                              const u8 *ssid, int ssid_len);
void svc_common_bss_update(struct bss_info *bss, 
                           struct cfg80211_bss *cfg_bss);
void svc_common_bss_list_flush(struct list_head *list);

/**
 * IE (Information Element) 처리
 */
const u8 *svc_common_find_ie(const u8 *ies, int ies_len, u8 eid);
const u8 *svc_common_find_ext_ie(const u8 *ies, int ies_len, u8 ext_eid);
const u8 *svc_common_find_vendor_ie(const u8 *ies, int ies_len, 
                                     u32 oui, u8 oui_type);
int svc_common_build_ie(u8 *buf, int buf_len, u8 eid, const u8 *data, int len);
int svc_common_build_vendor_ie(u8 *buf, int buf_len, u32 oui, u8 oui_type,
                                const u8 *data, int len);

/**
 * cfg80211 이벤트 래퍼
 */
void svc_common_report_scan_done(struct vif_context *vif, bool aborted);
void svc_common_report_scan_result(struct vif_context *vif,
                                    struct cfg80211_bss *bss);
void svc_common_report_connect_result(struct vif_context *vif, 
                                       const u8 *bssid, u16 status,
                                       const u8 *req_ie, size_t req_ie_len,
                                       const u8 *resp_ie, size_t resp_ie_len);
void svc_common_report_disconnect(struct vif_context *vif, u16 reason,
                                   bool from_ap);
void svc_common_report_roamed(struct vif_context *vif,
                               struct cfg80211_roam_info *info);

/**
 * 채널 유틸리티
 */
int svc_common_freq_to_channel(int freq);
int svc_common_channel_to_freq(int channel, enum nl80211_band band);
enum nl80211_band svc_common_channel_to_band(int channel);

/**
 * MAC 주소 유틸리티
 */
bool svc_common_is_broadcast_addr(const u8 *addr);
bool svc_common_is_multicast_addr(const u8 *addr);
bool svc_common_is_zero_addr(const u8 *addr);
```

---

## 4. 주목할 특성

### 4.1 다형성 (Polymorphism)

`service_ops` 구조체를 통해 `vif_manager`가 서비스 타입과 무관하게 동작:

```c
/* vif_manager는 구체적인 서비스 타입을 몰라도 됨 */
int vif_manager_dispatch_connect(struct vif_context *vif,
                                  struct cfg80211_connect_params *params)
{
    /* 각 서비스의 connect 함수 호출 (다형성) */
    if (vif->svc_ops && vif->svc_ops->connect)
        return vif->svc_ops->connect(vif->service, params);
    return -EOPNOTSUPP;
}
```

### 4.2 위임 패턴 (Delegation)

P2P가 GO/GC 역할에 따라 기존 모듈을 재사용:

```
                        ┌─────────────┐
                        │   svc_p2p   │
                        └──────┬──────┘
                               │
               ┌───────────────┼───────────────┐
               │ GO mode       │               │ GC mode
               ▼               │               ▼
        ┌─────────────┐        │        ┌─────────────┐
        │   sme_ap    │        │        │   sme_sta   │
        │ (재사용)     │        │        │ (재사용)     │
        └─────────────┘        │        └─────────────┘
                               │
                   (P2P specific: p2p_mgmt)
```

### 4.3 God Module 방지

| 모듈 | 책임 | 권장 LOC |
|------|------|----------|
| vif_manager | VIF CRUD, 라우팅, 정책 검사 | < 500 |
| svc_sta | STA 상태 머신, 정책 결정 | < 1000 |
| svc_mhs | AP 상태 머신, STA 관리 | < 800 |
| svc_p2p | P2P 상태 머신, GO/GC 위임 | < 1000 |
| service_common | Stateless 유틸리티 | < 500 |

### 4.4 의존성 규칙

```
Layer 2 (cfg80211_if) ──▶ vif_manager ──▶ svc_* ──▶ Core (Layer 4)
                              │
                              ▼
                       service_common

규칙:
- vif_manager는 svc_*를 알지만, svc_*는 vif_manager를 직접 호출 안함
- svc_* 간 직접 호출 금지 (vif_manager 통해서만)
- service_common은 모든 svc_*가 사용 가능 (stateless)
- FW 이벤트는 vif_manager가 dispatch
```
# Layer 5: FW Message Layer

## 1. 개요

### 1.1 역할

Driver와 Firmware 간의 메시지 통신을 담당한다. 메시지 라우팅, 직렬화/역직렬화, 버전 관리를 수행한다.

### 1.2 설계 목표

- 메시지 카테고리별 분리 (MLME, MA, DEBUG, TEST)
- FW 메시지 포맷 변경 시 영향 범위 최소화
- 메시지 버전 호환성 관리

### 1.3 메시지 방향

| 방향 | 타입 | 설명 |
|------|------|------|
| Driver → FW | REQ (Request) | 드라이버가 FW에 요청 |
| FW → Driver | CFM (Confirm) | REQ에 대한 FW 응답 |
| FW → Driver | IND (Indication) | FW가 자발적으로 알림 |

---

## 2. 모듈 구성

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 5: FW Message Layer                                                      │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                           msg_router                                     │   │
│  │                    (메시지 라우팅, 송수신 관리)                            │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                     │                                           │
│       ┌─────────────┬───────────────┼───────────────┬─────────────┐            │
│       ▼             ▼               ▼               ▼             ▼            │
│  ┌─────────┐  ┌─────────┐     ┌─────────┐     ┌───────────┐ ┌───────────┐     │
│  │msg_mlme │  │ msg_ma  │     │msg_debug│     │msg_wlanlite│ │msg_system │     │
│  └─────────┘  └─────────┘     └─────────┘     └───────────┘ └───────────┘     │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    msg_builder / msg_parser                              │   │
│  │                    (직렬화/역직렬화 유틸리티)                              │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 메시지 구조

### 3.1 공통 헤더

```c
/* msg_common.h */

/**
 * 메시지 카테고리
 */
enum msg_category {
    MSG_CAT_SYSTEM = 0,      /* 시스템 제어 */
    MSG_CAT_MLME,            /* Management */
    MSG_CAT_MA,              /* Data Path */
    MSG_CAT_DEBUG,           /* 디버그 */
    MSG_CAT_WLANLITE,        /* RF 테스트 */
    MSG_CAT_MAX,
};

/**
 * 메시지 방향/타입
 */
enum msg_type {
    MSG_TYPE_REQ = 0,        /* Request: Driver → FW */
    MSG_TYPE_CFM,            /* Confirm: FW → Driver (REQ 응답) */
    MSG_TYPE_IND,            /* Indication: FW → Driver (자발적) */
};

/**
 * 공통 메시지 헤더 (모든 메시지에 포함)
 */
struct msg_header {
    u16 msg_id;              /* 메시지 ID (카테고리 내 고유) */
    u16 msg_len;             /* 헤더 제외한 바디 길이 */
    u8 category;             /* enum msg_category */
    u8 type;                 /* enum msg_type */
    u8 vif_id;               /* VIF ID (0~2) */
    u8 seq_num;              /* 시퀀스 번호 */
    u16 status;              /* CFM일 때 결과 코드 */
    u16 reserved;
} __packed;

#define MSG_HEADER_SIZE  sizeof(struct msg_header)
#define MSG_MAX_SIZE     4096
#define MSG_MAX_BODY     (MSG_MAX_SIZE - MSG_HEADER_SIZE)
```

### 3.2 메시지 ID 정의

```c
/* msg_mlme_ids.h */

/**
 * MLME 메시지 ID
 */
enum mlme_msg_id {
    /* Scan */
    MLME_SCAN_REQ = 0x0001,
    MLME_SCAN_CFM,
    MLME_SCAN_DONE_IND,
    MLME_SCAN_RESULT_IND,
    
    /* Connect */
    MLME_CONNECT_REQ = 0x0010,
    MLME_CONNECT_CFM,
    MLME_CONNECT_IND,
    
    /* Disconnect */
    MLME_DISCONNECT_REQ = 0x0020,
    MLME_DISCONNECT_CFM,
    MLME_DISCONNECT_IND,
    
    /* Roaming */
    MLME_ROAM_REQ = 0x0030,
    MLME_ROAM_CFM,
    MLME_ROAM_START_IND,
    MLME_ROAM_COMPLETE_IND,
    
    /* AP */
    MLME_START_AP_REQ = 0x0040,
    MLME_START_AP_CFM,
    MLME_STOP_AP_REQ,
    MLME_STOP_AP_CFM,
    MLME_STA_CONNECT_IND,
    MLME_STA_DISCONNECT_IND,
    
    /* Key */
    MLME_ADD_KEY_REQ = 0x0050,
    MLME_ADD_KEY_CFM,
    MLME_DEL_KEY_REQ,
    MLME_DEL_KEY_CFM,
    
    /* Power Save */
    MLME_SET_PS_REQ = 0x0060,
    MLME_SET_PS_CFM,
    
    /* Misc Indications */
    MLME_RSSI_IND = 0x0070,
    MLME_BEACON_LOSS_IND,
    MLME_MIC_FAILURE_IND,
    
    /* 11k/11v */
    MLME_NEIGHBOR_REP_REQ = 0x0080,
    MLME_NEIGHBOR_REP_IND,
    MLME_BTM_REQ_IND,
    MLME_BTM_RESP_REQ,
};

/* msg_ma_ids.h */

/**
 * MA (Data Path) 메시지 ID
 */
enum ma_msg_id {
    /* TX */
    MA_TX_REQ = 0x0001,
    MA_TX_CFM,
    
    /* RX */
    MA_RX_IND = 0x0010,
    
    /* Block Ack */
    MA_ADDBA_REQ = 0x0020,
    MA_ADDBA_CFM,
    MA_DELBA_REQ,
    MA_DELBA_CFM,
    MA_ADDBA_IND,
    MA_DELBA_IND,
    
    /* Flow Control */
    MA_FLOW_CTRL_IND = 0x0030,
};

/* msg_debug_ids.h */

/**
 * DEBUG 메시지 ID
 */
enum debug_msg_id {
    DEBUG_SET_LOG_LEVEL_REQ = 0x0001,
    DEBUG_SET_LOG_LEVEL_CFM,
    DEBUG_GET_FW_VERSION_REQ,
    DEBUG_GET_FW_VERSION_CFM,
    DEBUG_TRIGGER_DUMP_REQ,
    DEBUG_DUMP_READY_IND,
    DEBUG_LOG_IND,
};

/* msg_wlanlite_ids.h */

/**
 * WLANLITE (RF Test) 메시지 ID
 */
enum wlanlite_msg_id {
    WLANLITE_START_REQ = 0x0001,
    WLANLITE_START_CFM,
    WLANLITE_STOP_REQ,
    WLANLITE_STOP_CFM,
    WLANLITE_TX_START_REQ,
    WLANLITE_TX_START_CFM,
    WLANLITE_TX_STOP_REQ,
    WLANLITE_TX_STOP_CFM,
    WLANLITE_RX_START_REQ,
    WLANLITE_RX_START_CFM,
    WLANLITE_RX_STOP_REQ,
    WLANLITE_RX_STOP_CFM,
    WLANLITE_RX_STAT_IND,
    WLANLITE_SET_CHANNEL_REQ,
    WLANLITE_SET_CHANNEL_CFM,
    WLANLITE_SET_TX_POWER_REQ,
    WLANLITE_SET_TX_POWER_CFM,
};

/* msg_system_ids.h */

/**
 * SYSTEM 메시지 ID
 */
enum system_msg_id {
    SYSTEM_INIT_REQ = 0x0001,
    SYSTEM_INIT_CFM,
    SYSTEM_DEINIT_REQ,
    SYSTEM_DEINIT_CFM,
    SYSTEM_SET_MAC_ADDR_REQ,
    SYSTEM_SET_MAC_ADDR_CFM,
    SYSTEM_SET_COUNTRY_REQ,
    SYSTEM_SET_COUNTRY_CFM,
    SYSTEM_FW_READY_IND,
    SYSTEM_ERROR_IND,
    SYSTEM_WATCHDOG_IND,
};
```

---

## 4. 모듈 상세

### 4.1 msg_router

| 항목 | 내용 |
|------|------|
| **역할** | 메시지 라우팅 및 송수신 관리 |
| **책임** | 카테고리별 핸들러 디스패치, 시퀀스 관리, 동기/비동기 처리 |
| **의존성** | OSAL, HIP (Layer 6) |

#### 인터페이스

```c
/* msg_router.h */

/**
 * 초기화/해제
 */
int msg_router_init(void);
void msg_router_deinit(void);

/**
 * 메시지 전송 (Driver → FW)
 */
int msg_router_send(enum msg_category cat, struct msg_header *hdr, void *body);

/**
 * 동기식 전송 (CFM 대기)
 */
int msg_router_send_sync(enum msg_category cat, struct msg_header *hdr, 
                         void *body, void *cfm_buf, size_t cfm_size,
                         unsigned long timeout_ms);

/**
 * 메시지 수신 (FW → Driver, HIP에서 호출)
 */
void msg_router_recv(void *data, size_t len);

/**
 * 카테고리별 핸들러 등록
 */
typedef void (*msg_handler_t)(u8 vif_id, u16 msg_id, void *body, u16 body_len);
int msg_router_register_handler(enum msg_category cat, msg_handler_t handler);
void msg_router_unregister_handler(enum msg_category cat);

/**
 * CFM 대기 콜백 등록 (비동기 CFM 처리)
 */
typedef void (*msg_cfm_callback_t)(u8 vif_id, u16 msg_id, u16 status, 
                                   void *body, u16 body_len, void *ctx);
int msg_router_register_cfm_callback(u8 seq_num, msg_cfm_callback_t callback,
                                     void *ctx, unsigned long timeout_ms);
```

#### 구현 예시

```c
/* msg_router.c */

struct msg_router {
    osal_mutex_t lock;
    
    /* 카테고리별 핸들러 */
    msg_handler_t handlers[MSG_CAT_MAX];
    
    /* 시퀀스 번호 */
    osal_atomic_t seq_num;
    
    /* 동기 CFM 대기 */
    struct {
        bool waiting;
        u8 seq_num;
        void *cfm_buf;
        size_t cfm_size;
        osal_completion_t completion;
        int result;
    } sync_wait;
    
    /* 비동기 CFM 콜백 */
    struct cfm_pending {
        bool used;
        u8 seq_num;
        msg_cfm_callback_t callback;
        void *ctx;
        osal_timer_t timer;
    } cfm_pending[16];
    
    /* 통계 */
    struct {
        u64 tx_count;
        u64 rx_count;
        u64 tx_errors;
        u64 rx_errors;
        u64 timeouts;
    } stats;
};

static struct msg_router g_router;

/**
 * 메시지 전송
 */
int msg_router_send(enum msg_category cat, struct msg_header *hdr, void *body)
{
    u8 *buf;
    size_t total_len;
    int ret;
    
    /* 헤더 설정 */
    hdr->category = cat;
    hdr->type = MSG_TYPE_REQ;
    hdr->seq_num = osal_atomic_inc_return(&g_router.seq_num) & 0xFF;
    
    /* 버퍼 구성 */
    total_len = MSG_HEADER_SIZE + hdr->msg_len;
    buf = osal_mem_alloc(total_len);
    if (!buf)
        return -ENOMEM;
    
    memcpy(buf, hdr, MSG_HEADER_SIZE);
    if (hdr->msg_len > 0 && body)
        memcpy(buf + MSG_HEADER_SIZE, body, hdr->msg_len);
    
    /* HIP로 전송 */
    ret = hip_send(buf, total_len);
    
    if (ret == 0)
        g_router.stats.tx_count++;
    else
        g_router.stats.tx_errors++;
    
    osal_mem_free(buf);
    return ret;
}

/**
 * 동기식 전송 (CFM 대기)
 */
int msg_router_send_sync(enum msg_category cat, struct msg_header *hdr,
                         void *body, void *cfm_buf, size_t cfm_size,
                         unsigned long timeout_ms)
{
    int ret;
    
    osal_mutex_lock(&g_router.lock);
    
    /* 동기 대기 설정 */
    g_router.sync_wait.waiting = true;
    g_router.sync_wait.cfm_buf = cfm_buf;
    g_router.sync_wait.cfm_size = cfm_size;
    osal_completion_reinit(&g_router.sync_wait.completion);
    
    osal_mutex_unlock(&g_router.lock);
    
    /* 전송 */
    ret = msg_router_send(cat, hdr, body);
    if (ret < 0) {
        g_router.sync_wait.waiting = false;
        return ret;
    }
    
    /* 시퀀스 번호 저장 */
    g_router.sync_wait.seq_num = hdr->seq_num;
    
    /* CFM 대기 */
    ret = osal_completion_wait_timeout(&g_router.sync_wait.completion, 
                                        timeout_ms);
    
    g_router.sync_wait.waiting = false;
    
    if (ret == -ETIMEDOUT) {
        g_router.stats.timeouts++;
        OSAL_ERR("Timeout waiting for CFM: cat=%d, id=0x%04x\n", 
                 cat, hdr->msg_id);
        return -ETIMEDOUT;
    }
    
    return g_router.sync_wait.result;
}

/**
 * 메시지 수신 (HIP에서 호출)
 */
void msg_router_recv(void *data, size_t len)
{
    struct msg_header *hdr;
    void *body;
    u16 body_len;
    
    if (len < MSG_HEADER_SIZE) {
        OSAL_ERR("Message too short: %zu\n", len);
        g_router.stats.rx_errors++;
        return;
    }
    
    hdr = (struct msg_header *)data;
    body = (u8 *)data + MSG_HEADER_SIZE;
    body_len = hdr->msg_len;
    
    /* 길이 검증 */
    if (MSG_HEADER_SIZE + body_len > len) {
        OSAL_ERR("Message body exceeds buffer: %u + %u > %zu\n",
                 MSG_HEADER_SIZE, body_len, len);
        g_router.stats.rx_errors++;
        return;
    }
    
    g_router.stats.rx_count++;
    
    OSAL_DBG("msg_recv: cat=%d, type=%d, id=0x%04x, vif=%d, seq=%d\n",
             hdr->category, hdr->type, hdr->msg_id, hdr->vif_id, hdr->seq_num);
    
    /* CFM 처리 */
    if (hdr->type == MSG_TYPE_CFM) {
        handle_cfm(hdr, body, body_len);
        return;
    }
    
    /* IND 처리: 카테고리별 핸들러로 디스패치 */
    if (hdr->category < MSG_CAT_MAX && g_router.handlers[hdr->category]) {
        g_router.handlers[hdr->category](hdr->vif_id, hdr->msg_id, 
                                          body, body_len);
    } else {
        OSAL_WARN("No handler for category %d\n", hdr->category);
    }
}

/**
 * CFM 처리
 */
static void handle_cfm(struct msg_header *hdr, void *body, u16 body_len)
{
    int i;
    
    /* 동기 대기 중인 경우 */
    if (g_router.sync_wait.waiting && 
        g_router.sync_wait.seq_num == hdr->seq_num) {
        
        /* CFM 데이터 복사 */
        if (g_router.sync_wait.cfm_buf && body_len > 0) {
            size_t copy_len = (body_len < g_router.sync_wait.cfm_size) ? 
                              body_len : g_router.sync_wait.cfm_size;
            memcpy(g_router.sync_wait.cfm_buf, body, copy_len);
        }
        
        g_router.sync_wait.result = (hdr->status == 0) ? 0 : -EIO;
        osal_completion_complete(&g_router.sync_wait.completion);
        return;
    }
    
    /* 비동기 콜백 확인 */
    for (i = 0; i < ARRAY_SIZE(g_router.cfm_pending); i++) {
        struct cfm_pending *pending = &g_router.cfm_pending[i];
        
        if (pending->used && pending->seq_num == hdr->seq_num) {
            osal_timer_stop(&pending->timer);
            
            if (pending->callback) {
                pending->callback(hdr->vif_id, hdr->msg_id, hdr->status,
                                  body, body_len, pending->ctx);
            }
            
            pending->used = false;
            return;
        }
    }
    
    OSAL_DBG("Unexpected CFM: seq=%d\n", hdr->seq_num);
}
```

---

### 4.2 msg_mlme

| 항목 | 내용 |
|------|------|
| **역할** | MLME 카테고리 메시지 처리 |
| **책임** | MLME 메시지 생성, MLME CFM/IND 파싱 및 전달 |
| **의존성** | msg_router, OSAL |

#### 메시지 바디 구조체

```c
/* msg_mlme.h */

/**
 * MLME_SCAN_REQ body
 */
struct mlme_scan_req_body {
    u8 scan_type;            /* 0: passive, 1: active */
    u8 n_channels;
    u16 dwell_time;          /* ms */
    u16 channels[64];        /* 채널 리스트 */
    u8 ssid[32];
    u8 ssid_len;
    u8 bssid[ETH_ALEN];      /* 특정 BSSID 스캔 시 */
    u8 reserved[3];
} __packed;

/**
 * MLME_SCAN_RESULT_IND body
 */
struct mlme_scan_result_ind_body {
    u8 bssid[ETH_ALEN];
    u8 ssid[32];
    u8 ssid_len;
    u8 channel;
    s8 rssi;
    u8 capability[2];
    u16 beacon_interval;
    u16 ie_len;
    u8 ies[0];               /* Variable length */
} __packed;

/**
 * MLME_CONNECT_REQ body
 */
struct mlme_connect_req_body {
    u8 bssid[ETH_ALEN];
    u8 ssid[32];
    u8 ssid_len;
    u8 channel;
    u8 band;
    u8 auth_type;
    u32 wpa_versions;
    u32 cipher_pairwise;
    u32 cipher_group;
    u32 akm_suite;
    u8 mfp;
    u8 reserved[3];
    u16 rsn_ie_len;
    u8 rsn_ie[256];
    u16 extra_ie_len;
    u8 extra_ie[512];
} __packed;

/**
 * MLME_CONNECT_IND body
 */
struct mlme_connect_ind_body {
    u16 status_code;         /* 0: success */
    u8 bssid[ETH_ALEN];
    u8 channel;
    u8 reserved;
    u16 req_ie_len;
    u16 resp_ie_len;
    u8 ies[0];               /* req_ie + resp_ie */
} __packed;

/**
 * MLME_DISCONNECT_IND body
 */
struct mlme_disconnect_ind_body {
    u16 reason_code;
    u8 from_ap;              /* 1: AP initiated */
    u8 reserved;
} __packed;

/**
 * MLME_START_AP_REQ body
 */
struct mlme_start_ap_req_body {
    u8 ssid[32];
    u8 ssid_len;
    u8 hidden_ssid;
    u8 channel;
    u8 bandwidth;            /* 20/40/80/160 */
    u8 band;
    u8 reserved;
    u16 beacon_interval;
    u8 dtim_period;
    u8 max_stations;
    u32 wpa_versions;
    u32 cipher_pairwise;
    u32 cipher_group;
    u32 akm_suite;
    u16 beacon_head_len;
    u16 beacon_tail_len;
    u8 beacon_data[0];       /* head + tail */
} __packed;

/**
 * MLME_RSSI_IND body
 */
struct mlme_rssi_ind_body {
    s8 rssi;
    s8 snr;
    u8 reserved[2];
} __packed;
```

#### 인터페이스

```c
/* msg_mlme.h */

/**
 * MLME 핸들러 초기화
 */
int msg_mlme_init(void);
void msg_mlme_deinit(void);

/**
 * MLME IND 콜백 타입
 */
typedef void (*mlme_ind_callback_t)(u8 vif_id, u16 msg_id, void *body, u16 len);

/**
 * MLME IND 콜백 등록
 */
void msg_mlme_register_ind_callback(mlme_ind_callback_t callback);
```

#### 구현 예시

```c
/* msg_mlme.c */

static mlme_ind_callback_t g_mlme_ind_callback;

/**
 * 초기화
 */
int msg_mlme_init(void)
{
    return msg_router_register_handler(MSG_CAT_MLME, mlme_msg_handler);
}

/**
 * MLME 메시지 핸들러 (msg_router에서 호출)
 */
static void mlme_msg_handler(u8 vif_id, u16 msg_id, void *body, u16 body_len)
{
    OSAL_DBG("mlme_msg_handler: vif=%d, id=0x%04x, len=%u\n",
             vif_id, msg_id, body_len);
    
    /* Core Layer(mlme_handler)로 전달 */
    if (g_mlme_ind_callback)
        g_mlme_ind_callback(vif_id, msg_id, body, body_len);
}

void msg_mlme_register_ind_callback(mlme_ind_callback_t callback)
{
    g_mlme_ind_callback = callback;
}
```

---

### 4.3 msg_ma

| 항목 | 내용 |
|------|------|
| **역할** | MA (Data Path) 카테고리 메시지 처리 |
| **책임** | TX/RX 메시지 처리, Block Ack 메시지 처리 |
| **의존성** | msg_router, OSAL |

#### 메시지 바디 구조체

```c
/* msg_ma.h */

/**
 * MA_TX_REQ body
 */
struct ma_tx_req_body {
    u8 priority;             /* AC */
    u8 flags;                /* TX flags */
    u16 frame_len;
    u32 cookie;              /* TX complete 식별용 */
    u8 frame[0];             /* 802.11 frame 또는 Ethernet frame */
} __packed;

#define MA_TX_FLAG_NO_ACK     BIT(0)
#define MA_TX_FLAG_NO_ENCRYPT BIT(1)
#define MA_TX_FLAG_MGMT       BIT(2)

/**
 * MA_TX_CFM body
 */
struct ma_tx_cfm_body {
    u32 cookie;
    u8 status;               /* 0: success */
    u8 retries;
    u8 reserved[2];
} __packed;

/**
 * MA_RX_IND body
 */
struct ma_rx_ind_body {
    s8 rssi;
    u8 channel;
    u8 flags;
    u8 reserved;
    u16 frame_len;
    u8 frame[0];             /* 802.11 frame 또는 Ethernet frame */
} __packed;

#define MA_RX_FLAG_DECRYPTED  BIT(0)
#define MA_RX_FLAG_MGMT       BIT(1)

/**
 * MA_ADDBA_REQ body
 */
struct ma_addba_req_body {
    u8 peer_addr[ETH_ALEN];
    u8 tid;
    u8 direction;            /* 0: initiator, 1: responder */
    u16 buf_size;
    u16 timeout;
} __packed;

/**
 * MA_FLOW_CTRL_IND body
 */
struct ma_flow_ctrl_ind_body {
    u8 ac;                   /* Affected AC */
    u8 stop;                 /* 1: stop, 0: resume */
    u8 reserved[2];
} __packed;
```

#### 구현 예시

```c
/* msg_ma.c */

static ma_ind_callback_t g_ma_ind_callback;

int msg_ma_init(void)
{
    return msg_router_register_handler(MSG_CAT_MA, ma_msg_handler);
}

static void ma_msg_handler(u8 vif_id, u16 msg_id, void *body, u16 body_len)
{
    switch (msg_id) {
    case MA_RX_IND:
        handle_rx_ind(vif_id, body, body_len);
        break;
        
    case MA_TX_CFM:
        handle_tx_cfm(vif_id, body, body_len);
        break;
        
    case MA_FLOW_CTRL_IND:
        handle_flow_ctrl_ind(vif_id, body, body_len);
        break;
        
    case MA_ADDBA_IND:
    case MA_DELBA_IND:
        /* BA 이벤트는 Core ma_handler로 */
        if (g_ma_ind_callback)
            g_ma_ind_callback(vif_id, msg_id, body, body_len);
        break;
        
    default:
        OSAL_DBG("Unknown MA message: 0x%04x\n", msg_id);
        break;
    }
}

static void handle_rx_ind(u8 vif_id, void *body, u16 body_len)
{
    struct ma_rx_ind_body *rx = body;
    struct sk_buff *skb;
    struct vif_context *vif;
    
    vif = vif_manager_get_vif(vif_id);
    if (!vif)
        return;
    
    /* skb 할당 */
    skb = dev_alloc_skb(rx->frame_len + 32);
    if (!skb) {
        vif->ndev->stats.rx_dropped++;
        return;
    }
    
    skb_reserve(skb, 16);
    memcpy(skb_put(skb, rx->frame_len), rx->frame, rx->frame_len);
    
    /* netdev로 전달 */
    netdev_if_rx(vif->ndev, skb);
}

static void handle_flow_ctrl_ind(u8 vif_id, void *body, u16 body_len)
{
    struct ma_flow_ctrl_ind_body *fc = body;
    struct vif_context *vif;
    
    vif = vif_manager_get_vif(vif_id);
    if (!vif)
        return;
    
    if (fc->stop)
        netdev_if_stop_queue(vif->ndev);
    else
        netdev_if_wake_queue(vif->ndev);
}
```

---

### 4.4 msg_debug

| 항목 | 내용 |
|------|------|
| **역할** | DEBUG 카테고리 메시지 처리 |
| **책임** | 로그 레벨, FW 버전, 덤프 관련 메시지 |
| **의존성** | msg_router, OSAL |

#### 메시지 바디 구조체

```c
/* msg_debug.h */

/**
 * DEBUG_GET_FW_VERSION_CFM body
 */
struct debug_fw_version_cfm_body {
    u32 major;
    u32 minor;
    u32 patch;
    u32 build;
    char build_date[32];
    char build_hash[16];
} __packed;

/**
 * DEBUG_LOG_IND body
 */
struct debug_log_ind_body {
    u8 level;
    u8 reserved[3];
    u32 timestamp;
    u16 len;
    char message[0];
} __packed;

/**
 * DEBUG_DUMP_READY_IND body
 */
struct debug_dump_ready_ind_body {
    u32 dump_size;
    u32 dump_addr;           /* FW 메모리 주소 */
    u8 dump_type;            /* 0: full, 1: partial */
    u8 reserved[3];
} __packed;
```

#### 인터페이스

```c
/* msg_debug.h */

int msg_debug_init(void);
void msg_debug_deinit(void);

int msg_debug_set_log_level(u8 level);
int msg_debug_get_fw_version(struct debug_fw_version_cfm_body *version);
int msg_debug_trigger_dump(void);
```

---

### 4.5 msg_wlanlite

| 항목 | 내용 |
|------|------|
| **역할** | WLANLITE (RF Test) 카테고리 메시지 처리 |
| **책임** | RF 테스트 모드 진입/종료, TX/RX 테스트 |
| **의존성** | msg_router, OSAL |

#### 메시지 바디 구조체

```c
/* msg_wlanlite.h */

/**
 * WLANLITE_TX_START_REQ body
 */
struct wlanlite_tx_start_req_body {
    u16 channel;
    u8 bandwidth;            /* 20/40/80/160 */
    u8 rate_idx;
    s8 tx_power;             /* dBm */
    u8 antenna;
    u16 packet_len;
    u32 packet_count;        /* 0: continuous */
    u32 packet_interval;     /* us */
} __packed;

/**
 * WLANLITE_RX_STAT_IND body
 */
struct wlanlite_rx_stat_ind_body {
    u32 total_packets;
    u32 fcs_errors;
    u32 phy_errors;
    s8 avg_rssi;
    s8 min_rssi;
    s8 max_rssi;
    u8 reserved;
} __packed;
```

---

### 4.6 msg_builder / msg_parser

| 항목 | 내용 |
|------|------|
| **역할** | 메시지 직렬화/역직렬화 유틸리티 |
| **책임** | 바이트 오더 변환, 버전 호환성 처리 |
| **의존성** | OSAL |

#### 인터페이스

```c
/* msg_builder.h */

/**
 * 메시지 빌더 컨텍스트
 */
struct msg_builder {
    u8 *buf;
    size_t size;
    size_t pos;
};

void msg_builder_init(struct msg_builder *b, void *buf, size_t size);
int msg_builder_put_u8(struct msg_builder *b, u8 val);
int msg_builder_put_u16(struct msg_builder *b, u16 val);
int msg_builder_put_u32(struct msg_builder *b, u32 val);
int msg_builder_put_bytes(struct msg_builder *b, const void *data, size_t len);
int msg_builder_put_mac(struct msg_builder *b, const u8 *mac);
size_t msg_builder_get_len(struct msg_builder *b);

/* msg_parser.h */

/**
 * 메시지 파서 컨텍스트
 */
struct msg_parser {
    const u8 *buf;
    size_t size;
    size_t pos;
};

void msg_parser_init(struct msg_parser *p, const void *buf, size_t size);
int msg_parser_get_u8(struct msg_parser *p, u8 *val);
int msg_parser_get_u16(struct msg_parser *p, u16 *val);
int msg_parser_get_u32(struct msg_parser *p, u32 *val);
int msg_parser_get_bytes(struct msg_parser *p, void *data, size_t len);
int msg_parser_get_mac(struct msg_parser *p, u8 *mac);
size_t msg_parser_remaining(struct msg_parser *p);
```

#### 구현 예시 (Endian 처리)

```c
/* msg_builder.c */

int msg_builder_put_u16(struct msg_builder *b, u16 val)
{
    if (b->pos + 2 > b->size)
        return -ENOSPC;
    
    /* Little endian으로 저장 (FW 프로토콜 규칙) */
    b->buf[b->pos++] = val & 0xFF;
    b->buf[b->pos++] = (val >> 8) & 0xFF;
    
    return 0;
}

int msg_builder_put_u32(struct msg_builder *b, u32 val)
{
    if (b->pos + 4 > b->size)
        return -ENOSPC;
    
    b->buf[b->pos++] = val & 0xFF;
    b->buf[b->pos++] = (val >> 8) & 0xFF;
    b->buf[b->pos++] = (val >> 16) & 0xFF;
    b->buf[b->pos++] = (val >> 24) & 0xFF;
    
    return 0;
}

/* msg_parser.c */

int msg_parser_get_u16(struct msg_parser *p, u16 *val)
{
    if (p->pos + 2 > p->size)
        return -ENODATA;
    
    *val = p->buf[p->pos] | (p->buf[p->pos + 1] << 8);
    p->pos += 2;
    
    return 0;
}
```

---

## 5. 디렉토리 구조

```
fw_msg/
├── msg_common.h         # 공통 정의 (헤더, 카테고리)
├── msg_router.c         # 메시지 라우팅
├── msg_router.h
│
├── msg_mlme.c           # MLME 카테고리
├── msg_mlme.h
├── msg_mlme_ids.h       # MLME 메시지 ID
│
├── msg_ma.c             # MA 카테고리
├── msg_ma.h
├── msg_ma_ids.h
│
├── msg_debug.c          # DEBUG 카테고리
├── msg_debug.h
├── msg_debug_ids.h
│
├── msg_wlanlite.c       # WLANLITE 카테고리
├── msg_wlanlite.h
├── msg_wlanlite_ids.h
│
├── msg_system.c         # SYSTEM 카테고리
├── msg_system.h
├── msg_system_ids.h
│
├── msg_builder.c        # 직렬화
├── msg_builder.h
├── msg_parser.c         # 역직렬화
└── msg_parser.h
```

---

## 6. 주목할 특성

### 6.1 메시지 흐름

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Driver                                                                     │
│                                                                             │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐          │
│  │   Core    │───▶│ msg_mlme  │───▶│msg_router │───▶│    HIP    │──▶ FW    │
│  │ (Layer 4) │    │ msg_ma    │    │           │    │ (Layer 6) │          │
│  └───────────┘    └───────────┘    └───────────┘    └───────────┘          │
│        ▲                                 │                                  │
│        │                                 │                                  │
│        │         ┌───────────┐           │          ┌───────────┐          │
│        └─────────│ msg_mlme  │◀──────────┘◀─────────│    HIP    │◀── FW    │
│                  │ msg_ma    │                      │ (Layer 6) │          │
│                  └───────────┘                      └───────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 동기 vs 비동기 처리

| 처리 방식 | 사용 시나리오 | 예시 |
|-----------|---------------|------|
| **동기** | CFM을 기다려야 하는 경우 | FW 버전 조회, 시스템 초기화 |
| **비동기** | CFM 없이 진행 가능 | 스캔 요청, 연결 요청 |

```c
/* 동기 처리 예시 */
int msg_debug_get_fw_version(struct debug_fw_version_cfm_body *version)
{
    struct msg_header hdr = {
        .msg_id = DEBUG_GET_FW_VERSION_REQ,
        .msg_len = 0,
    };
    
    /* CFM 올 때까지 블록 */
    return msg_router_send_sync(MSG_CAT_DEBUG, &hdr, NULL,
                                version, sizeof(*version), 1000);
}

/* 비동기 처리 예시 */
int mlme_req_scan(struct vif_context *vif, struct mlme_scan_req *req)
{
    struct msg_header hdr = {
        .msg_id = MLME_SCAN_REQ,
        .msg_len = sizeof(struct mlme_scan_req_body),
        .vif_id = vif->vif_id,
    };
    
    /* 바로 리턴, 결과는 MLME_SCAN_DONE_IND로 */
    return msg_router_send(MSG_CAT_MLME, &hdr, &body);
}
```

### 6.3 버전 호환성

```c
/* FW 메시지 버전 관리 */
#define MSG_VERSION_MAJOR  1
#define MSG_VERSION_MINOR  0

/* 버전 협상 (시스템 초기화 시) */
int msg_system_negotiate_version(u8 *fw_major, u8 *fw_minor)
{
    struct system_version_req req = {
        .driver_major = MSG_VERSION_MAJOR,
        .driver_minor = MSG_VERSION_MINOR,
    };
    struct system_version_cfm cfm;
    int ret;
    
    ret = msg_router_send_sync(MSG_CAT_SYSTEM, ...);
    if (ret < 0)
        return ret;
    
    *fw_major = cfm.fw_major;
    *fw_minor = cfm.fw_minor;
    
    /* 버전 호환성 검사 */
    if (cfm.fw_major != MSG_VERSION_MAJOR) {
        OSAL_ERR("FW version mismatch: driver=%d.%d, fw=%d.%d\n",
                 MSG_VERSION_MAJOR, MSG_VERSION_MINOR,
                 cfm.fw_major, cfm.fw_minor);
        return -EPROTONOSUPPORT;
    }
    
    return 0;
}
```

### 6.4 의존성 규칙

```
Core (Layer 4)
     │
     │ mlme_req_xxx(), ma_tx_submit()
     ▼
┌─────────────────────────────────────────────────────┐
│  FW Message Layer (Layer 5)                         │
│                                                     │
│  msg_mlme, msg_ma, msg_debug ◀────▶ msg_router     │
│                                          │          │
│  msg_builder, msg_parser (유틸리티)       │          │
└─────────────────────────────────────────────────────┘
                                           │
                                           │ hip_send(), hip_recv callback
                                           ▼
                                    HIP (Layer 6)

규칙:
- Core → msg_* → msg_router → HIP 단방향
- HIP → msg_router → msg_* → Core (callback)
- msg_builder/parser는 stateless 유틸리티
```

### 6.5 에러 처리

```c
/* 전송 실패 */
ret = msg_router_send(MSG_CAT_MLME, &hdr, &body);
if (ret < 0) {
    OSAL_ERR("Failed to send MLME message: %d\n", ret);
    return ret;
}

/* 타임아웃 */
ret = msg_router_send_sync(..., 1000);
if (ret == -ETIMEDOUT) {
    OSAL_ERR("FW not responding\n");
    /* 복구 시도 또는 에러 리포트 */
    return ret;
}

/* FW 에러 응답 */
if (cfm.status != 0) {
    OSAL_ERR("FW returned error: %d\n", cfm.status);
    return -EIO;
}
```

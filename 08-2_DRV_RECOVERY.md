# Layer 0: Driver Lifecycle - Recovery

## 1. 개요

### 1.1 역할

FW 장애(hang, crash, watchdog) 발생 시 드라이버를 자동으로 복구한다. 사용자 세션(VIF, 연결 상태)을 최대한 보존하면서 FW를 재시작한다.

### 1.2 설계 목표

- 자동 복구: 사용자 개입 없이 FW 장애 복구
- 세션 보존: VIF, 연결 정보 유지 (Silent Recovery)
- 복구 실패 대응: 단계적 복구 전략
- 최소 다운타임: 빠른 복구로 서비스 영향 최소화

### 1.3 Recovery 유형

| 유형 | 설명 | VIF 상태 | 연결 상태 |
|------|------|----------|-----------|
| **Silent Recovery** | FW만 재시작, 상위 레이어 상태 유지 | 유지 | 재연결 시도 |
| **Soft Recovery** | FW + Service 레이어 재시작 | 유지 | 재연결 시도 |
| **Full Recovery** | 전체 드라이버 재초기화 | 삭제 후 재생성 | 새로 연결 |

---

## 2. Recovery 트리거

### 2.1 장애 감지 소스

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Recovery Trigger Sources                                                       │
│                                                                                 │
│  ┌──────────────────┐                                                          │
│  │   HIP Layer      │                                                          │
│  │                  │                                                          │
│  │  • FW Watchdog   │─────┐                                                    │
│  │  • Link Down     │     │                                                    │
│  │  • DMA Error     │     │                                                    │
│  └──────────────────┘     │                                                    │
│                           │                                                    │
│  ┌──────────────────┐     │      ┌──────────────────┐                         │
│  │  FW Message      │     │      │                  │                         │
│  │                  │     ├─────▶│  drv_recovery    │                         │
│  │  • Error IND     │─────┤      │                  │                         │
│  │  • Timeout       │     │      │  • 장애 분류     │                         │
│  │  • Invalid Resp  │     │      │  • 복구 전략    │                         │
│  └──────────────────┘     │      │  • 복구 실행    │                         │
│                           │      │                  │                         │
│  ┌──────────────────┐     │      └──────────────────┘                         │
│  │  Service Layer   │     │                                                    │
│  │                  │     │                                                    │
│  │  • Beacon Loss   │─────┘                                                    │
│  │  • Repeated Fail │                                                          │
│  │  • State Mismatch│                                                          │
│  └──────────────────┘                                                          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Recovery 원인 정의

```c
/* drv_recovery.h */

/**
 * Recovery 원인
 */
enum recovery_reason {
    RECOVERY_REASON_NONE = 0,
    
    /* HIP 레벨 */
    RECOVERY_REASON_FW_WATCHDOG,         /* FW watchdog 만료 */
    RECOVERY_REASON_FW_CRASH,            /* FW crash 시그널 */
    RECOVERY_REASON_LINK_DOWN,           /* PCIe/MIF 링크 다운 */
    RECOVERY_REASON_DMA_ERROR,           /* DMA 에러 */
    RECOVERY_REASON_BUS_ERROR,           /* 버스 에러 */
    
    /* FW Message 레벨 */
    RECOVERY_REASON_FW_ERROR_IND,        /* FW 에러 indication */
    RECOVERY_REASON_MSG_TIMEOUT,         /* 메시지 응답 타임아웃 */
    RECOVERY_REASON_INVALID_RESPONSE,    /* 잘못된 FW 응답 */
    RECOVERY_REASON_PROTOCOL_ERROR,      /* 프로토콜 에러 */
    
    /* Service 레벨 */
    RECOVERY_REASON_BEACON_LOSS,         /* 반복적 Beacon 손실 */
    RECOVERY_REASON_REPEATED_FAILURE,    /* 반복 실패 */
    RECOVERY_REASON_STATE_MISMATCH,      /* 상태 불일치 */
    
    /* 수동 트리거 */
    RECOVERY_REASON_USER_REQUEST,        /* 사용자/디버그 요청 */
    RECOVERY_REASON_STRESS_TEST,         /* 스트레스 테스트 */
    
    RECOVERY_REASON_MAX,
};

/**
 * Recovery 원인별 심각도
 */
enum recovery_severity {
    RECOVERY_SEVERITY_LOW,               /* Silent recovery 시도 */
    RECOVERY_SEVERITY_MEDIUM,            /* Soft recovery 시도 */
    RECOVERY_SEVERITY_HIGH,              /* Full recovery 필요 */
    RECOVERY_SEVERITY_CRITICAL,          /* 복구 불가, 에러 상태 */
};
```

### 2.3 장애 감지 및 보고

```c
/* hip_core.c - HIP에서 장애 감지 */

/**
 * FW Watchdog 인터럽트 핸들러
 */
static void handle_fw_watchdog(void)
{
    OSAL_ERR("FW Watchdog triggered!\n");
    
    /* 덤프 트리거 (가능하면) */
    trigger_fw_dump_if_possible();
    
    /* Recovery 요청 */
    lifecycle_trigger_recovery(RECOVERY_REASON_FW_WATCHDOG);
}

/**
 * 링크 다운 감지
 */
static void handle_link_down(void)
{
    OSAL_ERR("Bus link down detected!\n");
    
    lifecycle_trigger_recovery(RECOVERY_REASON_LINK_DOWN);
}

/* msg_router.c - 메시지 타임아웃 */

/**
 * 메시지 응답 타임아웃
 */
static void handle_msg_timeout(u16 msg_id, u8 seq_num)
{
    static int consecutive_timeouts = 0;
    
    consecutive_timeouts++;
    
    OSAL_ERR("Message timeout: id=0x%04x, seq=%d, consecutive=%d\n",
             msg_id, seq_num, consecutive_timeouts);
    
    /* 연속 3회 타임아웃 시 Recovery */
    if (consecutive_timeouts >= 3) {
        consecutive_timeouts = 0;
        lifecycle_trigger_recovery(RECOVERY_REASON_MSG_TIMEOUT);
    }
}

/* svc_sta.c - 서비스에서 장애 감지 */

/**
 * 반복적 Beacon 손실
 */
static void handle_repeated_beacon_loss(struct vif_context *vif)
{
    static int beacon_loss_count = 0;
    
    beacon_loss_count++;
    
    if (beacon_loss_count >= 5) {
        OSAL_ERR("Repeated beacon loss detected\n");
        beacon_loss_count = 0;
        lifecycle_trigger_recovery(RECOVERY_REASON_BEACON_LOSS);
    }
}
```

---

## 3. Recovery 상태 머신

### 3.1 Recovery 단계

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Recovery State Machine                                                         │
│                                                                                 │
│                        장애 감지                                                 │
│                           │                                                     │
│                           ▼                                                     │
│                  ┌─────────────────┐                                           │
│                  │  RECOVERY_INIT  │                                           │
│                  │  (복구 시작)     │                                           │
│                  └────────┬────────┘                                           │
│                           │                                                     │
│                           ▼                                                     │
│                  ┌─────────────────┐                                           │
│                  │ RECOVERY_FREEZE │                                           │
│                  │  (요청 차단)     │                                           │
│                  └────────┬────────┘                                           │
│                           │                                                     │
│                           ▼                                                     │
│                  ┌─────────────────┐                                           │
│                  │  RECOVERY_SAVE  │                                           │
│                  │  (상태 저장)     │                                           │
│                  └────────┬────────┘                                           │
│                           │                                                     │
│                           ▼                                                     │
│                  ┌─────────────────┐                                           │
│                  │ RECOVERY_RESET  │                                           │
│                  │  (FW 리셋)      │                                           │
│                  └────────┬────────┘                                           │
│                           │                                                     │
│                           ▼                                                     │
│                  ┌─────────────────┐                                           │
│                  │ RECOVERY_RELOAD │                                           │
│                  │  (FW 재로드)    │                                           │
│                  └────────┬────────┘                                           │
│                           │                                                     │
│                           ▼                                                     │
│                  ┌─────────────────┐                                           │
│                  │RECOVERY_RESTORE │                                           │
│                  │  (상태 복원)     │                                           │
│                  └────────┬────────┘                                           │
│                           │                                                     │
│              ┌────────────┴────────────┐                                       │
│              ▼                         ▼                                       │
│     ┌─────────────────┐       ┌─────────────────┐                             │
│     │RECOVERY_COMPLETE│       │ RECOVERY_FAILED │                             │
│     │  (복구 성공)     │       │  (복구 실패)     │                             │
│     └────────┬────────┘       └────────┬────────┘                             │
│              │                         │                                       │
│              ▼                         ▼                                       │
│         State → RUNNING           재시도 또는 ERROR                            │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Recovery 상태 정의

```c
/* drv_recovery.h */

/**
 * Recovery 단계
 */
enum recovery_phase {
    RECOVERY_PHASE_NONE = 0,
    RECOVERY_PHASE_INIT,             /* 복구 시작 */
    RECOVERY_PHASE_FREEZE,           /* 새 요청 차단 */
    RECOVERY_PHASE_SAVE,             /* 상태 저장 */
    RECOVERY_PHASE_PRE_RECOVERY,     /* 레이어별 pre_recovery */
    RECOVERY_PHASE_RESET,            /* FW 리셋 */
    RECOVERY_PHASE_RELOAD,           /* FW 재로드 */
    RECOVERY_PHASE_POST_RECOVERY,    /* 레이어별 post_recovery */
    RECOVERY_PHASE_RESTORE,          /* 상태 복원 */
    RECOVERY_PHASE_COMPLETE,         /* 복구 완료 */
    RECOVERY_PHASE_FAILED,           /* 복구 실패 */
};

/**
 * Recovery 컨텍스트
 */
struct recovery_context {
    /* 현재 상태 */
    enum recovery_phase phase;
    enum recovery_reason reason;
    enum recovery_severity severity;
    
    /* 저장된 상태 */
    struct saved_state {
        int vif_count;
        struct vif_saved_state vifs[MAX_VIF_COUNT];
    } saved;
    
    /* 재시도 */
    int retry_count;
    int max_retries;
    
    /* 타이밍 */
    u64 start_time;
    u64 end_time;
    
    /* 통계 */
    struct recovery_stats {
        u32 total_recoveries;
        u32 silent_recoveries;
        u32 soft_recoveries;
        u32 full_recoveries;
        u32 failed_recoveries;
        u64 total_downtime_ms;
        u64 last_recovery_time;
    } stats;
    
    /* 동기화 */
    osal_mutex_t lock;
    osal_completion_t done;
    struct work_struct work;
};
```

---

## 4. 상태 저장 및 복원

### 4.1 저장할 상태

```c
/* drv_recovery.h */

/**
 * VIF별 저장 상태
 */
struct vif_saved_state {
    bool valid;
    
    /* VIF 기본 정보 */
    int vif_id;
    enum vif_type type;
    char name[IFNAMSIZ];
    u8 mac_addr[ETH_ALEN];
    
    /* STA 모드 상태 */
    struct {
        bool was_connected;
        u8 bssid[ETH_ALEN];
        u8 ssid[IEEE80211_MAX_SSID_LEN];
        size_t ssid_len;
        int channel;
        enum nl80211_band band;
        
        /* 보안 정보 */
        u32 wpa_versions;
        u32 cipher_pairwise;
        u32 cipher_group;
        u32 akm_suite;
        u8 pmk[32];
        bool has_pmk;
        
        /* 추가 파라미터 */
        bool power_save;
    } sta;
    
    /* AP 모드 상태 */
    struct {
        bool was_started;
        u8 ssid[IEEE80211_MAX_SSID_LEN];
        size_t ssid_len;
        int channel;
        enum nl80211_chan_width width;
        u16 beacon_interval;
        u8 dtim_period;
        bool hidden_ssid;
        
        /* 연결된 STA 목록 */
        int sta_count;
        u8 sta_macs[MAX_AP_STATIONS][ETH_ALEN];
    } ap;
};

/**
 * 글로벌 저장 상태
 */
struct global_saved_state {
    /* 국가 코드 */
    char country_code[3];
    
    /* Power 설정 */
    bool power_save_enabled;
    
    /* 기타 설정 */
    u8 log_level;
};
```

### 4.2 상태 저장 구현

```c
/* drv_recovery.c */

/**
 * 현재 상태 저장
 */
static int save_current_state(struct recovery_context *ctx)
{
    struct vif_context *vif;
    int i;
    
    OSAL_INFO("Saving current state...\n");
    
    memset(&ctx->saved, 0, sizeof(ctx->saved));
    
    /* 각 VIF 상태 저장 */
    for (i = 0; i < MAX_VIF_COUNT; i++) {
        vif = vif_manager_get_vif(i);
        if (!vif)
            continue;
        
        save_vif_state(vif, &ctx->saved.vifs[i]);
        ctx->saved.vif_count++;
    }
    
    /* 글로벌 상태 저장 */
    save_global_state(&ctx->saved.global);
    
    OSAL_INFO("  Saved %d VIFs\n", ctx->saved.vif_count);
    
    return 0;
}

/**
 * VIF 상태 저장
 */
static void save_vif_state(struct vif_context *vif, 
                           struct vif_saved_state *saved)
{
    saved->valid = true;
    saved->vif_id = vif->vif_id;
    saved->type = vif->type;
    strncpy(saved->name, vif->name, IFNAMSIZ);
    memcpy(saved->mac_addr, vif->mac_addr, ETH_ALEN);
    
    switch (vif->type) {
    case VIF_TYPE_STA:
        save_sta_state(vif, &saved->sta);
        break;
        
    case VIF_TYPE_AP:
    case VIF_TYPE_MHS:
        save_ap_state(vif, &saved->ap);
        break;
        
    case VIF_TYPE_P2P_CLIENT:
    case VIF_TYPE_P2P_GO:
        /* P2P는 recovery 후 재협상 필요 */
        break;
        
    default:
        break;
    }
}

/**
 * STA 상태 저장
 */
static void save_sta_state(struct vif_context *vif,
                           struct vif_sta_saved_state *saved)
{
    struct svc_sta_context *sta = vif->service;
    
    if (vif->state == VIF_STATE_CONNECTED) {
        saved->was_connected = true;
        memcpy(saved->bssid, sta->bssid, ETH_ALEN);
        memcpy(saved->ssid, sta->ssid, sta->ssid_len);
        saved->ssid_len = sta->ssid_len;
        saved->channel = sta->channel;
        saved->band = sta->band;
        
        /* 보안 정보 */
        saved->wpa_versions = sta->crypto.wpa_versions;
        saved->cipher_pairwise = sta->crypto.cipher_pairwise;
        saved->cipher_group = sta->crypto.cipher_group;
        saved->akm_suite = sta->crypto.akm_suite;
        
        /* PMK 저장 (있으면) */
        if (sta->has_pmk) {
            memcpy(saved->pmk, sta->pmk, sizeof(saved->pmk));
            saved->has_pmk = true;
        }
        
        saved->power_save = sta->power_save_enabled;
        
        OSAL_INFO("    STA: connected to %pM, SSID=%.*s\n",
                  saved->bssid, (int)saved->ssid_len, saved->ssid);
    }
}

/**
 * AP 상태 저장
 */
static void save_ap_state(struct vif_context *vif,
                          struct vif_ap_saved_state *saved)
{
    struct svc_mhs_context *ap = vif->service;
    int i;
    
    if (vif->state == VIF_STATE_AP_STARTED) {
        saved->was_started = true;
        memcpy(saved->ssid, ap->ssid, ap->ssid_len);
        saved->ssid_len = ap->ssid_len;
        saved->channel = ap->channel;
        saved->width = ap->width;
        saved->beacon_interval = ap->beacon_interval;
        saved->dtim_period = ap->dtim_period;
        saved->hidden_ssid = ap->hidden_ssid;
        
        /* 연결된 STA 목록 */
        saved->sta_count = 0;
        for (i = 0; i < ap->connected_sta_count && i < MAX_AP_STATIONS; i++) {
            memcpy(saved->sta_macs[i], ap->connected_stas[i].mac, ETH_ALEN);
            saved->sta_count++;
        }
        
        OSAL_INFO("    AP: SSID=%.*s, %d STAs connected\n",
                  (int)saved->ssid_len, saved->ssid, saved->sta_count);
    }
}
```

### 4.3 상태 복원 구현

```c
/* drv_recovery.c */

/**
 * 저장된 상태 복원
 */
static int restore_saved_state(struct recovery_context *ctx)
{
    int i, ret;
    int restored = 0;
    
    OSAL_INFO("Restoring saved state...\n");
    
    /* 글로벌 상태 먼저 복원 */
    restore_global_state(&ctx->saved.global);
    
    /* 각 VIF 복원 */
    for (i = 0; i < MAX_VIF_COUNT; i++) {
        struct vif_saved_state *saved = &ctx->saved.vifs[i];
        
        if (!saved->valid)
            continue;
        
        ret = restore_vif_state(saved);
        if (ret < 0) {
            OSAL_WARN("  Failed to restore VIF %d: %d\n", i, ret);
            continue;
        }
        
        restored++;
    }
    
    OSAL_INFO("  Restored %d/%d VIFs\n", restored, ctx->saved.vif_count);
    
    return 0;
}

/**
 * VIF 상태 복원
 */
static int restore_vif_state(struct vif_saved_state *saved)
{
    struct vif_context *vif;
    int ret;
    
    OSAL_INFO("  Restoring VIF %d (%s)...\n", saved->vif_id, saved->name);
    
    /* VIF가 이미 존재하는지 확인 */
    vif = vif_manager_get_vif(saved->vif_id);
    if (!vif) {
        /* VIF 재생성 필요 (Full Recovery의 경우) */
        vif = vif_manager_create_vif_with_id(saved->vif_id, 
                                              saved->type,
                                              saved->name);
        if (!vif) {
            OSAL_ERR("    Failed to recreate VIF\n");
            return -ENOMEM;
        }
    }
    
    /* MAC 주소 복원 */
    memcpy(vif->mac_addr, saved->mac_addr, ETH_ALEN);
    
    /* 타입별 복원 */
    switch (saved->type) {
    case VIF_TYPE_STA:
        ret = restore_sta_state(vif, &saved->sta);
        break;
        
    case VIF_TYPE_AP:
    case VIF_TYPE_MHS:
        ret = restore_ap_state(vif, &saved->ap);
        break;
        
    default:
        ret = 0;
        break;
    }
    
    return ret;
}

/**
 * STA 상태 복원 (재연결)
 */
static int restore_sta_state(struct vif_context *vif,
                             struct vif_sta_saved_state *saved)
{
    struct cfg80211_connect_params params;
    int ret;
    
    if (!saved->was_connected) {
        OSAL_INFO("    STA was not connected, skip\n");
        return 0;
    }
    
    OSAL_INFO("    Reconnecting to %pM...\n", saved->bssid);
    
    /* 연결 파라미터 구성 */
    memset(&params, 0, sizeof(params));
    params.bssid = saved->bssid;
    params.ssid = saved->ssid;
    params.ssid_len = saved->ssid_len;
    params.channel = ieee80211_get_channel(wiphy, 
                        ieee80211_channel_to_frequency(saved->channel, 
                                                       saved->band));
    
    /* 보안 설정 */
    params.crypto.wpa_versions = saved->wpa_versions;
    params.crypto.ciphers_pairwise[0] = saved->cipher_pairwise;
    params.crypto.n_ciphers_pairwise = 1;
    params.crypto.cipher_group = saved->cipher_group;
    params.crypto.akm_suites[0] = saved->akm_suite;
    params.crypto.n_akm_suites = 1;
    
    /* PMK가 있으면 설정 (빠른 재연결) */
    if (saved->has_pmk) {
        vif_manager_set_pmk(vif, saved->pmk, sizeof(saved->pmk));
    }
    
    /* 재연결 시도 (비동기) */
    ret = vif_manager_connect_async(vif, &params);
    if (ret < 0) {
        OSAL_WARN("    Reconnect request failed: %d\n", ret);
        return ret;
    }
    
    OSAL_INFO("    Reconnect initiated\n");
    return 0;
}

/**
 * AP 상태 복원 (AP 재시작)
 */
static int restore_ap_state(struct vif_context *vif,
                            struct vif_ap_saved_state *saved)
{
    struct cfg80211_ap_settings settings;
    int ret;
    
    if (!saved->was_started) {
        OSAL_INFO("    AP was not started, skip\n");
        return 0;
    }
    
    OSAL_INFO("    Restarting AP...\n");
    
    /* AP 설정 구성 */
    memset(&settings, 0, sizeof(settings));
    settings.ssid = saved->ssid;
    settings.ssid_len = saved->ssid_len;
    settings.chandef.chan = ieee80211_get_channel(wiphy,
                               ieee80211_channel_to_frequency(saved->channel,
                                   saved->channel < 14 ? NL80211_BAND_2GHZ : 
                                                          NL80211_BAND_5GHZ));
    settings.chandef.width = saved->width;
    settings.beacon_interval = saved->beacon_interval;
    settings.dtim_period = saved->dtim_period;
    settings.hidden_ssid = saved->hidden_ssid;
    
    /* AP 재시작 */
    ret = vif_manager_start_ap(vif, &settings);
    if (ret < 0) {
        OSAL_WARN("    AP restart failed: %d\n", ret);
        return ret;
    }
    
    OSAL_INFO("    AP restarted, waiting for STAs to reconnect\n");
    
    /* 이전에 연결된 STA들은 자동으로 재연결 시도할 것 */
    
    return 0;
}
```

---

## 5. Recovery 실행

### 5.1 Recovery 메인 로직

```c
/* drv_recovery.c */

static struct recovery_context g_recovery;

/**
 * Recovery 트리거 (외부에서 호출)
 */
int lifecycle_trigger_recovery(enum recovery_reason reason)
{
    osal_mutex_lock(&g_recovery.lock);
    
    /* 이미 Recovery 중이면 스킵 */
    if (g_recovery.phase != RECOVERY_PHASE_NONE) {
        OSAL_WARN("Recovery already in progress\n");
        osal_mutex_unlock(&g_recovery.lock);
        return -EBUSY;
    }
    
    /* Recovery 불가 상태 검사 */
    if (lifecycle_get_state() == DRV_STATE_STOPPING ||
        lifecycle_get_state() == DRV_STATE_UNLOADED) {
        OSAL_WARN("Cannot recover: driver stopping\n");
        osal_mutex_unlock(&g_recovery.lock);
        return -EINVAL;
    }
    
    g_recovery.reason = reason;
    g_recovery.severity = get_severity(reason);
    g_recovery.retry_count = 0;
    g_recovery.start_time = osal_get_time_ms();
    
    OSAL_INFO("========================================\n");
    OSAL_INFO("Recovery triggered: reason=%s\n", reason_to_str(reason));
    OSAL_INFO("========================================\n");
    
    osal_mutex_unlock(&g_recovery.lock);
    
    /* 워크큐에서 Recovery 실행 */
    schedule_work(&g_recovery.work);
    
    return 0;
}

/**
 * Recovery 워커
 */
static void recovery_work_handler(struct work_struct *work)
{
    int ret;
    
    ret = execute_recovery(&g_recovery);
    
    if (ret < 0 && g_recovery.retry_count < g_recovery.max_retries) {
        /* 재시도 */
        g_recovery.retry_count++;
        OSAL_WARN("Recovery failed, retry %d/%d\n",
                  g_recovery.retry_count, g_recovery.max_retries);
        
        /* 잠시 대기 후 재시도 */
        msleep(1000);
        schedule_work(&g_recovery.work);
        return;
    }
    
    if (ret < 0) {
        /* 최종 실패 */
        OSAL_ERR("Recovery failed after %d retries\n", g_recovery.retry_count);
        g_recovery.phase = RECOVERY_PHASE_FAILED;
        g_recovery.stats.failed_recoveries++;
        lifecycle_set_state(DRV_STATE_ERROR);
    } else {
        /* 성공 */
        g_recovery.phase = RECOVERY_PHASE_COMPLETE;
        g_recovery.stats.total_recoveries++;
    }
    
    g_recovery.end_time = osal_get_time_ms();
    g_recovery.stats.total_downtime_ms += 
        (g_recovery.end_time - g_recovery.start_time);
    g_recovery.stats.last_recovery_time = g_recovery.end_time;
    
    /* 완료 알림 */
    complete(&g_recovery.done);
    
    g_recovery.phase = RECOVERY_PHASE_NONE;
}

/**
 * Recovery 실행
 */
static int execute_recovery(struct recovery_context *ctx)
{
    int ret;
    
    /* Phase 1: INIT */
    ctx->phase = RECOVERY_PHASE_INIT;
    lifecycle_set_state(DRV_STATE_RECOVERING);
    
    /* Phase 2: FREEZE - 새 요청 차단 */
    ctx->phase = RECOVERY_PHASE_FREEZE;
    ret = freeze_operations();
    if (ret < 0)
        return ret;
    
    /* Phase 3: SAVE - 상태 저장 */
    ctx->phase = RECOVERY_PHASE_SAVE;
    ret = save_current_state(ctx);
    if (ret < 0)
        goto err_unfreeze;
    
    /* Phase 4: PRE_RECOVERY - 레이어별 정리 */
    ctx->phase = RECOVERY_PHASE_PRE_RECOVERY;
    ret = notify_layers_pre_recovery();
    if (ret < 0)
        goto err_unfreeze;
    
    /* Phase 5: RESET - FW 리셋 */
    ctx->phase = RECOVERY_PHASE_RESET;
    ret = reset_firmware();
    if (ret < 0)
        goto err_unfreeze;
    
    /* Phase 6: RELOAD - FW 재로드 */
    ctx->phase = RECOVERY_PHASE_RELOAD;
    ret = reload_firmware();
    if (ret < 0)
        goto err_unfreeze;
    
    /* Phase 7: POST_RECOVERY - 레이어별 복구 */
    ctx->phase = RECOVERY_PHASE_POST_RECOVERY;
    ret = notify_layers_post_recovery();
    if (ret < 0)
        goto err_unfreeze;
    
    /* Phase 8: RESTORE - 상태 복원 */
    ctx->phase = RECOVERY_PHASE_RESTORE;
    ret = restore_saved_state(ctx);
    if (ret < 0)
        OSAL_WARN("State restore incomplete\n");
    
    /* 성공 */
    lifecycle_set_state(DRV_STATE_RUNNING);
    unfreeze_operations();
    
    OSAL_INFO("========================================\n");
    OSAL_INFO("Recovery complete (took %llu ms)\n",
              osal_get_time_ms() - ctx->start_time);
    OSAL_INFO("========================================\n");
    
    return 0;

err_unfreeze:
    unfreeze_operations();
    return ret;
}
```

### 5.2 레이어별 Recovery 콜백

```c
/* drv_recovery.c */

/**
 * 레이어 pre_recovery 호출
 */
static int notify_layers_pre_recovery(void)
{
    const struct layer_ops *ops;
    int i;
    
    OSAL_INFO("Notifying layers: pre_recovery\n");
    
    /* Stop 순서대로 (SERVICE → FW_MSG → HIP) */
    static const enum layer_id pre_order[] = {
        LAYER_SERVICE,
        LAYER_CORE,
        LAYER_FW_MSG,
        LAYER_HIP,
    };
    
    for (i = 0; i < ARRAY_SIZE(pre_order); i++) {
        ops = layer_registry_get(pre_order[i]);
        if (ops && ops->pre_recovery) {
            OSAL_INFO("  %s pre_recovery...\n", ops->name);
            ops->pre_recovery();
        }
    }
    
    return 0;
}

/**
 * 레이어 post_recovery 호출
 */
static int notify_layers_post_recovery(void)
{
    const struct layer_ops *ops;
    int i, ret;
    
    OSAL_INFO("Notifying layers: post_recovery\n");
    
    /* Start 순서대로 (HIP → FW_MSG → SERVICE) */
    static const enum layer_id post_order[] = {
        LAYER_HIP,
        LAYER_FW_MSG,
        LAYER_CORE,
        LAYER_SERVICE,
    };
    
    for (i = 0; i < ARRAY_SIZE(post_order); i++) {
        ops = layer_registry_get(post_order[i]);
        if (ops && ops->post_recovery) {
            OSAL_INFO("  %s post_recovery...\n", ops->name);
            ret = ops->post_recovery();
            if (ret < 0) {
                OSAL_ERR("  %s post_recovery failed: %d\n", ops->name, ret);
                return ret;
            }
        }
    }
    
    return 0;
}
```

### 5.3 각 레이어의 Recovery 구현 예시

```c
/* vif_manager.c - Service Layer */

static int service_pre_recovery(void)
{
    int i;
    
    /* 모든 VIF의 진행 중인 작업 취소 */
    for (i = 0; i < MAX_VIF_COUNT; i++) {
        struct vif_context *vif = vif_manager_get_vif(i);
        if (!vif)
            continue;
        
        /* 스캔 취소 */
        if (vif->scan_in_progress)
            cancel_scan(vif);
        
        /* 연결 진행 취소 */
        if (vif->connect_in_progress)
            cancel_connect(vif);
        
        /* 상태를 RECOVERY로 마킹 */
        vif->recovery_in_progress = true;
    }
    
    return 0;
}

static int service_post_recovery(void)
{
    int i;
    
    /* VIF 상태 정리 */
    for (i = 0; i < MAX_VIF_COUNT; i++) {
        struct vif_context *vif = vif_manager_get_vif(i);
        if (!vif)
            continue;
        
        vif->recovery_in_progress = false;
        
        /* FW에 VIF 재등록 */
        register_vif_to_fw(vif);
    }
    
    return 0;
}

/* hip_core.c - HIP Layer */

static int hip_pre_recovery(void)
{
    /* TX 큐 정리 */
    flush_tx_queue();
    
    /* 인터럽트 비활성화 */
    disable_interrupts();
    
    return 0;
}

static int hip_post_recovery(void)
{
    int ret;
    
    /* 버스 재초기화 (필요시) */
    ret = reinit_bus();
    if (ret < 0)
        return ret;
    
    /* 인터럽트 재활성화 */
    enable_interrupts();
    
    /* HIP start */
    return hip_start();
}
```

---

## 6. Recovery 유형별 처리

### 6.1 Silent Recovery

```c
/**
 * Silent Recovery: FW만 재시작, VIF 유지
 */
static int execute_silent_recovery(struct recovery_context *ctx)
{
    OSAL_INFO("Executing Silent Recovery\n");
    
    /* VIF 구조는 유지, FW 상태만 리셋 */
    
    /* 1. FW 리셋 */
    hip_fw_reset();
    
    /* 2. FW 재로드 */
    reload_firmware();
    
    /* 3. VIF 재등록 (FW에) */
    reregister_all_vifs_to_fw();
    
    /* 4. 연결 복원 */
    restore_connections();
    
    return 0;
}
```

### 6.2 Soft Recovery

```c
/**
 * Soft Recovery: FW + Service 레이어 재시작
 */
static int execute_soft_recovery(struct recovery_context *ctx)
{
    OSAL_INFO("Executing Soft Recovery\n");
    
    /* 1. Service 레이어 stop */
    service_layer_stop();
    
    /* 2. FW 리셋 & 재로드 */
    hip_fw_reset();
    reload_firmware();
    
    /* 3. Service 레이어 restart */
    service_layer_start();
    
    /* 4. VIF 및 연결 복원 */
    restore_saved_state(ctx);
    
    return 0;
}
```

### 6.3 Full Recovery

```c
/**
 * Full Recovery: 전체 드라이버 재초기화
 */
static int execute_full_recovery(struct recovery_context *ctx)
{
    OSAL_INFO("Executing Full Recovery\n");
    
    /* 1. 전체 stop 시퀀스 */
    stop_all_layers();
    
    /* 2. FW 리셋 */
    hip_fw_reset();
    
    /* 3. 전체 deinit (OSAL 제외) */
    deinit_layers_except_osal();
    
    /* 4. 전체 재초기화 */
    init_layers_except_osal();
    
    /* 5. FW 재로드 */
    reload_firmware();
    
    /* 6. 전체 start */
    start_all_layers();
    
    /* 7. 상태 복원 (VIF 재생성 포함) */
    restore_saved_state(ctx);
    
    return 0;
}
```

### 6.4 Recovery 전략 선택

```c
/**
 * 원인에 따른 Recovery 전략 결정
 */
static enum recovery_type select_recovery_type(enum recovery_reason reason)
{
    switch (reason) {
    /* Silent Recovery로 충분한 경우 */
    case RECOVERY_REASON_MSG_TIMEOUT:
    case RECOVERY_REASON_BEACON_LOSS:
    case RECOVERY_REASON_FW_ERROR_IND:
        return RECOVERY_TYPE_SILENT;
    
    /* Soft Recovery 필요 */
    case RECOVERY_REASON_FW_WATCHDOG:
    case RECOVERY_REASON_PROTOCOL_ERROR:
    case RECOVERY_REASON_STATE_MISMATCH:
        return RECOVERY_TYPE_SOFT;
    
    /* Full Recovery 필요 */
    case RECOVERY_REASON_FW_CRASH:
    case RECOVERY_REASON_LINK_DOWN:
    case RECOVERY_REASON_DMA_ERROR:
    case RECOVERY_REASON_BUS_ERROR:
        return RECOVERY_TYPE_FULL;
    
    default:
        return RECOVERY_TYPE_SOFT;
    }
}
```

---

## 7. 디버그 및 테스트

### 7.1 Recovery 테스트 인터페이스

```c
/* drv_recovery_debug.c */

/**
 * debugfs를 통한 수동 Recovery 트리거
 */
static ssize_t trigger_recovery_write(struct file *file,
                                      const char __user *buf,
                                      size_t count, loff_t *ppos)
{
    char cmd[32];
    enum recovery_reason reason;
    
    if (copy_from_user(cmd, buf, min(count, sizeof(cmd) - 1)))
        return -EFAULT;
    
    if (strncmp(cmd, "silent", 6) == 0)
        reason = RECOVERY_REASON_USER_REQUEST;
    else if (strncmp(cmd, "soft", 4) == 0)
        reason = RECOVERY_REASON_FW_WATCHDOG;
    else if (strncmp(cmd, "full", 4) == 0)
        reason = RECOVERY_REASON_FW_CRASH;
    else
        return -EINVAL;
    
    lifecycle_trigger_recovery(reason);
    
    return count;
}

/**
 * Recovery 통계 출력
 */
static ssize_t recovery_stats_read(struct file *file,
                                   char __user *buf,
                                   size_t count, loff_t *ppos)
{
    char stats_buf[512];
    int len;
    
    len = snprintf(stats_buf, sizeof(stats_buf),
                   "Total recoveries: %u\n"
                   "  Silent: %u\n"
                   "  Soft: %u\n"
                   "  Full: %u\n"
                   "Failed: %u\n"
                   "Total downtime: %llu ms\n"
                   "Last recovery: %llu\n",
                   g_recovery.stats.total_recoveries,
                   g_recovery.stats.silent_recoveries,
                   g_recovery.stats.soft_recoveries,
                   g_recovery.stats.full_recoveries,
                   g_recovery.stats.failed_recoveries,
                   g_recovery.stats.total_downtime_ms,
                   g_recovery.stats.last_recovery_time);
    
    return simple_read_from_buffer(buf, count, ppos, stats_buf, len);
}
```

### 7.2 Recovery 로그 예시

```
[WIFI] ========================================
[WIFI] Recovery triggered: reason=FW_WATCHDOG
[WIFI] ========================================
[WIFI] Phase: FREEZE
[WIFI]   Operations frozen
[WIFI] Phase: SAVE
[WIFI] Saving current state...
[WIFI]   VIF 0 (wlan0): STA connected to aa:bb:cc:dd:ee:ff
[WIFI]   Saved 1 VIFs
[WIFI] Phase: PRE_RECOVERY
[WIFI] Notifying layers: pre_recovery
[WIFI]   SERVICE pre_recovery...
[WIFI]   CORE pre_recovery...
[WIFI]   FW_MSG pre_recovery...
[WIFI]   HIP pre_recovery...
[WIFI] Phase: RESET
[WIFI]   FW reset complete
[WIFI] Phase: RELOAD
[WIFI]   FW downloading...
[WIFI]   FW booting...
[WIFI]   FW ready
[WIFI] Phase: POST_RECOVERY
[WIFI] Notifying layers: post_recovery
[WIFI]   HIP post_recovery...
[WIFI]   FW_MSG post_recovery...
[WIFI]   CORE post_recovery...
[WIFI]   SERVICE post_recovery...
[WIFI] Phase: RESTORE
[WIFI] Restoring saved state...
[WIFI]   Restoring VIF 0 (wlan0)...
[WIFI]     Reconnecting to aa:bb:cc:dd:ee:ff...
[WIFI]     Reconnect initiated
[WIFI]   Restored 1/1 VIFs
[WIFI] ========================================
[WIFI] Recovery complete (took 2341 ms)
[WIFI] ========================================
```

---

## 8. 디렉토리 구조

```
lifecycle/
├── drv_recovery.c           # Recovery 메인 로직
├── drv_recovery.h
├── drv_recovery_state.c     # 상태 저장/복원
└── drv_recovery_debug.c     # 디버그 인터페이스
```

# Layer 0: Driver Lifecycle - Power Management

## 1. 개요

### 1.1 역할

시스템 전력 상태 변경(Suspend/Resume)에 따른 드라이버 전력 관리를 담당한다. 각 레이어의 suspend/resume 순서를 조율하고, WoWLAN(Wake on Wireless LAN) 기능을 관리한다.

### 1.2 설계 목표

- 시스템 suspend/resume 시 안정적인 드라이버 상태 전환
- WoWLAN 지원으로 특정 패턴에서 시스템 깨우기
- 빠른 resume으로 사용자 경험 최적화
- 전력 소비 최소화

### 1.3 전력 상태

| 상태 | 설명 | WiFi 상태 |
|------|------|-----------|
| **RUNNING** | 정상 동작 | 완전 활성 |
| **SUSPENDED** | 시스템 sleep | FW sleep 또는 WoWLAN 모드 |
| **RUNTIME_SUSPENDED** | 런타임 PM | 절전 모드 (연결 유지) |

---

## 2. Suspend/Resume 흐름

### 2.1 전체 시퀀스

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  System Suspend Sequence                                                        │
│                                                                                 │
│  PM Core: pm_suspend()                                                          │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 1: Prepare                                                         │   │
│  │   새 요청 차단, 진행 작업 완료 대기                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 2: Layer Suspend                                                   │   │
│  │   SERVICE → CORE → FW_MSG → HIP 순서로 suspend                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 3: FW Suspend                                                      │   │
│  │   WoWLAN 설정 또는 FW sleep                                               │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 4: Bus Suspend                                                     │   │
│  │   PCIe/MIF 버스 절전 모드                                                 │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│       SUSPENDED                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  System Resume Sequence                                                         │
│                                                                                 │
│  PM Core: pm_resume()                                                           │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 1: Bus Resume                                                      │   │
│  │   PCIe/MIF 버스 활성화                                                    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 2: FW Resume                                                       │   │
│  │   FW wake up, 상태 확인                                                   │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 3: Layer Resume                                                    │   │
│  │   HIP → FW_MSG → CORE → SERVICE 순서로 resume                            │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 4: Restore                                                         │   │
│  │   연결 상태 확인, 필요시 재연결                                            │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│       RUNNING                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 자료 구조

### 3.1 Power 컨텍스트

```c
/* drv_power.h */

/**
 * WoWLAN 트리거 타입
 */
enum wowlan_trigger {
    WOWLAN_TRIG_NONE = 0,
    WOWLAN_TRIG_MAGIC_PKT      = BIT(0),   /* Magic Packet */
    WOWLAN_TRIG_DISCONNECT     = BIT(1),   /* AP 연결 해제 */
    WOWLAN_TRIG_GTK_REKEY_FAIL = BIT(2),   /* GTK rekey 실패 */
    WOWLAN_TRIG_PATTERN_MATCH  = BIT(3),   /* 패턴 매칭 */
    WOWLAN_TRIG_ANY            = BIT(4),   /* 모든 패킷 */
};

/**
 * WoWLAN 패턴
 */
struct wowlan_pattern {
    u8 *pattern;
    u8 *mask;
    int pattern_len;
    int pkt_offset;
};

/**
 * WoWLAN 설정
 */
struct wowlan_config {
    u32 triggers;
    
    /* 패턴 매칭 */
    int n_patterns;
    struct wowlan_pattern patterns[MAX_WOWLAN_PATTERNS];
    
    /* GTK rekey */
    struct {
        u8 kek[NL80211_KEK_LEN];
        u8 kck[NL80211_KCK_LEN];
        u8 replay_ctr[NL80211_REPLAY_CTR_LEN];
    } gtk_rekey;
};

/**
 * Power 컨텍스트
 */
struct power_context {
    /* 현재 상태 */
    bool suspended;
    bool wowlan_enabled;
    
    /* WoWLAN 설정 */
    struct wowlan_config wowlan;
    
    /* Wake 원인 (resume 시) */
    enum wowlan_trigger wake_reason;
    
    /* Suspend 전 상태 저장 */
    struct {
        int connected_vif_count;
        bool power_save_was_enabled;
    } saved;
    
    /* 타이밍 */
    u64 suspend_time;
    u64 resume_time;
    
    /* 통계 */
    struct {
        u32 suspend_count;
        u32 resume_count;
        u32 suspend_failures;
        u32 resume_failures;
        u32 wowlan_wakeups;
        u64 total_suspend_time_ms;
    } stats;
    
    /* 동기화 */
    osal_mutex_t lock;
};
```

### 3.2 PM 콜백 구조

```c
/* drv_power.h */

/**
 * 플랫폼별 PM ops
 */
#ifdef CONFIG_PM_SLEEP
static const struct dev_pm_ops wifi_pm_ops = {
    .suspend = wifi_pm_suspend,
    .resume = wifi_pm_resume,
    .freeze = wifi_pm_suspend,      /* Hibernate */
    .thaw = wifi_pm_resume,
    .poweroff = wifi_pm_suspend,
    .restore = wifi_pm_resume,
};
#endif

#ifdef CONFIG_PM
static const struct dev_pm_ops wifi_runtime_pm_ops = {
    .runtime_suspend = wifi_runtime_suspend,
    .runtime_resume = wifi_runtime_resume,
    .runtime_idle = wifi_runtime_idle,
};
#endif
```

---

## 4. Suspend 구현

### 4.1 Suspend 메인 함수

```c
/* drv_power.c */

static struct power_context g_power;

/**
 * System Suspend 콜백
 */
int wifi_pm_suspend(struct device *dev)
{
    int ret;
    
    OSAL_INFO("========================================\n");
    OSAL_INFO("WiFi Suspend Start\n");
    OSAL_INFO("========================================\n");
    
    osal_mutex_lock(&g_power.lock);
    
    /* 이미 suspended면 스킵 */
    if (g_power.suspended) {
        OSAL_WARN("Already suspended\n");
        osal_mutex_unlock(&g_power.lock);
        return 0;
    }
    
    g_power.suspend_time = osal_get_time_ms();
    
    /* State 변경 */
    lifecycle_set_state(DRV_STATE_SUSPENDING);
    
    /* Phase 1: Prepare */
    ret = suspend_prepare();
    if (ret < 0)
        goto err_out;
    
    /* Phase 2: Layer Suspend */
    ret = suspend_all_layers();
    if (ret < 0)
        goto err_restore;
    
    /* Phase 3: FW Suspend (WoWLAN 설정) */
    ret = suspend_firmware();
    if (ret < 0)
        goto err_resume_layers;
    
    /* Phase 4: Bus Suspend */
    ret = suspend_bus();
    if (ret < 0)
        goto err_resume_fw;
    
    /* 완료 */
    g_power.suspended = true;
    lifecycle_set_state(DRV_STATE_SUSPENDED);
    g_power.stats.suspend_count++;
    
    osal_mutex_unlock(&g_power.lock);
    
    OSAL_INFO("========================================\n");
    OSAL_INFO("WiFi Suspend Complete\n");
    OSAL_INFO("========================================\n");
    
    return 0;

err_resume_fw:
    resume_firmware();
err_resume_layers:
    resume_all_layers();
err_restore:
    lifecycle_set_state(DRV_STATE_RUNNING);
err_out:
    g_power.stats.suspend_failures++;
    osal_mutex_unlock(&g_power.lock);
    OSAL_ERR("Suspend failed: %d\n", ret);
    return ret;
}
```

### 4.2 Phase 1: Prepare

```c
/* drv_power.c */

/**
 * Suspend 준비
 */
static int suspend_prepare(void)
{
    int ret;
    
    OSAL_INFO("Phase 1: Preparing suspend\n");
    
    /* 새 요청 차단 */
    block_new_requests();
    
    /* 진행 중인 스캔 취소 */
    cancel_all_pending_scans();
    
    /* 진행 중인 작업 완료 대기 */
    ret = wait_for_pending_operations(SUSPEND_TIMEOUT_MS);
    if (ret < 0) {
        OSAL_WARN("  Timeout waiting for pending ops\n");
        /* 강제 진행 */
    }
    
    /* 현재 상태 저장 */
    save_power_state();
    
    OSAL_INFO("  Prepare done\n");
    return 0;
}

/**
 * 상태 저장
 */
static void save_power_state(void)
{
    int i;
    
    g_power.saved.connected_vif_count = 0;
    
    for (i = 0; i < MAX_VIF_COUNT; i++) {
        struct vif_context *vif = vif_manager_get_vif(i);
        if (vif && vif->state == VIF_STATE_CONNECTED)
            g_power.saved.connected_vif_count++;
    }
    
    g_power.saved.power_save_was_enabled = 
        power_mgmt_is_enabled();
    
    OSAL_INFO("  Saved: %d connected VIFs\n", 
              g_power.saved.connected_vif_count);
}
```

### 4.3 Phase 2: Layer Suspend

```c
/* drv_power.c */

/**
 * Suspend 순서 (상위 → 하위)
 */
static const enum layer_id suspend_order[] = {
    LAYER_CUSTOMER,      /* 1. 고객 확장 먼저 */
    LAYER_SERVICE,       /* 2. 서비스 */
    LAYER_CORE,          /* 3. 코어 */
    LAYER_FW_MSG,        /* 4. FW 메시지 */
    LAYER_HIP,           /* 5. HIP (마지막) */
};

#define SUSPEND_ORDER_COUNT  ARRAY_SIZE(suspend_order)

/**
 * 모든 레이어 Suspend
 */
static int suspend_all_layers(void)
{
    const struct layer_ops *ops;
    int i, ret;
    
    OSAL_INFO("Phase 2: Suspending layers\n");
    
    for (i = 0; i < SUSPEND_ORDER_COUNT; i++) {
        enum layer_id id = suspend_order[i];
        
        ops = layer_registry_get(id);
        if (!ops || !ops->suspend)
            continue;
        
        OSAL_INFO("  Suspending %s...\n", ops->name);
        
        ret = ops->suspend();
        if (ret < 0) {
            OSAL_ERR("  %s suspend failed: %d\n", ops->name, ret);
            
            /* 이미 suspend된 레이어들 resume */
            resume_layers_up_to(i);
            return ret;
        }
        
        OSAL_INFO("  %s suspended\n", ops->name);
    }
    
    OSAL_INFO("  All layers suspended\n");
    return 0;
}

/**
 * 부분 resume (에러 복구용)
 */
static void resume_layers_up_to(int index)
{
    const struct layer_ops *ops;
    int i;
    
    /* 역순으로 resume */
    for (i = index - 1; i >= 0; i--) {
        enum layer_id id = suspend_order[i];
        
        ops = layer_registry_get(id);
        if (ops && ops->resume) {
            OSAL_INFO("  Resuming %s (rollback)...\n", ops->name);
            ops->resume();
        }
    }
}
```

### 4.4 Phase 3: FW Suspend

```c
/* drv_power.c */

/**
 * FW Suspend (WoWLAN 설정)
 */
static int suspend_firmware(void)
{
    int ret;
    
    OSAL_INFO("Phase 3: Suspending firmware\n");
    
    if (g_power.wowlan_enabled && g_power.saved.connected_vif_count > 0) {
        /* WoWLAN 모드로 진입 */
        OSAL_INFO("  Configuring WoWLAN...\n");
        ret = configure_wowlan();
        if (ret < 0) {
            OSAL_WARN("  WoWLAN config failed, using deep sleep\n");
            goto deep_sleep;
        }
        
        ret = msg_system_enter_wowlan(&g_power.wowlan);
        if (ret < 0) {
            OSAL_WARN("  WoWLAN enter failed: %d\n", ret);
            goto deep_sleep;
        }
        
        OSAL_INFO("  WoWLAN enabled, triggers=0x%x\n", 
                  g_power.wowlan.triggers);
    } else {
deep_sleep:
        /* Deep sleep 모드 */
        OSAL_INFO("  Entering deep sleep...\n");
        ret = msg_system_enter_sleep();
        if (ret < 0) {
            OSAL_ERR("  Deep sleep failed: %d\n", ret);
            return ret;
        }
    }
    
    OSAL_INFO("  FW suspended\n");
    return 0;
}

/**
 * WoWLAN 설정 구성
 */
static int configure_wowlan(void)
{
    struct vif_context *vif;
    int i;
    
    /* 연결된 VIF의 GTK rekey 정보 수집 */
    for (i = 0; i < MAX_VIF_COUNT; i++) {
        vif = vif_manager_get_vif(i);
        if (!vif || vif->state != VIF_STATE_CONNECTED)
            continue;
        
        if (vif->type == VIF_TYPE_STA) {
            /* GTK rekey 정보 복사 */
            if (vif_has_gtk_rekey_info(vif)) {
                get_gtk_rekey_info(vif, &g_power.wowlan.gtk_rekey);
                g_power.wowlan.triggers |= WOWLAN_TRIG_GTK_REKEY_FAIL;
            }
            
            /* 기본적으로 disconnect 시 깨우기 */
            g_power.wowlan.triggers |= WOWLAN_TRIG_DISCONNECT;
            break;
        }
    }
    
    return 0;
}
```

### 4.5 Phase 4: Bus Suspend

```c
/* drv_power.c */

/**
 * 버스 Suspend
 */
static int suspend_bus(void)
{
    OSAL_INFO("Phase 4: Suspending bus\n");
    
    /* HIP 레이어를 통해 버스 suspend */
    return hip_suspend();
}
```

### 4.6 각 레이어의 Suspend 구현 예시

```c
/* vif_manager.c - Service Layer */

int service_layer_suspend(void)
{
    int i;
    
    /* 모든 VIF에 suspend 알림 */
    for (i = 0; i < MAX_VIF_COUNT; i++) {
        struct vif_context *vif = vif_manager_get_vif(i);
        if (!vif)
            continue;
        
        /* 워크큐 플러시 */
        flush_vif_workqueue(vif);
        
        /* netdev 큐 정지 */
        if (vif->ndev)
            netif_device_detach(vif->ndev);
        
        /* 타이머 정지 */
        stop_vif_timers(vif);
    }
    
    return 0;
}

/* hip_mif.c - MIF 버스 */

int hip_mif_suspend(void *bus_priv)
{
    struct hip_mif_context *mif = bus_priv;
    
    /* 인터럽트 비활성화 */
    writel(0, mif->reg_base + MIF_REG_INT_ENABLE);
    
    /* MIF 절전 모드 */
    /* 플랫폼별 구현 */
    
    return 0;
}

/* hip_pcie.c - PCIe 버스 */

int hip_pcie_suspend(void *bus_priv)
{
    struct hip_pcie_context *pcie = bus_priv;
    
    /* NAPI 비활성화 */
    napi_disable(&pcie->napi);
    
    /* 인터럽트 비활성화 */
    writel(0, pcie->bar0 + PCIE_REG_INT_ENABLE);
    synchronize_irq(pci_irq_vector(pcie->pdev, 0));
    
    /* PCIe D3 상태로 전환 */
    pci_save_state(pcie->pdev);
    pci_disable_device(pcie->pdev);
    pci_set_power_state(pcie->pdev, PCI_D3hot);
    
    return 0;
}
```

---

## 5. Resume 구현

### 5.1 Resume 메인 함수

```c
/* drv_power.c */

/**
 * System Resume 콜백
 */
int wifi_pm_resume(struct device *dev)
{
    int ret;
    
    OSAL_INFO("========================================\n");
    OSAL_INFO("WiFi Resume Start\n");
    OSAL_INFO("========================================\n");
    
    osal_mutex_lock(&g_power.lock);
    
    /* suspended 상태가 아니면 스킵 */
    if (!g_power.suspended) {
        OSAL_WARN("Not suspended\n");
        osal_mutex_unlock(&g_power.lock);
        return 0;
    }
    
    g_power.resume_time = osal_get_time_ms();
    
    /* State 변경 */
    lifecycle_set_state(DRV_STATE_RESUMING);
    
    /* Phase 1: Bus Resume */
    ret = resume_bus();
    if (ret < 0)
        goto err_out;
    
    /* Phase 2: FW Resume */
    ret = resume_firmware();
    if (ret < 0)
        goto err_suspend_bus;
    
    /* Phase 3: Layer Resume */
    ret = resume_all_layers();
    if (ret < 0)
        goto err_suspend_fw;
    
    /* Phase 4: Restore */
    ret = resume_restore();
    if (ret < 0)
        OSAL_WARN("Restore incomplete\n");
    
    /* 완료 */
    g_power.suspended = false;
    lifecycle_set_state(DRV_STATE_RUNNING);
    g_power.stats.resume_count++;
    
    /* Suspend 시간 통계 */
    g_power.stats.total_suspend_time_ms += 
        (g_power.resume_time - g_power.suspend_time);
    
    osal_mutex_unlock(&g_power.lock);
    
    /* 요청 차단 해제 */
    unblock_new_requests();
    
    OSAL_INFO("========================================\n");
    OSAL_INFO("WiFi Resume Complete (suspended %llu ms)\n",
              g_power.resume_time - g_power.suspend_time);
    OSAL_INFO("========================================\n");
    
    return 0;

err_suspend_fw:
    suspend_firmware();
err_suspend_bus:
    suspend_bus();
err_out:
    g_power.stats.resume_failures++;
    lifecycle_set_state(DRV_STATE_ERROR);
    osal_mutex_unlock(&g_power.lock);
    OSAL_ERR("Resume failed: %d\n", ret);
    return ret;
}
```

### 5.2 Phase 1: Bus Resume

```c
/* drv_power.c */

/**
 * 버스 Resume
 */
static int resume_bus(void)
{
    int ret;
    
    OSAL_INFO("Phase 1: Resuming bus\n");
    
    ret = hip_resume();
    if (ret < 0) {
        OSAL_ERR("  Bus resume failed: %d\n", ret);
        return ret;
    }
    
    OSAL_INFO("  Bus resumed\n");
    return 0;
}
```

### 5.3 Phase 2: FW Resume

```c
/* drv_power.c */

/**
 * FW Resume
 */
static int resume_firmware(void)
{
    int ret;
    
    OSAL_INFO("Phase 2: Resuming firmware\n");
    
    /* FW가 깨어있는지 확인 */
    if (!hip_is_ready()) {
        OSAL_WARN("  FW not ready, waiting...\n");
        ret = wait_for_fw_ready(FW_RESUME_TIMEOUT_MS);
        if (ret < 0) {
            OSAL_ERR("  FW not responding\n");
            
            /* Recovery 트리거 */
            lifecycle_trigger_recovery(RECOVERY_REASON_FW_CRASH);
            return ret;
        }
    }
    
    /* Wake 원인 확인 */
    check_wake_reason();
    
    /* WoWLAN 해제 */
    if (g_power.wowlan_enabled) {
        ret = msg_system_exit_wowlan();
        if (ret < 0)
            OSAL_WARN("  WoWLAN exit failed: %d\n", ret);
    }
    
    OSAL_INFO("  FW resumed\n");
    return 0;
}

/**
 * Wake 원인 확인
 */
static void check_wake_reason(void)
{
    struct msg_wowlan_wakeup_ind ind;
    
    if (msg_system_get_wakeup_reason(&ind) == 0) {
        g_power.wake_reason = ind.trigger;
        
        if (ind.trigger != WOWLAN_TRIG_NONE) {
            g_power.stats.wowlan_wakeups++;
            OSAL_INFO("  Wake reason: %s\n", 
                      wowlan_trigger_to_str(ind.trigger));
            
            /* User space에 알림 */
            cfg80211_report_wowlan_wakeup(ind.trigger);
        }
    }
}
```

### 5.4 Phase 3: Layer Resume

```c
/* drv_power.c */

/**
 * Resume 순서 (하위 → 상위, suspend의 역순)
 */
static const enum layer_id resume_order[] = {
    LAYER_HIP,           /* 1. HIP 먼저 */
    LAYER_FW_MSG,        /* 2. FW 메시지 */
    LAYER_CORE,          /* 3. 코어 */
    LAYER_SERVICE,       /* 4. 서비스 */
    LAYER_CUSTOMER,      /* 5. 고객 확장 (마지막) */
};

#define RESUME_ORDER_COUNT  ARRAY_SIZE(resume_order)

/**
 * 모든 레이어 Resume
 */
static int resume_all_layers(void)
{
    const struct layer_ops *ops;
    int i, ret;
    
    OSAL_INFO("Phase 3: Resuming layers\n");
    
    for (i = 0; i < RESUME_ORDER_COUNT; i++) {
        enum layer_id id = resume_order[i];
        
        ops = layer_registry_get(id);
        if (!ops || !ops->resume)
            continue;
        
        OSAL_INFO("  Resuming %s...\n", ops->name);
        
        ret = ops->resume();
        if (ret < 0) {
            OSAL_ERR("  %s resume failed: %d\n", ops->name, ret);
            return ret;
        }
        
        OSAL_INFO("  %s resumed\n", ops->name);
    }
    
    OSAL_INFO("  All layers resumed\n");
    return 0;
}
```

### 5.5 Phase 4: Restore

```c
/* drv_power.c */

/**
 * 상태 복원
 */
static int resume_restore(void)
{
    int i;
    
    OSAL_INFO("Phase 4: Restoring state\n");
    
    /* 연결 상태 확인 */
    for (i = 0; i < MAX_VIF_COUNT; i++) {
        struct vif_context *vif = vif_manager_get_vif(i);
        if (!vif)
            continue;
        
        if (vif->type == VIF_TYPE_STA && 
            vif->state == VIF_STATE_CONNECTED) {
            
            /* 연결 상태 검증 */
            verify_sta_connection(vif);
        }
    }
    
    OSAL_INFO("  Restore done\n");
    return 0;
}

/**
 * STA 연결 상태 검증
 */
static void verify_sta_connection(struct vif_context *vif)
{
    int rssi;
    
    /* Beacon 수신 확인 */
    rssi = vif_manager_get_rssi(vif);
    
    if (rssi == 0 || rssi < -100) {
        OSAL_WARN("  VIF %d: connection may be lost\n", vif->vif_id);
        
        /* 짧은 스캔으로 AP 존재 확인 */
        trigger_connection_check(vif);
    } else {
        OSAL_INFO("  VIF %d: still connected, RSSI=%d\n", 
                  vif->vif_id, rssi);
    }
}
```

### 5.6 각 레이어의 Resume 구현 예시

```c
/* vif_manager.c - Service Layer */

int service_layer_resume(void)
{
    int i;
    
    /* 모든 VIF resume */
    for (i = 0; i < MAX_VIF_COUNT; i++) {
        struct vif_context *vif = vif_manager_get_vif(i);
        if (!vif)
            continue;
        
        /* netdev 활성화 */
        if (vif->ndev)
            netif_device_attach(vif->ndev);
        
        /* 타이머 재시작 */
        restart_vif_timers(vif);
    }
    
    return 0;
}

/* hip_pcie.c - PCIe 버스 */

int hip_pcie_resume(void *bus_priv)
{
    struct hip_pcie_context *pcie = bus_priv;
    int ret;
    
    /* PCIe D0 상태로 전환 */
    pci_set_power_state(pcie->pdev, PCI_D0);
    ret = pci_enable_device(pcie->pdev);
    if (ret < 0)
        return ret;
    
    pci_restore_state(pcie->pdev);
    pci_set_master(pcie->pdev);
    
    /* 인터럽트 재활성화 */
    writel(0xFFFFFFFF, pcie->bar0 + PCIE_REG_INT_ENABLE);
    
    /* NAPI 활성화 */
    napi_enable(&pcie->napi);
    
    return 0;
}
```

---

## 6. WoWLAN (Wake on WLAN)

### 6.1 WoWLAN 설정 인터페이스

```c
/* drv_power.h */

/**
 * WoWLAN 설정 (cfg80211에서 호출)
 */
int drv_power_set_wowlan(struct wiphy *wiphy,
                         struct cfg80211_wowlan *wowlan);

/**
 * WoWLAN 지원 Capability
 */
void drv_power_set_wowlan_caps(struct wiphy *wiphy);
```

### 6.2 WoWLAN Capability 설정

```c
/* drv_power.c */

/**
 * WoWLAN 지원 기능 설정
 */
void drv_power_set_wowlan_caps(struct wiphy *wiphy)
{
    static struct wiphy_wowlan_support wowlan_support = {
        .flags = WIPHY_WOWLAN_MAGIC_PKT |
                 WIPHY_WOWLAN_DISCONNECT |
                 WIPHY_WOWLAN_GTK_REKEY_FAILURE |
                 WIPHY_WOWLAN_SUPPORTS_GTK_REKEY,
        .n_patterns = MAX_WOWLAN_PATTERNS,
        .pattern_max_len = MAX_WOWLAN_PATTERN_LEN,
        .pattern_min_len = 1,
    };
    
    wiphy->wowlan = &wowlan_support;
}
```

### 6.3 WoWLAN 설정 처리

```c
/* drv_power.c */

/**
 * cfg80211 WoWLAN 설정 처리
 */
int drv_power_set_wowlan(struct wiphy *wiphy,
                         struct cfg80211_wowlan *wowlan)
{
    int i;
    
    osal_mutex_lock(&g_power.lock);
    
    if (!wowlan) {
        /* WoWLAN 비활성화 */
        g_power.wowlan_enabled = false;
        g_power.wowlan.triggers = WOWLAN_TRIG_NONE;
        g_power.wowlan.n_patterns = 0;
        osal_mutex_unlock(&g_power.lock);
        return 0;
    }
    
    /* 트리거 변환 */
    g_power.wowlan.triggers = 0;
    
    if (wowlan->magic_pkt)
        g_power.wowlan.triggers |= WOWLAN_TRIG_MAGIC_PKT;
    
    if (wowlan->disconnect)
        g_power.wowlan.triggers |= WOWLAN_TRIG_DISCONNECT;
    
    if (wowlan->gtk_rekey_failure)
        g_power.wowlan.triggers |= WOWLAN_TRIG_GTK_REKEY_FAIL;
    
    if (wowlan->any)
        g_power.wowlan.triggers |= WOWLAN_TRIG_ANY;
    
    /* 패턴 복사 */
    g_power.wowlan.n_patterns = 0;
    for (i = 0; i < wowlan->n_patterns && i < MAX_WOWLAN_PATTERNS; i++) {
        struct wowlan_pattern *dst = &g_power.wowlan.patterns[i];
        struct cfg80211_pkt_pattern *src = &wowlan->patterns[i];
        
        dst->pattern = osal_mem_alloc(src->pattern_len);
        dst->mask = osal_mem_alloc(DIV_ROUND_UP(src->pattern_len, 8));
        
        if (!dst->pattern || !dst->mask) {
            /* 정리 후 에러 */
            free_wowlan_patterns();
            osal_mutex_unlock(&g_power.lock);
            return -ENOMEM;
        }
        
        memcpy(dst->pattern, src->pattern, src->pattern_len);
        memcpy(dst->mask, src->mask, DIV_ROUND_UP(src->pattern_len, 8));
        dst->pattern_len = src->pattern_len;
        dst->pkt_offset = src->pkt_offset;
        
        g_power.wowlan.n_patterns++;
    }
    
    g_power.wowlan_enabled = true;
    
    OSAL_INFO("WoWLAN configured: triggers=0x%x, patterns=%d\n",
              g_power.wowlan.triggers, g_power.wowlan.n_patterns);
    
    osal_mutex_unlock(&g_power.lock);
    return 0;
}
```

---

## 7. Runtime PM

### 7.1 Runtime PM 개요

```c
/* drv_power.c */

/**
 * Runtime PM: 유휴 상태에서 자동 절전
 * - 트래픽 없을 때 저전력 모드
 * - 연결은 유지 (PS 모드)
 */

/**
 * Runtime Suspend
 */
int wifi_runtime_suspend(struct device *dev)
{
    OSAL_DBG("Runtime suspend\n");
    
    /* 연결 유지하면서 저전력 모드 */
    enable_deep_power_save();
    
    return 0;
}

/**
 * Runtime Resume
 */
int wifi_runtime_resume(struct device *dev)
{
    OSAL_DBG("Runtime resume\n");
    
    /* 일반 Power Save로 복귀 */
    disable_deep_power_save();
    
    return 0;
}

/**
 * Runtime Idle - autosuspend 가능 여부
 */
int wifi_runtime_idle(struct device *dev)
{
    /* 진행 중인 작업이 있으면 idle 거부 */
    if (has_pending_operations())
        return -EBUSY;
    
    return 0;
}
```

### 7.2 Runtime PM 활성화

```c
/* drv_init.c */

/**
 * Runtime PM 초기화
 */
static void init_runtime_pm(struct device *dev)
{
#ifdef CONFIG_PM
    pm_runtime_set_autosuspend_delay(dev, RUNTIME_PM_DELAY_MS);
    pm_runtime_use_autosuspend(dev);
    pm_runtime_enable(dev);
#endif
}

/**
 * Runtime PM 정리
 */
static void cleanup_runtime_pm(struct device *dev)
{
#ifdef CONFIG_PM
    pm_runtime_disable(dev);
#endif
}
```

---

## 8. 디버그 인터페이스

### 8.1 Power 상태 조회

```c
/* drv_power_debug.c */

/**
 * Power 상태 debugfs
 */
static ssize_t power_status_read(struct file *file,
                                 char __user *buf,
                                 size_t count, loff_t *ppos)
{
    char status_buf[512];
    int len;
    
    len = snprintf(status_buf, sizeof(status_buf),
                   "Suspended: %s\n"
                   "WoWLAN enabled: %s\n"
                   "WoWLAN triggers: 0x%x\n"
                   "Last wake reason: %s\n"
                   "\n"
                   "Statistics:\n"
                   "  Suspend count: %u\n"
                   "  Resume count: %u\n"
                   "  Suspend failures: %u\n"
                   "  Resume failures: %u\n"
                   "  WoWLAN wakeups: %u\n"
                   "  Total suspend time: %llu ms\n",
                   g_power.suspended ? "yes" : "no",
                   g_power.wowlan_enabled ? "yes" : "no",
                   g_power.wowlan.triggers,
                   wowlan_trigger_to_str(g_power.wake_reason),
                   g_power.stats.suspend_count,
                   g_power.stats.resume_count,
                   g_power.stats.suspend_failures,
                   g_power.stats.resume_failures,
                   g_power.stats.wowlan_wakeups,
                   g_power.stats.total_suspend_time_ms);
    
    return simple_read_from_buffer(buf, count, ppos, status_buf, len);
}

/**
 * 수동 Suspend/Resume 테스트
 */
static ssize_t power_test_write(struct file *file,
                                const char __user *buf,
                                size_t count, loff_t *ppos)
{
    char cmd[16];
    
    if (copy_from_user(cmd, buf, min(count, sizeof(cmd) - 1)))
        return -EFAULT;
    
    if (strncmp(cmd, "suspend", 7) == 0)
        wifi_pm_suspend(NULL);
    else if (strncmp(cmd, "resume", 6) == 0)
        wifi_pm_resume(NULL);
    
    return count;
}
```

### 8.2 로그 예시

```
[WIFI] ========================================
[WIFI] WiFi Suspend Start
[WIFI] ========================================
[WIFI] Phase 1: Preparing suspend
[WIFI]   Saved: 1 connected VIFs
[WIFI]   Prepare done
[WIFI] Phase 2: Suspending layers
[WIFI]   Suspending CUSTOMER...
[WIFI]   CUSTOMER suspended
[WIFI]   Suspending SERVICE...
[WIFI]   SERVICE suspended
[WIFI]   Suspending CORE...
[WIFI]   CORE suspended
[WIFI]   Suspending FW_MSG...
[WIFI]   FW_MSG suspended
[WIFI]   Suspending HIP...
[WIFI]   HIP suspended
[WIFI]   All layers suspended
[WIFI] Phase 3: Suspending firmware
[WIFI]   Configuring WoWLAN...
[WIFI]   WoWLAN enabled, triggers=0x3
[WIFI]   FW suspended
[WIFI] Phase 4: Suspending bus
[WIFI] ========================================
[WIFI] WiFi Suspend Complete
[WIFI] ========================================

... (시스템 sleep) ...

[WIFI] ========================================
[WIFI] WiFi Resume Start
[WIFI] ========================================
[WIFI] Phase 1: Resuming bus
[WIFI]   Bus resumed
[WIFI] Phase 2: Resuming firmware
[WIFI]   Wake reason: DISCONNECT
[WIFI]   FW resumed
[WIFI] Phase 3: Resuming layers
[WIFI]   Resuming HIP...
[WIFI]   HIP resumed
[WIFI]   Resuming FW_MSG...
[WIFI]   FW_MSG resumed
[WIFI]   Resuming CORE...
[WIFI]   CORE resumed
[WIFI]   Resuming SERVICE...
[WIFI]   SERVICE resumed
[WIFI]   Resuming CUSTOMER...
[WIFI]   CUSTOMER resumed
[WIFI]   All layers resumed
[WIFI] Phase 4: Restoring state
[WIFI]   VIF 0: connection may be lost
[WIFI]   Restore done
[WIFI] ========================================
[WIFI] WiFi Resume Complete (suspended 28532 ms)
[WIFI] ========================================
```

---

## 9. 디렉토리 구조

```
lifecycle/
├── drv_power.c              # Suspend/Resume 메인 로직
├── drv_power.h
├── drv_power_wowlan.c       # WoWLAN 관련
└── drv_power_debug.c        # 디버그 인터페이스
```

---

## 10. 주목할 특성

### 10.1 Suspend/Resume 순서 요약

```
Suspend 순서 (상위 → 하위):
  CUSTOMER → SERVICE → CORE → FW_MSG → HIP → FW → BUS

Resume 순서 (하위 → 상위):
  BUS → FW → HIP → FW_MSG → CORE → SERVICE → CUSTOMER
```

### 10.2 에러 복구 전략

| 단계 | 실패 시 대응 |
|------|-------------|
| Layer Suspend | 이미 suspend된 레이어 resume, 실패 반환 |
| FW Suspend | Deep sleep 시도, 그래도 실패 시 반환 |
| Bus Resume | Recovery 트리거 |
| FW Resume | Recovery 트리거 |
| Layer Resume | 실패한 레이어 로깅, 계속 진행 |

### 10.3 WoWLAN vs Deep Sleep

| 항목 | WoWLAN | Deep Sleep |
|------|--------|------------|
| 연결 유지 | 유지 | 끊김 |
| Wake 가능 | 특정 이벤트 | 불가 |
| 전력 소비 | 낮음~중간 | 매우 낮음 |
| Resume 속도 | 빠름 | 느림 (재연결 필요) |

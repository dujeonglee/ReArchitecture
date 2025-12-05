# Layer 0: Driver Lifecycle - Init/Deinit

## 1. 개요

### 1.1 역할

드라이버 초기화(probe)와 종료(remove) 시퀀스를 관리한다. 레이어별 의존성을 고려한 순차적 초기화/종료를 보장한다.

### 1.2 설계 목표

- 레이어 의존성 기반 순차 초기화
- 실패 시 자동 rollback
- 부분 초기화 상태 방지
- 종료 시 리소스 누수 방지

---

## 2. 초기화 시퀀스

### 2.1 전체 흐름

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Driver Probe Sequence                                                          │
│                                                                                 │
│  platform_driver_probe()                                                        │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 1: Layer Registration                                              │   │
│  │   각 레이어가 layer_registry에 ops 등록                                    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 2: Layer Init                                                      │   │
│  │   OSAL → HIP → FW_MSG → KERNEL_IF → SERVICE → CORE → CUSTOMER           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 3: FW Load                                                         │   │
│  │   FW 다운로드 → FW 부팅 → FW Ready 대기                                    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 4: Layer Start                                                     │   │
│  │   HIP start → FW_MSG start → SERVICE start                               │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 5: Ready                                                           │   │
│  │   State → RUNNING, wiphy 등록                                             │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Phase 1: Layer Registration

```c
/* drv_init.c */

/**
 * 모든 레이어의 ops를 registry에 등록
 * 이 단계에서는 실제 초기화 없음, 등록만 수행
 */
static int register_all_layers(void)
{
    int ret;
    
    OSAL_INFO("Phase 1: Registering layers\n");
    
    /* 각 레이어 모듈에서 구현한 register 함수 호출 */
    ret = osal_layer_register();
    if (ret < 0) {
        OSAL_ERR("Failed to register OSAL layer: %d\n", ret);
        return ret;
    }
    
    ret = hip_layer_register();
    if (ret < 0)
        goto err_osal;
    
    ret = fw_msg_layer_register();
    if (ret < 0)
        goto err_hip;
    
    ret = kernel_if_layer_register();
    if (ret < 0)
        goto err_fw_msg;
    
    ret = service_layer_register();
    if (ret < 0)
        goto err_kernel_if;
    
    ret = core_layer_register();
    if (ret < 0)
        goto err_service;
    
    ret = customer_layer_register();
    if (ret < 0)
        goto err_core;
    
    OSAL_INFO("All layers registered\n");
    return 0;

err_core:
    layer_registry_unregister(LAYER_CORE);
err_service:
    layer_registry_unregister(LAYER_SERVICE);
err_kernel_if:
    layer_registry_unregister(LAYER_KERNEL_IF);
err_fw_msg:
    layer_registry_unregister(LAYER_FW_MSG);
err_hip:
    layer_registry_unregister(LAYER_HIP);
err_osal:
    layer_registry_unregister(LAYER_OSAL);
    return ret;
}
```

### 2.3 Phase 2: Layer Init

```c
/* drv_init.c */

/**
 * 초기화 순서 (의존성 기반)
 */
static const enum layer_id init_order[] = {
    LAYER_OSAL,          /* 1. 가장 먼저 - 다른 모든 레이어가 의존 */
    LAYER_HIP,           /* 2. 버스 인터페이스 */
    LAYER_FW_MSG,        /* 3. FW 메시지 (HIP 의존) */
    LAYER_KERNEL_IF,     /* 4. 커널 인터페이스 */
    LAYER_SERVICE,       /* 5. 서비스 매니저 */
    LAYER_CORE,          /* 6. 코어 프로토콜 */
    LAYER_CUSTOMER,      /* 7. 고객 확장 (마지막) */
};

#define INIT_ORDER_COUNT  ARRAY_SIZE(init_order)

/**
 * 초기화된 레이어 추적 (rollback용)
 */
struct init_tracker {
    enum layer_id layers[LAYER_MAX];
    int count;
};

static struct init_tracker g_init_tracker;

/**
 * 레이어 순차 초기화
 */
static int init_all_layers(void *platform_data)
{
    const struct layer_ops *ops;
    int i, ret;
    
    OSAL_INFO("Phase 2: Initializing layers\n");
    
    memset(&g_init_tracker, 0, sizeof(g_init_tracker));
    
    for (i = 0; i < INIT_ORDER_COUNT; i++) {
        enum layer_id id = init_order[i];
        
        ops = layer_registry_get(id);
        if (!ops) {
            OSAL_ERR("Layer %d not registered\n", id);
            ret = -ENOENT;
            goto err_rollback;
        }
        
        OSAL_INFO("  Initializing %s...\n", ops->name);
        
        if (ops->init) {
            ret = ops->init(platform_data);
            if (ret < 0) {
                OSAL_ERR("  %s init failed: %d\n", ops->name, ret);
                goto err_rollback;
            }
        }
        
        /* 성공한 레이어 기록 */
        g_init_tracker.layers[g_init_tracker.count++] = id;
        
        OSAL_INFO("  %s initialized\n", ops->name);
    }
    
    OSAL_INFO("All layers initialized\n");
    return 0;

err_rollback:
    /* 이미 초기화된 레이어들 역순으로 deinit */
    OSAL_WARN("Rolling back initialization...\n");
    rollback_init();
    return ret;
}

/**
 * 초기화 롤백
 */
static void rollback_init(void)
{
    const struct layer_ops *ops;
    int i;
    
    /* 역순으로 deinit */
    for (i = g_init_tracker.count - 1; i >= 0; i--) {
        enum layer_id id = g_init_tracker.layers[i];
        
        ops = layer_registry_get(id);
        if (ops && ops->deinit) {
            OSAL_INFO("  Rolling back %s...\n", ops->name);
            ops->deinit();
        }
    }
    
    g_init_tracker.count = 0;
}
```

### 2.4 Phase 3: FW Load

```c
/* drv_init.c */

/**
 * FW 로드 시퀀스
 */
static int load_firmware(void *platform_data)
{
    const struct firmware *fw;
    int ret;
    
    OSAL_INFO("Phase 3: Loading firmware\n");
    
    /* State 변경 */
    lifecycle_set_state(DRV_STATE_FW_LOADING);
    
    /* FW 파일 요청 */
    ret = request_firmware(&fw, get_fw_filename(platform_data), 
                           get_device(platform_data));
    if (ret < 0) {
        OSAL_ERR("Failed to load firmware file: %d\n", ret);
        return ret;
    }
    
    OSAL_INFO("  FW loaded: %s, size=%zu\n", 
              get_fw_filename(platform_data), fw->size);
    
    /* FW 다운로드 */
    ret = hip_fw_download(fw->data, fw->size);
    if (ret < 0) {
        OSAL_ERR("FW download failed: %d\n", ret);
        release_firmware(fw);
        return ret;
    }
    
    release_firmware(fw);
    
    /* State 변경 */
    lifecycle_set_state(DRV_STATE_FW_BOOTING);
    
    /* FW 부팅 */
    ret = hip_fw_boot();
    if (ret < 0) {
        OSAL_ERR("FW boot failed: %d\n", ret);
        return ret;
    }
    
    /* FW Ready 대기 */
    ret = wait_for_fw_ready(FW_BOOT_TIMEOUT_MS);
    if (ret < 0) {
        OSAL_ERR("FW ready timeout\n");
        return ret;
    }
    
    /* FW 버전 확인 */
    log_fw_version();
    
    OSAL_INFO("Firmware ready\n");
    return 0;
}

/**
 * FW Ready 대기
 */
static int wait_for_fw_ready(unsigned long timeout_ms)
{
    unsigned long deadline = jiffies + msecs_to_jiffies(timeout_ms);
    
    while (!hip_is_ready()) {
        if (time_after(jiffies, deadline))
            return -ETIMEDOUT;
        
        msleep(100);
    }
    
    return 0;
}

/**
 * FW 버전 로깅
 */
static void log_fw_version(void)
{
    struct debug_fw_version_cfm_body version;
    
    if (msg_debug_get_fw_version(&version) == 0) {
        OSAL_INFO("FW Version: %u.%u.%u.%u\n",
                  version.major, version.minor, 
                  version.patch, version.build);
        OSAL_INFO("FW Build: %s (%s)\n", 
                  version.build_date, version.build_hash);
    }
}
```

### 2.5 Phase 4: Layer Start

```c
/* drv_init.c */

/**
 * Start 순서 (FW 로드 후 활성화가 필요한 레이어만)
 */
static const enum layer_id start_order[] = {
    LAYER_HIP,           /* 1. 버스 통신 시작 */
    LAYER_FW_MSG,        /* 2. 메시지 핸들러 시작 */
    LAYER_SERVICE,       /* 3. 서비스 매니저 시작 */
};

#define START_ORDER_COUNT  ARRAY_SIZE(start_order)

/**
 * Start된 레이어 추적 (rollback용)
 */
static struct init_tracker g_start_tracker;

/**
 * 레이어 Start
 */
static int start_all_layers(void)
{
    const struct layer_ops *ops;
    int i, ret;
    
    OSAL_INFO("Phase 4: Starting layers\n");
    
    memset(&g_start_tracker, 0, sizeof(g_start_tracker));
    
    for (i = 0; i < START_ORDER_COUNT; i++) {
        enum layer_id id = start_order[i];
        
        ops = layer_registry_get(id);
        if (!ops)
            continue;
        
        if (ops->start) {
            OSAL_INFO("  Starting %s...\n", ops->name);
            
            ret = ops->start();
            if (ret < 0) {
                OSAL_ERR("  %s start failed: %d\n", ops->name, ret);
                goto err_rollback;
            }
            
            g_start_tracker.layers[g_start_tracker.count++] = id;
            
            OSAL_INFO("  %s started\n", ops->name);
        }
    }
    
    OSAL_INFO("All layers started\n");
    return 0;

err_rollback:
    OSAL_WARN("Rolling back start sequence...\n");
    rollback_start();
    return ret;
}

/**
 * Start 롤백
 */
static void rollback_start(void)
{
    const struct layer_ops *ops;
    int i;
    
    for (i = g_start_tracker.count - 1; i >= 0; i--) {
        enum layer_id id = g_start_tracker.layers[i];
        
        ops = layer_registry_get(id);
        if (ops && ops->stop) {
            OSAL_INFO("  Stopping %s...\n", ops->name);
            ops->stop();
        }
    }
    
    g_start_tracker.count = 0;
}
```

### 2.6 Phase 5: Ready

```c
/* drv_init.c */

/**
 * 최종 활성화
 */
static int finalize_init(void)
{
    int ret;
    
    OSAL_INFO("Phase 5: Finalizing\n");
    
    /* wiphy 등록 */
    ret = cfg80211_if_register_wiphy();
    if (ret < 0) {
        OSAL_ERR("Failed to register wiphy: %d\n", ret);
        return ret;
    }
    
    /* State → RUNNING */
    lifecycle_set_state(DRV_STATE_RUNNING);
    
    OSAL_INFO("Driver ready\n");
    return 0;
}
```

### 2.7 전체 Probe 함수

```c
/* drv_init.c */

/**
 * 드라이버 초기화 메인 함수
 */
int drv_init_probe(void *platform_data)
{
    int ret;
    
    OSAL_INFO("========================================\n");
    OSAL_INFO("WiFi Driver Probe Start\n");
    OSAL_INFO("========================================\n");
    
    /* State → PROBING */
    lifecycle_set_state(DRV_STATE_PROBING);
    
    /* Phase 1: Layer 등록 */
    ret = register_all_layers();
    if (ret < 0)
        goto err_out;
    
    /* Phase 2: Layer 초기화 */
    ret = init_all_layers(platform_data);
    if (ret < 0)
        goto err_unregister;
    
    /* State → INITIALIZED */
    lifecycle_set_state(DRV_STATE_INITIALIZED);
    
    /* Phase 3: FW 로드 */
    ret = load_firmware(platform_data);
    if (ret < 0)
        goto err_deinit;
    
    /* Phase 4: Layer Start */
    ret = start_all_layers();
    if (ret < 0)
        goto err_fw_unload;
    
    /* Phase 5: 최종 활성화 */
    ret = finalize_init();
    if (ret < 0)
        goto err_stop;
    
    OSAL_INFO("========================================\n");
    OSAL_INFO("WiFi Driver Probe Complete\n");
    OSAL_INFO("========================================\n");
    
    return 0;

err_stop:
    rollback_start();
err_fw_unload:
    hip_fw_reset();
err_deinit:
    rollback_init();
err_unregister:
    unregister_all_layers();
err_out:
    lifecycle_set_state(DRV_STATE_ERROR);
    OSAL_ERR("Driver probe failed: %d\n", ret);
    return ret;
}
```

---

## 3. 종료 시퀀스

### 3.1 전체 흐름

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Driver Remove Sequence                                                         │
│                                                                                 │
│  platform_driver_remove()                                                       │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 1: Prepare                                                         │   │
│  │   State → STOPPING, 새 요청 차단                                          │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 2: VIF Cleanup                                                     │   │
│  │   모든 VIF 삭제, 연결 해제                                                 │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 3: Layer Stop                                                      │   │
│  │   SERVICE stop → FW_MSG stop → HIP stop                                  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 4: FW Shutdown                                                     │   │
│  │   FW에 shutdown 알림, FW reset                                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 5: Layer Deinit                                                    │   │
│  │   CUSTOMER → CORE → SERVICE → KERNEL_IF → FW_MSG → HIP → OSAL           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 6: Unregister                                                      │   │
│  │   wiphy 해제, layer registry 정리                                        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Phase 1: Prepare

```c
/* drv_deinit.c */

/**
 * 종료 준비
 */
static int prepare_shutdown(void)
{
    OSAL_INFO("Phase 1: Preparing shutdown\n");
    
    /* State → STOPPING */
    lifecycle_set_state(DRV_STATE_STOPPING);
    
    /* 새로운 요청 차단 */
    block_new_requests();
    
    /* 진행 중인 작업 완료 대기 */
    wait_for_pending_operations(SHUTDOWN_TIMEOUT_MS);
    
    return 0;
}

/**
 * 새 요청 차단
 */
static void block_new_requests(void)
{
    /* cfg80211 ops에서 STOPPING 상태 검사 */
    /* 새 scan, connect 등 거부됨 */
    OSAL_INFO("  New requests blocked\n");
}

/**
 * 진행 중인 작업 대기
 */
static int wait_for_pending_operations(unsigned long timeout_ms)
{
    unsigned long deadline = jiffies + msecs_to_jiffies(timeout_ms);
    
    /* 진행 중인 스캔 대기 */
    while (vif_manager_has_pending_scan()) {
        if (time_after(jiffies, deadline)) {
            OSAL_WARN("  Timeout waiting for pending scan\n");
            break;
        }
        msleep(100);
    }
    
    /* 진행 중인 연결 대기 */
    while (vif_manager_has_pending_connect()) {
        if (time_after(jiffies, deadline)) {
            OSAL_WARN("  Timeout waiting for pending connect\n");
            break;
        }
        msleep(100);
    }
    
    OSAL_INFO("  Pending operations completed\n");
    return 0;
}
```

### 3.3 Phase 2: VIF Cleanup

```c
/* drv_deinit.c */

/**
 * 모든 VIF 정리
 */
static void cleanup_all_vifs(void)
{
    struct vif_context *vif;
    int i;
    
    OSAL_INFO("Phase 2: Cleaning up VIFs\n");
    
    /* 모든 VIF 순회 */
    for (i = 0; i < MAX_VIF_COUNT; i++) {
        vif = vif_manager_get_vif(i);
        if (!vif)
            continue;
        
        OSAL_INFO("  Cleaning VIF %d (%s)\n", i, vif->name);
        
        /* 연결 해제 */
        if (vif->state == VIF_STATE_CONNECTED) {
            OSAL_INFO("    Disconnecting...\n");
            vif_manager_disconnect(vif, WLAN_REASON_DEAUTH_LEAVING);
        }
        
        /* AP 모드면 중지 */
        if (vif->type == VIF_TYPE_AP && vif->state == VIF_STATE_AP_STARTED) {
            OSAL_INFO("    Stopping AP...\n");
            vif_manager_stop_ap(vif);
        }
        
        /* VIF 삭제 */
        OSAL_INFO("    Destroying VIF...\n");
        vif_manager_destroy_vif(vif);
    }
    
    OSAL_INFO("  All VIFs cleaned up\n");
}
```

### 3.4 Phase 3: Layer Stop

```c
/* drv_deinit.c */

/**
 * Stop 순서 (Start의 역순)
 */
static const enum layer_id stop_order[] = {
    LAYER_SERVICE,       /* 1. 서비스 먼저 중지 */
    LAYER_FW_MSG,        /* 2. 메시지 핸들러 중지 */
    LAYER_HIP,           /* 3. 버스 통신 중지 */
};

#define STOP_ORDER_COUNT  ARRAY_SIZE(stop_order)

/**
 * 레이어 Stop
 */
static void stop_all_layers(void)
{
    const struct layer_ops *ops;
    int i;
    
    OSAL_INFO("Phase 3: Stopping layers\n");
    
    for (i = 0; i < STOP_ORDER_COUNT; i++) {
        enum layer_id id = stop_order[i];
        
        ops = layer_registry_get(id);
        if (!ops)
            continue;
        
        if (ops->stop) {
            OSAL_INFO("  Stopping %s...\n", ops->name);
            ops->stop();
            OSAL_INFO("  %s stopped\n", ops->name);
        }
    }
    
    OSAL_INFO("All layers stopped\n");
}
```

### 3.5 Phase 4: FW Shutdown

```c
/* drv_deinit.c */

/**
 * FW 종료
 */
static void shutdown_firmware(void)
{
    OSAL_INFO("Phase 4: Shutting down firmware\n");
    
    /* FW에 shutdown 알림 */
    if (hip_is_ready()) {
        msg_system_send_deinit();
        
        /* FW 응답 대기 (짧게) */
        msleep(100);
    }
    
    /* FW Reset */
    hip_fw_reset();
    
    OSAL_INFO("  Firmware shut down\n");
}
```

### 3.6 Phase 5: Layer Deinit

```c
/* drv_deinit.c */

/**
 * Deinit 순서 (Init의 역순)
 */
static const enum layer_id deinit_order[] = {
    LAYER_CUSTOMER,      /* 7. 고객 확장 먼저 */
    LAYER_CORE,          /* 6. 코어 프로토콜 */
    LAYER_SERVICE,       /* 5. 서비스 매니저 */
    LAYER_KERNEL_IF,     /* 4. 커널 인터페이스 */
    LAYER_FW_MSG,        /* 3. FW 메시지 */
    LAYER_HIP,           /* 2. 버스 인터페이스 */
    LAYER_OSAL,          /* 1. 가장 마지막 */
};

#define DEINIT_ORDER_COUNT  ARRAY_SIZE(deinit_order)

/**
 * 레이어 Deinit
 */
static void deinit_all_layers(void)
{
    const struct layer_ops *ops;
    int i;
    
    OSAL_INFO("Phase 5: Deinitializing layers\n");
    
    for (i = 0; i < DEINIT_ORDER_COUNT; i++) {
        enum layer_id id = deinit_order[i];
        
        ops = layer_registry_get(id);
        if (!ops)
            continue;
        
        if (ops->deinit) {
            OSAL_INFO("  Deinitializing %s...\n", ops->name);
            ops->deinit();
            OSAL_INFO("  %s deinitialized\n", ops->name);
        }
    }
    
    OSAL_INFO("All layers deinitialized\n");
}
```

### 3.7 Phase 6: Unregister

```c
/* drv_deinit.c */

/**
 * 최종 정리
 */
static void finalize_shutdown(void)
{
    OSAL_INFO("Phase 6: Finalizing shutdown\n");
    
    /* wiphy 해제 */
    cfg80211_if_unregister_wiphy();
    
    /* Layer registry 정리 */
    unregister_all_layers();
    
    /* State → UNLOADED */
    lifecycle_set_state(DRV_STATE_UNLOADED);
    
    OSAL_INFO("  Shutdown finalized\n");
}

/**
 * 모든 레이어 등록 해제
 */
static void unregister_all_layers(void)
{
    int i;
    
    for (i = 0; i < DEINIT_ORDER_COUNT; i++) {
        layer_registry_unregister(deinit_order[i]);
    }
}
```

### 3.8 전체 Remove 함수

```c
/* drv_deinit.c */

/**
 * 드라이버 종료 메인 함수
 */
void drv_deinit_remove(void)
{
    OSAL_INFO("========================================\n");
    OSAL_INFO("WiFi Driver Remove Start\n");
    OSAL_INFO("========================================\n");
    
    /* 이미 종료 중이면 스킵 */
    if (lifecycle_get_state() == DRV_STATE_STOPPING ||
        lifecycle_get_state() == DRV_STATE_UNLOADED) {
        OSAL_WARN("Driver already stopping or unloaded\n");
        return;
    }
    
    /* Phase 1: 준비 */
    prepare_shutdown();
    
    /* Phase 2: VIF 정리 */
    cleanup_all_vifs();
    
    /* Phase 3: Layer Stop */
    stop_all_layers();
    
    /* Phase 4: FW 종료 */
    shutdown_firmware();
    
    /* Phase 5: Layer Deinit */
    deinit_all_layers();
    
    /* Phase 6: 최종 정리 */
    finalize_shutdown();
    
    OSAL_INFO("========================================\n");
    OSAL_INFO("WiFi Driver Remove Complete\n");
    OSAL_INFO("========================================\n");
}
```

---

## 4. 에러 처리

### 4.1 초기화 실패 시 Rollback

```c
/**
 * 초기화 중 실패 시나리오
 */

/* Case 1: Layer Init 실패 */
init_all_layers() 실패
    │
    ├──▶ 이미 init된 레이어들 역순 deinit
    ├──▶ State → ERROR
    └──▶ probe 실패 반환

/* Case 2: FW Load 실패 */
load_firmware() 실패
    │
    ├──▶ 모든 레이어 deinit (rollback_init)
    ├──▶ State → ERROR
    └──▶ probe 실패 반환

/* Case 3: Layer Start 실패 */
start_all_layers() 실패
    │
    ├──▶ 이미 start된 레이어들 stop
    ├──▶ FW reset
    ├──▶ 모든 레이어 deinit
    ├──▶ State → ERROR
    └──▶ probe 실패 반환
```

### 4.2 종료 중 에러 처리

```c
/**
 * 종료 시 에러는 무시하고 계속 진행
 * 리소스 누수 방지가 최우선
 */
static void stop_all_layers(void)
{
    const struct layer_ops *ops;
    int i;
    
    for (i = 0; i < STOP_ORDER_COUNT; i++) {
        enum layer_id id = stop_order[i];
        
        ops = layer_registry_get(id);
        if (!ops || !ops->stop)
            continue;
        
        /* 에러가 발생해도 계속 진행 */
        ops->stop();
        
        /* 에러 로깅만 수행 */
    }
}
```

---

## 5. 동기화 고려사항

### 5.1 초기화 중 요청 차단

```c
/**
 * cfg80211 ops에서 상태 검사
 */
static int cfg80211_if_scan(struct wiphy *wiphy,
                            struct cfg80211_scan_request *request)
{
    enum drv_state state = lifecycle_get_state();
    
    /* RUNNING 상태에서만 허용 */
    if (state != DRV_STATE_RUNNING) {
        OSAL_WARN("Scan rejected: driver state=%d\n", state);
        return -EBUSY;
    }
    
    return vif_manager_dispatch_scan(...);
}
```

### 5.2 종료 중 진행 작업 대기

```c
/**
 * Reference counting으로 진행 작업 추적
 */
struct operation_tracker {
    atomic_t pending_count;
    struct completion all_done;
};

static struct operation_tracker g_op_tracker;

/* 작업 시작 시 */
void track_operation_start(void)
{
    atomic_inc(&g_op_tracker.pending_count);
}

/* 작업 완료 시 */
void track_operation_end(void)
{
    if (atomic_dec_and_test(&g_op_tracker.pending_count))
        complete(&g_op_tracker.all_done);
}

/* 종료 시 대기 */
int wait_for_all_operations(unsigned long timeout_ms)
{
    if (atomic_read(&g_op_tracker.pending_count) == 0)
        return 0;
    
    return wait_for_completion_timeout(&g_op_tracker.all_done,
                                        msecs_to_jiffies(timeout_ms));
}
```

---

## 6. 디렉토리 구조

```
lifecycle/
├── drv_init.c               # 초기화 시퀀스
├── drv_init.h
├── drv_deinit.c             # 종료 시퀀스
├── layer_registry.c         # 레이어 등록
└── layer_registry.h
```

---

## 7. 로그 예시

### 7.1 정상 Probe 로그

```
[WIFI] ========================================
[WIFI] WiFi Driver Probe Start
[WIFI] ========================================
[WIFI] Phase 1: Registering layers
[WIFI] All layers registered
[WIFI] Phase 2: Initializing layers
[WIFI]   Initializing OSAL...
[WIFI]   OSAL initialized
[WIFI]   Initializing HIP...
[WIFI]   HIP initialized
[WIFI]   Initializing FW_MSG...
[WIFI]   FW_MSG initialized
[WIFI]   Initializing KERNEL_IF...
[WIFI]   KERNEL_IF initialized
[WIFI]   Initializing SERVICE...
[WIFI]   SERVICE initialized
[WIFI]   Initializing CORE...
[WIFI]   CORE initialized
[WIFI]   Initializing CUSTOMER...
[WIFI]   CUSTOMER initialized
[WIFI] All layers initialized
[WIFI] Phase 3: Loading firmware
[WIFI]   FW loaded: wifi_fw.bin, size=524288
[WIFI]   FW download complete
[WIFI]   FW booting...
[WIFI] FW Version: 1.2.3.456
[WIFI] FW Build: Dec 06 2025 (abc123)
[WIFI] Firmware ready
[WIFI] Phase 4: Starting layers
[WIFI]   Starting HIP...
[WIFI]   HIP started
[WIFI]   Starting FW_MSG...
[WIFI]   FW_MSG started
[WIFI]   Starting SERVICE...
[WIFI]   SERVICE started
[WIFI] All layers started
[WIFI] Phase 5: Finalizing
[WIFI] Driver ready
[WIFI] ========================================
[WIFI] WiFi Driver Probe Complete
[WIFI] ========================================
```

### 7.2 정상 Remove 로그

```
[WIFI] ========================================
[WIFI] WiFi Driver Remove Start
[WIFI] ========================================
[WIFI] Phase 1: Preparing shutdown
[WIFI]   New requests blocked
[WIFI]   Pending operations completed
[WIFI] Phase 2: Cleaning up VIFs
[WIFI]   Cleaning VIF 0 (wlan0)
[WIFI]     Disconnecting...
[WIFI]     Destroying VIF...
[WIFI]   All VIFs cleaned up
[WIFI] Phase 3: Stopping layers
[WIFI]   Stopping SERVICE...
[WIFI]   SERVICE stopped
[WIFI]   Stopping FW_MSG...
[WIFI]   FW_MSG stopped
[WIFI]   Stopping HIP...
[WIFI]   HIP stopped
[WIFI] All layers stopped
[WIFI] Phase 4: Shutting down firmware
[WIFI]   Firmware shut down
[WIFI] Phase 5: Deinitializing layers
[WIFI]   Deinitializing CUSTOMER...
[WIFI]   CUSTOMER deinitialized
[WIFI]   Deinitializing CORE...
[WIFI]   CORE deinitialized
[WIFI]   Deinitializing SERVICE...
[WIFI]   SERVICE deinitialized
[WIFI]   Deinitializing KERNEL_IF...
[WIFI]   KERNEL_IF deinitialized
[WIFI]   Deinitializing FW_MSG...
[WIFI]   FW_MSG deinitialized
[WIFI]   Deinitializing HIP...
[WIFI]   HIP deinitialized
[WIFI]   Deinitializing OSAL...
[WIFI]   OSAL deinitialized
[WIFI] All layers deinitialized
[WIFI] Phase 6: Finalizing shutdown
[WIFI]   Shutdown finalized
[WIFI] ========================================
[WIFI] WiFi Driver Remove Complete
[WIFI] ========================================
```

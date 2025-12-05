# Layer 2: Kernel Interface Layer

## 1. 개요

### 1.1 역할

Linux 커널 서브시스템(cfg80211, netdev, debugfs 등)과의 인터페이스를 제공하고, 커널 버전별 차이를 흡수한다.

### 1.2 설계 목표

- 커널 버전 호환성 격리
- cfg80211/nl80211 인터페이스 추상화
- 디버그 인터페이스 통합

### 1.3 Linux 전용 레이어

이 레이어는 Linux 전용이다. Windows에서는 NDIS/WDF 기반의 별도 인터페이스 레이어가 필요하다.

---

## 2. 모듈 구성

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 2: Kernel Interface Layer                                               │
│                                                                                 │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐                │
│  │ cfg80211_if      │ │ vendor_nl80211   │ │ netdev_if        │                │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘                │
│  ┌──────────────────┐ ┌──────────────────┐                                     │
│  │ debugfs_if       │ │ procfs_if        │                                     │
│  └──────────────────┘ └──────────────────┘                                     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 모듈 상세

### 3.1 cfg80211_if

| 항목 | 내용 |
|------|------|
| **역할** | cfg80211 ops 구현 및 이벤트 전달 |
| **책임** | cfg80211_ops 콜백 핸들링, cfg80211 이벤트 래핑, wiphy 등록/해제 |
| **의존성** | OSAL, cfg80211 (kernel) |

#### 인터페이스

```c
/* cfg80211_if.h */

/**
 * 초기화/해제
 */
int cfg80211_if_init(struct device *dev);
void cfg80211_if_deinit(void);

/**
 * wiphy 관리
 */
struct wiphy *cfg80211_if_get_wiphy(void);
int cfg80211_if_register_wiphy(void);
void cfg80211_if_unregister_wiphy(void);

/**
 * Capability 설정
 */
void cfg80211_if_set_bands(struct wiphy *wiphy, 
                           struct ieee80211_supported_band *band_2g,
                           struct ieee80211_supported_band *band_5g,
                           struct ieee80211_supported_band *band_6g);
void cfg80211_if_set_interface_modes(struct wiphy *wiphy, uint32_t modes);
void cfg80211_if_set_cipher_suites(struct wiphy *wiphy, 
                                    const u32 *suites, int n_suites);

/**
 * 이벤트 래퍼 (커널 버전 호환성 처리)
 */
void cfg80211_if_scan_done(struct cfg80211_scan_request *request, bool aborted);
void cfg80211_if_sched_scan_results(struct wiphy *wiphy, u64 reqid);
void cfg80211_if_connect_result(struct net_device *ndev,
                                 const u8 *bssid,
                                 const u8 *req_ie, size_t req_ie_len,
                                 const u8 *resp_ie, size_t resp_ie_len,
                                 u16 status);
void cfg80211_if_disconnect(struct net_device *ndev, u16 reason, bool from_ap);
void cfg80211_if_roamed(struct net_device *ndev,
                        struct cfg80211_roam_info *info);
```

#### cfg80211_ops 등록 예시

```c
/* cfg80211_if.c */

static int cfg80211_if_scan(struct wiphy *wiphy,
                            struct cfg80211_scan_request *request)
{
    struct vif_context *vif = wdev_to_vif(request->wdev);
    
    /* Service Manager로 위임 */
    return vif_manager_dispatch_scan(vif, request);
}

static int cfg80211_if_connect(struct wiphy *wiphy,
                               struct net_device *ndev,
                               struct cfg80211_connect_params *params)
{
    struct vif_context *vif = ndev_to_vif(ndev);
    
    /* Service Manager로 위임 */
    return vif_manager_dispatch_connect(vif, params);
}

static int cfg80211_if_start_ap(struct wiphy *wiphy,
                                struct net_device *ndev,
                                struct cfg80211_ap_settings *settings)
{
    struct vif_context *vif = ndev_to_vif(ndev);
    return vif_manager_dispatch_start_ap(vif, settings);
}

static const struct cfg80211_ops wifi_cfg80211_ops = {
    /* VIF 관리 */
    .add_virtual_intf = cfg80211_if_add_vif,
    .del_virtual_intf = cfg80211_if_del_vif,
    .change_virtual_intf = cfg80211_if_change_vif,
    
    /* Scan */
    .scan = cfg80211_if_scan,
    .abort_scan = cfg80211_if_abort_scan,
    
    /* Connect/Disconnect */
    .connect = cfg80211_if_connect,
    .disconnect = cfg80211_if_disconnect,
    
    /* AP */
    .start_ap = cfg80211_if_start_ap,
    .stop_ap = cfg80211_if_stop_ap,
    .change_beacon = cfg80211_if_change_beacon,
    
    /* Key 관리 */
    .add_key = cfg80211_if_add_key,
    .del_key = cfg80211_if_del_key,
    .set_default_key = cfg80211_if_set_default_key,
    
    /* P2P */
    .remain_on_channel = cfg80211_if_remain_on_channel,
    .mgmt_tx = cfg80211_if_mgmt_tx,
    
    /* ... */
};
```

#### 커널 버전 호환성 처리

```c
/* cfg80211_if_compat.h */

#include <linux/version.h>

/* 커널 4.8+ scan_done API 변경 */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(4, 8, 0)
static inline void cfg80211_if_scan_done(struct cfg80211_scan_request *request,
                                          bool aborted)
{
    struct cfg80211_scan_info info = { .aborted = aborted };
    cfg80211_scan_done(request, &info);
}
#else
static inline void cfg80211_if_scan_done(struct cfg80211_scan_request *request,
                                          bool aborted)
{
    cfg80211_scan_done(request, aborted);
}
#endif

/* 커널 4.12+ roam API 변경 */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(4, 12, 0)
static inline void cfg80211_if_roamed(struct net_device *ndev,
                                       struct cfg80211_roam_info *info)
{
    cfg80211_roamed(ndev, info, GFP_KERNEL);
}
#else
static inline void cfg80211_if_roamed(struct net_device *ndev,
                                       struct cfg80211_roam_info *info)
{
    cfg80211_roamed(ndev, info->channel, info->bssid,
                    info->req_ie, info->req_ie_len,
                    info->resp_ie, info->resp_ie_len, GFP_KERNEL);
}
#endif

/* 커널 4.17+ connect_done API */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(4, 17, 0)
static inline void cfg80211_if_connect_result(struct net_device *ndev,
                                               const u8 *bssid,
                                               const u8 *req_ie, size_t req_ie_len,
                                               const u8 *resp_ie, size_t resp_ie_len,
                                               u16 status)
{
    struct cfg80211_connect_resp_params params = {
        .status = status,
        .bssid = bssid,
        .req_ie = req_ie,
        .req_ie_len = req_ie_len,
        .resp_ie = resp_ie,
        .resp_ie_len = resp_ie_len,
    };
    cfg80211_connect_done(ndev, &params, GFP_KERNEL);
}
#else
static inline void cfg80211_if_connect_result(struct net_device *ndev,
                                               const u8 *bssid,
                                               const u8 *req_ie, size_t req_ie_len,
                                               const u8 *resp_ie, size_t resp_ie_len,
                                               u16 status)
{
    cfg80211_connect_result(ndev, bssid, req_ie, req_ie_len,
                            resp_ie, resp_ie_len, status, GFP_KERNEL);
}
#endif
```

---

### 3.2 vendor_nl80211

| 항목 | 내용 |
|------|------|
| **역할** | Vendor specific nl80211 command 처리 |
| **책임** | Private command 등록, 파싱, 응답 생성 |
| **의존성** | OSAL, cfg80211_if |

#### Vendor Command 정의

```c
/* vendor_nl80211.h */

#define WIFI_VENDOR_OUI    0x001234  /* 예시 OUI */

enum wifi_vendor_commands {
    WIFI_VENDOR_CMD_GET_CAPABILITIES = 0,
    WIFI_VENDOR_CMD_GET_RSSI = 0x100,
    WIFI_VENDOR_CMD_ROAM_CONTROL,
    WIFI_VENDOR_CMD_SET_COUNTRY,
    WIFI_VENDOR_CMD_TEST_MODE = 0x400,
    WIFI_VENDOR_CMD_CUSTOMER_START = 0x1000,  /* 고객 확장 시작점 */
};

enum wifi_vendor_attrs {
    WIFI_VENDOR_ATTR_INVALID = 0,
    WIFI_VENDOR_ATTR_RSSI,
    WIFI_VENDOR_ATTR_CHANNEL,
    WIFI_VENDOR_ATTR_STATUS,
    WIFI_VENDOR_ATTR_DATA,
    WIFI_VENDOR_ATTR_MAX,
};
```

#### 구현 예시

```c
/* vendor_nl80211.c */

static const struct nla_policy wifi_vendor_policy[WIFI_VENDOR_ATTR_MAX] = {
    [WIFI_VENDOR_ATTR_RSSI] = { .type = NLA_S32 },
    [WIFI_VENDOR_ATTR_CHANNEL] = { .type = NLA_U32 },
    [WIFI_VENDOR_ATTR_STATUS] = { .type = NLA_U32 },
    [WIFI_VENDOR_ATTR_DATA] = { .type = NLA_BINARY },
};

static int vendor_cmd_get_rssi(struct wiphy *wiphy, struct wireless_dev *wdev,
                               const void *data, int data_len)
{
    struct vif_context *vif = wdev_to_vif(wdev);
    struct sk_buff *reply;
    int rssi;
    
    rssi = vif_manager_get_rssi(vif);
    
    reply = cfg80211_vendor_cmd_alloc_reply_skb(wiphy, 32);
    if (!reply)
        return -ENOMEM;
    
    nla_put_s32(reply, WIFI_VENDOR_ATTR_RSSI, rssi);
    
    return cfg80211_vendor_cmd_reply(reply);
}

static const struct wiphy_vendor_command wifi_vendor_commands[] = {
    {
        .info = { .vendor_id = WIFI_VENDOR_OUI, 
                  .subcmd = WIFI_VENDOR_CMD_GET_RSSI },
        .flags = WIPHY_VENDOR_CMD_NEED_WDEV | WIPHY_VENDOR_CMD_NEED_RUNNING,
        .doit = vendor_cmd_get_rssi,
        .policy = wifi_vendor_policy,
        .maxattr = WIFI_VENDOR_ATTR_MAX - 1,
    },
    /* ... */
};

int vendor_nl80211_init(struct wiphy *wiphy)
{
    wiphy->vendor_commands = wifi_vendor_commands;
    wiphy->n_vendor_commands = ARRAY_SIZE(wifi_vendor_commands);
    return 0;
}
```

---

### 3.3 netdev_if

| 항목 | 내용 |
|------|------|
| **역할** | net_device 관리 및 ops 구현 |
| **책임** | netdev 생성/삭제, TX/RX 인터페이스, 통계 관리 |
| **의존성** | OSAL, cfg80211_if |

#### 인터페이스

```c
/* netdev_if.h */

struct net_device *netdev_if_create(struct vif_context *vif, 
                                     enum nl80211_iftype type,
                                     const char *name);
void netdev_if_destroy(struct net_device *ndev);

/* TX (커널에서 호출) */
netdev_tx_t netdev_if_start_xmit(struct sk_buff *skb, struct net_device *ndev);

/* RX (HIP에서 호출) */
void netdev_if_rx(struct net_device *ndev, struct sk_buff *skb);

/* 상태 관리 */
void netdev_if_carrier_on(struct net_device *ndev);
void netdev_if_carrier_off(struct net_device *ndev);
void netdev_if_wake_queue(struct net_device *ndev);
void netdev_if_stop_queue(struct net_device *ndev);
```

#### 구현 예시

```c
/* netdev_if.c */

static netdev_tx_t netdev_if_start_xmit(struct sk_buff *skb, 
                                         struct net_device *ndev)
{
    struct vif_context *vif = netdev_priv(ndev);
    int ret;
    
    if (!vif->active) {
        dev_kfree_skb_any(skb);
        ndev->stats.tx_dropped++;
        return NETDEV_TX_OK;
    }
    
    /* MA Handler로 전달 */
    ret = ma_tx_submit(vif, skb);
    if (ret < 0) {
        if (ret == -ENOSPC) {
            netif_stop_queue(ndev);
            return NETDEV_TX_BUSY;
        }
        dev_kfree_skb_any(skb);
        ndev->stats.tx_dropped++;
        return NETDEV_TX_OK;
    }
    
    ndev->stats.tx_packets++;
    ndev->stats.tx_bytes += skb->len;
    
    return NETDEV_TX_OK;
}

void netdev_if_rx(struct net_device *ndev, struct sk_buff *skb)
{
    skb->dev = ndev;
    skb->protocol = eth_type_trans(skb, ndev);
    
    ndev->stats.rx_packets++;
    ndev->stats.rx_bytes += skb->len;
    
    netif_receive_skb(skb);
}

static const struct net_device_ops wifi_netdev_ops = {
    .ndo_open = netdev_if_open,
    .ndo_stop = netdev_if_stop,
    .ndo_start_xmit = netdev_if_start_xmit,
    .ndo_set_mac_address = netdev_if_set_mac_address,
    .ndo_get_stats64 = netdev_if_get_stats64,
};
```

---

### 3.4 debugfs_if

| 항목 | 내용 |
|------|------|
| **역할** | debugfs 인터페이스 제공 |
| **책임** | 런타임 디버그 정보 노출, 디버그 명령 수신 |
| **의존성** | OSAL |

#### 구현 예시

```c
/* debugfs_if.c */

static struct dentry *g_debugfs_root;

static ssize_t debugfs_read_status(struct file *file, char __user *buf,
                                   size_t count, loff_t *ppos)
{
    struct vif_context *vif = file->private_data;
    char status_buf[512];
    int len;
    
    len = snprintf(status_buf, sizeof(status_buf),
                   "VIF: %d\nType: %s\nState: %s\nRSSI: %d dBm\n",
                   vif->vif_id, vif_type_to_str(vif->type),
                   vif_state_to_str(vif->state), vif->rssi);
    
    return simple_read_from_buffer(buf, count, ppos, status_buf, len);
}

static const struct file_operations debugfs_status_fops = {
    .owner = THIS_MODULE,
    .open = simple_open,
    .read = debugfs_read_status,
};

int debugfs_if_init(void)
{
    g_debugfs_root = debugfs_create_dir("wifi_driver", NULL);
    return g_debugfs_root ? 0 : -ENOMEM;
}

int debugfs_if_add_vif(struct vif_context *vif)
{
    char name[32];
    
    snprintf(name, sizeof(name), "vif%d", vif->vif_id);
    vif->debugfs_dir = debugfs_create_dir(name, g_debugfs_root);
    
    debugfs_create_file("status", 0444, vif->debugfs_dir, vif, 
                        &debugfs_status_fops);
    
    return 0;
}
```

---

### 3.5 procfs_if

| 항목 | 내용 |
|------|------|
| **역할** | procfs 인터페이스 제공 |
| **책임** | 테스트 모드 인터페이스, 생산 라인 지원 |
| **의존성** | OSAL |

---

## 4. 디렉토리 구조

```
kernel_if/
├── cfg80211_if.c           # cfg80211 ops 구현
├── cfg80211_if.h
├── cfg80211_if_compat.h    # 커널 버전 호환성
├── vendor_nl80211.c        # vendor command
├── vendor_nl80211.h
├── netdev_if.c             # net_device 관리
├── netdev_if.h
├── debugfs_if.c
├── debugfs_if.h
├── procfs_if.c
└── procfs_if.h
```

---

## 5. 주목할 특성

### 5.1 커널 버전 의존성 격리

```
cfg80211_if.c         # 버전 독립 로직
       │
       ▼
cfg80211_if_compat.h  # 버전별 호환성 매크로
       │
       ▼
cfg80211 kernel API   # 실제 커널 API
```

**설계 원칙:**

- 버전별 분기는 `*_compat.h`에만 존재
- 상위 레이어는 버전 차이를 인식하지 못함
- 새 커널 버전 지원 시 `*_compat.h`만 수정

### 5.2 의존성

```
┌─────────────────────────────────────────┐
│        Service Manager (Layer 3)        │
└────────────────┬────────────────────────┘
                 │ 호출
                 ▼
┌─────────────────────────────────────────┐
│         Kernel Interface (Layer 2)      │
│  cfg80211_if, vendor_nl80211, netdev_if │
└────────────────┬────────────────────────┘
                 │ 사용
                 ▼
┌─────────────────────────────────────────┐
│          Linux Kernel APIs              │
│   cfg80211, net_device, debugfs, etc    │
└─────────────────────────────────────────┘
```

- **외부 의존성**: cfg80211, net_device, debugfs (Linux 커널)
- **내부 의존성**: OSAL (Layer 1)
- **의존하는 모듈**: Service Manager (Layer 3)

### 5.3 cfg80211 콜백 흐름

```
User Space (wpa_supplicant)
       │
       ▼ (nl80211)
┌──────────────────┐
│    cfg80211      │
└────────┬─────────┘
         │ cfg80211_ops callback
         ▼
┌──────────────────┐
│  cfg80211_if     │  ← 여기서 vif 찾고 파라미터 변환
└────────┬─────────┘
         │ vif_manager_dispatch_xxx()
         ▼
┌──────────────────┐
│  vif_manager     │  (Layer 3)
└──────────────────┘
```

### 5.4 이벤트 역방향 흐름

```
┌──────────────────┐
│    Service       │  (Layer 3)
└────────┬─────────┘
         │ cfg80211_if_xxx() 호출
         ▼
┌──────────────────┐
│  cfg80211_if     │  ← 커널 버전 호환성 처리
└────────┬─────────┘
         │ cfg80211_xxx()
         ▼
┌──────────────────┐
│    cfg80211      │  → User Space로 이벤트 전달
└──────────────────┘
```
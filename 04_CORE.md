# Layer 4: Core (802.11 Protocol Layers)

## 1. 개요

### 1.1 역할

802.11 표준 프로토콜 로직을 구현한다. 서비스 타입별 Management Entity와 공통 MLME/MA Handler로 구성된다.

### 1.2 설계 목표

- 802.11 표준 레이어 개념 반영 (SME, MLME, MA)
- 서비스별 Management Entity 분리 (God Module 방지)
- 프로토콜 로직과 정책 결정 분리

### 1.3 Layer 3 vs Layer 4 역할 분리

| Layer | 역할 | 예시 |
|-------|------|------|
| **Layer 3 (Service)** | 정책 결정, 상태 머신 orchestration | "로밍할까 말까", "RSSI 낮으면 스캔 트리거" |
| **Layer 4 (Core)** | 프로토콜 로직, MLME primitive 생성 | "auth frame 생성", "assoc req 파라미터 세팅" |

**흐름 예시:**

```
svc_sta: "지금 RSSI 낮으니까 로밍하자" (정책 결정)
    │
    ▼
sme_sta.roam(): "roam 절차 시작, auth→assoc 시퀀스 실행" (프로토콜)
    │
    ▼
mlme_handler: "MLME_ROAM_REQ 메시지 생성해서 FW로" (메시지)
```

---

## 2. 모듈 구성

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 4: Core (802.11 Protocol Layers)                                        │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Management Entities (서비스별)                                          │   │
│  │                                                                          │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────┐│   │
│  │  │   sme_sta     │  │    sme_ap     │  │   p2p_mgmt    │  │ nan_mgmt  ││   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘  └───────────┘│   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ MLME Handler (공통)                                                     │   │
│  │ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐                  │   │
│  │ │   mlme_req    │ │   mlme_cfm    │ │   mlme_ind    │                  │   │
│  │ └───────────────┘ └───────────────┘ └───────────────┘                  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ MA Handler (Data Path)                                                  │   │
│  │ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐│   │
│  │ │   tx_ctrl     │ │   rx_ctrl     │ │   ba_mgmt     │ │     qos       ││   │
│  │ └───────────────┘ └───────────────┘ └───────────────┘ └───────────────┘│   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐   │
│  │      security        │ │     power_mgmt       │ │     regulatory       │   │
│  └──────────────────────┘ └──────────────────────┘ └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Management Entities

### 3.1 sme_sta (Station Management Entity)

| 항목 | 내용 |
|------|------|
| **역할** | STA 모드 802.11 프로토콜 처리 |
| **책임** | Scan, Auth, Assoc, Deauth, Roam 프로토콜 구현 |
| **호출자** | svc_sta, svc_p2p (GC 모드) |
| **권장 LOC** | < 800 |

#### 서브 모듈 구조

```
sme_sta/
├── sme_sta.c           # 메인 인터페이스
├── sme_sta_scan.c      # 스캔 요청/결과 처리
├── sme_sta_connect.c   # 연결 시퀀스 (auth → assoc)
├── sme_sta_auth.c      # Authentication 프로토콜
├── sme_sta_assoc.c     # Association 프로토콜
├── sme_sta_roam.c      # 로밍 프로토콜 (11k/11v/11r)
└── sme_sta_pm.c        # Power Save 모드
```

#### 인터페이스

```c
/* sme_sta.h */

/**
 * Scan
 */
int sme_sta_scan_request(struct vif_context *vif, 
                         struct cfg80211_scan_request *request);
void sme_sta_scan_abort(struct vif_context *vif);

/**
 * Connect/Disconnect
 */
int sme_sta_connect(struct vif_context *vif, 
                    struct cfg80211_connect_params *params);
int sme_sta_disconnect(struct vif_context *vif, u16 reason_code);

/**
 * Roaming
 */
int sme_sta_roam_start(struct vif_context *vif, const u8 *target_bssid);
void sme_sta_roam_abort(struct vif_context *vif);

/**
 * Power Save
 */
int sme_sta_set_power_save(struct vif_context *vif, bool enable);

/**
 * 11k/11v/11r
 */
int sme_sta_send_neighbor_report_req(struct vif_context *vif);
int sme_sta_send_btm_query(struct vif_context *vif, u8 reason);

/**
 * Key 관리
 */
int sme_sta_add_key(struct vif_context *vif, u8 key_idx, bool pairwise,
                    const u8 *mac_addr, struct key_params *params);
int sme_sta_del_key(struct vif_context *vif, u8 key_idx, bool pairwise,
                    const u8 *mac_addr);

/**
 * PMKSA
 */
int sme_sta_set_pmksa(struct vif_context *vif, struct cfg80211_pmksa *pmksa);
int sme_sta_del_pmksa(struct vif_context *vif, struct cfg80211_pmksa *pmksa);
```

#### 구현 예시 (Connect)

```c
/* sme_sta_connect.c */

int sme_sta_connect(struct vif_context *vif,
                    struct cfg80211_connect_params *params)
{
    struct mlme_connect_req req;
    int ret;
    
    OSAL_DBG("sme_sta_connect: ssid=%.*s, bssid=%pM\n",
             (int)params->ssid_len, params->ssid, params->bssid);
    
    /* MLME 요청 구성 */
    memset(&req, 0, sizeof(req));
    
    /* 기본 정보 */
    if (params->bssid)
        memcpy(req.bssid, params->bssid, ETH_ALEN);
    memcpy(req.ssid, params->ssid, params->ssid_len);
    req.ssid_len = params->ssid_len;
    
    /* 채널 정보 */
    if (params->channel) {
        req.channel = params->channel->hw_value;
        req.band = params->channel->band;
    }
    
    /* Auth 타입 */
    req.auth_type = params->auth_type;
    
    /* Security IE 구성 */
    ret = build_security_ies(vif, &req, params);
    if (ret < 0)
        return ret;
    
    /* 추가 IE */
    if (params->ie && params->ie_len) {
        memcpy(req.extra_ie, params->ie, params->ie_len);
        req.extra_ie_len = params->ie_len;
    }
    
    /* HT/VHT/HE capability */
    build_ht_vht_he_caps(vif, &req);
    
    /* MLME Handler로 전달 */
    return mlme_req_connect(vif, &req);
}

static int build_security_ies(struct vif_context *vif,
                              struct mlme_connect_req *req,
                              struct cfg80211_connect_params *params)
{
    struct cfg80211_crypto_settings *crypto = &params->crypto;
    
    if (crypto->wpa_versions) {
        req->wpa_versions = crypto->wpa_versions;
        
        /* Cipher suites */
        if (crypto->n_ciphers_pairwise > 0)
            req->cipher_pairwise = crypto->ciphers_pairwise[0];
        req->cipher_group = crypto->cipher_group;
        
        /* AKM */
        if (crypto->n_akm_suites > 0)
            req->akm_suite = crypto->akm_suites[0];
        
        /* RSN IE 생성 */
        req->rsn_ie_len = security_build_rsn_ie(req->rsn_ie, 
                                                 sizeof(req->rsn_ie),
                                                 crypto);
    }
    
    /* MFP */
    req->mfp = crypto->mfp;
    
    return 0;
}
```

#### 구현 예시 (Roaming with 11k/11v/11r)

```c
/* sme_sta_roam.c */

/**
 * BTM (BSS Transition Management) Request 처리
 */
int sme_sta_process_btm_request(struct vif_context *vif,
                                 struct btm_request *btm_req)
{
    struct btm_response resp;
    struct btm_candidate *best = NULL;
    int best_score = 0;
    int i;
    
    memset(&resp, 0, sizeof(resp));
    resp.dialog_token = btm_req->dialog_token;
    
    /* 후보 AP 평가 */
    for (i = 0; i < btm_req->candidate_count; i++) {
        struct btm_candidate *cand = &btm_req->candidate_list[i];
        int score = evaluate_btm_candidate(vif, cand);
        
        if (score > best_score) {
            best_score = score;
            best = cand;
        }
    }
    
    if (best && best_score > 0) {
        resp.status = BTM_STATUS_ACCEPT;
        memcpy(resp.target_bssid, best->bssid, ETH_ALEN);
        
        /* 로밍 시작 */
        sme_sta_roam_start(vif, best->bssid);
    } else {
        resp.status = BTM_STATUS_REJECT_NO_SUITABLE_CANDIDATES;
    }
    
    return send_btm_response(vif, &resp);
}

/**
 * 11r Fast Transition
 */
int sme_sta_ft_roam(struct vif_context *vif, const u8 *target_bssid)
{
    struct mlme_ft_roam_req req;
    
    memset(&req, 0, sizeof(req));
    memcpy(req.target_bssid, target_bssid, ETH_ALEN);
    
    /* FT IE 구성 */
    req.mdie_len = build_mdie(req.mdie, sizeof(req.mdie), vif->ft_ctx->mdid);
    req.ftie_len = build_ftie(req.ftie, sizeof(req.ftie), vif->ft_ctx);
    
    return mlme_req_ft_roam(vif, &req);
}
```

---

### 3.2 sme_ap (AP Management Entity)

| 항목 | 내용 |
|------|------|
| **역할** | AP 모드 802.11 프로토콜 처리 |
| **책임** | AP Start/Stop, Beacon, STA 인증/연결 관리 |
| **호출자** | svc_mhs, svc_p2p (GO 모드) |
| **권장 LOC** | < 700 |

#### 서브 모듈 구조

```
sme_ap/
├── sme_ap.c            # 메인 인터페이스
├── sme_ap_start.c      # AP 시작/중지
├── sme_ap_beacon.c     # Beacon 관리
├── sme_ap_sta_mgmt.c   # 연결된 STA 관리
└── sme_ap_acs.c        # Auto Channel Selection
```

#### 인터페이스

```c
/* sme_ap.h */

/**
 * AP 시작/중지
 */
int sme_ap_start(struct vif_context *vif, struct cfg80211_ap_settings *settings);
int sme_ap_stop(struct vif_context *vif);

/**
 * Beacon
 */
int sme_ap_update_beacon(struct vif_context *vif, 
                         struct cfg80211_beacon_data *beacon);

/**
 * STA 관리
 */
int sme_ap_del_station(struct vif_context *vif, const u8 *mac, u16 reason);
int sme_ap_get_station_info(struct vif_context *vif, const u8 *mac,
                            struct station_info *sinfo);

/**
 * ACS
 */
int sme_ap_trigger_acs(struct vif_context *vif);
int sme_ap_get_acs_result(struct vif_context *vif, int *channel);
```

#### 구현 예시

```c
/* sme_ap_start.c */

int sme_ap_start(struct vif_context *vif, struct cfg80211_ap_settings *settings)
{
    struct mlme_start_ap_req req;
    int ret;
    
    memset(&req, 0, sizeof(req));
    
    /* 기본 정보 */
    memcpy(req.ssid, settings->ssid, settings->ssid_len);
    req.ssid_len = settings->ssid_len;
    req.hidden_ssid = settings->hidden_ssid;
    
    /* 채널 정보 */
    req.channel = settings->chandef.chan->hw_value;
    req.bandwidth = nl80211_chan_width_to_bw(settings->chandef.width);
    req.band = settings->chandef.chan->band;
    
    /* Beacon */
    req.beacon_interval = settings->beacon_interval;
    req.dtim_period = settings->dtim_period;
    
    /* Beacon IE 복사 */
    ret = build_ap_beacon_ies(&req, settings);
    if (ret < 0)
        return ret;
    
    /* Security */
    if (settings->crypto.wpa_versions) {
        req.privacy = true;
        req.wpa_versions = settings->crypto.wpa_versions;
        req.cipher_pairwise = settings->crypto.ciphers_pairwise[0];
        req.cipher_group = settings->crypto.cipher_group;
        req.akm_suite = settings->crypto.akm_suites[0];
    }
    
    /* HT/VHT/HE capability */
    if (settings->ht_cap)
        memcpy(&req.ht_cap, settings->ht_cap, sizeof(req.ht_cap));
    if (settings->vht_cap)
        memcpy(&req.vht_cap, settings->vht_cap, sizeof(req.vht_cap));
    
    return mlme_req_start_ap(vif, &req);
}
```

---

### 3.3 p2p_mgmt (P2P Management)

| 항목 | 내용 |
|------|------|
| **역할** | Wi-Fi Direct 프로토콜 처리 |
| **책임** | P2P Discovery, GO Negotiation, Provision Discovery, NOA |
| **호출자** | svc_p2p |
| **위임** | GO→sme_ap, GC→sme_sta |
| **권장 LOC** | < 800 |

#### 인터페이스

```c
/* p2p_mgmt.h */

/**
 * Discovery
 */
int p2p_mgmt_start_find(struct vif_context *vif, unsigned int timeout);
int p2p_mgmt_stop_find(struct vif_context *vif);
int p2p_mgmt_listen(struct vif_context *vif, int channel, unsigned int duration);

/**
 * GO Negotiation
 */
int p2p_mgmt_connect(struct vif_context *vif, const u8 *peer_addr,
                     int go_intent, int freq);
int p2p_mgmt_reject_nego(struct vif_context *vif, const u8 *peer_addr);

/**
 * Provision Discovery
 */
int p2p_mgmt_prov_disc_req(struct vif_context *vif, const u8 *peer_addr,
                           u16 config_methods);

/**
 * Group
 */
int p2p_mgmt_start_go(struct vif_context *vif, int freq);
int p2p_mgmt_stop_group(struct vif_context *vif);

/**
 * NOA (Notice of Absence)
 */
int p2p_mgmt_set_noa(struct vif_context *vif, int count, int start, 
                     int duration, int interval);
```

#### 재사용 패턴 (위임)

```c
/* p2p_mgmt.c */

/**
 * P2P GO로 동작할 때는 sme_ap를 재사용
 */
int p2p_mgmt_start_go_ap(struct vif_context *vif, 
                         struct cfg80211_ap_settings *settings)
{
    /* P2P specific IE 추가 */
    add_p2p_ie_to_beacon(settings);
    
    /* sme_ap로 위임 */
    return sme_ap_start(vif, settings);
}

/**
 * P2P GC로 동작할 때는 sme_sta를 재사용
 */
int p2p_mgmt_gc_connect(struct vif_context *vif,
                        struct cfg80211_connect_params *params)
{
    /* P2P specific IE 추가 */
    add_p2p_ie_to_assoc_req(params);
    
    /* sme_sta로 위임 */
    return sme_sta_connect(vif, params);
}
```

---

### 3.4 nan_mgmt (NAN Management)

| 항목 | 내용 |
|------|------|
| **역할** | NAN (Neighbor Awareness Networking) 프로토콜 처리 |
| **책임** | NAN Enable/Disable, Publish/Subscribe, NDP, Ranging |
| **호출자** | svc_nan |
| **권장 LOC** | < 700 |

#### 인터페이스

```c
/* nan_mgmt.h */

/**
 * NAN Enable/Disable
 */
int nan_mgmt_enable(struct vif_context *vif, struct nan_enable_params *params);
int nan_mgmt_disable(struct vif_context *vif);

/**
 * Publish/Subscribe
 */
int nan_mgmt_publish(struct vif_context *vif, struct nan_publish_params *params);
int nan_mgmt_cancel_publish(struct vif_context *vif, u8 publish_id);
int nan_mgmt_subscribe(struct vif_context *vif, struct nan_subscribe_params *params);
int nan_mgmt_cancel_subscribe(struct vif_context *vif, u8 subscribe_id);

/**
 * NDP (NAN Data Path)
 */
int nan_mgmt_ndp_request(struct vif_context *vif, struct nan_ndp_req_params *params);
int nan_mgmt_ndp_response(struct vif_context *vif, struct nan_ndp_resp_params *params);
int nan_mgmt_ndp_terminate(struct vif_context *vif, u8 ndp_id);

/**
 * Ranging
 */
int nan_mgmt_ranging_request(struct vif_context *vif, 
                             struct nan_ranging_params *params);
```

---

## 4. MLME Handler

### 4.1 역할

| 항목 | 내용 |
|------|------|
| **역할** | MLME 메시지 처리 (Management Layer Management Entity) |
| **책임** | MLME REQ 생성, MLME CFM/IND 디스패치 |
| **방향** | REQ: Driver → FW, CFM/IND: FW → Driver |

### 4.2 메시지 흐름

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│  sme_sta    │         │ mlme_handler│         │  msg_router │
│  sme_ap     │         │             │         │  (Layer 5)  │
│  p2p_mgmt   │         │             │         │             │
└──────┬──────┘         └──────┬──────┘         └──────┬──────┘
       │                       │                       │
       │ sme_xxx()             │                       │
       │──────────────────────▶│                       │
       │                       │ mlme_req_xxx()        │
       │                       │──────────────────────▶│───────▶ FW
       │                       │                       │
       │                       │   MLME_XXX_CFM        │
       │                       │◀──────────────────────│◀─────── FW
       │ callback              │                       │
       │◀──────────────────────│                       │
```

### 4.3 인터페이스

```c
/* mlme_handler.h */

/**
 * MLME Request (Driver → FW)
 */
int mlme_req_scan(struct vif_context *vif, struct mlme_scan_req *req);
int mlme_req_connect(struct vif_context *vif, struct mlme_connect_req *req);
int mlme_req_disconnect(struct vif_context *vif, u16 reason_code);
int mlme_req_roam(struct vif_context *vif, struct mlme_roam_req *req);
int mlme_req_start_ap(struct vif_context *vif, struct mlme_start_ap_req *req);
int mlme_req_stop_ap(struct vif_context *vif);
int mlme_req_add_key(struct vif_context *vif, struct mlme_add_key_req *req);
int mlme_req_del_key(struct vif_context *vif, struct mlme_del_key_req *req);

/**
 * MLME Confirm/Indication 핸들러 등록
 */
typedef void (*mlme_cfm_handler_t)(struct vif_context *vif, void *cfm);
typedef void (*mlme_ind_handler_t)(struct vif_context *vif, void *ind);

void mlme_register_cfm_handler(enum mlme_cfm_type type, mlme_cfm_handler_t handler);
void mlme_register_ind_handler(enum mlme_ind_type type, mlme_ind_handler_t handler);

/**
 * FW 메시지 수신 시 호출 (msg_router에서 호출)
 */
void mlme_handle_cfm(struct vif_context *vif, enum mlme_cfm_type type, void *cfm);
void mlme_handle_ind(struct vif_context *vif, enum mlme_ind_type type, void *ind);
```

### 4.4 구현 예시

```c
/* mlme_handler.c */

int mlme_req_scan(struct vif_context *vif, struct mlme_scan_req *req)
{
    struct msg_hdr hdr;
    struct msg_body_scan_req body;
    
    /* 메시지 헤더 */
    hdr.id = MLME_SCAN_REQ;
    hdr.vif_id = vif->vif_id;
    hdr.len = sizeof(body);
    
    /* 메시지 바디 */
    memset(&body, 0, sizeof(body));
    body.scan_type = req->scan_type;
    body.n_channels = req->n_channels;
    memcpy(body.channels, req->channels, req->n_channels * sizeof(u16));
    memcpy(body.ssid, req->ssid, req->ssid_len);
    body.ssid_len = req->ssid_len;
    body.dwell_time = req->dwell_time;
    
    /* FW Message Layer로 전달 */
    return msg_router_send(MSG_CAT_MLME, &hdr, &body);
}

int mlme_req_connect(struct vif_context *vif, struct mlme_connect_req *req)
{
    struct msg_hdr hdr;
    struct msg_body_connect_req body;
    
    hdr.id = MLME_CONNECT_REQ;
    hdr.vif_id = vif->vif_id;
    hdr.len = sizeof(body);
    
    memset(&body, 0, sizeof(body));
    memcpy(body.bssid, req->bssid, ETH_ALEN);
    memcpy(body.ssid, req->ssid, req->ssid_len);
    body.ssid_len = req->ssid_len;
    body.channel = req->channel;
    body.auth_type = req->auth_type;
    body.wpa_versions = req->wpa_versions;
    body.cipher_pairwise = req->cipher_pairwise;
    body.cipher_group = req->cipher_group;
    body.akm_suite = req->akm_suite;
    body.mfp = req->mfp;
    
    /* IE 복사 */
    if (req->rsn_ie_len > 0) {
        memcpy(body.rsn_ie, req->rsn_ie, req->rsn_ie_len);
        body.rsn_ie_len = req->rsn_ie_len;
    }
    
    return msg_router_send(MSG_CAT_MLME, &hdr, &body);
}

/**
 * FW에서 온 Indication 처리
 */
void mlme_handle_ind(struct vif_context *vif, enum mlme_ind_type type, void *ind)
{
    mlme_ind_handler_t handler = g_mlme_ind_handlers[type];
    
    if (handler)
        handler(vif, ind);
    else
        OSAL_WARN("No handler for MLME IND type %d\n", type);
}
```

---

## 5. MA Handler (Data Path)

### 5.1 역할

| 항목 | 내용 |
|------|------|
| **역할** | Data Path 제어 |
| **책임** | TX/RX 제어, Block Ack 관리, QoS |

### 5.2 서브 모듈

| 서브 모듈 | 역할 |
|-----------|------|
| tx_ctrl | TX 큐 관리, flow control, TX complete 처리 |
| rx_ctrl | RX 패킷 분류, reordering, 상위 전달 |
| ba_mgmt | Block Ack 세션 관리 (ADDBA, DELBA) |
| qos | WMM/QoS 파라미터 관리 |

### 5.3 인터페이스

```c
/* ma_handler.h */

/**
 * TX
 */
int ma_tx_submit(struct vif_context *vif, struct sk_buff *skb);
void ma_tx_complete(struct vif_context *vif, u16 seq_num, bool success);
void ma_tx_flow_control(struct vif_context *vif, bool stop);

/**
 * RX
 */
void ma_rx_packet(struct vif_context *vif, struct sk_buff *skb);

/**
 * Block Ack
 */
int ma_ba_session_start(struct vif_context *vif, const u8 *peer, u8 tid);
int ma_ba_session_stop(struct vif_context *vif, const u8 *peer, u8 tid);

/**
 * QoS
 */
int ma_set_wmm_params(struct vif_context *vif, struct wmm_params *params);
```

### 5.4 TX 흐름

```
┌──────────────┐
│  netdev_if   │  (Layer 2)
└──────┬───────┘
       │ ma_tx_submit(skb)
       ▼
┌──────────────┐
│   tx_ctrl    │  - 큐 선택 (AC)
│              │  - Flow control 검사
│              │  - 헤더 정보 추가
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  msg_router  │  (Layer 5)
└──────┬───────┘
       │ MA_TX_REQ
       ▼
┌──────────────┐
│     HIP      │  (Layer 6)
└──────────────┘
```

---

## 6. 공통 모듈

### 6.1 security

| 항목 | 내용 |
|------|------|
| **역할** | 보안 관련 공통 기능 |
| **책임** | RSN IE 생성/파싱, Key 관리, MFP 처리 |

```c
/* security.h */

int security_build_rsn_ie(u8 *buf, size_t buf_len,
                          struct cfg80211_crypto_settings *crypto);
int security_parse_rsn_ie(const u8 *ie, size_t ie_len,
                          struct rsn_ie_data *data);
int security_install_key(struct vif_context *vif, 
                         struct key_params *params);
```

### 6.2 power_mgmt

| 항목 | 내용 |
|------|------|
| **역할** | 전력 관리 공통 기능 |
| **책임** | Power Save 모드, UAPSD, WoWLAN |

```c
/* power_mgmt.h */

int power_mgmt_set_ps_mode(struct vif_context *vif, bool enable);
int power_mgmt_set_uapsd(struct vif_context *vif, u8 ac_mask);
int power_mgmt_configure_wowlan(struct vif_context *vif,
                                 struct cfg80211_wowlan *wowlan);
```

### 6.3 regulatory

| 항목 | 내용 |
|------|------|
| **역할** | 규제 도메인 관리 |
| **책임** | 국가 코드 설정, 채널 가용성 검사 |

```c
/* regulatory.h */

int regulatory_set_country(const char *alpha2);
int regulatory_get_country(char *alpha2);
bool regulatory_is_channel_allowed(int channel, enum nl80211_band band);
int regulatory_get_max_tx_power(int channel, enum nl80211_band band);
```

---

## 7. 디렉토리 구조

```
core/
├── sme_sta/
│   ├── sme_sta.c
│   ├── sme_sta.h
│   ├── sme_sta_scan.c
│   ├── sme_sta_connect.c
│   ├── sme_sta_roam.c
│   └── sme_sta_pm.c
│
├── sme_ap/
│   ├── sme_ap.c
│   ├── sme_ap.h
│   ├── sme_ap_start.c
│   ├── sme_ap_beacon.c
│   └── sme_ap_sta_mgmt.c
│
├── p2p_mgmt/
│   ├── p2p_mgmt.c
│   ├── p2p_mgmt.h
│   ├── p2p_discovery.c
│   ├── p2p_nego.c
│   └── p2p_noa.c
│
├── nan_mgmt/
│   ├── nan_mgmt.c
│   ├── nan_mgmt.h
│   ├── nan_discovery.c
│   └── nan_ndp.c
│
├── mlme_handler.c
├── mlme_handler.h
├── ma_handler.c
├── ma_handler.h
├── security.c
├── security.h
├── power_mgmt.c
├── power_mgmt.h
├── regulatory.c
└── regulatory.h
```

---

## 8. 주목할 특성

### 8.1 802.11 레이어 매핑

```
┌─────────────────────────────────────────────┐
│  802.11 Standard      │  Driver Module      │
├─────────────────────────────────────────────┤
│  SME (STA)            │  sme_sta            │
│  SME (AP)             │  sme_ap             │
│  MLME                 │  mlme_handler       │
│  MA (MAC Access)      │  ma_handler         │
│  Security             │  security           │
│  Power Management     │  power_mgmt         │
└─────────────────────────────────────────────┘
```

### 8.2 코드 재사용 (위임 패턴)

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

### 8.3 Callback 패턴 (역방향 통신)

```c
/* FW 이벤트가 올라올 때 */

/* 1. msg_router가 수신 */
msg_router_recv() {
    /* 2. MLME 카테고리면 mlme_handler로 */
    mlme_handle_ind(vif, type, ind);
}

/* 3. mlme_handler가 등록된 핸들러 호출 */
mlme_handle_ind() {
    handler = g_mlme_ind_handlers[type];
    handler(vif, ind);  /* → sme_sta에서 등록한 핸들러 */
}

/* 4. sme_sta 핸들러 */
sme_sta_connect_ind_handler(vif, ind) {
    /* 5. Service Layer로 콜백 */
    vif->svc_ops->handle_fw_event(vif->service, event);
}
```

### 8.4 의존성 규칙

```
svc_* (Layer 3) ──▶ sme_*/p2p_mgmt/nan_mgmt ──▶ mlme_handler ──▶ msg_router (Layer 5)
                         │
                         ▼
                   ma_handler ──▶ msg_router (Layer 5)
                         │
                         ▼
              security, power_mgmt, regulatory (공통)

규칙:
- Layer 3 → Layer 4 → Layer 5 단방향
- 역방향은 Callback으로만
- 같은 레이어 내에서도 순환 참조 금지
```
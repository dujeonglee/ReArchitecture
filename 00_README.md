# Linux WiFi FullMAC Driver 모듈화 구조 설계

## 1. 개요

### 1.1 목적

본 문서는 Linux WiFi FullMAC 드라이버의 최적화된 모듈 구조를 정의한다. 이 설계는 다음의 핵심 원칙을 기반으로 한다:

- **Single Responsibility Principle (SRP)**: 각 모듈은 하나의 명확한 책임만 가진다
- **God Module 방지**: 과도하게 비대한 모듈 없이 적절한 크기로 분리
- **순환 참조 제거**: 단방향 의존성을 통한 명확한 계층 구조
- **OS 추상화**: Linux/Windows 공용화를 위한 플랫폼 독립적 설계
- **고객 확장성**: Core 기능과 고객별 커스터마이징 분리

### 1.2 대상 시스템

| 항목 | 설명 |
|------|------|
| 드라이버 타입 | FullMAC (MAC sublayer가 Firmware에서 처리) |
| Bus Interface | MIF (onchip), PCIe (discrete) → HIP로 추상화 |
| Kernel Interface | cfg80211 직접 연동, nl80211 vendor command 지원 |
| 지원 서비스 | STA, MHS (Mobile Hotspot), P2P, NAN |
| 멀티 VIF | 최대 3개 VIF 동시 운용 |
| FW 메시지 | MLME, MA, DEBUG, WLANLITE/TEST |

---

## 2. 전체 아키텍처

### 2.1 레이어 다이어그램

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              User Space                                         │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         cfg80211 / nl80211 (Kernel)                             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
══════════════════════════════════════╪═══════════════════════════════════════════
                                      │         DRIVER BOUNDARY
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: OS Abstraction Layer (OSAL)                                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│  LAYER 2: Kernel Interface Layer                                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│  LAYER 3: Service Manager                                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│  LAYER 4: Core (802.11 Protocol Layers)                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│  LAYER 5: FW Message Layer                                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│  LAYER 6: HIP (Host Interface Protocol)                                        │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
══════════════════════════════════════╪═══════════════════════════════════════════
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Firmware (FullMAC)                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════════
 LAYER 7: Customer Extensions (Side Layer - Core와 분리)
═══════════════════════════════════════════════════════════════════════════════════
```

### 2.2 의존성 방향 (순환 참조 방지)

```
OSAL ← Kernel_IF ← Service_Mgr ← Core ← FW_Msg ← HIP
  ↑                     ↑           ↑
  └─────────────────────┴───────────┴──── Customer Hooks (선택적 주입)

규칙:
- 화살표 방향으로만 의존 (상위 → 하위)
- 같은 레이어 내 모듈 간 의존 최소화
- Customer는 Core에 hook으로만 연결, 직접 의존 금지
```

### 2.3 레이어별 문서

| 레이어 | 문서 | 설명 |
|--------|------|------|
| Layer 1 | [01_OSAL.md](01_OSAL.md) | OS Abstraction Layer |
| Layer 2 | [02_KERNEL_IF.md](02_KERNEL_IF.md) | Kernel Interface Layer |
| Layer 3 | [03_SERVICE_MANAGER.md](03_SERVICE_MANAGER.md) | Service Manager |
| Layer 4 | [04_CORE.md](04_CORE.md) | Core (802.11 Protocol) |
| Layer 5 | [05_FW_MESSAGE.md](05_FW_MESSAGE.md) | FW Message Layer |
| Layer 6 | [06_HIP.md](06_HIP.md) | Host Interface Protocol |
| Layer 7 | [07_CUSTOMER_EXTENSION.md](07_CUSTOMER_EXTENSION.md) | Customer Extensions |

---

## 3. 설계 원칙 요약

### 3.1 Single Responsibility Principle

각 모듈은 하나의 명확한 책임만 가진다:

| 잘못된 설계 | 올바른 설계 |
|-------------|-------------|
| `wifi_manager.c` (3000 LOC, 모든 것 처리) | `vif_manager.c` (라우팅만) + `svc_sta.c` (STA 로직) + ... |
| `mlme.c` (STA/AP/P2P 모두 처리) | `sme_sta.c` + `sme_ap.c` + `p2p_mgmt.c` |

### 3.2 God Module 방지

| 모듈 | 권장 LOC | 책임 |
|------|----------|------|
| vif_manager | < 500 | VIF 라이프사이클, 라우팅 |
| svc_sta | < 1000 | STA 상태 머신, 정책 결정 |
| sme_sta | < 800 | STA 프로토콜 구현 |
| mlme_handler | < 600 | MLME 메시지 처리 |

### 3.3 의존성 규칙

```c
/* 올바른 의존성 */
svc_sta.c → sme_sta.c → mlme_handler.c → msg_router.c

/* 금지된 의존성 (순환 참조) */
sme_sta.c → svc_sta.c  // ❌ Core가 Service를 호출하면 안됨
msg_router.c → sme_sta.c  // ❌ 하위가 상위를 직접 호출하면 안됨
```

**역방향 통신은 Callback으로:**

```c
/* FW 이벤트가 올라올 때 */
msg_router → mlme_handler → (registered callback) → sme_sta → svc_sta
```

### 3.4 플랫폼 독립성

```
┌─────────────────────────────────────────────────────────┐
│                    공용 코드 (Core)                      │
│  Service Manager, Core, FW Message Layer                │
│  → Linux/Windows 동일                                   │
├─────────────────────────────────────────────────────────┤
│                    OSAL Interface                        │
├─────────────┬───────────────────────────┬───────────────┤
│ OSAL Linux  │                           │ OSAL Windows  │
│ Kernel IF   │                           │ NDIS/WDF IF   │
│ HIP Linux   │                           │ HIP Windows   │
└─────────────┴───────────────────────────┴───────────────┘
```

---

## 4. 주요 흐름 예시

### 4.1 STA Connect 흐름

```
User Space: wpa_supplicant
       │
       ▼ (nl80211 connect)
┌──────────────────┐
│   cfg80211_if    │  Layer 2
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   vif_manager    │  Layer 3 - VIF 찾고 서비스로 라우팅
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│     svc_sta      │  Layer 3 - 상태 전이 (DISCONNECTED → CONNECTING)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│     sme_sta      │  Layer 4 - Connect 프로토콜 구성
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  mlme_handler    │  Layer 4 - MLME_CONNECT_REQ 생성
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   msg_router     │  Layer 5 - 메시지 직렬화
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│       HIP        │  Layer 6 - 버스로 전송
└────────┬─────────┘
         │
         ▼
      Firmware
```

### 4.2 FW Event (Connect Complete) 흐름

```
      Firmware
         │
         ▼
┌──────────────────┐
│       HIP        │  Layer 6 - 버스에서 수신
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   msg_router     │  Layer 5 - 메시지 파싱, 카테고리별 분배
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  mlme_handler    │  Layer 4 - MLME_CONNECT_IND 처리
└────────┬─────────┘
         │
         ▼ (registered callback)
┌──────────────────┐
│     sme_sta      │  Layer 4 - 프로토콜 상태 업데이트
└────────┬─────────┘
         │
         ▼ (service callback)
┌──────────────────┐
│     svc_sta      │  Layer 3 - 상태 전이 (CONNECTING → CONNECTED)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   cfg80211_if    │  Layer 2 - cfg80211_connect_result() 호출
└────────┬─────────┘
         │
         ▼
     cfg80211 → User Space
```

---

## 5. 디렉토리 구조 (권장)

```
driver/
├── osal/                           # Layer 1
│   ├── include/
│   │   └── osal.h
│   ├── linux/
│   │   ├── osal_memory_linux.c
│   │   ├── osal_sync_linux.c
│   │   └── ...
│   └── windows/
│       ├── osal_memory_windows.c
│       └── ...
│
├── kernel_if/                      # Layer 2 (Linux specific)
│   ├── cfg80211_if.c
│   ├── cfg80211_if_compat.h
│   ├── vendor_nl80211.c
│   ├── netdev_if.c
│   ├── debugfs_if.c
│   └── procfs_if.c
│
├── service/                        # Layer 3
│   ├── vif_manager.c
│   ├── svc_sta.c
│   ├── svc_mhs.c
│   ├── svc_p2p.c
│   ├── svc_nan.c
│   ├── svc_test.c
│   └── service_common.c
│
├── core/                           # Layer 4
│   ├── sme_sta/
│   │   ├── sme_sta_scan.c
│   │   ├── sme_sta_connect.c
│   │   └── ...
│   ├── sme_ap/
│   ├── p2p_mgmt/
│   ├── nan_mgmt/
│   ├── mlme_handler.c
│   ├── ma_handler.c
│   ├── security.c
│   ├── power_mgmt.c
│   └── regulatory.c
│
├── fw_msg/                         # Layer 5
│   ├── msg_router.c
│   ├── msg_mlme.c
│   ├── msg_ma.c
│   ├── msg_debug.c
│   ├── msg_wlanlite.c
│   ├── msg_builder.c
│   └── msg_parser.c
│
├── hip/                            # Layer 6
│   ├── hip_core.c
│   ├── hip_mif.c
│   └── hip_pcie.c
│
└── customer/                       # Layer 7
    ├── customer_hook.c
    ├── cust_vendor_a/
    └── cust_vendor_b/
```

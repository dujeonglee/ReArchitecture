# Layer 6: HIP (Host Interface Protocol)

## 1. 개요

### 1.1 역할

Host(Driver)와 Firmware 간의 물리적 통신 인터페이스를 추상화한다. 버스 타입(MIF, PCIe)에 관계없이 상위 레이어에 동일한 인터페이스를 제공한다.

### 1.2 설계 목표

- 버스 추상화: MIF(onchip)와 PCIe(discrete) 통합
- 상위 레이어 독립성: FW Message Layer가 버스 타입을 인식하지 않음
- 고성능 데이터 전송: Zero-copy, DMA 활용

### 1.3 지원 버스 타입

| 버스 타입 | 설명 | 사용 환경 |
|-----------|------|-----------|
| MIF (Memory Interface) | AP와 WiFi 칩이 동일 SoC에 있는 경우 | Mobile (onchip) |
| PCIe | 별도 WiFi 칩이 PCIe로 연결된 경우 | PC, Discrete chip |

---

## 2. 모듈 구성

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 6: HIP (Host Interface Protocol)                                        │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                            hip_core                                      │   │
│  │                     (공통 인터페이스, 버스 추상화)                          │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                     │                                           │
│                    ┌────────────────┴────────────────┐                         │
│                    ▼                                 ▼                         │
│            ┌─────────────┐                   ┌─────────────┐                   │
│            │   hip_mif   │                   │  hip_pcie   │                   │
│            │ (MIF 버스)   │                   │ (PCIe 버스) │                   │
│            └─────────────┘                   └─────────────┘                   │
│                    │                                 │                         │
│                    ▼                                 ▼                         │
│            ┌─────────────┐                   ┌─────────────┐                   │
│            │  MIF H/W    │                   │  PCIe H/W   │                   │
│            └─────────────┘                   └─────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 모듈 상세

### 3.1 hip_core

| 항목 | 내용 |
|------|------|
| **역할** | HIP 공통 인터페이스 및 버스 추상화 |
| **책임** | 버스 드라이버 등록, 메시지 송수신 API, 초기화/종료 |
| **의존성** | OSAL, hip_mif 또는 hip_pcie |

#### 핵심 자료 구조

```c
/* hip_core.h */

/**
 * 버스 타입
 */
enum hip_bus_type {
    HIP_BUS_MIF,
    HIP_BUS_PCIE,
};

/**
 * HIP 상태
 */
enum hip_state {
    HIP_STATE_STOPPED,
    HIP_STATE_STARTING,
    HIP_STATE_STARTED,
    HIP_STATE_STOPPING,
    HIP_STATE_ERROR,
};

/**
 * 버스 드라이버 인터페이스 (다형성)
 */
struct hip_bus_ops {
    /* 초기화/종료 */
    int (*init)(void *bus_priv);
    void (*deinit)(void *bus_priv);
    int (*start)(void *bus_priv);
    void (*stop)(void *bus_priv);
    
    /* 데이터 전송 */
    int (*send)(void *bus_priv, void *data, size_t len);
    int (*send_skb)(void *bus_priv, struct sk_buff *skb);
    
    /* FW 제어 */
    int (*fw_download)(void *bus_priv, const u8 *fw_data, size_t fw_len);
    int (*fw_boot)(void *bus_priv);
    int (*fw_reset)(void *bus_priv);
    
    /* 전력 관리 */
    int (*suspend)(void *bus_priv);
    int (*resume)(void *bus_priv);
    
    /* 상태 조회 */
    bool (*is_ready)(void *bus_priv);
    int (*get_fw_status)(void *bus_priv);
};

/**
 * HIP 컨텍스트
 */
struct hip_context {
    enum hip_bus_type bus_type;
    enum hip_state state;
    
    /* 버스 드라이버 */
    const struct hip_bus_ops *bus_ops;
    void *bus_priv;
    
    /* 수신 콜백 */
    hip_recv_callback_t recv_callback;
    void *recv_callback_ctx;
    
    /* 워크큐 */
    osal_workqueue_t *workqueue;
    
    /* 통계 */
    struct hip_stats {
        u64 tx_packets;
        u64 tx_bytes;
        u64 rx_packets;
        u64 rx_bytes;
        u64 tx_errors;
        u64 rx_errors;
    } stats;
    
    /* 동기화 */
    osal_mutex_t lock;
    osal_spinlock_t tx_lock;
};
```

#### 인터페이스

```c
/* hip_core.h */

/**
 * 수신 콜백 타입
 */
typedef void (*hip_recv_callback_t)(void *data, size_t len, void *ctx);

/**
 * 초기화/종료
 */
int hip_init(enum hip_bus_type bus_type, void *bus_priv);
void hip_deinit(void);

/**
 * 시작/중지
 */
int hip_start(void);
void hip_stop(void);

/**
 * 데이터 송신 (msg_router에서 호출)
 */
int hip_send(void *data, size_t len);
int hip_send_skb(struct sk_buff *skb);

/**
 * 수신 콜백 등록 (msg_router가 등록)
 */
void hip_register_recv_callback(hip_recv_callback_t callback, void *ctx);
void hip_unregister_recv_callback(void);

/**
 * FW 관리
 */
int hip_fw_download(const u8 *fw_data, size_t fw_len);
int hip_fw_boot(void);
int hip_fw_reset(void);

/**
 * 전력 관리
 */
int hip_suspend(void);
int hip_resume(void);

/**
 * 상태 조회
 */
enum hip_state hip_get_state(void);
bool hip_is_ready(void);
void hip_get_stats(struct hip_stats *stats);
```

#### 구현 예시

```c
/* hip_core.c */

static struct hip_context g_hip;

/**
 * 초기화
 */
int hip_init(enum hip_bus_type bus_type, void *bus_priv)
{
    int ret;
    
    memset(&g_hip, 0, sizeof(g_hip));
    
    g_hip.bus_type = bus_type;
    g_hip.bus_priv = bus_priv;
    g_hip.state = HIP_STATE_STOPPED;
    
    osal_mutex_init(&g_hip.lock);
    osal_spin_lock_init(&g_hip.tx_lock);
    
    /* 버스 타입에 따른 ops 설정 */
    switch (bus_type) {
    case HIP_BUS_MIF:
        g_hip.bus_ops = hip_mif_get_ops();
        break;
    case HIP_BUS_PCIE:
        g_hip.bus_ops = hip_pcie_get_ops();
        break;
    default:
        OSAL_ERR("Unknown bus type: %d\n", bus_type);
        return -EINVAL;
    }
    
    /* 워크큐 생성 */
    g_hip.workqueue = osal_workqueue_create_singlethread("hip_wq");
    if (!g_hip.workqueue)
        return -ENOMEM;
    
    /* 버스 드라이버 초기화 */
    ret = g_hip.bus_ops->init(g_hip.bus_priv);
    if (ret < 0) {
        OSAL_ERR("Bus init failed: %d\n", ret);
        osal_workqueue_destroy(g_hip.workqueue);
        return ret;
    }
    
    OSAL_INFO("HIP initialized: bus=%s\n", 
              bus_type == HIP_BUS_MIF ? "MIF" : "PCIe");
    
    return 0;
}

/**
 * 시작
 */
int hip_start(void)
{
    int ret;
    
    osal_mutex_lock(&g_hip.lock);
    
    if (g_hip.state != HIP_STATE_STOPPED) {
        osal_mutex_unlock(&g_hip.lock);
        return -EBUSY;
    }
    
    g_hip.state = HIP_STATE_STARTING;
    
    ret = g_hip.bus_ops->start(g_hip.bus_priv);
    if (ret < 0) {
        g_hip.state = HIP_STATE_ERROR;
        osal_mutex_unlock(&g_hip.lock);
        return ret;
    }
    
    g_hip.state = HIP_STATE_STARTED;
    
    osal_mutex_unlock(&g_hip.lock);
    
    OSAL_INFO("HIP started\n");
    return 0;
}

/**
 * 데이터 송신
 */
int hip_send(void *data, size_t len)
{
    unsigned long flags;
    int ret;
    
    if (g_hip.state != HIP_STATE_STARTED)
        return -ENODEV;
    
    osal_spin_lock_irqsave(&g_hip.tx_lock, &flags);
    
    ret = g_hip.bus_ops->send(g_hip.bus_priv, data, len);
    
    if (ret == 0) {
        g_hip.stats.tx_packets++;
        g_hip.stats.tx_bytes += len;
    } else {
        g_hip.stats.tx_errors++;
    }
    
    osal_spin_unlock_irqrestore(&g_hip.tx_lock, flags);
    
    return ret;
}

/**
 * 데이터 수신 (버스 드라이버에서 호출)
 * 주의: 인터럽트 컨텍스트에서 호출될 수 있음
 */
void hip_rx_handler(void *data, size_t len)
{
    g_hip.stats.rx_packets++;
    g_hip.stats.rx_bytes += len;
    
    if (g_hip.recv_callback)
        g_hip.recv_callback(data, len, g_hip.recv_callback_ctx);
}

/**
 * FW 다운로드
 */
int hip_fw_download(const u8 *fw_data, size_t fw_len)
{
    int ret;
    
    OSAL_INFO("Downloading FW: %zu bytes\n", fw_len);
    
    ret = g_hip.bus_ops->fw_download(g_hip.bus_priv, fw_data, fw_len);
    if (ret < 0) {
        OSAL_ERR("FW download failed: %d\n", ret);
        return ret;
    }
    
    OSAL_INFO("FW download complete\n");
    return 0;
}

/**
 * FW 부팅
 */
int hip_fw_boot(void)
{
    int ret;
    int timeout = 5000;  /* 5초 */
    
    OSAL_INFO("Booting FW...\n");
    
    ret = g_hip.bus_ops->fw_boot(g_hip.bus_priv);
    if (ret < 0)
        return ret;
    
    /* FW Ready 대기 */
    while (timeout > 0) {
        if (g_hip.bus_ops->is_ready(g_hip.bus_priv)) {
            OSAL_INFO("FW boot complete\n");
            return 0;
        }
        osal_msleep(100);
        timeout -= 100;
    }
    
    OSAL_ERR("FW boot timeout\n");
    return -ETIMEDOUT;
}
```

---

### 3.2 hip_mif (MIF Bus Driver)

| 항목 | 내용 |
|------|------|
| **역할** | MIF (Memory Interface) 버스 드라이버 |
| **책임** | Shared memory 통신, 인터럽트 처리, MIF 특화 제어 |
| **의존성** | OSAL, MIF H/W |

#### MIF 통신 구조

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  MIF Communication Architecture                                                 │
│                                                                                 │
│  ┌─────────────────────┐              ┌─────────────────────┐                  │
│  │      AP (Host)      │              │    WiFi Subsystem   │                  │
│  │                     │              │      (Firmware)     │                  │
│  │  ┌───────────────┐  │   Shared     │  ┌───────────────┐  │                  │
│  │  │   hip_mif     │  │   Memory     │  │   FW MIF      │  │                  │
│  │  │               │◀─┼──────────────┼─▶│               │  │                  │
│  │  └───────────────┘  │              │  └───────────────┘  │                  │
│  │         │           │              │         │           │                  │
│  │         │ IRQ       │              │         │ IRQ       │                  │
│  │         ▼           │              │         ▼           │                  │
│  │  ┌───────────────┐  │              │  ┌───────────────┐  │                  │
│  │  │  MIF Driver   │  │              │  │  MIF Driver   │  │                  │
│  │  └───────────────┘  │              │  └───────────────┘  │                  │
│  └─────────────────────┘              └─────────────────────┘                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 자료 구조

```c
/* hip_mif.h */

/**
 * MIF 링 버퍼 (공유 메모리)
 */
struct mif_ring {
    u32 write_idx;           /* Producer index */
    u32 read_idx;            /* Consumer index */
    u32 size;                /* Ring size */
    u8 *buf;                 /* Ring buffer */
};

/**
 * MIF 컨텍스트
 */
struct hip_mif_context {
    /* 레지스터 베이스 */
    void __iomem *reg_base;
    
    /* 공유 메모리 */
    void *shared_mem;
    size_t shared_mem_size;
    dma_addr_t shared_mem_dma;
    
    /* TX/RX 링 버퍼 */
    struct mif_ring tx_ring;
    struct mif_ring rx_ring;
    
    /* 인터럽트 */
    int irq;
    
    /* 상태 */
    bool fw_ready;
    
    /* 동기화 */
    osal_spinlock_t lock;
    
    /* 통계 */
    struct {
        u64 irq_count;
        u64 tx_full_count;
        u64 rx_empty_count;
    } stats;
};

/**
 * MIF 레지스터 오프셋
 */
#define MIF_REG_INT_STATUS      0x00
#define MIF_REG_INT_ENABLE      0x04
#define MIF_REG_INT_CLEAR       0x08
#define MIF_REG_TX_DOORBELL     0x10
#define MIF_REG_RX_DOORBELL     0x14
#define MIF_REG_FW_STATUS       0x20
#define MIF_REG_FW_BOOT         0x24

/**
 * 인터럽트 비트
 */
#define MIF_INT_TX_COMPLETE     BIT(0)
#define MIF_INT_RX_READY        BIT(1)
#define MIF_INT_FW_READY        BIT(2)
#define MIF_INT_FW_ERROR        BIT(3)
```

#### 인터페이스

```c
/* hip_mif.h */

/**
 * bus_ops 획득
 */
const struct hip_bus_ops *hip_mif_get_ops(void);

/**
 * 플랫폼 드라이버 등록 (probe에서 호출)
 */
int hip_mif_probe(struct platform_device *pdev);
int hip_mif_remove(struct platform_device *pdev);
```

#### 구현 예시

```c
/* hip_mif.c */

static struct hip_mif_context g_mif;

/**
 * 초기화
 */
static int mif_init(void *bus_priv)
{
    struct hip_mif_context *mif = bus_priv;
    int ret;
    
    osal_spin_lock_init(&mif->lock);
    
    /* 공유 메모리 할당 */
    mif->shared_mem_size = MIF_SHARED_MEM_SIZE;
    mif->shared_mem = osal_dma_alloc(mif->shared_mem_size, 
                                      &mif->shared_mem_dma);
    if (!mif->shared_mem)
        return -ENOMEM;
    
    /* 링 버퍼 초기화 */
    init_ring_buffers(mif);
    
    /* 인터럽트 등록 */
    ret = request_irq(mif->irq, mif_irq_handler, IRQF_SHARED,
                      "wifi_mif", mif);
    if (ret < 0) {
        osal_dma_free(mif->shared_mem, mif->shared_mem_size,
                      mif->shared_mem_dma);
        return ret;
    }
    
    return 0;
}

/**
 * 시작
 */
static int mif_start(void *bus_priv)
{
    struct hip_mif_context *mif = bus_priv;
    
    /* 인터럽트 활성화 */
    writel(MIF_INT_RX_READY | MIF_INT_FW_READY | MIF_INT_FW_ERROR,
           mif->reg_base + MIF_REG_INT_ENABLE);
    
    return 0;
}

/**
 * 데이터 전송
 */
static int mif_send(void *bus_priv, void *data, size_t len)
{
    struct hip_mif_context *mif = bus_priv;
    struct mif_ring *ring = &mif->tx_ring;
    unsigned long flags;
    u32 write_idx, next_write;
    
    osal_spin_lock_irqsave(&mif->lock, &flags);
    
    write_idx = ring->write_idx;
    next_write = (write_idx + 1) % ring->size;
    
    /* 링 버퍼 Full 검사 */
    if (next_write == ring->read_idx) {
        mif->stats.tx_full_count++;
        osal_spin_unlock_irqrestore(&mif->lock, flags);
        return -ENOSPC;
    }
    
    /* 데이터 복사 */
    memcpy(ring->buf + (write_idx * MIF_SLOT_SIZE), data, len);
    
    /* Write index 업데이트 */
    ring->write_idx = next_write;
    
    /* Doorbell 레지스터로 FW에 알림 */
    writel(1, mif->reg_base + MIF_REG_TX_DOORBELL);
    
    osal_spin_unlock_irqrestore(&mif->lock, flags);
    
    return 0;
}

/**
 * 인터럽트 핸들러
 */
static irqreturn_t mif_irq_handler(int irq, void *dev_id)
{
    struct hip_mif_context *mif = dev_id;
    u32 status;
    
    status = readl(mif->reg_base + MIF_REG_INT_STATUS);
    if (!status)
        return IRQ_NONE;
    
    /* 인터럽트 클리어 */
    writel(status, mif->reg_base + MIF_REG_INT_CLEAR);
    
    mif->stats.irq_count++;
    
    /* RX 처리 */
    if (status & MIF_INT_RX_READY)
        mif_rx_handler(mif);
    
    /* FW Ready */
    if (status & MIF_INT_FW_READY) {
        mif->fw_ready = true;
        OSAL_INFO("FW ready interrupt\n");
    }
    
    /* FW Error */
    if (status & MIF_INT_FW_ERROR) {
        OSAL_ERR("FW error interrupt\n");
        /* 에러 복구 처리 */
    }
    
    return IRQ_HANDLED;
}

/**
 * RX 처리
 */
static void mif_rx_handler(struct hip_mif_context *mif)
{
    struct mif_ring *ring = &mif->rx_ring;
    u32 read_idx, write_idx;
    void *data;
    size_t len;
    
    write_idx = ring->write_idx;  /* FW가 업데이트 */
    read_idx = ring->read_idx;
    
    while (read_idx != write_idx) {
        data = ring->buf + (read_idx * MIF_SLOT_SIZE);
        len = get_packet_len(data);  /* 패킷 길이 추출 */
        
        /* 상위 레이어로 전달 */
        hip_rx_handler(data, len);
        
        read_idx = (read_idx + 1) % ring->size;
    }
    
    ring->read_idx = read_idx;
    
    /* Doorbell로 FW에 알림 (공간 생김) */
    writel(1, mif->reg_base + MIF_REG_RX_DOORBELL);
}

/**
 * FW 다운로드
 */
static int mif_fw_download(void *bus_priv, const u8 *fw_data, size_t fw_len)
{
    struct hip_mif_context *mif = bus_priv;
    
    /* MIF는 보통 별도의 FW 다운로드 인터페이스 사용 */
    /* 예: PMU를 통한 다운로드 또는 별도 메모리 맵 */
    
    /* 구현은 플랫폼에 따라 다름 */
    return platform_specific_fw_download(fw_data, fw_len);
}

/**
 * bus_ops
 */
static const struct hip_bus_ops mif_bus_ops = {
    .init = mif_init,
    .deinit = mif_deinit,
    .start = mif_start,
    .stop = mif_stop,
    .send = mif_send,
    .send_skb = mif_send_skb,
    .fw_download = mif_fw_download,
    .fw_boot = mif_fw_boot,
    .fw_reset = mif_fw_reset,
    .suspend = mif_suspend,
    .resume = mif_resume,
    .is_ready = mif_is_ready,
    .get_fw_status = mif_get_fw_status,
};

const struct hip_bus_ops *hip_mif_get_ops(void)
{
    return &mif_bus_ops;
}
```

---

### 3.3 hip_pcie (PCIe Bus Driver)

| 항목 | 내용 |
|------|------|
| **역할** | PCIe 버스 드라이버 |
| **책임** | PCIe 통신, DMA 전송, MSI 인터럽트 처리 |
| **의존성** | OSAL, PCIe H/W |

#### PCIe 통신 구조

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PCIe Communication Architecture                                                │
│                                                                                 │
│  ┌─────────────────────┐                    ┌─────────────────────┐            │
│  │      Host CPU       │                    │    WiFi Chip        │            │
│  │                     │                    │                     │            │
│  │  ┌───────────────┐  │      PCIe         │  ┌───────────────┐  │            │
│  │  │   hip_pcie    │  │◀────────────────▶ │  │   FW PCIe     │  │            │
│  │  │               │  │                    │  │               │  │            │
│  │  └───────┬───────┘  │                    │  └───────────────┘  │            │
│  │          │          │                    │                     │            │
│  │          ▼          │                    │                     │            │
│  │  ┌───────────────┐  │                    │                     │            │
│  │  │  Host Memory  │  │                    │                     │            │
│  │  │               │  │    DMA Transfer    │                     │            │
│  │  │  TX Ring ─────┼──┼──────────────────▶ │                     │            │
│  │  │  RX Ring ◀────┼──┼────────────────────│                     │            │
│  │  │  Completion   │  │                    │                     │            │
│  │  └───────────────┘  │                    │                     │            │
│  └─────────────────────┘                    └─────────────────────┘            │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 자료 구조

```c
/* hip_pcie.h */

/**
 * PCIe 링 버퍼 디스크립터
 */
struct pcie_desc {
    u64 buf_addr;            /* DMA 주소 */
    u32 buf_len;             /* 버퍼 길이 */
    u32 flags;               /* 플래그 */
} __packed;

#define PCIE_DESC_FLAG_OWN      BIT(31)  /* 소유권: 1=FW, 0=Host */
#define PCIE_DESC_FLAG_EOP      BIT(30)  /* End of Packet */
#define PCIE_DESC_FLAG_SOP      BIT(29)  /* Start of Packet */

/**
 * PCIe 링 버퍼
 */
struct pcie_ring {
    struct pcie_desc *descs;     /* 디스크립터 배열 */
    dma_addr_t descs_dma;
    
    void **bufs;                 /* 버퍼 포인터 배열 */
    dma_addr_t *bufs_dma;
    
    u32 size;                    /* 링 크기 */
    u32 head;                    /* Host가 관리 */
    u32 tail;                    /* FW가 관리 (completion) */
    
    osal_spinlock_t lock;
};

/**
 * PCIe 컨텍스트
 */
struct hip_pcie_context {
    struct pci_dev *pdev;
    
    /* BAR 매핑 */
    void __iomem *bar0;          /* 레지스터 */
    void __iomem *bar2;          /* 메모리 (옵션) */
    
    /* TX/RX 링 */
    struct pcie_ring tx_ring;
    struct pcie_ring rx_ring;
    
    /* MSI 인터럽트 */
    int msi_vectors;
    
    /* NAPI */
    struct napi_struct napi;
    
    /* 상태 */
    bool fw_ready;
    bool link_up;
    
    /* 통계 */
    struct {
        u64 irq_count;
        u64 tx_dma_errors;
        u64 rx_dma_errors;
    } stats;
};

/**
 * PCIe 레지스터 오프셋
 */
#define PCIE_REG_INT_STATUS     0x00
#define PCIE_REG_INT_ENABLE     0x04
#define PCIE_REG_TX_HEAD        0x10
#define PCIE_REG_TX_TAIL        0x14
#define PCIE_REG_RX_HEAD        0x20
#define PCIE_REG_RX_TAIL        0x24
#define PCIE_REG_FW_STATUS      0x30
#define PCIE_REG_FW_BOOT        0x34
```

#### 구현 예시

```c
/* hip_pcie.c */

/**
 * 초기화
 */
static int pcie_init(void *bus_priv)
{
    struct hip_pcie_context *pcie = bus_priv;
    int ret;
    
    /* PCI 활성화 */
    ret = pci_enable_device(pcie->pdev);
    if (ret < 0)
        return ret;
    
    /* Bus master 활성화 */
    pci_set_master(pcie->pdev);
    
    /* DMA 마스크 설정 */
    ret = dma_set_mask_and_coherent(&pcie->pdev->dev, DMA_BIT_MASK(64));
    if (ret) {
        ret = dma_set_mask_and_coherent(&pcie->pdev->dev, DMA_BIT_MASK(32));
        if (ret)
            goto err_disable;
    }
    
    /* BAR 매핑 */
    pcie->bar0 = pci_iomap(pcie->pdev, 0, 0);
    if (!pcie->bar0) {
        ret = -ENOMEM;
        goto err_disable;
    }
    
    /* MSI 활성화 */
    ret = pci_alloc_irq_vectors(pcie->pdev, 1, 4, PCI_IRQ_MSI);
    if (ret < 0)
        goto err_unmap;
    pcie->msi_vectors = ret;
    
    /* 인터럽트 핸들러 등록 */
    ret = request_irq(pci_irq_vector(pcie->pdev, 0), pcie_irq_handler,
                      0, "wifi_pcie", pcie);
    if (ret < 0)
        goto err_free_vectors;
    
    /* TX/RX 링 버퍼 할당 */
    ret = alloc_rings(pcie);
    if (ret < 0)
        goto err_free_irq;
    
    /* NAPI 초기화 */
    netif_napi_add(&pcie->dummy_ndev, &pcie->napi, pcie_napi_poll, 64);
    
    return 0;
    
err_free_irq:
    free_irq(pci_irq_vector(pcie->pdev, 0), pcie);
err_free_vectors:
    pci_free_irq_vectors(pcie->pdev);
err_unmap:
    pci_iounmap(pcie->pdev, pcie->bar0);
err_disable:
    pci_disable_device(pcie->pdev);
    return ret;
}

/**
 * 데이터 전송 (Zero-copy)
 */
static int pcie_send_skb(void *bus_priv, struct sk_buff *skb)
{
    struct hip_pcie_context *pcie = bus_priv;
    struct pcie_ring *ring = &pcie->tx_ring;
    struct pcie_desc *desc;
    dma_addr_t dma_addr;
    unsigned long flags;
    u32 head, next_head;
    
    /* DMA 매핑 */
    dma_addr = dma_map_single(&pcie->pdev->dev, skb->data, skb->len,
                               DMA_TO_DEVICE);
    if (dma_mapping_error(&pcie->pdev->dev, dma_addr)) {
        pcie->stats.tx_dma_errors++;
        return -EIO;
    }
    
    osal_spin_lock_irqsave(&ring->lock, &flags);
    
    head = ring->head;
    next_head = (head + 1) % ring->size;
    
    /* 링 Full 검사 */
    if (next_head == ring->tail) {
        osal_spin_unlock_irqrestore(&ring->lock, flags);
        dma_unmap_single(&pcie->pdev->dev, dma_addr, skb->len, DMA_TO_DEVICE);
        return -ENOSPC;
    }
    
    /* 디스크립터 설정 */
    desc = &ring->descs[head];
    desc->buf_addr = dma_addr;
    desc->buf_len = skb->len;
    desc->flags = PCIE_DESC_FLAG_OWN | PCIE_DESC_FLAG_SOP | PCIE_DESC_FLAG_EOP;
    
    /* skb 저장 (TX complete에서 해제용) */
    ring->bufs[head] = skb;
    ring->bufs_dma[head] = dma_addr;
    
    /* Head 업데이트 */
    ring->head = next_head;
    
    /* Doorbell: FW에 알림 */
    writel(next_head, pcie->bar0 + PCIE_REG_TX_HEAD);
    
    osal_spin_unlock_irqrestore(&ring->lock, flags);
    
    return 0;
}

/**
 * 인터럽트 핸들러
 */
static irqreturn_t pcie_irq_handler(int irq, void *dev_id)
{
    struct hip_pcie_context *pcie = dev_id;
    u32 status;
    
    status = readl(pcie->bar0 + PCIE_REG_INT_STATUS);
    if (!status)
        return IRQ_NONE;
    
    /* 인터럽트 비활성화 */
    writel(0, pcie->bar0 + PCIE_REG_INT_ENABLE);
    
    pcie->stats.irq_count++;
    
    /* NAPI 스케줄 */
    napi_schedule(&pcie->napi);
    
    return IRQ_HANDLED;
}

/**
 * NAPI 폴링
 */
static int pcie_napi_poll(struct napi_struct *napi, int budget)
{
    struct hip_pcie_context *pcie = container_of(napi, struct hip_pcie_context, napi);
    int work_done = 0;
    
    /* TX Complete 처리 */
    pcie_tx_complete(pcie);
    
    /* RX 처리 */
    work_done = pcie_rx_process(pcie, budget);
    
    if (work_done < budget) {
        napi_complete(napi);
        /* 인터럽트 재활성화 */
        writel(0xFFFFFFFF, pcie->bar0 + PCIE_REG_INT_ENABLE);
    }
    
    return work_done;
}

/**
 * TX Complete 처리
 */
static void pcie_tx_complete(struct hip_pcie_context *pcie)
{
    struct pcie_ring *ring = &pcie->tx_ring;
    u32 tail, old_tail;
    
    /* FW가 업데이트한 tail 읽기 */
    tail = readl(pcie->bar0 + PCIE_REG_TX_TAIL);
    old_tail = ring->tail;
    
    while (old_tail != tail) {
        struct sk_buff *skb = ring->bufs[old_tail];
        dma_addr_t dma_addr = ring->bufs_dma[old_tail];
        
        /* DMA 언매핑 */
        dma_unmap_single(&pcie->pdev->dev, dma_addr, skb->len, DMA_TO_DEVICE);
        
        /* skb 해제 */
        dev_kfree_skb_any(skb);
        
        ring->bufs[old_tail] = NULL;
        old_tail = (old_tail + 1) % ring->size;
    }
    
    ring->tail = tail;
}

/**
 * RX 처리
 */
static int pcie_rx_process(struct hip_pcie_context *pcie, int budget)
{
    struct pcie_ring *ring = &pcie->rx_ring;
    int processed = 0;
    u32 head;
    
    head = ring->head;
    
    while (processed < budget) {
        struct pcie_desc *desc = &ring->descs[head];
        struct sk_buff *skb;
        void *buf;
        u32 len;
        
        /* 소유권 검사 */
        if (desc->flags & PCIE_DESC_FLAG_OWN)
            break;
        
        len = desc->buf_len;
        buf = ring->bufs[head];
        
        /* DMA 동기화 */
        dma_sync_single_for_cpu(&pcie->pdev->dev, ring->bufs_dma[head],
                                 len, DMA_FROM_DEVICE);
        
        /* 상위 레이어로 전달 */
        hip_rx_handler(buf, len);
        
        /* 새 버퍼 할당하고 디스크립터 재설정 */
        refill_rx_buffer(pcie, head);
        
        head = (head + 1) % ring->size;
        processed++;
    }
    
    ring->head = head;
    
    /* FW에 새로운 버퍼 알림 */
    writel(head, pcie->bar0 + PCIE_REG_RX_HEAD);
    
    return processed;
}

/**
 * FW 다운로드 (BAR를 통한 직접 쓰기)
 */
static int pcie_fw_download(void *bus_priv, const u8 *fw_data, size_t fw_len)
{
    struct hip_pcie_context *pcie = bus_priv;
    u32 offset = 0;
    
    OSAL_INFO("PCIe FW download: %zu bytes\n", fw_len);
    
    /* FW 메모리 영역에 직접 쓰기 */
    while (offset < fw_len) {
        size_t chunk = min(fw_len - offset, (size_t)4096);
        memcpy_toio(pcie->bar2 + FW_MEM_OFFSET + offset, 
                    fw_data + offset, chunk);
        offset += chunk;
    }
    
    return 0;
}

/**
 * bus_ops
 */
static const struct hip_bus_ops pcie_bus_ops = {
    .init = pcie_init,
    .deinit = pcie_deinit,
    .start = pcie_start,
    .stop = pcie_stop,
    .send = pcie_send,
    .send_skb = pcie_send_skb,
    .fw_download = pcie_fw_download,
    .fw_boot = pcie_fw_boot,
    .fw_reset = pcie_fw_reset,
    .suspend = pcie_suspend,
    .resume = pcie_resume,
    .is_ready = pcie_is_ready,
    .get_fw_status = pcie_get_fw_status,
};

const struct hip_bus_ops *hip_pcie_get_ops(void)
{
    return &pcie_bus_ops;
}
```

---

## 4. 디렉토리 구조

```
hip/
├── hip_core.c           # 공통 인터페이스
├── hip_core.h
│
├── hip_mif.c            # MIF 버스 드라이버
├── hip_mif.h
├── hip_mif_platform.c   # MIF 플랫폼 드라이버
│
├── hip_pcie.c           # PCIe 버스 드라이버
├── hip_pcie.h
└── hip_pcie_ids.h       # PCIe Vendor/Device ID
```

---

## 5. 주목할 특성

### 5.1 버스 추상화 (다형성)

```c
/* hip_core가 버스 타입을 추상화 */

struct hip_bus_ops {
    int (*init)(void *bus_priv);
    int (*send)(void *bus_priv, void *data, size_t len);
    /* ... */
};

/* 상위 레이어(msg_router)는 버스 타입을 모름 */
int hip_send(void *data, size_t len)
{
    /* 실제 버스 드라이버로 위임 */
    return g_hip.bus_ops->send(g_hip.bus_priv, data, len);
}
```

### 5.2 MIF vs PCIe 비교

| 특성 | MIF | PCIe |
|------|-----|------|
| **연결 방식** | Shared Memory | DMA Ring Buffer |
| **인터럽트** | Platform IRQ | MSI/MSI-X |
| **데이터 복사** | memcpy to shared mem | Zero-copy DMA |
| **지연 시간** | 낮음 (onchip) | 상대적으로 높음 |
| **처리량** | 메모리 대역폭 제한 | PCIe 대역폭 (Gen3: 8GT/s) |
| **전력** | 낮음 | 상대적으로 높음 |
| **FW 다운로드** | PMU 또는 별도 인터페이스 | BAR 직접 쓰기 |

### 5.3 흐름 제어

```c
/* TX Flow Control */

/* 1. 버스 드라이버에서 버퍼 부족 감지 */
if (next_head == ring->tail) {
    /* FW Message Layer에 알림 */
    msg_router_flow_control(true);  /* stop */
    return -ENOSPC;
}

/* 2. TX Complete 후 공간 생기면 */
pcie_tx_complete() {
    if (ring_space_available(ring) > threshold) {
        msg_router_flow_control(false);  /* resume */
    }
}

/* 3. netdev 레벨까지 전파 */
ma_flow_ctrl_handler() {
    if (stop)
        netdev_if_stop_queue(vif->ndev);
    else
        netdev_if_wake_queue(vif->ndev);
}
```

### 5.4 전력 관리

```c
/* Suspend */
int hip_suspend(void)
{
    /* 1. 진행 중인 TX 완료 대기 */
    wait_for_tx_complete();
    
    /* 2. 버스 드라이버 suspend */
    g_hip.bus_ops->suspend(g_hip.bus_priv);
    
    /* 3. 상태 업데이트 */
    g_hip.state = HIP_STATE_STOPPED;
    
    return 0;
}

/* Resume */
int hip_resume(void)
{
    int ret;
    
    /* 1. 버스 드라이버 resume */
    ret = g_hip.bus_ops->resume(g_hip.bus_priv);
    if (ret < 0)
        return ret;
    
    /* 2. FW 상태 확인 */
    if (!g_hip.bus_ops->is_ready(g_hip.bus_priv)) {
        /* FW 재시작 필요 */
        ret = hip_fw_boot();
        if (ret < 0)
            return ret;
    }
    
    /* 3. 상태 업데이트 */
    g_hip.state = HIP_STATE_STARTED;
    
    return 0;
}
```

### 5.5 의존성 규칙

```
FW Message Layer (Layer 5)
         │
         │ hip_send(), hip_recv_callback
         ▼
┌─────────────────────────────────────────────────────┐
│  HIP (Layer 6)                                      │
│                                                     │
│  hip_core ◀────▶ hip_mif 또는 hip_pcie            │
│     │                     │                         │
│     │                     │                         │
│     ▼                     ▼                         │
│  OSAL              Platform/PCI Driver             │
└─────────────────────────────────────────────────────┘
                           │
                           ▼
                      Hardware

규칙:
- 상위 레이어는 hip_core 인터페이스만 사용
- hip_mif/hip_pcie는 외부에 노출되지 않음
- 버스 드라이버는 hip_bus_ops 인터페이스 구현
```

### 5.6 에러 처리

```c
/* 링크 에러 감지 */
static void pcie_check_link(struct hip_pcie_context *pcie)
{
    u16 link_status;
    
    pcie_capability_read_word(pcie->pdev, PCI_EXP_LNKSTA, &link_status);
    
    if (!(link_status & PCI_EXP_LNKSTA_DLLLA)) {
        OSAL_ERR("PCIe link down!\n");
        pcie->link_up = false;
        
        /* 상위 레이어에 알림 */
        hip_notify_error(HIP_ERROR_LINK_DOWN);
    }
}

/* FW 에러 처리 */
static void handle_fw_error(struct hip_pcie_context *pcie)
{
    u32 fw_status = readl(pcie->bar0 + PCIE_REG_FW_STATUS);
    
    OSAL_ERR("FW error: status=0x%08x\n", fw_status);
    
    /* FW 덤프 트리거 */
    msg_debug_trigger_dump();
    
    /* 복구 시도 */
    hip_fw_reset();
}
```

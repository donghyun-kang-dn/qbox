# [V200] ADD-SOL: Virtualization - 기술 검토 리포트

**검토 대상:** `[V200] ADD-SOL: Virtualization` v0.3 (2026-03-24)  
**작성자:** 강동현  
**검토일:** 2026-03-24  

---

## 1. 문서 개요

본 문서는 V200 SoC의 가상화 아키텍처 솔루션 문서(ADD-SOL: Virtualization v0.3)에서 제안한 **"Fault-less Restricted SVA"** 메커니즘에 대한 기술 검토를 수행한 결과입니다.

### 핵심 제안 요약

| 항목 | 내용 |
|------|------|
| **가상화 기반** | S-IOV (Scalable I/O Virtualization) 기반 다중 커맨드 큐 |
| **주소 변환** | PASID 기반 Shared Virtual Addressing (Fault-less Restricted SVA) |
| **TLB 구조** | Global-TLB (Central-SATU) + Per-engine local-TLB (Local-SATU) |
| **메모리 관리** | 세그먼트 + 정적 TLB, Pinned memory 기반, 동적 PTW 배제 |
| **격리 수준** | 기능 격리 O, 성능 격리 X (의도적 미지원) |
| **P2P 호환성** | 단일 UVA 포인터로 호스트/디바이스/P2P 접근 동일 보장 |

---

## 2. v0.3 주요 변경점 분석: NSID → CTX_ID 전환

### 2.1 변경 배경

v0.2까지 사용되던 **NSID** 개념은 디바이스 내부 관리에는 적합했으나, BAR mapped memory를 통한 **P2P(Peer-to-Peer) 환경**에서 target device로 활용되는 시나리오와 대응되지 않는 문제가 발견되었습니다.

### 2.2 긍정적 평가

- **CTX_ID ↔ PASID 1:1 매핑**은 표준 PCIe 가상화 체계와 자연스럽게 통합됩니다. NSID처럼 별도의 매핑 레이어를 추가하지 않으므로, 호스트 소프트웨어 스택(IOMMU, 드라이버)과의 인터페이스가 단순해집니다.
- `v200_bind_context()` API를 통해 CTX_ID를 부여하는 방식은 보안 측면에서도 견고합니다. VM 내 유저가 임의의 CTX_ID를 날조할 수 없도록 하드웨어 + VDCM이 담당하는 설계가 적절합니다.

### 2.3 우려 사항

- **CTX_ID 재사용 정책 미정의:** 컨텍스트가 해제된 후 CTX_ID를 재사용할 때, 기존 TLB 엔트리의 무효화(invalidation) 순서와 타이밍에 대한 명시적 프로토콜이 문서에 없습니다. Race condition이 발생할 수 있는 경계 조건입니다.
- **CTX_ID 상한이 SQ Group 수에 종속:** 최대 active context 수가 SQ Group 수로 bound되는 것은 시스템 규모 확장에 제약이 될 수 있습니다. SQ Group 수가 하드웨어 리소스(전용 SRAM 크기)에 의해 결정되므로, 향후 확장성에 대한 trade-off 분석이 필요합니다.

---

## 3. Fault-less Restricted SVA 메커니즘 평가

### 3.1 설계 철학 평가

표준 SVA(ATS/PRI)와의 비교 테이블이 문서에 잘 정리되어 있습니다. 이 접근법의 핵심 가치는 다음과 같습니다:

| 강점 | 평가 |
|------|------|
| **결정론적 성능** | 동적 PTW와 TLB Miss로 인한 레이턴시 스파이크를 원천 차단. 가속기 워크로드에서 **1-Cycle 주소 변환**은 매우 큰 장점 |
| **하드웨어 복잡도 감소** | ATS/ATC/PRI 미구현으로 실리콘 면적 절감 효과가 큼 |
| **P2P 주소 동일성** | `get_acc_pa(CTX_ID, UVA) == get_host_pa(...)` 보장은 프로그래밍 모델을 크게 단순화 |

이 선택은 **가속기 특화 SoC**라는 V200의 포지셔닝에 적합합니다. 범용 GPU처럼 임의의 유저 메모리에 on-demand로 접근해야 하는 워크로드가 아니므로, Pinned memory 기반 운용은 합리적입니다.

### 3.2 세그먼트 기반 주소 변환 상세 검토

```
segment_id, segment_va = find_segment(VA)
page_size, segment_ptr = segment_info[segment_id]
VPN = segment_va // page_size
offset = segment_va % page_size
PPN = TLB[segment_ptr + VPN]
accelerator_physical_address = PPN * page_size + offset
```

**검토 의견:**

1. **세그먼트 수 (8 또는 16):** 문서에서 8개 또는 16개를 고려 중이라고 밝혔습니다. 주소 비교 로직의 critical path를 고려할 때, 8개가 timing closure에 유리할 것입니다. 단, DB 워크로드의 다양한 메모리 영역(인덱스, 데이터, 메타데이터, 임시 버퍼 등)을 고려하면 8개로는 부족할 수 있습니다. **권장: 16개로 설계하되, 비교기를 병렬 파이프라인으로 구현하여 timing 확보.**

2. **정적 vs 동적 세그먼트 엔트리 할당:** 문서에서 `8 × 256 = 2048` 또는 `16 × 128` 등의 정적 할당을 언급했습니다. 정적 할당은 구현이 단순하고 timing이 예측 가능하지만, 특정 세그먼트에 대규모 매핑이 집중되는 불균형 시나리오에 취약합니다. **권장: 정적 할당 기본, 세그먼트 체이닝(문서에서 이미 언급)을 통한 유연성 확보를 병행.**

3. **`find_segment(VA)` 의 구현:** 8~16개 세그먼트의 base address를 동시에 비교하는 CAM 로직이 필요합니다. 각 세그먼트의 크기가 가변적이므로, base + size range 비교기가 되어야 합니다. 이 로직이 1 cycle에 수행되어야 한다면, 세그먼트 수가 critical path를 결정합니다.

### 3.3 페이지 승격(Promotion) 메커니즘

**Temporal VA(TVA)** 를 활용한 승격 절차가 제안되어 있습니다:

```
Host driver → 거대 페이지 후보 PPN 탐색
            → Admin 권한의 migration 명령 전달
FW → TVA → New PA 매핑 등록
   → DMA로 데이터 복사 (VA → TVA)
   → TVA 파기, 기존 매핑 파기
   → VA → New PA 새 매핑 등록
```

**검토 의견:**

1. **Quiescing 범위:** 문서에서 "해당 CTX_ID에 대한 명령은 수행될 수 없어야 함"이라고 명시했습니다. 이는 올바른 요구사항이나, **Quiescing의 구체적인 프로토콜**(drain 완료 신호, 타임아웃 처리 등)이 정의되어야 합니다.

2. **TVA 영역 충돌 방지:** TVA가 기존 VA 공간과 겹치지 않도록 보장하는 메커니즘이 문서에 명시되어야 합니다. "별도의 등록 규칙이 존재해야 함 (privileged)"이라고만 언급되어 있어, 구체적인 TVA 주소 범위 관리 정책이 필요합니다.

3. **승격 실패 시 롤백:** DMA 중간에 에러가 발생하면 어떻게 되는지에 대한 에러 핸들링이 문서에 없습니다. 부분 복사 상태에서의 데이터 일관성 보장 방안이 필요합니다.

---

## 4. Global-TLB + Local-TLB 아키텍처 평가

### 4.1 구조

- **Central-SATU (Global TLB):** 시스템 단일, 모든 활성 SQ Group의 페이지 매핑 정보 관리
- **Local-SATU:** 각 엔진 별 부착, 현재 실행 중인 CTX_ID의 매핑 정보 캐시

### 4.2 긍정적 평가

- **Context preload (ping-pong 운용):** 엔진 별 Command FIFO 기반으로, 다음 CTX_ID의 TLB를 미리 로드하는 것은 context switch 오버헤드를 효과적으로 숨깁니다. 이는 GPU의 warp scheduling과 유사한 접근으로, 가속기 활용률을 높이는 좋은 설계입니다.
- **동적 PTW 배제:** "static한 behavior 보장"이라는 원칙은 가속기에서 중요한 결정론적 성능의 핵심입니다.

### 4.3 우려 사항 및 개선 권고

1. **Global TLB 저장 위치:** 문서에서 SPM vs 전용 모듈을 비교하고, "전용 모듈 구현 방향"이라고 결정했습니다. 이 경우 약 1MiB(128 × 8KiB) 수준의 SRAM이 필요합니다. 전용 모듈의 SRAM이 SQ Group 최대 수를 직접 제한하므로, **향후 제품 라인업별로 SRAM 크기를 구성 가능하게(configurable) 설계**하는 것을 권장합니다.

2. **Local-SATU의 크기 및 갱신 정책:** Local TLB의 엔트리 수, 갱신(preload) 시 latency, 그리고 preload 실패 시의 fallback 경로가 문서에 정의되어 있지 않습니다. 이는 후속 Functional Specification에서 반드시 다루어야 할 항목입니다.

3. **TLB Invalidation 브로드캐스트:** `v200_device_alloc()`이나 승격(promotion) 시 매핑이 변경되면, Central-SATU 뿐 아니라 해당 CTX_ID의 매핑을 캐시하고 있는 모든 Local-SATU에도 invalidation이 전파되어야 합니다. 이 브로드캐스트 메커니즘과 소요 cycle이 명시되어야 합니다.

---

## 5. S-IOV 기반 Command Queue 아키텍처 평가

### 5.1 SQ Group 개념

- SQ Group = CQ 1개 + SQ 1~32개를 동시에 생성
- NVMe 표준의 CQ 생성 → SQ 생성 흐름을 단순화
- 동일 PASID를 공유하므로 메모리 격리 단위와 일치

**평가:** NVMe 스타일을 기반으로 하면서도 가속기 특성에 맞게 단순화한 것은 좋은 설계 결정입니다. 다만 아래 사항들의 보완이 필요합니다:

### 5.2 검토 사항

1. **SQ 간 우선순위/QoS:** 동일 SQ Group 내 32개 SQ 간의 스케줄링 정책이 문서에 없습니다. Round-Robin인지, 우선순위 기반인지, 또는 weighted fair queuing인지 정의가 필요합니다.

2. **Queue Overflow 처리:** SQ가 가득 찼을 때의 back-pressure 메커니즘이 명시되어야 합니다. 도어벨 기반 제출에서, SQ full 상태의 감지를 유저 공간에서 어떻게 처리하는지(polling vs. event) 정의 필요.

3. **FW용 opcode set:** "세부 명령은 추가 필드(sw-defined)를 통해 결정"이라고 되어 있으나, FW SQ와 일반 SQ 간의 우선순위 관계, FW 명령의 latency 보장 등이 아직 미정의입니다.

---

## 6. VM Management (VDCM) 평가

### 6.1 긍정적 측면

- **mdev 기반 할당:** Linux `mediated device` 프레임워크를 활용하는 것은 기존 에코시스템과의 호환성이 좋습니다.
- **제어 평면/데이터 평면 분리:** BAR01의 0x0~0x8000 영역을 하이퍼바이저 전용으로 격리하는 설계는 보안 모델로서 적절합니다.

### 6.2 우려 사항

1. **Admin 명령의 Native 동작 요구:** "Admin 명령은 VM 환경에서도 native하게 동작할 수 있도록 구성되어야 함"은 중요한 요구사항이나, 구체적으로 어떤 Admin 명령들이 있고, 각각이 VM 환경에서 어떻게 변환되는지의 목록이 필요합니다.

2. **FLR 안전성:** Queue abort → delete queues → delete namespaces 흐름은 올바르나, **abort 중 진행 중인 DMA 트랜잭션의 완료 대기(drain) 타임아웃**이 정의되어야 합니다. 무한 대기 방지를 위한 watchdog 메커니즘이 필요합니다.

3. **보안 이슈 미해결:** 문서의 Target SVA 관련 보안 이슈 항목에 "?"로만 표기되어 있습니다. BAR를 통해 들어오는 외부 접근에 대한 보안 검증(특히 P2P 시나리오에서 다른 디바이스가 잘못된 주소로 접근하는 경우)에 대한 방어 메커니즘이 반드시 정의되어야 합니다.

---

## 7. API 설계 평가

### 7.1 `v200_device_alloc()` 흐름

`ioctl(ALLOC)` → `mmap()` → 드라이버 `.mmap` 콜백에서 UVA 확정 → ASQ를 통해 HW 동기화하는 흐름은 Linux 커널의 메모리 관리 체계를 잘 활용한 설계입니다.

**우려:**
- `mmap()` 과 ASQ 명령 사이의 **원자성(atomicity):** mmap은 성공했으나 ASQ 명령이 실패하면 불일치 상태가 됩니다. 이 경우의 롤백 절차가 필요합니다.
- 멀티스레드 환경에서 동일 context에 대한 동시 `v200_device_alloc()` 호출 시의 직렬화 메커니즘이 명시되어야 합니다.

### 7.2 `v200_host_register()`

Pinned memory 전략은 Fault-less 보장의 핵심입니다. 다만:
- **호스트 메모리 부족 시의 graceful degradation** 정책이 필요합니다. 과도한 pinning은 호스트 OS 전체의 메모리 관리에 영향을 줄 수 있습니다.
- `pin_user_pages_fast()` 실패 시의 에러 리포팅 경로가 정의되어야 합니다.

---

## 8. 코드베이스(Qbox) 관점에서의 통합 검토

현재 코드베이스는 **Qbox** — SystemC/TLM-2.0 기반 가상 플랫폼 시뮬레이션 프레임워크입니다. 이 프로젝트는 QEMU를 C++로 래핑하여 SystemC 모델로 노출하는 구조로, V200 SoC의 가상화 아키텍처를 시뮬레이션/검증하기 위한 기반 인프라로 활용될 수 있습니다.

### 통합 시 고려 사항

1. **`qemu-components/`에 V200 VDPU 모델 추가 필요:** S-IOV 기반의 다중 커맨드 큐, SATU 주소 변환, SQ Group 디스패칭 등의 동작을 시뮬레이션하기 위한 TLM 모델 구현이 필요합니다.

2. **NVMe 컴포넌트 재활용:** 기존 `qemu-components/` 내 NVMe 관련 코드가 존재하며, SQ/CQ 기반 커맨드 인터페이스가 NVMe 스타일이므로 이를 기반으로 확장할 수 있습니다.

3. **IOMMU/PASID 시뮬레이션:** 현재 Qbox에서 IOMMU 관련 시뮬레이션 지원이 제한적일 수 있으므로, PASID 기반 주소 변환의 시뮬레이션을 위한 추가 모델 개발이 필요할 수 있습니다.

---

## 9. 종합 평가 및 권고 사항

### 9.1 종합 평가

| 영역 | 평가 | 등급 |
|------|------|------|
| **아키텍처 방향성** | S-IOV + Fault-less Restricted SVA는 가속기 특화 SoC에 적합한 pragmatic 접근 | **적합** |
| **주소 변환 설계** | 세그먼트 + 정적 TLB 기반의 1-cycle 변환은 결정론적 성능 보장에 효과적 | **적합** |
| **P2P 호환성** | 단일 UVA 동일성 보장은 프로그래밍 모델의 핵심 가치 | **우수** |
| **보안 모델** | CTX_ID 바인딩, BAR 격리는 적절하나, Target 접근 보안이 미정의 | **보완 필요** |
| **승격 메커니즘** | TVA 기반 접근은 창의적이나, 에러 핸들링/롤백이 미정의 | **보완 필요** |
| **문서 완성도** | 핵심 개념은 잘 전달되나, 일부 critical path의 상세 사양 부재 | **보완 필요** |

### 9.2 핵심 권고 사항 (우선순위순)

1. **[Critical]** Target SVA 보안 모델 정의 — BAR를 통한 외부 접근에 대한 방어 메커니즘 구체화
2. **[Critical]** CTX_ID 재사용 시 TLB invalidation 프로토콜 정의
3. **[High]** 승격(Promotion) 실패 시 롤백 절차 및 에러 핸들링 정의
4. **[High]** Local-SATU invalidation 브로드캐스트 메커니즘 명세
5. **[High]** FLR drain 타임아웃 및 watchdog 정의
6. **[Medium]** SQ 간 스케줄링 정책 정의
7. **[Medium]** `v200_device_alloc()` 원자성 보장 방안
8. **[Low]** 세그먼트 수 최종 결정 (16개 권장) 및 비교기 구현 방안

### 9.3 결론

V200의 **Fault-less Restricted SVA** 제안은 가속기 특화 SoC에 맞는 실용적이고 효율적인 가상화 메커니즘입니다. 표준 SVA의 무거운 하드웨어(ATS/ATC/PRI)를 배제하면서도, PASID 기반 주소 공간 격리와 단일 UVA 동일성이라는 핵심 가치를 달성하는 접근은 기술적으로 타당합니다.

다만, v0.3 문서는 아직 Draft 상태로, 위에서 지적한 보안 모델, 에러 핸들링, invalidation 프로토콜 등의 상세 정의가 후속 버전에서 반드시 보완되어야 합니다. 특히 **Target 접근 보안**과 **CTX_ID 재사용 프로토콜**은 실리콘 결함으로 이어질 수 있는 critical path이므로, Functional Specification 단계 진입 전에 해결되어야 합니다.

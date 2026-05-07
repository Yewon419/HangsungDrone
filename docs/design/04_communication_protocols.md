# 04. 통신 프로토콜 디테일 + 시간 sync (D)

> 작성일: 2026-05-07
> 단계: D. 통신 프로토콜 디테일 (락) — E(장애·재시작 시나리오) 진입 전 기준선
> 관련: `01_system_architecture.md`, `02_interface_catalog.md`, `03_data_flow.md`

---

## 0. 목적

B의 통신 방식(REST·WebSocket·gRPC·RTSP·Crazyradio) 위에, **운영 디테일** 정의:
- 시간 동기화
- Crazyradio 채널 관리
- HTTPS 인증·권한
- WebSocket 재연결·하트비트
- (메모) 메시지 직렬화, TLS·암호화

---

## 1. 시간 동기화

| # | 결정 | 락 |
|---|---|---|
| 1a | 프로토콜 | **Skybrush 자체 sync** (Crazyradio 통해 시각 분배). 마스터 시계 백업으로 NTP |
| 1b | 정밀도 목표 | **1-10ms** |

### 근거
- 융합형 안무(정적 → 동적 전환) 시 30대 동시 트리거 필수
- 50ms 어긋남 = 1m/s 비행 시 5cm — 시각적으로 안 맞음
- Skybrush는 Crazyradio sync로 기본 5-10ms 달성 (검증된 표준)
- PTP는 하드웨어 지원 필요해 부담 큼, NTP는 정밀도 부족

### 마스터 시계
- **field-controller PC 시각이 마스터**
- 외부 NTP 동기화 가능할 때 주기적으로 PC 시각 갱신
- 박람회장 인터넷 가용 가정 (가용 안 되면 PC 시각만으로 운용)

---

## 2. Crazyradio 채널 관리

| # | 결정 | 락 |
|---|---|---|
| 2a | 동글 수 | **3-4개** |
| 2b | 채널 분산 전략 | **정적** |
| 2c | 사전 RF 스캔 | **셋업 데이 측정** |

### 운영
- 드론 8-10대를 동글 1개에 그룹화 (Crazyflie 표준 권장)
- 시연 ID당 채널 풀 사전 할당 (예: 채널 [10, 30, 50, 70])
- 셋업 데이에 RF 스캔 도구로 박람회장 측정 → 충돌 채널 회피
- WiFi 1/6/11 회피 + LED EMI 영역 회피
- 시연 중 호핑 안 함 (latency 위험)

### 셋업 데이 RF 스캔 절차
1. 박람회장 도착
2. RF 스캔 도구(Crazyradio + 자체 스크립트) 실행 → 2.4GHz 대역 활용도 측정
3. 활용도 낮은 4채널 선정 → 동글 4개 각각 할당
4. 드론 재구성 (그룹별 채널 매핑)

---

## 3. HTTPS 인증·권한

| # | 결정 | 락 |
|---|---|---|
| 3a | 토큰 방식 | **JWT** (stateless) |
| 3b | 권한 모델 | **RBAC** (4 역할) |
| 3c | 소셜 로그인 | **미통합** (1단계 자체 인증, 추후 카카오 옵션) |

### 4 역할 정의
| 역할 | 권한 |
|---|---|
| `customer` | 본인 예약·결제·미리보기·계약서 |
| `designer` | 디자인 업로드·견적 산출 (오너) |
| `operator` | 시연 운영 대시보드, 텔레메트리 조회 |
| `system` | 모듈 간 service-to-service 호출 |

### 토큰 정책
- **access token**: 15분 만료
- **refresh token**: 7일 만료, DB 저장 + 암호화 (ADR-0001 준수)
- 토큰 회전 (refresh 사용 시 새 refresh 발급, 이전 무효화)
- 로그아웃: refresh 무효화

---

## 4. WebSocket 재연결·하트비트

| # | 결정 | 락 |
|---|---|---|
| 4a | 하트비트 간격 | **5초** |
| 4b | 재연결 전략 | **지수백오프** (1s → 2s → 4s → 8s → 16s → 30s 상한) |
| 4c | 끊김 중 메시지 | **큐잉** (backend 보관) |
| 4d | 라이브러리 | **표준 라이브러리** (Socket.IO 또는 Python 진영 동급) |

### 운영
- 양쪽 모두 5초마다 ping/pong
- 3회 (15초) 응답 없으면 끊김 감지 → 재연결 시작
- 재연결 시 클라이언트가 마지막 수신 메시지 ID 전송 → backend가 그 이후 메시지 재전송
- backend 큐 보관 기간: 24시간 (이후 자동 파기)

### 적용 범위
- I2: backend → frontend 알림
- I7: field-controller → backend 텔레메트리

---

## 5. (메모) 메시지 직렬화

- REST / WebSocket: **JSON** (사람 읽기 가능, 표준)
- gRPC: **Protobuf** (gRPC 강제 양식)
- 본격 개발 시 압축 옵션 검토 (텔레메트리 30대 × 수십 Hz면 대역폭 영향)

---

## 6. (메모) TLS·암호화

| 채널 | 정책 |
|---|---|
| REST (외부) | TLS 1.3 강제, 평문 차단 |
| WebSocket (외부) | wss:// 강제 |
| gRPC (같은 머신) | mTLS 또는 Unix socket 권한 격리 (본격 개발 시) |
| Crazyradio | Crazyflie firmware encryption 옵션 활성화 (펌웨어 빌드 시) |
| RTSP (카메라) | 카메라 지원 시 RTSPS, 아니면 같은 LAN 격리 |

---

## 7. 다음 단계

- ✅ A. 모듈 책임 경계 → `01_system_architecture.md`
- ✅ B. 인터페이스 카탈로그 → `02_interface_catalog.md`
- ✅ C. 데이터 흐름도 → `03_data_flow.md`
- ✅ D. 통신 프로토콜 디테일 → 본 파일
- ✅ E. 장애·재시작 시나리오 → `05_failure_scenarios.md`
- **시스템 아키텍처 5단계 완료**

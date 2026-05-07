# 03. 데이터 흐름도 (C)

> 작성일: 2026-05-06
> 단계: C. 데이터 흐름도 (락) — D(통신 프로토콜 디테일) 진입 전 기준선
> 관련: `01_system_architecture.md`, `02_interface_catalog.md`
> 범위: **정상 흐름만** (사고·장애 분기는 E. 장애 시나리오에서)

---

## 0. 목적

A의 모듈 책임 + B의 인터페이스 위에, **시간 순서대로 어떤 모듈이 어떤 데이터를 어떤 순서로 주고받는지** 시퀀스로 정의. 시연 PR 영상 ≠ 시스템 흐름 — 이 문서는 시스템 관점.

5개 사이클:
1. 주문 (예약 → 견적 → 결제)
2. 디자인 (안무 작성 → 검증 → 컨펌)
3. 시연 준비 (D-N ~ D-1)
4. 시연 당일 (셋업 → 실행 → 철수)
5. 사후 (보고 → 정산)

---

## 1. 주문 사이클

```mermaid
sequenceDiagram
    actor User as 고객
    participant FE as frontend
    participant BE as backend
    actor Owner as 예원
    participant Toss

    User->>FE: 캘린더 페이지 접속
    FE->>BE: GET /availability (I1)
    BE-->>FE: 사용 가능 일정 (I2)
    User->>FE: 날짜·시간 선택, 폼 작성·제출
    FE->>BE: POST /quote-request (I1)
    BE->>BE: DB에 견적 요청 저장
    BE->>Owner: 알림 (이메일/카톡)
    Owner->>BE: 견적 산출·입력 (수동)
    BE-->>User: WebSocket 알림 "견적 나왔어요" (I2)
    User->>FE: 견적 페이지 확인
    User->>FE: 결제 진행
    FE->>Toss: 결제 위젯 호출
    Toss-->>BE: 결제 콜백
    BE->>BE: DB 일정 확정
    BE->>BE: 계약서 자동 생성
    BE-->>User: WebSocket 알림 "결제 완료" (I2)
    BE->>User: 이메일 (계약서 첨부)
```

**핵심**: 견적은 *수동 (예원)*, 나머지는 *자동*. 반자동 SaaS의 정의.

---

## 2. 디자인 사이클

```mermaid
sequenceDiagram
    actor Designer as 디자이너
    participant DS as design-system
    participant BE as backend
    actor User as 고객
    participant FE as frontend

    Designer->>BE: 행사 정보·요구사항 조회
    BE-->>Designer: 메타데이터
    Designer->>DS: 패턴 선택·파라미터 설정
    DS->>DS: 알고리즘 보간 (안무 생성)
    DS->>DS: 충돌·물리 검증
    Note over DS: 검증 통과 시만 다음 단계
    DS->>DS: 3D 시뮬레이션 영상 생성 (Skybrush Live)
    Designer->>DS: 검수·미세 조정
    Designer->>DS: 최종 결과 확정
    DS->>BE: 안무 (.skyc) + 시뮬 영상 URL + 검증 결과 업로드 (I3)
    BE->>BE: DB에 디자인 자산 저장
    BE-->>User: WebSocket 알림 "안무 미리보기 준비됐어요" (I2)
    User->>FE: 미리보기 페이지 접속
    FE->>BE: GET /design-preview (I1)
    BE-->>FE: 시뮬 영상 URL + 메타 (I2)
    User->>FE: 영상 확인 → 컨펌
    FE->>BE: POST /design-confirm (I1)
    BE->>BE: 안무 최종 락 (변경 불가)
    BE-->>User: 알림 "디자인 확정" (I2)
```

**핵심**: 안무는 *알고리즘 자동 생성 + 사람 검수 + 고객 컨펌* 3단계 게이트. 미리보기 = 영업 자산.

---

## 3. 시연 준비 사이클 (D-N ~ D-1)

```mermaid
sequenceDiagram
    participant BE as backend
    participant Log as logistics
    participant Carrier as 운송업체 API
    actor Ops as 운영자

    Note over BE,Log: D-N (예: 30일 전)
    BE->>Log: 행사 일정 자동 등록 (I4)

    Note over Log: D-7 (1주 전)
    Log->>Log: 인벤토리 점검 (드론 30·베이스 4·동글 2·케이블·LED 데크·예비 부품)
    Log-->>BE: 인벤토리 상태 보고 (I5)

    Note over Log,Carrier: D-3
    Log->>Carrier: 배송 요청
    Carrier-->>Log: 배송 일정 확정

    Note over Log: D-1
    Log-->>BE: 운송 진행 상태 (I5)
    Log->>Ops: 셋업 일정·체크리스트 전달

    Note over Ops: 도착
    Ops->>Log: 현장 도착 확인
    Log-->>BE: 도착 상태 (I5)
```

**핵심**: 시연 D-N부터 자동 트리거되는 일정 체인. 1인 운영 부담 최소화.

---

## 4. 시연 당일 사이클

```mermaid
sequenceDiagram
    actor Ops as 운영자
    participant FC as field-controller
    participant SM as safety-monitor
    participant DF as drone-firmware (30대)
    participant LH as Lighthouse Base
    participant Cam as Camera
    participant BE as backend

    Note over Ops: 셋업 단계 (시연 2-4시간 전)
    Ops->>LH: 베이스 4개 설치·캘리브레이션
    Ops->>Cam: 카메라 설치
    Ops->>FC: PC 부팅, field-controller 실행
    Ops->>DF: 드론 30대 배치, 전원 ON

    FC->>BE: 안무 패키지 (.skyc) 다운로드 요청 (I6)
    BE-->>FC: 안무 + 행사 메타 (I6)

    LH-->>DF: IR 빔 송출 (외부)
    DF->>DF: Lighthouse 위치 자체 인식

    FC->>DF: 안무 30대에 분배 (I11, Crazyradio)
    DF-->>FC: 다운로드 완료 ACK (I12)

    FC->>SM: geofence 정의 + 시작 신호 (I8, gRPC)
    Cam-->>SM: 영상 스트림 시작 (RTSP, 외부)
    SM->>SM: CV 모니터링 시작

    Note over Ops: 시연 시작 트리거
    Ops->>FC: 시연 시작 버튼 (운영 대시보드)
    FC->>DF: 시작 sync 신호 (I11)

    par 안무 실행 (수 분)
        DF->>DF: 안무 자체 실행 (LED·이동)
        DF-->>FC: 위치·상태·배터리 (I12, 수 Hz)
        FC-->>BE: 텔레메트리 스트리밍 (I7, WebSocket)
        SM-->>FC: (정상) 이상 이벤트 없음 (I9)
    end

    Note over DF: 안무 종료
    DF->>DF: 자동 착륙
    DF-->>FC: 착륙 완료 보고 (I12)
    FC->>SM: 종료 신호 (I8)
    FC-->>BE: 시연 종료 보고 (I7)

    Note over Ops: 철수 단계
    Ops->>DF: 드론 회수
    Ops->>LH: 베이스 회수
    Ops->>Cam: 카메라 회수
```

**핵심**: 시연 자체는 *드론 자율 실행* (펌웨어 안무) + *현장 시스템 모니터링* (텔레메트리·안전). field-controller가 명령을 매 프레임 송출하지 않음 (사전 다운로드 + sync 모델).

---

## 5. 사후 사이클

```mermaid
sequenceDiagram
    actor Ops as 운영자
    participant FC as field-controller
    participant BE as backend
    actor User as 고객
    actor Owner as 예원
    participant Log as logistics

    Note over Ops,FC: 시연 종료 직후
    FC-->>BE: 시연 영상·이벤트 로그 업로드 (I7 + I10 보강 채널)
    BE->>BE: 영상·로그 영구 저장

    Note over Ops,Log: 장비 회수
    Ops->>Log: 회수 완료 보고
    Log-->>BE: 인벤토리 갱신 (I5)

    Note over BE: D+1 ~ D+3
    BE->>Owner: 시연 결과 요약 + 영상 자료 정리 요청
    Owner->>BE: 고객 보고 자료 승인
    BE->>User: 보고 영상 + 시연 리포트 전송 (이메일·카톡)

    Note over BE: D+7
    BE->>BE: 정산 워크플로 (결제 정산, 운영팀 정산)
    BE->>User: 사후 만족도 설문 발송
    BE->>BE: 계약 종료 처리
```

**핵심**: 사후는 *backend 자동* 흐름이 대부분. 예원 개입은 보고 자료 *승인* 정도.

---

## 6. 사이클 간 관계

```mermaid
graph LR
    Order[1. 주문] -->|일정 확정 + 행사 메타| Design[2. 디자인]
    Design -->|확정 안무 .skyc| Prep[3. 시연 준비]
    Prep -->|장비 도착| Show[4. 시연 당일]
    Show -->|영상·로그| Post[5. 사후]
    Post -.->|만족도·재구매| Order
```

5개 사이클이 *순차적이지만 일부 병행* (디자인 진행 중에도 시연 준비 시작 가능, D-N가 충분히 길면).

---

## 7. 다음 단계

- ✅ A. 모듈 책임 경계 → `01_system_architecture.md`
- ✅ B. 인터페이스 카탈로그 → `02_interface_catalog.md`
- ✅ C. 데이터 흐름도 → 본 파일
- ✅ D. 통신 프로토콜 디테일 → `04_communication_protocols.md`
- **E. 장애·재시작 시나리오** — 다음 (시스템 아키텍처 5단계 마지막)

# ADR 0001: 데이터베이스 보안 정책

- **Status**: Accepted
- **Date**: 2026-05-06
- **Context**: 시스템 아키텍처 A 단계 결정 (`../01_system_architecture.md` 참조)

---

## Context

`backend/` DB가 다음 민감 데이터 다수 보유:

- 고객 PII (이름·연락처)
- 박람회 주최측·인맥 정보
- 결제 토큰 (카드번호 자체는 Toss가 보유, 우리 DB엔 토큰만)
- 안무 디자인 자산 (사업적 IP)
- 드론 텔레메트리 (감사·분석용)
- 카메라 영상 (사고 시 증거)
- 인증 토큰

본 레포가 public이므로 **코드는 공개되지만 운영 DB·시크릿은 절대 비공개**가 되어야 함. 따라서 보안 정책을 코드 변경과 분리해 ADR로 명시.

---

## Decision

### 1. DB 호스팅 — 매니지드 (PostgreSQL 호환)

- **Supabase 또는 Neon 등 매니지드 PostgreSQL** 사용
- 자체 호스팅 회피 — 1인 운영자에게 보안 패치·백업·암호화 자동화 필수
- 최종 선택(Supabase vs Neon vs 기타)은 backend 본격 개발 시점에 가격·기능 비교 후 결정
- 회피 옵션:
  - AWS RDS: 운영 부담 큼 (1인 운영에 과함)
  - 셀프호스팅: 보안 리스크 큼 (CVE 패치 자동화 부담)

### 2. 암호화

- **At rest**: 매니지드 기본 (AES-256, 매니지드 SaaS가 자동 적용)
- **In transit**: TLS 1.3 강제 (매니지드 기본, 평문 연결 차단)
- **Column-level 추가 암호화** (애플리케이션 레벨):
  - PII (이름·연락처)
  - 박람회 주최측·인맥 정보
  - 결제 토큰
  - 인증 토큰 (refresh token 등)
- 알고리즘 후보: pgcrypto AES, 또는 backend 레벨 Fernet/AES-GCM
- 키 관리 방식은 본격 개발 시 결정 (환경 변수 / AWS KMS / HashiCorp Vault)

### 3. PII 분리

- **단일 DB 안에서 PII 별도 테이블**
- 비즈니스 테이블에서 외래키로 참조
- PII 테이블 접근 권한 별도 (read-only 권한 분리, 변경 시 audit 강제)
- 결제 카드번호는 우리 DB에 절대 저장 안 함 — Toss 토큰만

### 4. 카메라 영상 처리

- **평시**: `safety-monitor/`가 자체 ring buffer (디스크 기반, 일정 주기 덮어쓰기, DB 저장 X)
- **사고 시점 ±2분**: `backend/`로 영구 전송
- **저장 전 얼굴 blur 처리**: OpenCV face detection + Gaussian blur
- **보유 기간**: 1년
- **자동 파기**: cron job 또는 backend 스케줄러

### 5. 감사 로그 (Audit Trail)

- `audit_log` 별도 테이블
- 추적 대상:
  - PII 조회·변경
  - 결제 정보 조회·변경
  - 안무 자산 조회·변경
  - 인증 시도 (성공·실패)
- 필드: `timestamp`, `user_id`, `action`, `target_table`, `target_id`, `ip`, `user_agent`
- `audit_log` 자체 변경 금지 (insert 후 read-only)
- Sentry는 *에러* 추적용, audit_log는 *비즈니스 보안 이벤트* 용 — 분리

---

## Consequences

### 긍정
- 매니지드 호스팅으로 운영 부담 폭감 (1인 운영자에 결정적)
- column-level 암호화로 DB 덤프 유출 시에도 핵심 데이터 보호
- audit_log로 사고 시 추적 가능 + 한국 개인정보보호법 대응
- 카메라 영상 얼굴 blur로 비촬영 동의 관객 보호

### 비용·트레이드오프
- 매니지드 호스팅 비용: 월 $25~$100 추정 (트래픽에 따라)
- column-level 암호화로 쿼리 성능 약간 저하 (PII 검색 시 복호화 비용)
- audit_log로 DB 쓰기 부하 약간 증가 (insert 한 번 더)
- 자체 호스팅 대비 종속성 증가 (vendor lock-in 일부)

---

## Open Questions

1. 매니지드 DB 최종 선택 (Supabase vs Neon vs 기타) — backend 본격 개발 시
2. column-level 암호화 키 관리 방식 (env vs KMS vs Vault)
3. `audit_log` 보존 기간 (현재 무기한, 추후 결정 필요)
4. Sentry 무료 티어 한도 초과 시 대응
5. 백업·복구 절차 표준화 (매니지드가 자동이지만 복구 테스트 정기 실행 필요)

---

## References

- `../01_system_architecture.md` § 2 (모호한 경계 결정 α, η, θ, ι)
- `../../../CLAUDE.md` § 3 (Public 레포 보안)
- 한국 개인정보보호법 — 카메라 영상 보유·익명화 근거

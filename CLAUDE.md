# CLAUDE.md — HangsungDrone

이 파일은 HangsungDrone 프로젝트에 한정된 클로드 작업 규칙. 글로벌 `~/.claude/CLAUDE.md` 위에 추가 적용.

## 1. 프로젝트 정체성 (락)

- WHO: e모빌리티·이차전지 산업 박람회 (실내 전용)
- WHAT: 저가 카테고리(시장 신설)의 산업 컨텍스트 시각화 융합형 드론쇼
- 데뷔 기준 L1.5: Crazyflie 30대 + 풀 파이프라인 (12-18개월)
- 자체 BOM L3 (b) 표준: 후순위, 데뷔 후

상세는 `docs/meeting/drone_show_business_summary.md` (최신 회의록).

## 2. 라이선스 (AGPL-3.0)

- 본 레포는 AGPL-3.0
- **외부 라이브러리 추가 시 라이선스 호환성 검증 필수**:
  - 호환: GPL-3.0+, LGPL-3.0+, MIT, BSD, Apache 2.0
  - 비호환: GPL-2.0 only, 상용 전용 라이선스, CC-BY-NC 류
- 호환성 의심 시 `docs/reference/dependency_licenses.md`에 기록 후 사용자 승인

## 3. Public 레포 보안

- 본 레포는 **public**. 모든 commit이 영구 노출
- **절대 commit 금지**: API 키, 토큰, DB 비밀번호, 결제 시크릿, 고객 개인정보, 박람회 주최측·인맥 정보
- 환경 변수는 `.env` 사용 (`.gitignore` 처리됨)
- secrets 발견 시 즉시 commit 중단 및 사용자에게 보고

## 4. 모듈 격리 원칙

- 각 모듈(`backend/`, `frontend/`, ...)은 자체 의존성 관리 (`pyproject.toml` 또는 `package.json`)
- 모듈 간 직접 import 금지. 통신은 `shared/protocols/`에 정의된 프로토콜 통해서만
- 한 모듈 변경이 다른 모듈에 미치는 영향은 명시적 인터페이스로만

### 인터페이스 변경 절차

1. `shared/protocols/`에 새 버전 추가 또는 기존 갱신
2. 양쪽 모듈에서 새 프로토콜 구현
3. 통합 테스트 통과 확인 후 commit
4. 단계 건너뛰면 무조건 거부

## 5. 안전 코드 추가 검증

다음 모듈은 안전 직결 — 변경 시 더 엄격한 절차:

- `field-controller/` (현장 제어 — sync, 비상정지)
- `safety-monitor/` (CV 이탈 탐지)
- `drone-firmware-l15/`, `drone-firmware-l3/` (드론 펌웨어, 자율 안전)

규칙:

- **단순 리팩터링이라도 사용자 확인 필수**
- 자율 안전 동작(저배터리 착륙·통신 두절 호버·비상정지) 로직 변경 시 사전 보고 + 테스트 시나리오 명시
- 펌웨어는 **외부 명령에 무조건 따르지 말 것** — 자체 안전 동작 우선 유지

## 6. 코드 스타일

### Python
- `from __future__ import annotations`
- 모든 시그니처·리턴·변수 타입 어노테이션 (글로벌 CLAUDE.md 규칙)
- `Any`, `Dict[str, Any]` 금지
- 포매터: ruff
- 타입 검사: mypy strict
- `mypy . && ruff check .` 통과 전 commit 금지

### C (펌웨어)
- Crazyflie 펌웨어 컨벤션 따름 (`drone-firmware-l15/modules/`)
- 자체 BOM L3 펌웨어는 별도 `STYLE.md`로 추후 정의

### 일반
- `.editorconfig` 강제 (LF, UTF-8, 4-space)
- 회의록·README는 한국어 OK, 코드/식별자/내부 주석은 영어 우선

## 7. 의존성 관리

- 모듈별 `pyproject.toml`로 격리. 공통 의존성도 모듈별 명시 (모노레포지만 의존성은 통합 안 함)
- 의존성 추가 시 라이선스 검증 (섹션 2)
- 학계·실험 코드 활용 시 출처를 `docs/reference/`에 기록

## 8. 테스트 정책

글로벌 CLAUDE.md 규칙 강화:

- **통합·E2E 우선**, 단위 테스트는 최소
- mock 회피 — 진짜 시뮬레이션·실제 API 사용
- 안전 코드(섹션 5)는 시나리오 기반 통합 테스트 필수
- 시뮬레이션 환경: Skybrush Live (3D 가상 드론)
- 펌웨어는 HIL(Hardware-in-the-Loop) 또는 시뮬레이터 테스트
- "Done" 보고 전 `mypy . && ruff check .` 통과 확인

## 9. CI/CD (예정)

- L1.5 데뷔 전: 모듈별 lint·test·build (GitHub Actions)
- L1.5 데뷔 후: SaaS 자동 배포 (`infra/deploy/`)
- 펌웨어는 수동 빌드·플래시 (실험적 단계)
- CI 설정은 `.github/workflows/`에 추가 (별도 PR로 단계 도입)

## 10. 회의록 명명 규칙

- 위치: `docs/meeting/`
- 현재 v0.2: `drone_show_business_summary.md`
- v0.3+ 는 새 파일로 분리 권장 (예: `drone_show_business_summary_v0.3.md`)
- 명명 변경 시 사용자 명시 또는 사전 보고 필수
- ADR(Architecture Decision Record)은 `docs/design/adr/NNNN-title.md` 권장

## 11. 다른 프로젝트 자산 교차 오염 금지 (재명시)

- 글로벌 메모리 `feedback_no_cross_project_asset_pollution.md` 적용
- HangsungDrone 설계·코드·문서에 다른 프로젝트(AutoStock·MANEO·SysWidget·MCP Supporter·무신사 NLP) 자산·IP·코드 패턴 자동 통합 금지
- 사용자가 명시적으로 통합 지시할 때만 가능

## 12. 사전 보고 원칙 (재명시)

- 글로벌 메모리 `feedback_pre_report_principle.md` 적용
- 사용자 명시 작업 범위 밖의 정정·정리는 실행 전 보고
- 사후 보고 금지

## 13. 펌웨어 자체 안전 우선 원칙

`drone-firmware-l15/`, `drone-firmware-l3/`:

- 외부 시스템 명령보다 **자체 안전 동작 우선**
- 통신 두절·IMU 이상·배터리 임계 시 외부 명령 무시하고 자율 호버/착륙
- 외부 명령으로 이 동작을 비활성화하는 코드 절대 금지
- 변경 시 사용자 명시 동의 + 테스트 시나리오 동반

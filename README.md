# HangsungDrone

실내 특화 드론 라이트쇼 사업 — e모빌리티 박람회 단일 ICP, 저가 카테고리 신설형.

## Status

- 단계: 정체성 락 완료 (Pre-MVP)
- 본격 시작: 2026 하반기
- 회의록: [docs/meeting/](docs/meeting/)

## Modules

| 디렉토리 | 책임 |
|---|---|
| `backend/` | [1] 중앙 SaaS 백엔드 — 주문 API, 견적 워크플로, DB, 계약서 자동 생성 |
| `frontend/` | [2] 고객용 웹 — 캘린더 예약, 폼, 결제 UI, 3D 미리보기 |
| `design-system/` | [3] 안무 디자인 시스템 — 패턴 라이브러리, 알고리즘 보간, 충돌·물리 검증, 3D 시뮬레이션 |
| `logistics/` | [4] 장비 이송 관리 — 인벤토리, 운송 일정 |
| `field-controller/` | [5] 현장 제어 시스템 — 안무 다운로드, sync, 비상정지, 운영 대시보드 |
| `safety-monitor/` | [6] 카메라 안전 모니터 — CV 기반 이탈·이상 탐지 |
| `drone-firmware-l15/` | [7] Crazyflie 펌웨어 커스터마이징 (L1.5 데뷔용) |
| `drone-firmware-l3/` | [8] 자체 BOM 펌웨어 (L3 후순위) |
| `shared/` | 공통 자산 — 프로토콜, 타입 정의 |
| `infra/` | Docker, IaC, 배포 |
| `docs/` | 회의록, 설계 문서, 외부 사양 정리 |
| `scripts/` | 빌드·배포·개발 헬퍼 |

## Tech Stack

- 백엔드·디자인·검증·CV: Python
- 프론트엔드: 추후 결정
- 펌웨어 (L1.5): C (Crazyflie 오픈소스 위)
- 펌웨어 (L3): C (ESP32-S3 또는 RP2040, 후순위)
- 위치 추적: Lighthouse Positioning System (IR 광학)
- 안무 디자인 백본: Skybrush Studio for Blender + Skybrush Live

## License

AGPL-3.0

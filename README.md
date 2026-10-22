# ktl-hub — KTL 정도검사 업무 자동화 공동개발 허브

연암공과대학교 캡스톤 디자인 프로젝트. 측정기기 정도검사 과정의 **기록 → 판독 → 병합 → 계산** 수작업을 줄이는 도구를 만든다.
이 레포는 코드가 아니라 **모두가 같이 보는 기준 문서**를 둔다. 코드는 파트별 레포에 있다.

> 처음 왔다면 순서대로: ① [역할분담](docs/04-역할분담.md)에서 내 파트 확인 → ② [협업 규칙](docs/05-협업규칙.md) → ③ [요구사항](docs/01-요구사항명세.md)에서 내 파트 ID → ④ [API·계약](docs/02-API명세.md) → ⑤ [UI 시나리오](docs/03-UI시나리오.md)

## 문서

| 문서 | 내용 |
|---|---|
| [01 요구사항 명세](docs/01-요구사항명세.md) | 파트별 기능·비기능 요구사항, 우선순위, 완료 기준 |
| [02 API 명세](docs/02-API명세.md) | 현장 서버 REST API, 모듈 계약, QR 스키마, 실험 PC 출력 형식 |
| [02a OpenAPI](docs/api/field-server.openapi.yaml) | 현장 서버 API 기계 판독용 (목업 서버·타입 생성에 사용) |
| [02b QR 테스트 벡터](docs/contracts/qr-test-vectors.json) | A-1·A-2가 같이 쓰는 QR 정답 예시 |
| [03 UI/UX 시나리오 대본](docs/03-UI시나리오.md) | 화면별 사용자 행동 → 화면 반응 → 예외 |
| [04 역할분담 · 제출물 형태](docs/04-역할분담.md) | 누가 무엇을 어떤 형태로 만들어 오는지 |
| [05 협업 규칙](docs/05-협업규칙.md) | 이슈·브랜치·PR·병합·시간 기록 |
| [06 일정 · 관문](docs/06-일정.md) | 11월 초 완성 일정, 관문별 확인 항목 |

## 레포 지도

| 레포 | 파트 | 담당 |
|---|---|---|
| [ktl-hub](https://github.com/YC-Dr-link/ktl-hub) | 공동 문서 | 희승(PM) |
| [ktl-flow-pi](https://github.com/YC-Dr-link/ktl-flow-pi) | A-1 접수대 라즈베리파이 | 희승 |
| [ktl-flow-pc](https://github.com/YC-Dr-link/ktl-flow-pc) | A-2 실험 PC 도구 | 경태 · 영진 |
| [ktl-field-server](https://github.com/YC-Dr-link/ktl-field-server) | B-1 현장 서버 (백엔드) | 여준 |
| [ktl-field-web](https://github.com/YC-Dr-link/ktl-field-web) | B-2 휴대폰 웹 · 관리 화면 (프론트) | 희승 |
| [ktl-field-modules](https://github.com/YC-Dr-link/ktl-field-modules) | B-3 사진 판독 · B-4 선별·병합 | 지호 · 정윤 / 탁 · 다운 |

## 지키는 원칙 (전 파트 공통)

- **인터넷 없이 동작**: 외부 API·CDN·클라우드·자동 업데이트 호출 금지
- **실제 데이터 커밋 금지**: 견본 CSV·사진·실명·업체명은 어떤 레포에도 올리지 않는다. 가짜 값으로 만든 `synthetic/` 파일만. CI의 `data-guard`가 막는다
- **원본은 그대로**: 입력 파일·사진은 읽기만, SHA-256 기록
- **판단은 시험자**: OCR·자동 계산은 보조. 화면에서 시험자가 확인해야 확정
- **파일 이름에 기대지 않기**: 파일 종류는 내용으로 판별

이 레포는 공개 레포다. KTL 내부 규정·시스템·장비에 관한 내용은 추측해서 쓰지 않고 "확인 필요"로 남긴다.

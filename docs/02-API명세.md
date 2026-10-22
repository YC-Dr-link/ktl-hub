# 02 API 명세 · 파트 간 계약

- 버전 1.0 (2026-10-22). 관문 1(10/25)에 **고정**한다. 이후 바꾸려면 ktl-hub 이슈 → 영향받는 파트 담당자 동의 → PM 승인.
- 기계 판독용 명세: [`api/field-server.openapi.yaml`](api/field-server.openapi.yaml). 이 문서와 다르면 **OpenAPI가 기준**이다.

| § | 계약 | 만드는 쪽 → 쓰는 쪽 |
|---|---|---|
| 1~2 | 현장 서버 REST API | B-1 여준 → B-2 희승 |
| 3 | 모듈 계약 (`Reader` · `Selector`) | B-3 지호·정윤, B-4 탁·다운 → B-1 여준 |
| 3.4 | `categories.yaml` | 전원 공용 (ktl-field-modules에 둔다) |
| 4 | QR 스키마 | A-1 희승 → A-2 경태·영진 |
| 5 | 통일 Excel 출력 형식 | A-2 → 시험자(CLAYDOX 입력) |
| 6 | Pi 내부 API | A-1 내부 (참고용) |

---

## 1. 현장 서버 공통 규칙

| 항목 | 규칙 |
|---|---|
| 휴대폰 API | `http://<노트북 IP>:8765/api/v1` — 에그 Wi-Fi 대역에서만 접근 |
| 관리 API | `http://127.0.0.1:8766/api/admin/v1` — 노트북 안에서만 (다른 기기 접속 불가) |
| 화면 제공 | 휴대폰 웹은 `:8765/`, 관리 화면은 `127.0.0.1:8766/`에서 서버가 정적 파일로 제공 |
| 형식 | 요청·응답 JSON(UTF-8). 업로드만 `multipart/form-data` |
| 시각 | ISO 8601, 노트북 로컬 시각 + 오프셋 (`2026-10-29T14:03:11+09:00`) |
| ID | 문자열. 접두사 + 숫자 (`case_12`, `ph_301`, `file_7`, `mg_3`, `ex_5`) |
| 인증 (휴대폰) | 쿠키 2개: `ktl_client`(단말 등록 시 발급, 90일) + `ktl_session`(시험자 로그인, 12시간). 모두 HttpOnly, SameSite=Strict |
| 인증 (관리) | 127.0.0.1에서만 열리므로 별도 로그인 없음 (S: 관리 비밀번호) |
| 목록 | `?limit=50&cursor=…` → `{ "items": [...], "next_cursor": null }` |

### 1.1 에러 형식

```json
{ "error": { "code": "PIN_LOCKED", "message": "PIN 5회 실패로 10분간 잠겼습니다.", "detail": { "retry_after": 600 } } }
```

| HTTP | code | 언제 |
|---|---|---|
| 400 | `VALIDATION` | 형식 오류 (`detail.fields`에 칸별 이유) |
| 401 | `CLIENT_UNKNOWN` · `SESSION_REQUIRED` | 미등록 단말 · 로그인 안 함 |
| 403 | `CLIENT_REVOKED` | 해제된 단말 |
| 404 | `NOT_FOUND` | — |
| 409 | `DUPLICATE_UPLOAD` · `CASE_COMPLETED` · `ALREADY_CONFIRMED` | 같은 업로드 재전송(기존 사진 반환) · 완료된 건 수정 · 확정된 값 재확정 |
| 410 | `PAIR_CODE_EXPIRED` | 등록 코드 5분 지남 |
| 413 | `FILE_TOO_LARGE` | 사진 20 MB 초과 |
| 423 | `PIN_LOCKED` | PIN 5회 실패 |
| 500 | `INTERNAL` | — |

### 1.2 상태 값

| 대상 | 상태 | 뜻 |
|---|---|---|
| 사진 `Photo.status` | `queued` → `reading` → `done` / `failed` / `deleted` | 판독 진행 |
| 값 `Value.status` | `auto` · `needs_check` · `confirmed` · `edited` | 자동 판독 · 확인 필요(신뢰도 < 0.8 또는 형식 오류) · 확정 · 수정 후 확정 |
| 접수 건 `Case.status` | `open` → `completed` | 완료 후에는 사진 추가·수정 불가 (관리 화면에서 다시 열기) |
| 병합 `Merge.status` | `queued` → `running` → `done` / `failed` | — |

---

## 2. 현장 서버 API (B-1 → B-2)

### 2.1 단말 · 로그인 (휴대폰)

| 메서드 | 경로 | 설명 | 요구사항 |
|---|---|---|---|
| POST | `/clients/pair` | 등록 코드로 단말 등록 → `ktl_client` 쿠키 | REQ-B1-10 |
| GET | `/testers` | 시험자 목록 (이름만) | REQ-B2-01 |
| POST | `/sessions` | 시험자 + PIN → `ktl_session` 쿠키 | REQ-B1-10 |
| GET | `/sessions/me` | 현재 단말·시험자 | — |
| DELETE | `/sessions` | 로그아웃 | — |

```http
POST /api/v1/clients/pair
{ "pair_code": "483920", "device_name": "현장폰-1" }

200 Set-Cookie: ktl_client=…
{ "client_id": "cl_3", "device_name": "현장폰-1", "paired_at": "2026-10-29T09:00:12+09:00" }
```

```http
POST /api/v1/sessions
{ "tester_id": "ts_2", "pin": "1234" }

200 Set-Cookie: ktl_session=…
{ "tester": { "id": "ts_2", "name": "시험자B" }, "expires_at": "2026-10-29T21:00:00+09:00" }
```

### 2.2 분류 · 접수 건

| 메서드 | 경로 | 설명 | 요구사항 |
|---|---|---|---|
| GET | `/categories` | `categories.yaml`을 JSON으로 (분류, 촬영 항목, 출력 칼럼) | REQ-B2-03 |
| GET | `/cases?status=open&q=` | 접수 건 목록 | REQ-B1-02 |
| POST | `/cases` | 새 건 | REQ-B1-02 |
| GET | `/cases/{case_id}` | 상세 (기기별·항목별 사진 상태 포함) | REQ-B1-02 |
| POST | `/cases/{case_id}/complete` | 건 완료 (미확정 값이 있으면 409 + 목록) | — |

```http
POST /api/v1/cases
{ "receipt_no": "26-000123-01", "category": "sampling", "unit_count": 2, "options": { "flowmeter_type": "defender" } }

201
{
  "id": "case_12", "receipt_no": "26-000123-01", "category": "sampling", "unit_count": 2,
  "status": "open", "created_by": "ts_2", "created_at": "2026-10-29T09:05:00+09:00",
  "units": [
    { "index": 1, "items": [ { "key": "cert", "label": "필증 (정도검사 증명서)", "required": true, "photo_id": null, "status": "empty" }, … ] },
    { "index": 2, "items": [ … ] }
  ]
}
```

- `receipt_no`는 `NN-NNNNNN-NN` 형식. 기기(세부 접수) 한 대가 `units[]` 한 칸이다.
- `items[].status`: `empty` · `queued` · `reading` · `done` · `needs_check` · `confirmed` · `failed`

### 2.3 사진

| 메서드 | 경로 | 설명 | 요구사항 |
|---|---|---|---|
| POST | `/cases/{case_id}/photos` | 업로드 → 즉시 `202 queued` | REQ-B1-03 |
| GET | `/cases/{case_id}/photos` | 건의 사진 목록 | — |
| GET | `/photos/{photo_id}` | 판독 상태·값·경고 | REQ-B1-04 |
| GET | `/photos/{photo_id}/image?variant=thumb` | 이미지 (`original` · `rotated` · `thumb`) | — |
| PATCH | `/photos/{photo_id}/values` | 값 수정·확정 | REQ-B1-06 |
| POST | `/photos/{photo_id}/retry` | 실패한 판독 다시 | — |
| DELETE | `/photos/{photo_id}` | 다시 찍기 (소프트 삭제, 사유) | — |

```http
POST /api/v1/cases/case_12/photos   (multipart/form-data)
file=@IMG_0001.jpg  unit_index=1  item_key=prec_1  client_upload_id=6f1c…(휴대폰이 만든 UUID)

202
{ "id": "ph_301", "case_id": "case_12", "unit_index": 1, "item_key": "prec_1", "status": "queued",
  "sha256": "9b2e…", "stored_name": "20261029_091102_prec_1.jpg" }
```

- 같은 `client_upload_id`를 다시 보내면 저장하지 않고 `409 DUPLICATE_UPLOAD` + 기존 사진을 돌려준다 (재시도 안전).
- 같은 칸에 새 사진을 올리면 이전 사진은 `deleted`(사유 `"다시 찍기"`)가 된다.

```http
GET /api/v1/photos/ph_301

200
{
  "id": "ph_301", "status": "done", "item_key": "prec_1", "reader": "flowmeter",
  "values": [
    { "key": "flow", "label": "유량", "value": "1.002", "unit": "L/min", "confidence": 0.93,
      "bbox": [412, 220, 690, 310], "status": "auto" },
    { "key": "reading_no", "label": "측정 번호", "value": "01 of 003", "unit": null, "confidence": 0.71,
      "bbox": [400, 120, 560, 160], "status": "needs_check" }
  ],
  "warnings": [ { "code": "PRE_MEASURE_SCREEN", "message": "측정 전 화면(0.000)일 수 있습니다." } ],
  "error": null, "read_ms": 2140
}
```

```http
PATCH /api/v1/photos/ph_301/values
{ "values": [ { "key": "reading_no", "value": "01 of 003" },
              { "key": "flow", "value": "1.020", "reason": "OCR이 소수점 위치를 잘못 읽음" } ],
  "confirm": true }

200  (갱신된 Photo)
```

- 값을 바꾸면 `reason` 필수(없으면 400). 바꾸지 않고 확정만 하는 값은 `reason` 생략.
- `confirm: true`면 보낸 값 전부 `confirmed`(바뀐 값은 `edited`). `needs_check` 값을 보내지 않고 확정하려 하면 400.

### 2.4 실시간 알림 (SSE)

`GET /api/v1/events` — `EventSource`로 연결. 쿠키 인증. 15초마다 `: ping`.

| event | data | 언제 |
|---|---|---|
| `photo.status` | `{ "photo_id", "case_id", "unit_index", "item_key", "status" }` | 상태 변경 |
| `photo.values` | `{ "photo_id", "case_id" }` | 판독 완료·값 수정 (화면은 `GET /photos/{id}`로 다시 받음) |
| `case.updated` | `{ "case_id" }` | 건 생성·완료 |
| `merge.status` · `export.done` | `{ "id", "case_id", "status" }` | 관리 화면용 (관리 API `/events`에도 같은 형식) |

### 2.5 관리 API (127.0.0.1:8766/api/admin/v1)

휴대폰 API의 `/cases`, `/photos`, `/events`는 관리 API에서도 같은 경로로 쓸 수 있다 (로그인 없이). 아래는 관리 전용.

| 메서드 | 경로 | 설명 | 요구사항 |
|---|---|---|---|
| POST | `/pair-codes` | 등록 코드 발급 → `{ code, expires_at, qr_svg }` | REQ-B2-08 |
| GET · DELETE | `/clients` · `/clients/{id}` | 단말 목록 · 해제 | REQ-B2-08 |
| GET · POST · PATCH | `/testers` · `/testers/{id}` | 시험자 관리 (PIN 재설정, 잠금 해제) | REQ-B2-08 |
| GET | `/review-queue` | "확인 필요" 값 모아 보기 | REQ-B2-06 |
| POST | `/cases/{id}/reopen` | 완료 건 다시 열기 (사유) | — |
| POST | `/cases/{id}/files` | 가스·라돈 파일 업로드 → 형식 판별 결과 | REQ-B1-09 |
| GET | `/files/{id}/series` | 그래프용 시계열 (B-4 `series()`) | REQ-B2-09 |
| POST | `/cases/{id}/merges` | 선별/병합 실행 → `202` | REQ-B1-09 |
| GET | `/merges/{id}` | 결과 표 | REQ-B1-09 |
| POST | `/cases/{id}/exports` | 내보내기 생성 | REQ-B1-08 |
| GET | `/exports/{id}` · `/exports/{id}/download` | 상태 · 파일 받기 | REQ-B1-08 |
| GET · PUT | `/settings` | 출력 인코딩, 보관 기간 등 | — |
| GET | `/audit?case_id=` | 작업 로그 (+ 해시 체인 검증 결과) | REQ-B1-11 |

```http
POST /api/admin/v1/cases/case_20/files   (multipart)  file=@logger_1min.csv
201 { "id": "file_7", "detected": { "kind": "gas_gl240", "interval_s": 60, "unit": "PPB", "rows": 1440, "span": [-500, 500] },
      "sha256": "…" }

GET /api/admin/v1/files/file_7/series
200 { "columns": ["t", "CH1"], "unit": "PPB", "rows": [["2026-10-29T10:00:00+09:00", 1.3], …] }

POST /api/admin/v1/cases/case_20/merges
{ "selector": "gas_select", "file_ids": ["file_7", "file_8"],
  "options": { "ranges": { "ZDS": ["2026-10-29T10:00:00+09:00", "2026-10-29T10:30:00+09:00"], "SDS": [ … ], "LT": [ … ] } } }
202 { "id": "mg_3", "status": "queued" }

GET /api/admin/v1/merges/mg_3
200 { "id": "mg_3", "status": "done", "selector": "gas_select",
      "table": { "columns": ["code", "v1", "v2", "…", "v10"], "rows": [["ZDS", "1.2", "1.3", …], …] },
      "warnings": [] }
```

라돈 병합 옵션: `{ "reference": "file_11", "align": "sequence" | "hour" }` (기준기 = 업로드한 파일 중 하나).

```http
POST /api/admin/v1/cases/case_12/exports
{ "kind": "photo_values", "format": "csv", "encoding": "cp949", "include_photos": true }
202 { "id": "ex_5", "status": "queued" }
```

| kind | 내용 | 행 |
|---|---|---|
| `photo_values` | 사진 업무(시료채취·먼지) 확정값 | 기기 1대 = 1행, 칼럼은 `categories.yaml`의 `export` 순서 |
| `merge_raw` | 가스·라돈 병합 결과 | xlsm RAW 시트에 붙여 넣는 배치 그대로 |

`format`: `csv` · `txt`(탭 구분) · `xlsx`. `encoding`: `cp949` · `utf-8`. `include_photos`면 zip(값 파일 + 워터마크 사진 사본).

---

## 3. 모듈 계약 (B-3 · B-4 → B-1)

모듈은 **파이썬 패키지**로 만들고 `ktl-field-modules` 레포에 둔다. 서버는 `pip install`로 설치해 이름으로 불러 쓴다. 계약 코드는 `ktl-field-modules/contract/ktl_contract/`에 있고 **모듈 팀은 이 파일을 고치지 않는다** (바꾸려면 이슈).

### 3.1 공통 타입

```python
@dataclass
class Value:
    key: str                     # categories.yaml의 칸 key (예: "flow", "approval_no")
    label: str                   # 화면 표시 이름
    value: str                   # 항상 문자열. 원본 표기 유지 ("1.020", "01 of 003")
    unit: str | None = None
    confidence: float = 1.0      # 0~1. 형식·범위 검사 실패면 0.5 이하로
    bbox: list[int] | None = None  # 받은 이미지 기준 [x1, y1, x2, y2]

@dataclass
class Warning:
    code: str                    # "PRE_MEASURE_SCREEN", "MULTIPLE_DOCS" …
    message: str

class ReadError(Exception):      # 아예 못 읽을 때 (화면을 못 찾음 등). 메시지는 한국어
    ...
```

### 3.2 Reader (B-3)

```python
class Reader(Protocol):
    name: str                                   # "cert" | "flowmeter" | "dust"
    def read(self, image_path: str, item: str) -> ReadResult: ...

@dataclass
class ReadResult:
    values: list[Value]
    warnings: list[Warning] = field(default_factory=list)
```

- 입력: 서버가 회전 보정한 **사본 경로**, 촬영 항목 key (`categories.yaml`의 `items[].key`)
- 금지: 네트워크, DB, 파일 쓰기(임시 폴더 제외), 전역 상태
- 시간: 1장 10초 안 (서버가 넘으면 강제 종료)
- 못 읽으면 `ReadError`, 일부만 읽으면 읽은 칸만 반환 (빈 칸은 서버가 `needs_check`로 채움)

### 3.3 Selector (B-4)

```python
class Selector(Protocol):
    name: str                                   # "gas_select" | "radon_merge"
    def detect(self, path: str) -> dict | None: ...      # 형식 판별. 모르는 파일이면 None
    def series(self, paths: list[str]) -> pandas.DataFrame: ...  # 그래프용. 칼럼 t + 채널들
    def run(self, paths: list[str], options: dict) -> SelectResult: ...

@dataclass
class SelectResult:
    table: pandas.DataFrame       # RAW 시트에 붙일 배치 그대로 (칼럼 순서 = 출력 순서)
    warnings: list[Warning] = field(default_factory=list)
    meta: dict = field(default_factory=dict)   # 계산 근거(선택 구간, 정렬 방식, 지시오차 등)
```

- 판정(합격/불합격)은 하지 않는다. 기존 xlsm/CLAYDOX가 한다.
- `options` 형식은 [§2.5](#25-관리-api-1270018766apiadminv1) 예시를 따른다.

### 3.4 categories.yaml

`ktl-field-modules/categories.yaml` 하나를 B-1·B-2·B-3·B-4가 같이 쓴다. 칸 매핑을 바꿀 때 코드는 안 바꾸고 이 파일만 바꾸는 것이 목표다.

```yaml
version: 1
categories:
  - category: sampling
    label: 시료채취
    phone: true                      # false면 휴대폰 분류 선택에 안 나옴 (가스·라돈)
    options:
      - { key: flowmeter_type, label: 기준 유량계, choices: [defender, alicat] }
    items:
      - { key: cert,   label: "필증 (정도검사 증명서)", reader: cert,      required: true }
      - { key: plate,  label: "형식승인표",             reader: none,      required: true }
      - { key: prec_1, label: "정밀오차 1",             reader: flowmeter, required: true }
      # … prec_2, prec_3, flow_1 ~ flow_3
    export: [photo_first, cert.prev_receipt, cert.approval_no, cert.maker, cert.serial,
             prec_1.flow, prec_2.flow, prec_3.flow, flow_1.flow, flow_2.flow, flow_3.flow]
  - category: gas
    label: 가스
    phone: false
    selector: gas_select
    items: []
    export: [merge_raw]
```

- `export`의 `prec_1.flow` = "항목 prec_1 사진에서 읽은 칸 flow 값". `photo_first` = 그 기기 첫 사진의 저장 파일명.
- 전체 파일: [ktl-field-modules/categories.yaml](https://github.com/YC-Dr-link/ktl-field-modules/blob/main/categories.yaml) (시료채취·먼지·가스·라돈). 먼지 칸 매핑은 임시안.

---

## 4. QR 스키마 (A-1 → A-2)

Pi LCD에 띄우고 실험 PC가 읽는다. **ASCII만** 쓴다 (스캐너가 키보드처럼 입력하므로).

```json
{"v":1,"rn":"25-123456-01","sn":"FV-0001","ap":"ABCDE-2013-1","md":"FLOW-03D","rg":22,"cn":"7Jew7JWU6rO164yA","ck":"51cf3e4f"}
```

| 키 | 뜻 | 형식 |
|---|---|---|
| `v` | 스키마 버전 | 정수 `1` (다른 값은 거부) |
| `rn` | 접수번호 | `NN-NNNNNN-NN` |
| `sn` | 제조번호 | 문자열 (공백 가능) |
| `ap` | 형식승인번호 | 문자열 |
| `md` | 모델 | 문자열 |
| `rg` | 최대 측정범위 (m/s) | 숫자 — A-2 반복성 분모 |
| `cn` | 업체명 | UTF-8 → Base64 URL-safe, `=` 제거 |
| `ck` | 체크값 | `ck`를 뺀 JSON(키 순서 위 표대로, 공백 없음)의 SHA-256 앞 8자리 hex |

- 정답 예시: [`contracts/qr-test-vectors.json`](contracts/qr-test-vectors.json). A-1·A-2 모두 이 파일로 테스트한다.
- 스캐너는 2D 이미저 USB(키보드 입력) 또는 웹캠. PC가 한글 입력 상태면 `{"v"…`가 한글 자모로 바뀌어 들어오므로, A-2는 입력을 받은 뒤 **두벌식 자모 → 영문 키**로 되돌리는 처리를 한다 (REQ-A2-02).

---

## 5. A-2 출력 형식 (통일 Excel)

CLAYDOX 입력은 시험자가 한다. 그래서 **시트 이름과 셀 위치만 고정**되면 된다. 아래는 초안이며 원본 성적서 양식을 받으면 맞춘다 (확인 필요).

- 파일명: `<접수번호-세부>_유속계.xlsx` (예: `25-123456-01-1_유속계.xlsx`), 접수번호 폴더에 저장

| 시트 | 내용 | 배치 |
|---|---|---|
| `일반사항` | 접수번호, 업체, 모델, 제조번호, 형식승인번호, 최대 측정범위, 시험일, 시험자, 대기압·온도·습도 최소/최대/평균 | A열 항목명, B열 값 (행 고정, 아래 표) |
| `반복직선성` | 반복성 ZERO/SPAN, 직선성 30/70 %, 판정 기준, 계산 근거 표 | 상단 요약 + 아래 원 값 표 |
| `민감도` | 각도 5개 × 30/70 % × 3회, 묶음별 평균 오차, 최댓값 | 표 + 그래프 2개(S) |
| `구조확인` | Pi 대조 결과 칸 (이번 범위에서는 빈 칸) | — |
| `raw data` | CSV 6개를 원래 행 번호 그대로 (1~13행 머리글 포함) | 파일마다 열 블록 |
| `출처` | 입력 파일명, SHA-256, 판별 결과, 도구 버전, 생성 시각 | 파일 1개 = 1행 |

`일반사항` 고정 행(초안): 2 접수번호 · 3 업체 · 4 모델 · 5 제조번호 · 6 형식승인번호 · 7 최대 측정범위 · 8 시험일 · 9 시험자 · 11~13 대기압(최소·최대·평균) · 14~16 온도 · 17~19 습도.

계산식 (합성 데이터 정답과 소수 셋째 자리까지 일치해야 함):

| 항목 | 식 | 기준 |
|---|---|---|
| 반복성 | 결과 10행 중 홀수 = ZERO, 짝수 = SPAN. 각각 `STDEV(측정 유속 5개) ÷ 최대 유속 × 100` | ≤ 2.0 % |
| 직선성 | 1~5행 = 30 %, 6~10행 = 70 %. ① `max(|기준−측정| ÷ 기준) × 100` ② `(평균 측정 − 평균 기준) ÷ 평균 기준 × 100` — 설정으로 선택 | ≤ 5.0 % |
| 민감도 | 30행 = 각도 5개(0°, −10°, −5°, +5°, +10°) × (30 %, 70 %) × 3회. 3행 묶음별 `|AVERAGE(오 차)|`의 최댓값 | ≤ 4.0 % |

---

## 6. Pi 내부 API (A-1, 참고용)

Pi 안에서만 쓰는 `http://127.0.0.1:8000/api`. 외부 계약이 아니므로 희승이 자유롭게 바꾼다.

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/devices?q=` | 기기 검색 (업체 별칭 포함) |
| GET · POST · PATCH | `/devices/{id}` | 기기 상세·등록·수정 |
| POST | `/receipts` | `{ device_id, receipt_no }` → `{ qr_text, qr_svg }` |
| POST | `/captures` | 카메라 촬영 + OCR → 대조 결과 (S) |
| GET | `/specs/{approval_no}` | 형식승인 기준정보 (S) |
| POST | `/backup` · `/restore` · `/import` | USB 백업·복원·CSV 일괄 등록 (S) |

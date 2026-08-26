# G-Bird 지원 사이트 — Cloudflare 이전 구현 명세서

> 이 문서는 **코딩 에이전트(Codex)와 사람이 함께 이 작업을 이어서 실행**하기 위한 정밀 명세입니다.
> 원본 요구사항은 `HANDOFF.md`(또는 최초 지시서)이고, 이 문서는 그 요구사항을 **현재 코드에 맞춰 실행 가능한 단위로 분해**한 것입니다.
>
> 작성 시점 기준으로 현재 저장소에는 `index.html`(번들된 프런트) 한 개만 있습니다. 아래 파일 구조/코드/스키마는 아직 **미구현**이며, 이 문서를 따라 새로 만들면 됩니다.

---

## 0. 진행 방식 (사람용 안내)

- 로컬에서 **Remote Control**을 켜고 **Codex**를 코딩 에이전트로 붙여, 이 문서의 각 섹션을 순서대로 구현하면 됩니다.
- 권장 구현 순서: **§3 저장소 구조 → §4 D1 스키마 → §5 API → §6 보안규칙 → §7 레이트리밋 → §8 마이그레이션 → §9 프런트 배선 → §10 배포 → §11 검증**.
- 각 섹션 끝의 `☑ 완료 기준`을 그대로 Codex 작업 단위(=커밋 단위)로 쓰면 좋습니다.
- 비밀값(관리자 비밀번호, 서명 키)은 **절대 저장소에 커밋하지 않습니다.** `wrangler secret put`으로만 주입합니다(§10).

---

## 1. 목표 (요구사항 요약)

Google Apps Script + Sheets → **Cloudflare Workers + D1(SQLite)** 로 이전. 무료 티어로 충분.

반드시 지킬 5가지 (원본 지시서):

1. 어떤 응답도 **요청자 본인 레코드**를 넘어서는 데이터를 포함하지 않는다. 전체 명단은 운영진 인증을 통과한 요청에만 반환.
2. 운영진 인증은 **서버에서** 수행한다. 비밀번호(또는 해시)는 Worker secret에 두고 클라이언트 코드에는 어떤 형태로도 두지 않는다.
3. 합격 여부는 **발표 시각 이후에만** 응답에 포함한다. 시각 판정은 서버가 `Asia/Seoul` 기준으로 한다. 클라이언트를 신뢰하지 않는다.
4. 면접 시각도 같은 방식으로 공개 시각 이후에만 반환한다.
5. 클라이언트 `localStorage`는 없애거나, 남기더라도 **서버 응답의 사본으로만** 쓴다. 서버가 원본이다.

---

## 2. 현재 코드 사실관계 (분석 결과 — Codex가 재조사할 필요 없음)

### 2.1 프런트 파일 구조 — `index.html`은 "번들" 파일이다

`index.html`(약 7.8MB, 386줄)은 Claude Design으로 만든 **번들 아티팩트**입니다. 대부분이 인라인 자산입니다.

- **line 372** (약 7.4MB): 폰트/이미지 등 인라인 자산 blob
- **line 384** (약 420KB): 실제 앱 HTML이 **JSON 문자열로 인코딩**되어 들어 있음
- 앱 로직 전체는 그 안의 `<script type="text/x-dc" ...>` 블록 **하나**에 있고, `class Component extends DCLogic` 형태입니다(React 유사, `this.state` / `this.setState` / `renderVals()`).

> **편집 방법 (중요):** 이 파일은 소스가 아니라 빌드 산출물입니다. 앱 로직을 바꾸려면 두 가지 길이 있습니다.
> - **(권장) Claude Design 편집기에서** 원본을 열어 수정 후 다시 export → `index.html` 교체. 디자인 워크플로가 보존됩니다.
> - **직접 패치:** line 384를 JSON 디코드 → 스크립트 수정 → 다시 JSON 인코드해 line 384에 되쓰기. 아래 스니펫 참고. (디자인 편집기 재편집 호환성은 깨질 수 있음)
>
> ```python
> # 디코드
> import json
> lines = open('index.html', encoding='utf-8').read().split('\n')
> app = json.loads(lines[383])          # 0-index → line 384
> open('app.html','w',encoding='utf-8').write(app)
> # ... app.html 수정 ...
> # 재인코드
> app = open('app.html',encoding='utf-8').read()
> lines[383] = json.dumps(app, ensure_ascii=False)
> open('index.html','w',encoding='utf-8').write('\n'.join(lines))
> ```

### 2.2 현재 데이터 흐름과 바꿔야 할 지점

앱 스크립트 안의 주요 심볼(디코드된 `app.html` 기준):

| 심볼 | 현재 동작 | 이전 후 |
|---|---|---|
| `const KEY = 'gbird-2026-fall'` | 지원자 명단 localStorage 키 | 캐시 용도로만 축소 또는 제거 (§9) |
| `const SKEY = 'gbird-2026-settings'` | 설정 localStorage 키 | `/api/settings` 응답 캐시로만 |
| `const ADMIN_HASH = '1o281lx-1md8mp0'` | 클라이언트 FNV 해시 비교 | **삭제** |
| `const APPS_SCRIPT = [...]` | 운영진 화면에 박힌 Apps Script 전문 | **삭제** (연동 UI 통째로 제거) |
| `this.props.sheetUrl` (Tweaks 기본값 `AKfycb...`) | Apps Script `/exec` 주소 주입 | **삭제**, 동일출처 `/api` 사용 |
| `state.settings.endpoint` | 위 주소 저장 | 제거 |
| `api(payload)` | 단일 엔드포인트에 `{action}` POST | 엔드포인트별 호출로 분리 (§9) |
| `syncNow(silent)` | `{action:'list'}`로 **전체 명단** 수신 | `POST /api/admin/list` (쿠키 필요) |
| `login()` | 로컬 `applicants`에서 이름+전화로 검색 | `POST /api/me` (서버가 본인 레코드 반환) |
| `submit()` | `{action:'submit', record}` | `POST /api/submit` (409 중복) |
| `saveEdit()` | `{action:'update', oldPhone, record}` | `POST /api/me/update` (마감 전만) |
| `setRsvp()` / `pushRsvpNote()` | `{action:'rsvp', ...}` | `POST /api/me/rsvp` |
| `setSlot(id, field, value)` | `{action:'admin', phone, time, result}` | `POST /api/admin/set` |
| `saveSettings()` | localStorage(SKEY) 저장만 | `POST /api/admin/settings` |
| `adminLogin` 핸들러 | `hash(pw) === ADMIN_HASH` | `POST /api/admin/login` |
| `resultPassed()` / `revealPassed()` | 클라 시각 판정 | 장식으로만 (서버가 필드 자체를 생략) (§6) |
| `persist(applicants)` | 전체 명단 localStorage 기록 | 지원자 브라우저에서 전체 명단 저장 금지 (§9) |

### 2.3 현재 페이로드/응답 계약 (기존 Apps Script `doPost`)

바꾸기 전 계약을 기록해 둡니다. 신설 Worker는 응답 **형태를 최대한 유지**해 프런트 변경을 줄입니다.

- `submit`: req `{record:{name,gender,phone,email,birth,dept,exp,avail,hobby,route[],routeEtc,note,submittedAt,time}}` → `{ok:true}` / 중복 시 `{ok:false,error:'duplicate'}`
- `list`: `{ok:true, rows:[ {name,gender,phone,email,birth,dept,exp,avail,hobby,note,submittedAt,time,rsvp,rsvpAt,rsvpNote,result} ]}`
- `rsvp`: req `{phone,rsvp:'yes'|'no',rsvpAt,rsvpNote}` → `{ok:true}`
- `update`: req `{oldPhone, record}` → `{ok:true}`
- `admin`: req `{phone, time?, result?}` → `{ok:true}`

---

## 3. 저장소 구조 (신설)

```
/
├─ index.html                 # 기존 번들 프런트 (배선만 교체, §9)
├─ public/                    # Worker가 서빙할 정적 파일
│  └─ index.html              # 위 파일을 여기로 이동/복사 (배포 시 정적 루트)
├─ worker/
│  ├─ index.ts                # fetch 진입점: /api/* 라우팅, 그 외 정적 서빙
│  ├─ router.ts               # 경로→핸들러
│  ├─ handlers/
│  │  ├─ submit.ts
│  │  ├─ me.ts                # /api/me, /api/me/update, /api/me/rsvp
│  │  ├─ settings.ts          # /api/settings (공개)
│  │  └─ admin.ts             # login/list/set/settings
│  ├─ lib/
│  │  ├─ auth.ts              # 세션 쿠키 서명/검증, 비밀번호 비교
│  │  ├─ time.ts              # KST 시각 판정
│  │  ├─ phone.ts             # 전화번호 정규화
│  │  ├─ ratelimit.ts         # IP 레이트리밋
│  │  └─ db.ts                # D1 쿼리 헬퍼
│  └─ types.ts
├─ migrations/
│  ├─ 0001_init.sql           # D1 스키마 (§4)
│  └─ seed_settings.sql       # 기본 설정/회차
├─ scripts/
│  └─ import-csv.mjs          # CSV → D1 일회성 임포트 (§8)
├─ wrangler.toml              # §10
├─ package.json
└─ docs/CLOUDFLARE-MIGRATION.md   # 이 문서
```

> 정적 서빙은 Cloudflare **Workers Static Assets**(`[assets] directory="./public"`)를 사용해 **단일 Worker가 정적 파일 + `/api`를 동일 출처로** 제공합니다. → CORS 불필요. (Pages + Functions로 해도 되지만, 아래 예시는 Workers 기준.)

☑ **완료 기준:** 위 디렉터리와 빈 파일 스캐폴드 생성, `package.json`/`wrangler.toml` 존재, `wrangler dev`가 뜬다.

---

## 4. D1 스키마

핵심 키는 **전화번호(정규화: 숫자만)** 이고, **회차(cohort)** 로 학기를 분리합니다.

`migrations/0001_init.sql`:

```sql
-- 지원자
CREATE TABLE applicants (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  cohort        TEXT    NOT NULL,              -- 예: '2026-fall'
  name          TEXT    NOT NULL,
  gender        TEXT,
  phone         TEXT    NOT NULL,              -- 표시용 원본(010-1234-5678)
  phone_norm    TEXT    NOT NULL,              -- 숫자만 (고유 비교용)
  email         TEXT,
  birth         TEXT,
  dept          TEXT,                          -- 학과/학위 및 재학학기
  exp           TEXT,                          -- 배드민턴 경험
  avail         TEXT,                          -- 참여 가능
  hobby         TEXT,
  route         TEXT,                          -- 유입 경로 (" / " 조합 문자열)
  note          TEXT,                          -- 하고 싶은 말
  submitted_at  TEXT,                          -- 제출시각 (KST 문자열)
  interview_time TEXT,                         -- 면접시각 (HH:mm) — 운영진이 입력
  rsvp          TEXT,                          -- '' | 'yes' | 'no'
  rsvp_at       TEXT,
  rsvp_note     TEXT,                          -- 불참/변경 사유
  result        TEXT,                          -- '' | '합격' | '불합격'
  created_at    TEXT    DEFAULT (datetime('now')),
  updated_at    TEXT    DEFAULT (datetime('now')),
  UNIQUE (cohort, phone_norm)
);
CREATE INDEX idx_applicants_cohort ON applicants (cohort);

-- 회차별 공통 설정 (날짜·장소·공개/발표 시각)
CREATE TABLE settings (
  cohort        TEXT PRIMARY KEY,              -- 예: '2026-fall'
  active        INTEGER NOT NULL DEFAULT 0,    -- 현재 활성 회차 1개만 1
  date          TEXT,                          -- 표시용 면접 날짜 '9/20 (일)'
  place         TEXT,
  close_at      TEXT,                          -- 접수 마감 (KST) '2026-09-16 22:00'
  reveal_at     TEXT,                          -- 면접시각 공개 (KST) '2026-09-18 20:00'
  iv_start_at   TEXT,                          -- 면접 시작 (KST) '2026-09-20 19:30'
  result_at     TEXT,                          -- 합격 발표 (KST) '2026-09-22 20:00'
  updated_at    TEXT    DEFAULT (datetime('now'))
);

-- 레이트리밋 (KV를 못 쓸 때의 D1 대안; §7)
CREATE TABLE rate_limits (
  bucket        TEXT PRIMARY KEY,              -- 예: 'me:1.2.3.4:2026-08-26T09:15'
  count         INTEGER NOT NULL,
  expires_at    INTEGER NOT NULL               -- epoch seconds
);
```

`migrations/seed_settings.sql` — 현재 값으로 시드:

```sql
INSERT INTO settings (cohort, active, date, place, close_at, reveal_at, iv_start_at, result_at)
VALUES ('2026-fall', 1, '9/20 (일)',
        'KAIST N3 류근철 스포츠 컴플렉스 B1층 주경기장',
        '2026-09-16 22:00', '2026-09-18 20:00', '2026-09-20 19:30', '2026-09-22 20:00');
```

☑ **완료 기준:** `wrangler d1 migrations apply`로 스키마가 로컬/원격에 적용된다. 시드 설정 1행 존재.

---

## 5. API 엔드포인트 계약

모두 `POST`, 본문 JSON, 응답 JSON. 동일 출처(`/api/...`). 활성 회차는 서버가 `settings.active=1`에서 읽습니다(클라이언트가 cohort를 지정하지 않음).

| 엔드포인트 | 인증 | 요청 | 응답(성공) | 규칙 |
|---|---|---|---|---|
| `/api/settings` | 없음 | `{}` | `{ok, settings:{date, place, close_at, reveal_at, iv_start_at, result_at}}` | 비밀값 없음. 공개용만 |
| `/api/submit` | 없음 | `{record}` | `{ok:true}` | 같은 회차+전화 중복이면 **409** `{ok:false,error:'duplicate'}`. 접수 마감 후면 **403** |
| `/api/me` | 이름+전화 | `{name, phone}` | `{ok, me:{...본인필드}}` | 본인 1건만. `interview_time`은 `reveal_at` 이후에만, `result`는 `result_at` 이후에만 **필드 포함**(그 전엔 키 자체 없음). 없으면 **404**. 레이트리밋(§7) |
| `/api/me/update` | 이름+전화 | `{name, phone, record}` | `{ok:true}` | `close_at` 이전에만. 이후 **403**. 전화 변경 시 회차 내 중복 검사 |
| `/api/me/rsvp` | 이름+전화 | `{name, phone, rsvp, rsvpNote?}` | `{ok:true}` | `reveal_at` 이후에만 허용 권장. `rsvp_at`은 서버가 KST로 스탬프 |
| `/api/admin/login` | 비밀번호 | `{password}` | `{ok:true}` + `Set-Cookie` | 실패 **401**. 성공 시 HttpOnly 세션 쿠키(§6.2). 레이트리밋 |
| `/api/admin/logout` | 쿠키 | `{}` | `{ok:true}` | 쿠키 만료 |
| `/api/admin/list` | 세션 쿠키 | `{}` | `{ok, rows:[...전체]}` | 쿠키 없으면 **401**. 전체 필드(면접시각·합격 포함, 시각 무관) |
| `/api/admin/set` | 세션 쿠키 | `{phone, interview_time?, result?}` | `{ok:true}` | 특정 지원자 수정 |
| `/api/admin/settings` | 세션 쿠키 | `{settings}` (쓰기) 또는 `{}` (읽기) | `{ok, settings}` | 회차 공통 설정 읽기/쓰기 |
| `/api/admin/cohort` | 세션 쿠키 | `{cohort, activate?}` | `{ok}` | 새 회차 생성/활성 전환 (매 학기 재사용, §12) |

**응답 필드 이름:** 프런트 변경을 줄이려면 `/api/me`·`/api/admin/list`의 지원자 객체를 현재 프런트가 쓰는 카멜케이스 키로 내려주는 게 편합니다:
`{name, gender, phone, email, birth, dept, exp, avail, hobby, note, submittedAt, time, rsvp, rsvpAt, rsvpNote, result}`
(DB의 `interview_time`→`time`, `submitted_at`→`submittedAt` 매핑은 핸들러에서.)

☑ **완료 기준:** 각 엔드포인트가 위 계약대로 응답. `curl`로 왕복 확인.

---

## 6. 보안 규칙 상세

### 6.1 시각 판정 (KST, 서버 권위)

- 설정의 `reveal_at` 등은 `'2026-09-18 20:00'` 형태. **Asia/Seoul(+09:00)** 로 해석.
- 판정: 저장 문자열에 타임존이 없으면 `+09:00`을 붙여 `Date.parse` → `Date.now()`와 비교. (기존 프런트 `ts()` 로직과 동일 규칙, `worker/lib/time.ts`로 이식.)
- `/api/me` 직렬화:
  - `now >= reveal_at` 일 때만 결과 객체에 `time` 키 추가.
  - `now >= result_at` 일 때만 `result` 키 추가.
  - **빈 값이 아니라 키 자체가 없어야 함.** (검증 항목 §11)

### 6.2 운영진 인증 (서버)

- secret `ADMIN_PASSWORD`(평문) 또는 `ADMIN_PASSWORD_SHA256`(hex). 비교는 **상수 시간**(`crypto.subtle` 또는 길이 확인 후 XOR 누적).
- 성공 시 **서명된 세션 토큰**을 HttpOnly 쿠키로:
  - 쿠키: `gb_admin=<token>; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=7200`
  - 토큰 = `base64url(payload) + '.' + base64url(HMAC-SHA256(payload, SESSION_SECRET))`, `payload = {exp: epoch+7200}`.
  - 검증: 서명 확인 + `exp` 확인. (D1 세션 테이블 없이 무상태로 가능.)
- 모든 `/api/admin/*`(login 제외)은 쿠키 검증 실패 시 **401**.
- 클라이언트에는 비밀번호/해시/토큰 payload 어떤 것도 두지 않음 — 쿠키는 HttpOnly라 JS에서 못 읽음.

### 6.3 최소 노출

- `/api/me`는 **본인 1건**만. 다른 사람 필드 절대 금지.
- `/api/settings`는 공개 안전한 필드만(비밀번호/엔드포인트/토큰 없음).
- 브라우저 개발자도구 Network 탭에서 응답 본문에 불필요 데이터가 없어야 함(§11).

☑ **완료 기준:** §11 검증 4항목 통과.

---

## 7. 레이트리밋

`/api/me`, `/api/me/update`, `/api/me/rsvp`, `/api/admin/login`은 이름+전화 또는 비번만으로 열리므로 무작위 대입 대상.

- 키: 클라이언트 IP는 `request.headers.get('CF-Connecting-IP')`.
- 정책(예시): IP당 분당 시도 **10회** 초과 시 **429**. 실패가 반복되면 **지연**(예: 연속 실패 5회↑면 처리 전 `await sleep(500~2000ms)`).
- 저장소 우선순위:
  1. **Workers KV** — 키 `rl:<scope>:<ip>:<yyyyMMddHHmm>`, `expirationTtl: 90`, 값=카운트. 가장 단순.
  2. KV 미사용 시 **D1 `rate_limits`** 테이블(§4) upsert + 만료 청소.
- `/api/admin/login` 실패는 더 엄격하게(분당 5회).

☑ **완료 기준:** 같은 IP로 빠르게 반복 호출 시 429가 뜬다.

---

## 8. 데이터 마이그레이션 (일회성)

이번 학기 데이터는 몇 건뿐 → **Sheets에서 CSV 내보내기 → D1 import** 로 충분.

- Google Sheets에서 "지원자" 시트를 CSV로 내보냄. 열 순서(원본):
  `성함, 성별, 전화번호, 이메일, 생년월일, 학과/학위 및 재학학기, 배드민턴 경험, 참여 가능, 취미, 유입 경로, 하고 싶은 말, 제출시각, 면접시각, 참석여부, 참석확인시각, 불참/변경 사유, 합격여부`
- `scripts/import-csv.mjs`:
  - CSV 파싱(따옴표/개행 처리) → 각 행을 `applicants` INSERT 문으로 변환.
  - `phone_norm = 숫자만`. `참석여부` `참석`→`yes`/`불참`→`no`/그외 `''`.
  - `cohort`는 인자로(`--cohort 2026-fall`).
  - 출력: `wrangler d1 execute <DB> --file=out.sql` 로 넣을 SQL, 또는 D1 HTTP API 직접 호출.
- 실행: `node scripts/import-csv.mjs applicants.csv --cohort 2026-fall > migrations/data_2026fall.sql` → `wrangler d1 execute ...`.

☑ **완료 기준:** 기존 지원자 수 = D1 `applicants` 행 수. 전화번호 정규화 확인.

---

## 9. 프런트엔드 배선 교체

**원칙:** 화면 흐름·문구·레이아웃 그대로. 데이터 주고받는 부분만 교체. `index.html`은 재작성 금지(§2.1 편집 방법).

### 9.1 삭제할 것

- `const ADMIN_HASH = ...`
- `const APPS_SCRIPT = [...]` 전체 및 이를 쓰는 UI: 운영진 "구글 시트 연동" 섹션(엔드포인트 입력 `sf.endpoint` input, 스크립트 복사/토글 버튼, `appsScript`/`showScript`/`copyScript`/`toggleScript`/`linked`/`linkLabel`/`sEndpoint` 관련 `renderVals` 필드와 마크업).
- `this.props.sheetUrl` 주입 및 `state.settings.endpoint` 사용 전부.
- `persist()`가 지원자 브라우저에 전체 `applicants`를 저장하는 동작(§9.4).

### 9.2 `api()` 분리 → 동일출처 `/api`

현재 단일 `api(payload)`를 엔드포인트별 얇은 래퍼로 교체:

```js
async function call(path, body, opts={}) {
  const r = await fetch('/api/' + path, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include',           // 관리자 쿠키 왕복
    body: JSON.stringify(body || {})
  });
  const data = await r.json().catch(() => ({}));
  return { status: r.status, ...data };
}
```

배선 매핑:

- `submit()` → `call('submit', { record })`; `status===409`면 중복 안내.
- `login()` → `call('me', { name, phone })`; 성공 시 **서버가 준 `me` 객체**로 화면 구성(로컬 `applicants` 검색 제거). 404면 "접수 내역 없음". 응답에 `time`/`result` 키가 **있을 때만** 해당 화면 노출.
- `saveEdit()` → `call('me/update', { name, phone, record })`; 403이면 "마감되어 수정 불가".
- `setRsvp()`/`pushRsvpNote()` → `call('me/rsvp', { name, phone, rsvp, rsvpNote })`.
- `syncNow()` → `call('admin/list', {})`; **401이면** 관리자 로그인 화면으로 유도.
- `adminLogin` 핸들러 → `call('admin/login', { password: s.af.pw })`; `status===200`이면 `adminOk=true`(쿠키는 자동 저장, JS는 토큰을 보관하지 않음). 아니면 오류.
- `setSlot()` → `call('admin/set', { phone, interview_time: time, result })`.
- `saveSettings()` → `call('admin/settings', { settings: sf })`; 읽기는 `call('admin/settings', {})`.
- 앱 시작 시 `call('settings', {})`로 공개 설정 로드(현재 `componentDidMount`의 시트 자동 로드 자리).

### 9.3 `resultPassed()` / `revealPassed()`

- 서버가 이미 필드를 생략하므로 **표시 판정에만** 사용. 로직은 남겨도 되지만 데이터 노출을 좌우하지 않음.
- `forceReveal`/`forceResult`(운영진 미리보기 토글)는 **남겨도 됨** — 단, 지원자 화면에서는 서버가 안 준 데이터를 억지로 만들 수 없으므로 진짜 유출로 이어지지 않음. 운영진 미리보기는 `admin/list`로 받은 전체 데이터에서 렌더.

### 9.4 `localStorage`

- **지원자 브라우저:** 전체 명단 저장 금지. 기껏해야 **본인 `me` 객체 1건**을 캐시로 저장 가능하되, 다음 `/api/me` 응답으로 덮어씀. `KEY`에 `applicants[]` 통째 저장하던 `persist()`는 제거/축소.
- **운영진 명단(`admin/list` 결과):** localStorage에 남기지 말고 **메모리(state)** 로만. (발표 전 합격여부가 디스크에 남지 않도록.)
- `SKEY`(설정)는 `/api/settings` 응답 캐시로만.

☑ **완료 기준:** 지원자 시크릿 탭에서 로그인해도 localStorage에 타인 정보/전체 명단/합격여부가 남지 않는다.

---

## 10. 배포

### 10.1 `wrangler.toml` (예시)

```toml
name = "gbird-apply"
main = "worker/index.ts"
compatibility_date = "2025-08-01"

[assets]
directory = "./public"
binding = "ASSETS"          # 정적 파일 서빙

[[d1_databases]]
binding = "DB"
database_name = "gbird"
database_id = "<wrangler d1 create 후 채움>"

# (선택) 레이트리밋용 KV
# [[kv_namespaces]]
# binding = "RL"
# id = "<wrangler kv namespace create 후>"
```

`worker/index.ts` 진입점 골격:

```ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    if (url.pathname.startsWith('/api/')) return route(request, env, url);
    return env.ASSETS.fetch(request);   // 그 외 정적 파일
  }
};
```

### 10.2 명령

```bash
npm i -D wrangler
npx wrangler d1 create gbird                       # database_id 받아 wrangler.toml에 기입
npx wrangler d1 migrations apply gbird --remote     # §4 스키마
npx wrangler d1 execute gbird --remote --file=migrations/seed_settings.sql
npx wrangler secret put ADMIN_PASSWORD              # 운영진 비밀번호
npx wrangler secret put SESSION_SECRET              # 세션 서명 키(랜덤 32B+)
npx wrangler deploy
```

### 10.3 GitHub 자동 배포

- Cloudflare 대시보드에서 이 저장소를 Worker에 연결하면 `main` 푸시 시 자동 배포. **또는** GitHub Actions에서 `cloudflare/wrangler-action`을 쓰고 리포 secret으로 `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`를 넣는다. 토큰/비밀은 저장소에 커밋하지 않는다.

### 10.4 기존 GitHub Pages / 포스터 QR

- **포스터 QR이 현재 GitHub Pages 주소를 가리킴.** 새 Workers 주소 확인 후, 기존 GitHub Pages `index.html`을 **리다이렉트 페이지로 교체**(`<meta http-equiv="refresh">` + JS `location.replace(...)`)해 옛 주소 방문자를 새 주소로 넘긴다. QR 재인쇄 전까지 유지.

☑ **완료 기준:** 배포 URL에서 `index.html`이 뜨고 `/api/*`가 동작. 옛 Pages 주소가 새 주소로 넘어감.

---

## 11. 검증 (반드시 통과)

- [ ] 시크릿 탭에서 지원서 제출 → D1에 정상 저장.
- [ ] 남의 이름+전화번호로 `/api/me` 호출 → **그 사람 것만** 나오고 타인 정보 없음.
- [ ] 운영진 세션 쿠키 없이 `/api/admin/list` 호출 → **401 거부**.
- [ ] `reveal_at` 이전 `/api/me` → 응답에 `time` 키가 **아예 없음**(빈 값 아님).
- [ ] `result_at` 이전 `/api/me` → 응답에 `result` 키가 **아예 없음**.
- [ ] 개발자도구 Network에서 지원자 응답 본문에 불필요 데이터 없음.
- [ ] 같은 IP 반복 호출 → 429 레이트리밋.
- [ ] `index.html` 안에 Apps Script 주소/전문/`ADMIN_HASH`가 **더는 없음**.

---

## 12. 매 학기 재사용

- 데이터에 `cohort`(예: `2026-fall`) 포함, 학기별 분리.
- 날짜·장소·공개/발표 시각은 운영진 화면에서 입력 → `/api/admin/settings`로 저장.
- 새 학기: 운영진 화면에서 `/api/admin/cohort`로 새 회차 생성·활성 전환 후 날짜만 입력. 코드 수정 불필요.

---

## 13. 참고

- 원본 지시서(문제 1~5, 목표, API 초안, 데이터 모델)는 최초 `HANDOFF.md`를 참조.
- 현재 `index.html` 운영진 화면의 Apps Script 전문은 데이터 구조 파악용으로만 참고하고, 이전 완료 후 §9.1대로 삭제.

# Global Daily News — 한국타이어 39개국 HR·시장 대시보드 (v4)

한국타이어 진출 **39개국**의 국가 프로필·환율·현지 뉴스·비즈니스/HR 동향을 매일 아침(KST 06시대) 자동 수집하여
**권역 탭 → 국가 칩** 클릭만으로 새로고침 없이 조회할 수 있는 대시보드를 GitHub Pages로 배포합니다.

---

## 1. 디렉토리 구조

```
├── requirements.txt
├── README.md
├── src/
│   ├── collector.py            # ① 환율·뉴스 수집 + 일괄 번역 + 아카이브 관리
│   └── build_site.py           # ② JSON을 템플릿에 임베드 → docs/index.html
├── template/
│   └── template.html           # Jinja2 셸 + 클라이언트 JS(권역 탭/국가 칩/차트/계산기)
├── data/
│   ├── countries_metadata.json # 39개국 기본 정보 (사람이 직접 관리하는 원본)
│   ├── countries_data.json     # 오늘자 수집 결과 (자동 생성)
│   └── translation_cache.json  # 영→한 번역 캐시 (자동 생성, 45일 보존)
├── docs/
│   ├── index.html              # 배포 산출물
│   └── archive/
│       ├── index.json          # 아카이브 날짜 목록 (날짜 피커 min/max)
│       └── YYYY-MM-DD.json     # 일별 스냅샷 (180일 보존)
└── .github/workflows/update.yml
```

**흐름**: `countries_metadata.json` → `collector.py` → `data/countries_data.json` + `docs/archive/{오늘}.json`
→ `build_site.py` → `docs/index.html` → GitHub Pages

---

## 2. 데이터 출처

| 항목 | 출처 | 갱신 |
|---|---|---|
| 총인구·GDP·물가·실업률·최저임금·거점 | `output/country_profile.json` (국가 기본 프로필 파이프라인) — 값이 없으면 `countries_metadata.json` | 월 1회 / 연 1회 |
| 수도 | `data/countries_metadata.json` | 사람이 직접 수정 |
| 환율 (현지통화/KRW, 1년 월말 추이) | Yahoo Finance (`yfinance`) | 매일 |
| 현지 주요 뉴스 / 비즈니스 동향 | Google News RSS | 매일 |
| 영문 기사 번역 | Google 번역 공개 엔드포인트(타임아웃 10초), 실패 시 MyMemory | 매일 (캐시) |

### 메타데이터 기준일 표시
`countries_metadata.json`을 아래처럼 객체 형식으로 바꾸고 `as_of`를 적으면 프로필 카드 하단에 "(2026-10-01 기준)"이 표시됩니다.
리스트 형식(현재)도 그대로 동작하며, 이 경우 기준일 문구는 숨겨집니다. 국가별로 `"as_of"`를 넣으면 그 값이 우선합니다.

```json
{ "as_of": "2026-10-01", "countries": [ { "code": "KR", ... }, ... ] }
```

---

## 3. 수집 로직 요약

### 환율
- 필요한 티커 전체를 `yf.download` 한 번으로 일괄 수집(호출 수 ↓, 차단 위험 ↓), 빠진 티커만 개별 재시도.
- USD·PAB(1:1 고정)는 `USDKRW=X`를 직접 사용.
- 그 외 통화는 직접 페어 `{CUR}KRW=X`와 USD 교차값(`{CUR}USD=X × USDKRW=X`, **같은 거래일끼리만** 곱함)을 비교해
  직접 페어가 30거래일 미만이거나, 교차값과 5% 이상 어긋나거나, 5일 이상 늦으면 교차값을 채택.
- **100단위 표기**: JPY·VND·IDR·HUF·KZT·CLP·COP.
- **소수 자릿수 자동**: 표시값 ≥100 → 1자리, ≥10 → 2자리, 그 미만 → 3자리 (`exchange_rate.decimals`).
- 수집 실패 시 이전 수집값을 쓰고 출처에 "· 이전 수집값"을 붙임. 이전 값도 없으면 "수집 실패"로 표시(추정치 곡선을 만들지 않음).

### 뉴스
- **한국**: 국내 Google News 상위 기사 / 국내 전용 쿼리.
- **해외 38개국**: `"국가명" (economy OR government OR …) when:3d` 영문 검색 → 부족하면 조건을 완화해 재검색.
- **관련성 필터**: `COUNTRY_NEWS_FILTERS`에 정의한 국가 별칭(국명·형용사·수도 등)이 **제목에** 있어야 채택.
  동명 지명/동음이의어는 제외어와 거부 정규식으로 걸러냄 (예: Panama City FL, chile pepper, Love is Blind NL, New Mexico).
- **비즈니스 동향**: 한국어 `"국가명" (노동 OR 임금 OR …)` 우선 → 부족분은 영문 `"Country" ("labor law" OR …)`로 채움.
- 링크 + 정규화 제목 기준 중복 제거, HTML 엔티티(&amp;nbsp; 등) 정리, 의미 없는 요약(제목 반복) 제거, 발행일 표시.
- 이전 실행의 기사를 재사용할 때도 같은 관련성 필터를 다시 적용.

### 번역
- 수집이 모두 끝난 뒤 영문 항목을 모아 **묶음 번역**(최대 20문장/1,500자씩) → 줄 수가 어긋나면 단건 번역.
- Google 실패 시 MyMemory, 단건 번역이 5건 연속 실패하면 중단하고 다음 실행에서 재시도.
- 결과는 `data/translation_cache.json`에 저장되어 같은 기사는 다시 번역하지 않음.

### 시간 예산 (워크플로 20분 제한 대비)
| 단계 | 기본 예산 | 넘으면 |
|---|---|---|
| 환율 개별 재시도 | 180초 (`FX_BUDGET_SEC`) | 남은 티커는 이전 수집값 |
| 뉴스 수집 | 480초 (`NEWS_BUDGET_SEC`), 오류 12회 연속 시 중단 | 남은 국가는 이전 기사 |
| 번역 | 240초 (`TRANSLATION_BUDGET_SEC`) | 남은 기사는 영문 원문 |

### 실행 로그 품질 요약
실행이 끝나면 다음이 로그에 남습니다: 환율 최신값을 못 받은 국가, 헤드라인 폴백 국가, 미번역 기사 수.

---

## 4. 로컬 실행

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python src/collector.py
python src/build_site.py
python -m http.server --directory docs   # http://localhost:8000
```

> `file://`로 열면 브라우저 보안 정책 때문에 과거 날짜(아카이브) 조회가 동작하지 않으니 위처럼 로컬 서버로 여세요.

테스트용 환경변수:
- `DISABLE_TRANSLATION=1` : 번역 호출 생략
- `OFFLINE=1` : 네트워크 호출 전부 생략 (캐시/폴백 경로만 실행)

---

## 5. 자동 실행 (GitHub Actions)

`.github/workflows/update.yml`
- **주 실행** `7 21 * * *` (KST 06:07), **백업 실행** `37 22 * * *` (KST 07:37).
  GitHub 스케줄은 정각에 몰려 지연·누락이 잦아 분을 비켜 두었고, 백업 실행은 오늘 아카이브가 이미 있으면 수집을 건너뜁니다.
- `workflow_dispatch`로 수동 실행 가능, `main` push 시에도 실행(README만 바뀐 경우 제외).
- 작업 30분·수집 20분 타임아웃, push 전 `git pull --rebase` + 3회 재시도.
- 커밋이 실패해도 Pages 배포는 진행.

### 최초 설정
1. Settings → Pages → Source: **GitHub Actions**
2. Settings → Actions → General → Workflow permissions: **Read and write**
3. Actions 탭에서 `Daily Global Brief Update` → **Run workflow**로 1회 수동 실행

> 60일 동안 저장소에 활동이 없으면 GitHub이 스케줄 실행을 자동 중지합니다. 매일 데이터 커밋이 생기므로 보통 문제없지만,
> 수집이 연속 실패하면 Actions 탭에서 다시 활성화하세요.

---

## 6. 국가/키워드 추가·수정

- **국가 추가**: `countries_metadata.json`에 항목을 넣고, `collector.py`의 `COUNTRY_NEWS_FILTERS`에 별칭을 추가하세요
  (별칭이 없으면 영문 국가명만으로 필터링합니다).
- **엉뚱한 기사가 섞이면**: 해당 국가의 `exclude`(검색 제외어)나 `reject`(제목 거부 정규식)에 패턴을 추가.
- **동향 키워드 변경**: `INDUSTRY_TOPICS`의 `kr_keywords` / `en_keywords` / `home_query`.
- **100단위 통화 변경**: `SMALL_UNIT_CURRENCIES`. 이전 캐시 값은 자동으로 새 단위로 환산됩니다.

---

## 7. 보안

뉴스 제목·요약·출처 등 외부 문자열은 화면 출력 시 모두 HTML 이스케이프되며, 링크는 `http(s)`만 허용합니다.

---

## 8. 국가 기본 프로필 자동 수집과의 연결

프로필 카드의 총인구·GDP·인플레이션·실업률·최저임금·주요 거점은 `country_profile` 파이프라인이 채웁니다
(자세한 내용: [README_country_profile.md](README_country_profile.md)).

```
매월 1일 09:00 KST  profile-monthly.yml → 인플레이션·실업률 수집 → output/country_profile.json 커밋 → update.yml 실행
매년 1월 1일 09:00  profile-annual.yml  → 총인구·GDP·최저임금 수집  → output/country_profile.json 커밋 → update.yml 실행
매일 06:07         update.yml          → collector.py 가 output/country_profile.json 값을 프로필 카드에 반영
```

- 자동 수집 값이 아직 없는 항목('-')은 기존 `data/countries_metadata.json` 값이 그대로 표시됩니다.
- 수도는 한글 표기를 위해 `countries_metadata.json` 값을 유지합니다.
- 프로필 카드 하단 출처 문구는 자동 수집 값을 쓴 국가에서 "IMF WEO · Trading Economics · Eurostat/ILOSTAT · HR 보정자료 (생성일 기준)"으로 바뀝니다.
- ⚠️ 이 저장소가 **공개(Public)** 라면 `config/min_wage_overrides.csv`, `data/`, `output/`도 공개됩니다.

---

## 9. 화면 구성 (v5: 오늘의 브리핑 + 국가 상세)

| 화면 | 주소 | 내용 |
|---|---|---|
| 오늘의 브리핑 | `#/briefing` (기본) | 조직(권역) 카드 필터 · 안전 확인 필요 보도 · HR·노동 주요 소식(법·제도 우선) · 글로벌 완성차·타이어 동향 |
| 국가 상세 | `#/country/DE` | 권역 탭 + 국가 칩(접기 가능) · 국가 개요/거점 · 환율·물가·실업률·최저임금 · 환율 계산기 · HR·노동 · 현지 주요 뉴스(정치·경제/사회·문화/안전·재난) · 공식 기관 · 완성차·타이어 |

### 뉴스 분류 (collector.py `NEWS_CATEGORIES`)
- **안전·재난**: 제목이 `SAFETY_TAGS`(지진·자연재해·무력충돌·테러·치안·시위·소요·환경·보건) 중 하나에 걸려야 채택하고 태그를 붙입니다.
  훈련·기념일·영화 등은 `SAFETY_EXCLUDE`로 제외합니다. 화면에는 '공식 경보가 아닌 확인이 필요한 보도'로 표시됩니다.
- **정치·경제 / 사회·문화**: 분류별 검색어 + 국가 별칭 필터. 새 기사가 없으면 이전 기사, 그것도 없으면 '보도 없음'.
- **HR·노동**: 기존 수집 + 제목에 법·개정·최저임금 등이 있으면 `법·제도` 배지(`LAW_RE`).
- **글로벌 완성차·타이어**: 국내 타이어·완성차 + 경쟁사(Michelin·Bridgestone 등) + 시장 전망 검색, 주제 태그 자동 부착.

### 공식 기관·보고서 (`config/official_sources.csv`)
`iso3|name|url|desc` 형식. `iso3`에 `ALL`은 전체 국가, `EU`는 EU 회원국에 공통으로 붙습니다.
현재 목록은 **초안**입니다 — 각국 담당자가 주소와 기관명을 확인해 보완해 주세요.

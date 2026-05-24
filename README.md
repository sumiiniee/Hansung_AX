# 한성대학교 규정 마스터 AI

> 2026 한성대학교 AX 프론티어 공모전 출품작

한성대학교의 **모든 규정(315편 · 1,618개 버전)** 을 학습한 RAG 챗봇입니다. 학생·교직원은 자연어로 질문하면 근거 조항과 담당부서·직통번호까지 함께 답변받고, 관리자는 회의자료 한 장만 올리면 AI가 개정안을 자동 추출해 미리보기 비교 후 한 번에 반영할 수 있습니다.

🔗 [바로 체험하기](https://web-production-90839.up.railway.app)

---

## 핵심 기능

| 기능 | 설명 |
|---|---|
| 규정 검색·답변 | 자연어 질문 → 근거 조항·담당부서·전화번호 포함 답변 (SSE 스트리밍) |
| 편(編)별 규정 목록 | 제1편 학교법인 ~ 제8편 학생군사교육단까지 사이트 트리 그대로 분류 |
| 개정 이력 추적 | 모든 규정의 과거 버전 보관, 신구조문 대비 |
| 규정 추가 | PDF/DOCX/HWP/HWPX 업로드 → 자동 청크화·임베딩·DB 등록 |
| 규정 개정 | 전체/조 단위 개정. 좌(현행) vs 우(개정안) 하이브리드 diff 미리보기 후 적용 |
| 회의록 기반 AI 개정 | 회의자료 → AI가 양식 판단·규정 매칭·개정안 추출 → 한 카드로 정리 |
| 일괄 치환 | 전 규정 단어 동시 변경. 영향 받는 규정 선택해 적용 |
| 활동 로그 | 추가/개정/일괄치환 시간순 피드. 추가는 삭제, 개정·치환은 되돌리기 |
| 충돌 분석 | 신규 규정 파일 업로드 → Claude가 기존 규정과의 모순·중복 분석 |
| AI 형식 검사 | 한성대 표준 형식(`제 N 조`, `①`, `1.` 등) 자동 점검 |

---

## 챗봇 검색 흐름

질문이 들어오면 다음 6단계로 답변을 만듭니다:

```
질문 입력
   ↓
[1] Claude가 키워드 확장 (동의어·관련 용어 5~10개)
[2] Upstage 임베딩(4096차원) → pgvector 검색으로 TOP 30 후보
[3] 키워드 LIKE 보강 (벡터 검색이 놓친 조항 추가)
[4] Voyage rerank → 진짜 관련 8개 선별
[5] 부서 매핑 + 컨텍스트 구성
[6] Claude SSE 스트리밍 답변 (글자 단위 실시간 표시)
```

답변 결과에는 본문 + 📞 담당부서·직통번호 + 📚 참조 조항 카드 + 💡 이어서 물어보기 + ⬇ PDF/Word 내보내기가 함께 나옵니다. 개정 비교 질문일 때는 🔍 버전 비교 모달 버튼도 추가됩니다.

---

## 규정 개정 — 미리보기 + 하이브리드 Diff

전체 규정 또는 조 단위로 개정 가능합니다.

1. 본문 직접 입력 또는 개정 파일 업로드 (조 단위 개정은 AI가 개정안 본문만 자동 추출)
2. **🔍 미리보기** 클릭 → 좌(현행) vs 우(개정안) **diff 모달**
3. **이대로 적용** → 옛 버전을 `revision_backups/`에 자동 저장 후 신 버전을 DB에 반영
4. 활동 로그에 ✏️(전체) 또는 📌(조 단위)로 기록됨 → 언제든 ↩️ 되돌리기 가능

### 어절+글자 하이브리드 Diff

차이 표시를 자연스럽게 보여주기 위해 두 단계로 처리합니다:

- **어절(공백 단위)** 로 먼저 diff
- 변경된 어절이 **숫자·구두점 섞인 토큰**(날짜·코드 등)이면 → **통째로** 빨강/파랑
- 변경된 어절이 **순한글**(조사·어미 변화)이면 → 어절 안에서 **글자 단위**로 더 세밀하게

| 예시 | diff 결과 |
|---|---|
| `(2026.4.10.)` ↔ `(2026.0.00.)` | 토큰 통째로 강조 |
| `촉진시키는` ↔ `촉진하는` | `시키` ↔ `하` 두 글자만 강조 |
| `이 연구원은` ↔ `본 연구원은` | `이` ↔ `본` 한 글자만 |

---

## 회의록 기반 AI 개정

회의자료(HWP·PDF·DOCX) 한 장이나 텍스트만 있으면 AI가 4단계로 처리합니다:

```
회의자료 업로드 (또는 텍스트 직접 입력)
   ↓
① AI가 양식 판단        — 표 양식인가, 줄글 양식인가?
   ↓
② 1차 규정 매칭         — 어떤 규정의 개정인지 history.json과 매칭
   2차 개정안만 추출     — 현행 본문은 건드리지 않음 (시스템에 정답이 있음)
   ↓
③ 시스템에서 현행 자동 로드 — history.json의 최신 버전 본문
   ↓
④ 통합 미리보기         — [시스템 현행 전체] vs [AI 추출 개정안 합성본]
                          하이브리드 diff로 변경된 부분만 강조
```

### HWP 표 파싱

HWP의 "현행규정 | 개정(안)" 두 컬럼 표는 다음 순서로 분리합니다:

- `hwp5html`로 HWP → HTML 변환 → `<table><tr><td>` 그대로 보존 → 헤더 표를 골라 셀별 좌/우 분리
- 위가 실패하면 `hwp5proc xml`로 XML 텍스트 노드 추출
- 그것도 실패하면 텍스트 패턴(`제 N 조`가 두 번 등장하는 구조)으로 좌/우 분리

분리된 좌/우 셀이 AI 프롬프트에 명시적으로 전달돼서 AI가 현행과 개정안을 헷갈리지 않습니다.

### 통합 카드

한 안건에 여러 조의 변경이 있어도 카드는 1개로 묶입니다:

- 시스템 현행을 조 단위로 분리
- AI가 추출한 조마다 같은 조를 교체
- 회의록에 없는 조는 시스템 현행 그대로 유지
- 신설된 부 칙·신규 조는 끝에 추가

카드에는 💡 개정 사유, 🔵 변경/신설된 조 목록, ▸ AI 추출 본문 보기·수정, 🔍 미리보기 버튼이 모여 있습니다. 미리보기를 누르면 시스템 현행 전체 vs 합성된 개정안 전체를 하이브리드 diff로 비교합니다.

---

## 활동 로그 사이드바

관리자 페이지 우측에 활동 피드가 깃허브 커밋 스타일로 표시됩니다.

| 종류 | 아이콘 | 색상 | 액션 |
|---|---|---|---|
| 규정 추가 | ➕ | 초록 | 🗑 삭제 (DB + 업로드 파일 제거) |
| 전체 규정 개정 | ✏️ | 파랑 | ↩️ 되돌리기 |
| 조 단위 개정 | 📌 | 파랑 | ↩️ 되돌리기 |
| 일괄 치환 | 🔁 | 호박 | ↩️ 되돌리기 |

시간순으로 통합되고, 사이드바는 토글 가능합니다(상태는 localStorage에 저장).

---

## AI 스택

| 단계 | 모델 | 역할 |
|---|---|---|
| 임베딩 | Upstage `solar-embedding-1-large` (4096차원) | 한국어 특화 벡터 변환 |
| 재순위화 | Voyage `rerank-2` | 임베딩 후보 중 진짜 관련 조항 선별 |
| 생성 | Anthropic `claude-sonnet-4-5` | 답변 · 충돌 분석 · 형식 검사 · 회의록 분석 · 텍스트 정돈 |
| Fallback | Groq `llama-4-scout-17b` | Claude 장애 시 자동 백업 |

---

## 보안·정확성

- **JWT 인증**: 관리자 토큰 24시간 유효, 매 요청 서버 검증
- **부서명 검증**: AI가 임의로 만든 부서명은 원본 source의 실제 부서명으로 자동 교체
- **출처 인용 강제**: 답변에 인용되지 않은 조항은 참조 카드에서 자동 제거
- **개정 미리보기**: 실제 DB 반영 전 하이브리드 diff로 확인 → 오타·실수 차단
- **자동 백업**: 모든 개정·치환은 `revision_backups/`에 보관 → 활동 로그에서 1-click 롤백
- **회의록 분석**: AI에게 개정안만 추출시키고 현행은 시스템에서 가져옴 → AI가 현행을 잘못 가져오는 실수 자체가 발생할 수 없음
- **업로드 규정 자동 백필**: DB의 `upload://*` 규정이 history.json에 누락돼 있으면 회의록 분석 직전 자동 등록 → 회의록 매칭 후보로 포함

---

## 기술 스택

| 영역 | 기술 |
|---|---|
| 백엔드 | FastAPI · psycopg2 · python-multipart · PyJWT |
| DB | PostgreSQL 16 + `pgvector` (4096차원 벡터) |
| 프론트 | 정적 HTML + Vanilla JS (Server-Sent Events) |
| 문서 파싱 | pdfplumber · python-docx · pyhwp(`hwp5txt`·`hwp5proc xml`·`hwp5html`) · lxml · BeautifulSoup4 |
| 크롤러 | requests · BeautifulSoup4 |
| 컨테이너 | Docker (PostgreSQL) |

---

## 프로젝트 구조

```
HSU/
├── server.py                       # FastAPI 메인 (모든 엔드포인트)
├── routers/teacher.py              # 충돌 분석 + PDF/Word 내보내기
├── crawler.py                      # rule.hansung.ac.kr 크롤러
├── make_rules_json.py              # history → 최신판 hansung_rules.json
├── build_db.py                     # Upstage 임베딩 + DB 적재
├── index.html                      # 메인 (챗봇 + 규정 목록)
├── upload.html                     # 관리자 페이지
├── login.html                      # 관리자 로그인
├── hansung_rules_history.json      # 315 규정 × 평균 5버전
├── hansung_rules.json              # 최신판
├── dept_phones.json                # 79개 부서 직통번호
├── uploads/                        # 업로드된 원본 파일
├── revision_backups/               # 개정 자동 백업
├── requirements.txt
└── .env                            # API 키 + DB URL (gitignore)
```

---

## 설치 및 실행

### 1. 사전 요구사항
- Python 3.11+
- Docker Desktop
- Windows: `pip install pyhwp` 후 `hwp5txt --help` 동작 확인

### 2. 클론
```bash
git clone https://github.com/sumiiniee/Hansung_AX.git
cd Hansung_AX
```

### 3. 환경 변수 (`.env`)
```env
ANTHROPIC_API_KEY=sk-ant-...
UPSTAGE_API_KEY=up_...
VOYAGE_API_KEY=pa-...
GROQ_API_KEY=gsk_...
DATABASE_URL=postgresql://postgres:비밀번호@localhost:5432/hansungrules
ADMIN_ID=admin
ADMIN_PW=비밀번호
SECRET_KEY=랜덤_32자_이상
```

### 4. 패키지 설치
```bash
pip install -r requirements.txt
```

### 5. PostgreSQL 컨테이너
```bash
docker run -d --name hansung-db \
  -e POSTGRES_PASSWORD=비밀번호 \
  -e POSTGRES_DB=hansungrules \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

### 6. DB 빌드 (15~30분)
```bash
python build_db.py
```

### 7. 서버 실행
```bash
python server.py
```

- 메인: `http://127.0.0.1:8000`
- 관리자: `http://127.0.0.1:8000/upload`

---

## 사용 예시

### 학생/교직원 — 챗봇

추천 카드 클릭 또는 직접 질문:
- "수강신청 방법을 알려주세요"
- "휴학 가능한 기간이 어떻게 되나요?"
- "미래플러스대학 학사운영 규정 개정 이력이 어떻게 되나요?"
- "장학금 신청 절차를 알려주세요"

### 관리자 — 규정 관리

`/upload` 페이지에서 5개 탭을 제공합니다.

**① 규정 추가**
PDF/DOCX/HWP 업로드 → 자동 텍스트 추출 → 청크화 → 임베딩 → DB 등록 → 활동 로그에 ➕ 표시

**② 규정 개정**
본문 직접 입력 또는 개정 파일 업로드(조 단위는 AI가 개정안 자동 추출) → 미리보기 모달에서 하이브리드 diff 확인 → 이대로 적용

**③ 회의록 기반**
- 입력 방식 토글: 📎 파일 / ✏️ 직접 입력
- 파일: HWP·PDF·DOCX·HWPX (표 양식 + 줄글 양식 모두 지원)
- 직접 입력: 텍스트 + 선택적 대상 규정 지정
- 결과: 안건당 카드 1개("📌 전체 규정 개정 — N개 조 변경/신설")

**④ 일괄 치환**
"옛 단어 → 새 단어" 입력 → 영향 받는 규정 미리보기 → 체크박스로 선택해 적용

**⑤ 충돌 분석**
신규/개정 규정 파일 업로드 → Claude가 기존 315편과 비교 → 모순·중복 보고서

---

## 데이터 현황

| 항목 | 수치 |
|---|---|
| 규정 수 | 315편 |
| 총 버전 수 | 1,618개 |
| 첨부 텍스트 | 315건 |
| DB 청크 | 약 43,000개 |
| 부서 매핑 | 79개 |
| 편 분류 | 315/315 (100%) |

---

## 주요 엔드포인트

| 메서드 | 경로 | 설명 |
|---|---|---|
| POST | `/query-stream` | 챗봇 답변 (SSE) |
| POST | `/query` | 챗봇 답변 (JSON) |
| POST | `/diff` | 버전 간 하이브리드 diff |
| GET | `/rules` | 편별 규정 목록 |
| POST | `/search-rules` | 규정 키워드 검색 |
| POST | `/upload-regulation` | 규정 추가 |
| POST | `/revise-preview` · `/revise-regulation` | 전체 개정 |
| POST | `/revise-article-preview` · `/revise-article` | 조 단위 개정 |
| POST | `/extract-meeting-revisions` | 회의록 파일 분석 |
| POST | `/extract-meeting-revisions-text` | 회의록 직접 입력 분석 |
| POST | `/bulk-replace/preview` · `/bulk-replace/apply` | 일괄 치환 |
| POST | `/conflict/analyze` | 충돌 분석 |
| POST | `/format-check` | AI 형식 검사 |
| GET | `/revision-backups` | 활동 로그 |
| POST | `/revision-rollback` | 되돌리기 |
| DELETE | `/uploaded-rules/{filename}` | 업로드 규정 삭제 |
| GET | `/uploads/{filename}` | 업로드 원본 파일 보기/다운로드 |

---

## 라이선스

학내 사용 + 공모전 제출용. 상업적 이용 금지.

---

**2026 한성대학교 AX 프론티어 공모전**

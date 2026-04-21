# Webex Calling & Message: 한국형 조직도 기반 연락처 서비스 (wxc-d-svc)

**1. 배경 및 문제점**
현재 Webex Calling 및 Message의 기본 연락처 검색 방식은 이름(Name) 검색에 집중되어 있습니다. 하지만 국내 기업 환경에서는 이름 외에도 **조직도(Org Chart) 기반의 탐색**이나 **휴대폰 번호, 내선 번호** 등 다양한 정보를 활용한 검색 방식이 필수적으로 요구됩니다..

**2. 기술적 도전 과제**
단순히 AD(Active Directory)에서 가져온 기본 정보만으로는 국내 기업 특유의 복잡하고 세분화된 조직 구조를 완벽하게 표현하기 어렵습니다. 특히 대규모 조직일수록 다음과 같은 관리가 필요합니다.
*   **계층화 설계:** 조직 단위를 Tier별로 세분화한 별도의 데이터베이스(DB) 구축 필요.
*   **정교한 관리:** 조직 규모에 비례하여 늘어나는 복합적인 조직 트리 구조를 단순하면서도 정확하게 구현.

**3. 해결 방안: wxc-d-svc (Webex Directory Service)**
이러한 요구사항을 충족하기 위해 별도의 DB를 통해 체계적인 조직 관리 기능을 제공합니다.
*   **Multi-Tier 구조:** 조직을 Tier 1, 2, 3으로 나누어 복잡한 구조를 명확하게 반영.
*   **다양한 검색 및 필터링:** 이름, 이메일은 물론 내선 및 휴대폰 번호 기반 검색 지원.
*   **관리 편의성:** CSV 대량 임포트 및 관리자 패널을 통한 효율적인 데이터 유지보수.

---

**관련 소스 코드:**  
🔗 [GitHub - nexpert-code/wxc-d-svc](https://github.com/nexpert-code/wxc-d-svc)




# Webex Directory Service (wxc-d-svc)

Webex Bot을 인증 수단으로 활용하는 **기업 임직원 디렉터리 서비스**입니다.  
FastAPI 백엔드 + React 프론트엔드(빌드 결과물 포함)로 구성되어 있으며, Docker 한 장으로 바로 실행할 수 있습니다.

---

## 주요 기능

| 기능 | 설명 |
|------|------|
| **Webex OTP 로그인** | 이메일 입력 → Webex Adaptive Card 수신 → 버튼 클릭으로 승인 |
| **직원 디렉터리** | Tier1 → Tier2 → Tier3 드릴다운 탐색 + 이름/이메일 검색 |
| **관리자 패널** | CSV 일괄 임포트, 직원 추가·수정·비활성화, 로그인 통계 |
| **역할 기반 접근** | viewer / editor / admin 3단계 권한 |
| **세션 관리** | HttpOnly 쿠키 + itsdangerous 서명 (CSRF 방어) |

---

## 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                    Browser (React SPA)                  │
└────────────────┬───────────────────────┬────────────────┘
                 │  REST API             │  Static Files
┌────────────────▼───────────────────────▼────────────────┐
│                   FastAPI Application                   │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  auth.py    │  │employees.py  │  │   admin.py    │  │
│  │             │  │              │  │               │  │
│  │ POST /auth/ │  │ GET /api/    │  │ POST/PATCH/   │  │
│  │   request   │  │   browse     │  │ DELETE        │  │
│  │ GET /auth/  │  │ GET /api/    │  │ /api/employees│  │
│  │   poll/{id} │  │   employees  │  │ POST /api/    │  │
│  │ POST /auth/ │  │ GET /api/    │  │   import/csv  │  │
│  │   webhook   │  │   orgs       │  │               │  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬───────┘  │
│         │                │                  │          │
│  ┌──────▼──────────────────────────────────▼───────┐  │
│  │              SQLAlchemy (Async)                  │  │
│  │      employees / admin_users / pending_auth      │  │
│  │           import_logs / login_logs               │  │
│  └──────────────────────┬───────────────────────────┘  │
│                         │                              │
└─────────────────────────┼──────────────────────────────┘
                          │
          ┌───────────────▼──────────────┐
          │  PostgreSQL (운영)            │
          │  SQLite (개발/로컬)           │
          └──────────────────────────────┘
                          
          ┌───────────────────────────────┐
          │       Webex Bot API           │
          │  (로그인 승인 카드 발송)        │
          └───────────────────────────────┘
```

---

## 프로젝트 구조

```
wxc-d-svc/
├── app/
│   ├── main.py          # FastAPI 앱 진입점, lifespan(초기 시드)
│   ├── config.py        # pydantic-settings 환경변수 설정
│   ├── database.py      # SQLAlchemy 비동기 엔진 + 세션
│   ├── models.py        # ORM 모델 (Employee, AdminUser, PendingAuth, …)
│   ├── schemas.py       # Pydantic 입출력 스키마
│   ├── routers/
│   │   ├── auth.py      # 인증 플로우 (요청 → Webex 카드 → 폴링 → 세션)
│   │   ├── employees.py # 직원 조회 API (browse, search, orgs)
│   │   └── admin.py     # 관리자 API (CRUD, CSV 임포트, 통계)
│   └── utils/
│       ├── session.py   # HttpOnly 쿠키 세션 발급·검증
│       ├── webex.py     # Webex API 헬퍼 (카드 발송, 액션 조회)
│       └── seeder.py    # CSV → DB 임포트 유틸
├── static/              # React 빌드 결과물 (SPA fallback 포함)
├── data.csv.example     # CSV 임포트 포맷 예시 (실제 데이터 제외)
├── .env.example         # 환경변수 템플릿
├── Dockerfile           # 프로덕션용 Docker 이미지
└── requirements.txt     # Python 의존성
```

---

## 인증 플로우

```
사용자 (Browser)              FastAPI                    Webex Bot
     │                           │                           │
     │── POST /auth/request ──►  │                           │
     │   { email: "a@ex.com" }   │                           │
     │                           │── send_approval_card ──►  │
     │                           │   (Adaptive Card DM)      │
     │◄── { state_token, ... } ──│                           │
     │                           │                     사용자가 카드에서
     │   (2초마다 폴링)           │                     [승인] 버튼 클릭
     │── GET /auth/poll/{token}─► │                           │
     │                           │◄── POST /auth/webhook ────│
     │                           │    (attachmentActions)    │
     │                           │  → pending.status = approved
     │── GET /auth/poll/{token}─► │                           │
     │◄── { status: "approved" } │                           │
     │   Set-Cookie: kd_session  │                           │
     │                           │                           │
     │  이후 API 호출은 쿠키로 자동 인증                       │
```

### 핵심 보안 설계

- **도메인 제한**: `ALLOWED_EMAIL_DOMAIN`에 설정된 도메인만 로그인 허용
- **직원 DB 검증**: 디렉터리 DB에 등록된 이메일만 승인 요청 가능  
- **Rate Limit**: IP 당 N회 실패 시 M분 잠금 (메모리 기반, 운영 시 Redis 권장)
- **세션 서명**: `itsdangerous.URLSafeTimedSerializer`로 쿠키 위변조 방지
- **HttpOnly + SameSite=Strict**: XSS/CSRF 방어

---

## 데이터 모델

```
employees          admin_users        pending_auth
──────────         ───────────        ─────────────────
id (PK)            id (PK)            id (PK)
sort_order         email (UQ)         state_token (UQ)
tier1              role               email
tier2              created_at         status (pending/approved/rejected)
tier3                                 expires_at
title_ko           import_logs        created_at
name_ko            ───────────
name_en            id (PK)            login_logs
extension          filename           ──────────
mobile             uploaded_by        id (PK)
email (UQ)         rows_added         email
is_active          rows_updated       logged_in_at
created_at         rows_skipped
updated_at         created_at
```

### CSV 임포트 포맷

`data.csv.example` 참조. 컬럼:

| 컬럼 | 설명 | 예시 |
|------|------|------|
| Sort | 표시 순서 | `1` |
| Tier1 | 최상위 조직 | `Engineering` |
| Tier2 | 중간 조직 | `Platform Team` |
| Tier3 | 하위 조직 | (선택) |
| 직책 | 한국어 직책 | `팀장` |
| 이름 | 한국어 이름 | `홍길동` |
| Name | 영문 이름 | `Gildong Hong` |
| Ext. | 내선 번호 | `822XXXXXXXXX` |
| Mobile | 휴대폰 | `010-XXXX-XXXX` |
| Email | 이메일 (필수) | `gildong@example.com` |

---

## 로컬 실행

### 1. 환경 설정

```bash
git clone https://github.com/nexpert-code/wxc-d-svc.git
cd wxc-d-svc

cp .env.example .env
# .env 파일을 열어 값 설정
```

### 2. Webex Bot 준비

1. [Webex Developer Portal](https://developer.webex.com)에서 Bot 생성
2. Bot Token 복사 → `.env`의 `WEBEX_BOT_TOKEN`에 입력
3. Webex Webhook 등록:
   - Resource: `attachmentActions`
   - Event: `created`  
   - Target URL: `https://your-server/auth/webhook`
   - Secret: `.env`의 `WEBEX_WEBHOOK_SECRET` 값

### 3. 패키지 설치 및 실행

```bash
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt

# 직원 데이터 준비 (data.csv.example을 복사해서 수정)
cp data.csv.example data.csv

uvicorn app.main:app --reload --port 8000
```

브라우저에서 `http://localhost:8000` 접속

### 4. Docker로 실행

```bash
# data.csv를 준비한 뒤
docker build -t wxc-d-svc .
docker run -p 8080:8080 \
  -e WEBEX_BOT_TOKEN=your-token \
  -e WEBEX_WEBHOOK_SECRET=your-secret \
  -e ADMIN_EMAIL=admin@example.com \
  -e ALLOWED_EMAIL_DOMAIN=@example.com \
  -e SECRET_KEY=your-random-secret \
  wxc-d-svc
```

---

## API 엔드포인트

### 인증 (`/auth`)

| Method | Path | 설명 |
|--------|------|------|
| POST | `/auth/request` | 로그인 요청 (Webex 카드 발송) |
| GET | `/auth/poll/{token}` | 승인 상태 폴링 |
| POST | `/auth/webhook` | Webex attachmentActions 수신 |
| POST | `/auth/logout` | 로그아웃 (쿠키 삭제) |
| GET | `/auth/me` | 현재 로그인 사용자 정보 |

### 직원 조회 (`/api`) — 로그인 필요

| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/browse` | 조직 드릴다운 탐색 |
| GET | `/api/employees` | 직원 목록 (검색·필터·페이징) |
| GET | `/api/employees/{id}` | 직원 단건 조회 |
| GET | `/api/orgs` | 조직 트리 |

### 관리자 (`/api/admin`) — editor/admin 권한

| Method | Path | 설명 |
|--------|------|------|
| POST | `/api/employees` | 직원 추가 |
| PATCH | `/api/employees/{id}` | 직원 수정 |
| DELETE | `/api/employees/{id}` | 직원 비활성화 |
| POST | `/api/import/csv` | CSV 일괄 임포트 |
| GET | `/api/import/logs` | 임포트 이력 |
| GET | `/api/admin/stats` | 대시보드 통계 |
| GET | `/api/admin/login-stats` | 로그인 현황 |

---

## 운영 환경 배포 (Cloud Run 예시)

```bash
gcloud run deploy wxc-d-svc \
  --source . \
  --region asia-northeast3 \
  --set-env-vars="APP_ENV=production,\
DATABASE_URL=postgresql://user:pass@host/db?sslmode=require,\
WEBEX_BOT_TOKEN=...,\
WEBEX_WEBHOOK_SECRET=...,\
ADMIN_EMAIL=admin@example.com,\
ALLOWED_EMAIL_DOMAIN=@example.com,\
SECRET_KEY=..." \
  --allow-unauthenticated \
  --port=8080
```

> **PostgreSQL 권장**: Neon, Supabase, Cloud SQL 등 무료 티어 활용 가능

---

## 기술 스택

| 분류 | 기술 |
|------|------|
| 백엔드 | Python 3.12, FastAPI 0.115, Uvicorn |
| ORM | SQLAlchemy 2.0 (Async), Alembic |
| DB | PostgreSQL (운영) / SQLite (개발) |
| 인증 | itsdangerous, HttpOnly Cookie |
| 외부 API | Webex REST API (httpx) |
| 프론트엔드 | React (빌드 결과물 — static/) |
| 컨테이너 | Docker |

---

## 라이선스

MIT License

# Code Citations

## License: MIT
https://github.com/colinch4/colinch4.github.io/blob/0000652ad1bc9ea54cea7572fcc631bcd2f5fe04/_posts/2023/09/8/2023-09-08-09-44-32-092958.md

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: 알 수 없음
https://github.com/kelvinromero/docker-introduction/blob/1cce910c1029119618d756394d7853a98bc6709a/hello-fastapi/Dockerfile

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: MIT
https://github.com/colinch4/colinch4.github.io/blob/0000652ad1bc9ea54cea7572fcc631bcd2f5fe04/_posts/2023/09/8/2023-09-08-09-44-32-092958.md

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: 알 수 없음
https://github.com/kelvinromero/docker-introduction/blob/1cce910c1029119618d756394d7853a98bc6709a/hello-fastapi/Dockerfile

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: MIT
https://github.com/colinch4/colinch4.github.io/blob/0000652ad1bc9ea54cea7572fcc631bcd2f5fe04/_posts/2023/09/8/2023-09-08-09-44-32-092958.md

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: 알 수 없음
https://github.com/kelvinromero/docker-introduction/blob/1cce910c1029119618d756394d7853a98bc6709a/hello-fastapi/Dockerfile

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: MIT
https://github.com/colinch4/colinch4.github.io/blob/0000652ad1bc9ea54cea7572fcc631bcd2f5fe04/_posts/2023/09/8/2023-09-08-09-44-32-092958.md

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: 알 수 없음
https://github.com/kelvinromero/docker-introduction/blob/1cce910c1029119618d756394d7853a98bc6709a/hello-fastapi/Dockerfile

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: MIT
https://github.com/colinch4/colinch4.github.io/blob/0000652ad1bc9ea54cea7572fcc631bcd2f5fe04/_posts/2023/09/8/2023-09-08-09-44-32-092958.md

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: 알 수 없음
https://github.com/kelvinromero/docker-introduction/blob/1cce910c1029119618d756394d7853a98bc6709a/hello-fastapi/Dockerfile

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: MIT
https://github.com/colinch4/colinch4.github.io/blob/0000652ad1bc9ea54cea7572fcc631bcd2f5fe04/_posts/2023/09/8/2023-09-08-09-44-32-092958.md

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: 알 수 없음
https://github.com/kelvinromero/docker-introduction/blob/1cce910c1029119618d756394d7853a98bc6709a/hello-fastapi/Dockerfile

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: MIT
https://github.com/colinch4/colinch4.github.io/blob/0000652ad1bc9ea54cea7572fcc631bcd2f5fe04/_posts/2023/09/8/2023-09-08-09-44-32-092958.md

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


## License: 알 수 없음
https://github.com/kelvinromero/docker-introduction/blob/1cce910c1029119618d756394d7853a98bc6709a/hello-fastapi/Dockerfile

```


현재 파일 편집 도구가 비활성화되어 있어 직접 파일을 수정할 수 없습니다. 아래 완성된 사양서를 드리니, 기존 문서 내용을 이 내용으로 교체해주세요.

```markdown
# 시스코 점심요정 🧚 — Webex Bot 개발 사양서

> **문서 버전**: v1.1 | **작성일**: 2026-03-24 | **상태**: 확인 완료

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| **봇 이름** | 시스코 점심요정 |
| **플랫폼** | Cisco Webex |
| **목적** | 직급·세대·팀을 초월한 3~4명 점심 페어링으로 사내 소통 활성화 |
| **핵심 가치** | 미지의 동료와의 만남, 음식 취향 매칭, 익명 감사 문화 |
| **대상 사용자** | 시스코 코리아 전 임직원 |
| **예상 참여 인원** | 30~100명 (소규모 파일럿 → 점진적 확대) |

### 봇이 해결하는 문제
- 같은 팀/직급끼리만 밥을 먹는 사일로 현상
- "오늘 뭐 먹지?" 매일 반복되는 메뉴 고민
- 새로운 동료를 알아갈 기회 부족

---

## 2. 기술 스택

| 구성요소 | 선택 | 근거 |
|---|---|---|
| **언어/프레임워크** | Python 3.11+ / FastAPI | 비동기 지원, 빠른 개발 |
| **Webex SDK** | `webexpythonsdk` v2.0+ | 공식 Python SDK, 자동 rate-limit 처리 |
| **DB** | Firestore (GCP) | 서버리스, 유연한 스키마, GCP 네이티브 |
| **스케줄러** | GCP Cloud Scheduler | Cloud Run과 연동, 서버리스 cron |
| **맛집 추천** | 네이버 검색 API (지역검색) | 한국 맛집 데이터 풍부 |
| **배포** | GCP Cloud Run | 서버리스, 자동 스케일링, Docker 기반 |
| **카드 UI** | Webex Adaptive Cards 1.3 | 인터랙티브 폼, 버튼, 날짜 선택 |
| **시크릿 관리** | GCP Secret Manager | 봇 토큰, API 키 안전 저장 |

---

## 3. 시스템 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                    GCP Cloud Run                      │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐  │
│  │  FastAPI App      │     │  Webhook Handlers      │  │
│  │  /cron/*          │     │  /webhook/messages     │  │
│  │  /admin/*         │     │  /webhook/cards        │  │
│  └────────┬─────────┘     └──────────┬─────────────┘  │
│           │                          │                │
│  ┌────────▼──────────────────────────▼─────────────┐  │
│  │              비즈니스 로직 레이어                  │  │
│  │  ScheduleService  │ PairingService               │  │
│  │  MenuService      │ SpaceService                 │  │
│  │  KudosService     │ UserService                  │  │
│  └─────────────────────┬───────────────────────────┘  │
│                        │                              │
│  ┌─────────────────────▼───────────────────────────┐  │
│  │              Firestore (GCP)                     │  │
│  │  users / weekly_responses / pairings /           │  │
│  │  menu_history / kudos                            │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         ▲                              ▲
         │                              │
  GCP Cloud Scheduler            Webex Platform
  (cron HTTP 트리거)             (Webhook 전달)
```

---

## 4. 데이터 모델 (Firestore 컬렉션)

### `users` — 등록된 사용자

| 필드 | 타입 | 설명 |
|---|---|---|
| `personId` | string | Webex Person ID (문서 ID로 사용) |
| `email` | string | Webex 이메일 |
| `displayName` | string | 표시 이름 |
| `location` | string \| null | 근무지 ("삼성동", "판교" 등) |
| `isActive` | boolean | 활성 여부 |
| `addedBy` | string | 등록한 사람의 personId |
| `lastMenus` | [string, string] | 최근 먹은 메뉴 2개 (추천 제외용) |
| `createdAt` | timestamp | 등록 시각 |

### `weekly_responses` — 주간 참여 응답

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 식별자 (예: "2026-W14") |
| `userId` | string | 사용자 personId |
| `availableDays` | [string] | 점심 페어링 원하는 요일 (["월", "화", "목"]) |
| `preferredMenus` | [string] | 먹고 싶은 메뉴 (["떡볶이", "초밥"]) |
| `confirmed` | boolean | 전날 확정 여부 |
| `respondedAt` | timestamp | 응답 시각 |

### `pairings` — 페어링 결과

| 필드 | 타입 | 설명 |
|---|---|---|
| `weekId` | string | 주차 |
| `date` | string | 점심 날짜 ("2026-03-27") |
| `members` | [string] | 조원 userId 배열 |
| `groupName` | string | 랜덤 조 이름 ("신나는 떡볶이단 🔥") |
| `spaceId` | string \| null | 생성된 Webex Space ID |
| `selectedMenu` | string | 선정된 메뉴 |
| `restaurant` | string \| null | 선택된 맛집 |
| `treaterId` | string \| null | 점심 쏘는 사람 |
| `treaterTitle` | string \| null | 점심킹 칭호 |
| `status` | string | "pending" \| "confirmed" \| "cancelled" |

### `kudos` — 익명 감사 메시지

| 필드 | 타입 | 설명 |
|---|---|---|
| `pairingId` | string | 해당 페어링 ID |
| `toUserId` | string | 점심 쏘는 사람 |
| `message` | string | 익명 감사 메시지 |
| `createdAt` | timestamp | 작성 시각 |

> ⚠️ `fromUserId`는 **저장하지 않음** — 완전한 익명 보장

---

## 5. 핵심 워크플로우 — 8단계

### 전체 흐름 타임라인

```
금요일 10:00  ──→  ① 주간 안내 카드 발송
금~일          ──→  ② 응답 수집 (페어링 원하는 날, 근무지, 메뉴)
해당일 전날 14:00 →  ③ "내일 점심 준비됐나요?" 확인
해당일 전날 18:00 →  ④ 페어링 실행 + 메뉴 추천
             직후 →  ⑤ 결과 알림 DM + 점심킹 접수
해당일 10:30   ──→  ⑥ Webex Space 생성 + 멤버 초대
해당일 12:00   ──→  ⑦ "즐거운 점심!" 인사 + 마무리
해당일 18:00   ──→  ⑧ Space 삭제 예고 + 삭제
```

---

### ① 주간 안내 (매주 금요일 10:00)

**트리거**: Cloud Scheduler → `POST /cron/weekly-invite`

**동작**:
1. `users` 컬렉션에서 `isActive=true` 전원 조회
2. 각 사용자에게 **1:1 Adaptive Card** 발송

**카드 내용**:
- 인사: "다음 주에 미지의 동료와 점심 같이 하는 거 어때요? 🧚"
- `Input.ChoiceSet` (multiSelect): **점심 페어링 원하는 날** 선택 (월~금)
  - 출근 요일이 아닌, 실제로 점심 페어링을 원하는 날만 선택하도록 안내
- `Input.ChoiceSet`: **근무지** (이전 저장값 기본 선택, 변경 가능)
  - 선택지: 삼성동, 판교, 기타(직접입력)
  - 최초 응답 시에만 질문, 이후에는 저장된 값 표시
- `Input.Text` (placeholder): **먹고 싶은 메뉴** 2~3개 (쉼표 구분)
  - 예: "떡볶이, 초밥, 베트남쌀국수"
- `Action.Submit`: "참여할게요! 🙋"
- `Action.Submit`: "이번 주는 패스 👋"

**응답 처리**:
- 참여 시 → `weekly_responses` 저장 + 확인 DM: "접수! 다음 주 [수,목] 점심 페어링이 기대되네요! ✨"
- 패스 시 → 기록만 남기고 다음 주에 다시 안내

---

### ② 응답 수집 (금~일, 상시)

**트리거**: Webhook — `attachmentActions.created`

**동작**:
1. `GET /attachment/actions/{actionId}`로 카드 입력값 조회
2. `weekly_responses` 컬렉션에 저장/업데이트
3. 근무지 변경 시 `users.location` 업데이트
4. 메뉴 선호 저장 (나중에 같은 음식 선호 사람 매칭에 활용)

---

### ③ 전날 확인 (해당일 전날 14:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-confirm`

**동작**:
1. 내일 요일에 참여 등록한 사용자 조회
2. 1:1 확인 카드 발송

**카드 내용**:
- "내일 점심 준비 됐나요? 🧚"
- `Action.Submit`: "당연하지! 😎"
- `Action.Submit`: "미안, 내일은 어려워 😢"

**응답 처리**:
- 확정 → `weekly_responses.confirmed = true`
- 취소 → `weekly_responses.confirmed = false`
- **미응답** → 아래 리마인더 프로세스 진행

#### 리마인더 (해당일 전날 17:00)

**트리거**: Cloud Scheduler → `POST /cron/day-before-remind`

**동작**:
1. 14:00 확인 카드에 **미응답**인 사용자 조회
2. 리마인더 DM 발송:
   > "아직 내일 점심 확인을 안 하셨어요! 🧚 1시간 내 응답 없으면 아쉽지만 불참으로 처리할게요~"
3. 18:00 페어링 시점까지 미응답 시 → **불참 처리** (`weekly_responses.confirmed = false`)

---

### ④ 페어링 실행 (전날 18:00)

**트리거**: Cloud Scheduler → `POST /cron/run-pairing`

#### 인원 부족 시 (확정 3명 미만)

참여자 전원에게 DM:
> "미안해요~ 내일은 점심요정이 좀 쉬려구요 😴 참여 인원이 적어서 요정도 쉬는 날! 다음 주엔 꼭 만나요! 🧚✨"

#### 페어링 알고리즘 (3명 이상)

```
1. 확정 인원을 location별로 그룹핑
2. 각 location 그룹 내에서:
   a. 메뉴 선호도 유사성 점수 계산 (교집합 기반)
   b. 이전 4주간 같은 그룹 이력 → 페널티 부여
   c. 점수 기반 탐욕적 그룹핑 (3~4명)
3. 남은 인원은 크로스-location 그룹으로 편성
```

**그룹 크기 규칙**:
| 확정 인원 | 그룹 편성 |
|---|---|
| 3명 | 3 |
| 4명 | 4 |
| 5명 | 3+2(X) → 5명 한 그룹 허용 |
| 6명 | 3+3 |
| 7명 | 4+3 |
| 8명 | 4+4 |
| 9명 | 3+3+3 |
| 10명 | 4+3+3 |
| 11명 | 4+4+3 |
| 12명 | 4+4+4 |

> 원칙: **다양한 사람과 만남**을 극대화. 매주 다른 멤버 조합이 되도록 이력 기반 분배.

---

### ⑤ 결과 알림 + 메뉴 추천 (페어링 직후)

**각 조원에게 1:1 DM**:
- "내일 **3명**과 함께 점심을 할 예정입니다! 🎉"
- 조원 이름은 아직 **비공개** (내일 스페이스에서 공개)

**메뉴 추천 로직**:
1. 조원 선호 메뉴 교집합 확인
2. `users.lastMenus`에 있는 메뉴 **제외** (최근 2회 중복 방지)
3. 교집합이 있으면 → 해당 메뉴 추천 + "오늘은 **떡볶이 데이**! 🔥"
4. 교집합이 없으면 → 봇이 랜덤 메뉴 선정

**맛집 추천 (네이버 검색 API)**:
- 쿼리: `"{근무지} {메뉴} 맛집"` (예: "삼성동 떡볶이 맛집")
- 상위 3개 결과를 카드로 표시 (가게명, 주소, 링크)

**메뉴 투표 카드**:
- `Input.ChoiceSet`: 맛집 3개 중 선택
- `Input.Text`: "조원들에게 한마디!" (선택)
- `Action.Submit`: "이걸로!" 
- `Action.Submit`: "🤑 점심값 내가 쏠게!"

**메뉴 미선정 시**: 투표 마감(당일 09:00)까지 합의 없으면 봇이 랜덤 선택 후 알림

---

### 점심킹 시스템 🤑

**점심 쏘겠다는 사람 발생 시**:

1. 재미있는 칭호 랜덤 부여:
   - "점심킹 👑"
   - "멋쟁이 물주 💰"
   - "사랑스런 호구 ❤️"
   - "전설의 지갑요정 🧚"
   - "오늘의 부자 💎"
   - "점심계의 산타 🎅"
   - "밥값히어로 🦸"

#### 점심킹 후보가 2명 이상일 경우 — 가위바위보 대결 ✊✌️🖐️

1. 후보들에게 각각 1:1 DM으로 가위바위보 카드 발송:
   > "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
   - `Action.Submit`: "✊ 바위" / "✌️ 가위" / "🖐️ 보"
2. 결과 판정 후, **전체 조원에게 익명 공지**:
   > "🎉 점심킹 후보 2명이 가위바위보를 해서 한 명이 이겼습니다! 누군지는 내일 점심때 공개! 🤫"
   - ⚠️ **이름, 성별, 직급 등 정체를 유추할 수 있는 어떤 힌트도 절대 포함하지 않음**
3. 무승부 시 → 재경기 (최대 3회), 3회 연속 무승부 → 공동 점심킹 인정

#### 점심킹 1명 확정 후

2. 다른 조원에게 DM:
   > "🎉 내일 누군가가 점심을 쏘겠다고 합니다! 감사한 마음을 익명으로 전해주세요!"

3. **Kudos 카드** 발송:
   - `Input.Text`: "감사 메시지를 적어주세요"
   - `Action.Submit`: "익명으로 전달하기"

4. 수집된 Kudos → 점심킹에게 **익명으로** 일괄 전달:
   > "당신에게 도착한 감사 편지 💌"
   > - "덕분에 행복한 점심이 될 것 같아요!"
   > - "진짜 멋지십니다 ㅎㅎ"
   > *(누가 보냈는지는 비밀이에요 🤫)*

---

### ⑥ Webex Space 생성 (당일 10:30)

**트리거**: Cloud Scheduler → `POST /cron/create-spaces`

**유쾌한 랜덤 조 이름 생성**:
- 패턴: `[형용사] + [명사] + [이모지]`
- 예시:
  - "신나는 떡볶이단 🔥"
  - "행복한 초밥클럽 🍣"
  - "용감한 점심원정대 ⚔️"
  - "멋진 라멘동호회 🍜"
  - "반짝이는 김치찌개파 ✨"

**동작**:
1. `POST /rooms` → 조 이름으로 Webex Space 생성
2. `POST /memberships` → 각 조원 + 봇 초대
3. 봇 첫 메시지:
   > "안녕하세요! 🧚 점심요정이 여러분을 연결해드렸어요!"
   > 
   > 👥 **오늘의 멤버**: 홍길동, 김철수, 이영희
   > 🍽️ **오늘의 메뉴**: 떡볶이
   > 📍 **추천 맛집**: [OO떡볶이 삼성점](링크)
   > 
   > 점심시간까지 자유롭게 대화 나눠보세요! 💬

4. 점심킹 있을 경우 추가 메시지:
   > "🎉 오늘의 **점심킹 👑** 님이 밥을 쏜다고 합니다! 감사~!"

---

### ⑦ 마무리 (당일 12:00)

**트리거**: Cloud Scheduler → `POST /cron/lunch-greeting`

**스페이스에 메시지**:
> "즐거운 점심시간 되세요! 🍽️ 맛있는 거 많이 드세요!"
> "다음 주 금요일에 점심요정이 다시 찾아올게요! 🧚✨ 또 만나요!"

**후처리**:
- `users.lastMenus` 업데이트 (오늘 먹은 메뉴 추가, 가장 오래된 것 제거)

---

### ⑧ Space 삭제 (당일 18:00)

**트리거**: Cloud Scheduler → `POST /cron/cleanup-spaces`

**동작**:
1. 오늘 생성된 모든 페어링 Space 조회
2. 각 Space에 삭제 예고 메시지 발송:
   > "오늘 즐거운 점심이었나요? 🧚 이 방은 잠시 후 사라질 예정이에요! 기억에 남는 대화가 있다면 지금 저장해두세요! 💾"
   > "다음 주에 새로운 조에서 또 만나요! ✨"
3. **5분 후** Space 삭제 (`DELETE /rooms/{roomId}`)
4. `pairings.spaceId = null` 업데이트

---

## 6. 사용자 관리

### 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `추가 {이메일}` | 새 사용자를 페어링 시스템에 등록 | `추가 hong@cisco.com` |
| `탈퇴` | 본인의 페어링 시스템 비활성화 | `탈퇴` |
| `내정보` | 등록된 근무지, 선호 메뉴 확인 | `내정보` |
| `도움말` | 명령어 안내 | `도움말` |

### 사용자 추가 플로우
1. 기존 사용자가 봇에게 1:1 DM: `추가 kim@cisco.com`
2. 봇이 `kim@cisco.com`을 `users` 컬렉션에 등록 (`isActive=true`)
3. 해당 사용자에게 환영 DM:
   > "안녕하세요! 🧚 시스코 점심요정입니다! 누군가가 당신을 점심 친구로 추천해주셨어요."
   > "매주 금요일에 다음 주 점심 페어링을 안내해드릴게요. 기대해주세요! ✨"
4. 그 주 금요일부터 주간 안내 수신 시작

### 사용자 탈퇴
- 봇에게 `탈퇴` 입력 → `users.isActive = false`
- > "아쉽지만 다음에 또 만나요! 언제든 '참여'라고 하면 돌아올 수 있어요 🧚"

---

### 관리자(어드민) 기능

> 관리자는 Firestore `users` 컬렉션에서 `isAdmin=true`인 사용자

#### 관리자 명령어

| 명령어 | 설명 | 예시 |
|---|---|---|
| `어드민 일괄추가 {이메일1}, {이메일2}, ...` | 여러 사용자 한번에 등록 | `어드민 일괄추가 a@cisco.com, b@cisco.com` |
| `어드민 일괄삭제 {이메일1}, {이메일2}, ...` | 여러 사용자 비활성화 | `어드민 일괄삭제 a@cisco.com` |
| `어드민 강제페어링` | 이번 주 페어링 즉시 실행 | `어드민 강제페어링` |
| `어드민 통계` | 통계 대시보드 카드 수신 | `어드민 통계` |

#### 통계 대시보드

`어드민 통계` 명령 시 Adaptive Card로 표시:

- **주간 참여율**: 이번 주 참여 인원 / 전체 활성 사용자
- **월간 참여율 추이**: 최근 4주 참여율 그래프 (텍스트 바 차트)
- **인기 메뉴 TOP 5**: 가장 많이 선호된 메뉴
- **점심킹 랭킹 TOP 5**: 가장 많이 점심을 쏜 사람 (칭호와 함께)
- **근무지별 참여 분포**: 삼성동 vs 판교 등

---

## 7. 외부 API 연동

### Webex API (`webexpythonsdk`)

```python
from webexpythonsdk import WebexAPI
api = WebexAPI(access_token=BOT_TOKEN)
```

| 기능 | 코드 |
|---|---|
| 1:1 메시지 | `api.messages.create(toPersonEmail=..., text=..., attachments=[card])` |
| 스페이스 생성 | `api.rooms.create(title="신나는 떡볶이단 🔥")` |
| 멤버 초대 | `api.memberships.create(roomId=..., personEmail=...)` |
| 카드 액션 조회 | `api.attachment_actions.get(actionId)` |
| Webhook 등록 | `api.webhooks.create(name, targetUrl, resource, event, secret)` |

**Webhook 등록 (초기 설정)**:
```python
# 메시지 수신 (명령어 처리)
api.webhooks.create(
    name="Messages",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/messages",
    resource="messages",
    event="created",
    secret=WEBHOOK_SECRET
)

# 카드 액션 수신 (Adaptive Card 폼 제출)
api.webhooks.create(
    name="Card Actions",
    targetUrl="https://{CLOUD_RUN_URL}/webhook/cards",
    resource="attachmentActions",
    event="created",
    secret=WEBHOOK_SECRET
)
```

### 네이버 검색 API (맛집 추천)

- **엔드포인트**: `GET https://openapi.naver.com/v1/search/local.json`
- **인증 헤더**: `X-Naver-Client-Id`, `X-Naver-Client-Secret`
- **쿼리 예시**: `query=삼성동 떡볶이 맛집&display=3&sort=comment`
- **활용 필드**: `title`, `address`, `roadAddress`, `link`, `category`

**근무지별 검색 키워드 매핑**:
| 근무지 | 검색 범위 |
|---|---|
| 삼성동 | "삼성동", "코엑스", "공항터미널" |
| 판교 | "판교", "판교역" |

---

## 8. Adaptive Card 설계 (5종)

### 카드 ① — 주간 안내 카드

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🧚 점심요정이 찾아왔어요!",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "TextBlock",
      "text": "다음 주에 미지의 동료와 점심 같이 하는 거 어때요?",
      "wrap": true
    },
    {
      "type": "Input.ChoiceSet",
      "id": "availableDays",
      "label": "📅 다음 주 점심 페어링 원하는 날을 선택해주세요 (복수 선택 가능)",
      "isMultiSelect": true,
      "choices": [
        {"title": "월요일", "value": "월"},
        {"title": "화요일", "value": "화"},
        {"title": "수요일", "value": "수"},
        {"title": "목요일", "value": "목"},
        {"title": "금요일", "value": "금"}
      ]
    },
    {
      "type": "Input.ChoiceSet",
      "id": "location",
      "label": "📍 근무지",
      "value": "${savedLocation}",
      "choices": [
        {"title": "삼성동 (시스코코리아)", "value": "삼성동"},
        {"title": "판교", "value": "판교"},
        {"title": "기타", "value": "기타"}
      ]
    },
    {
      "type": "Input.Text",
      "id": "preferredMenus",
      "label": "🍽️ 먹고 싶은 메뉴가 있다면? (2~3개, 쉼표로 구분)",
      "placeholder": "예: 떡볶이, 초밥, 베트남쌀국수",
      "isMultiline": false
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "참여할게요! 🙋",
      "data": {"action": "participate"}
    },
    {
      "type": "Action.Submit",
      "title": "이번 주는 패스 👋",
      "data": {"action": "pass"}
    }
  ]
}
```

### 카드 ② — 전날 확인 카드
- TextBlock: "내일 점심 준비 됐나요? 🧚"
- Action.Submit: "당연하지! 😎" (`{"action":"confirm"}`)
- Action.Submit: "미안, 내일은 어려워 😢" (`{"action":"cancel"}`)

### 카드 ③ — 메뉴 투표 + 맛집 선택 카드
- TextBlock: "오늘은 **{메뉴} 데이**! 🔥"
- Input.ChoiceSet: 맛집 3곳 (네이버 API 결과)
- Input.Text: "조원들에게 한마디!"
- Action.Submit: "이걸로!" (`{"action":"vote"}`)
- Action.Submit: "🤑 점심값 내가 쏠게!" (`{"action":"treat"}`)

### 카드 ④ — 익명 Kudos 카드
- TextBlock: "🎉 누군가가 점심을 쏘겠다고 합니다! 감사 메시지를 남겨주세요."
- Input.Text: "감사 메시지" (multiline)
- Action.Submit: "익명으로 전달하기 💌"

### 카드 ⑤ — 가위바위보 카드 (점심킹 중복 시)
- TextBlock: "점심킹 후보가 여러 명이에요! 가위바위보로 결정합니다! ✊✌️🖐️"
- Action.Submit: "✊ 바위" (`{"action":"rps", "choice":"rock"}`)
- Action.Submit: "✌️ 가위" (`{"action":"rps", "choice":"scissors"}`)
- Action.Submit: "🖐️ 보" (`{"action":"rps", "choice":"paper"}`)"

---

## 9. 배포 및 운영 (GCP)

### Cloud Run 배포

**Dockerfile**:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**환경변수** (GCP Secret Manager에서 관리):
- `WEBEX_BOT_TOKEN`: Webex Bot Access Token
- `WEBHOOK_SECRET`: Webhook 서명 검증 시크릿
- `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`: 네이버 검색 API
- `GCP_PROJECT_ID`: Firestore 프로젝트 ID
```

---

## 10. 릴리즈 로드맵

### MVP (v1.0) — 핵심 루프
- [ ] Webex Bot 생성 + GCP 프로젝트 세팅
- [ ] 사용자 등록/탈퇴 (`추가`, `탈퇴`, `도움말`)
- [ ] 주간 안내 카드 (페어링 원하는 날 선택)
- [ ] 전날 확인 + 17시 리마인더
- [ ] 페어링 알고리즘 (근무지별 그룹핑)
- [ ] Webex Space 생성 + 멤버 초대 + 메뉴 알림
- [ ] Space 당일 18시 예고 후 삭제

### v1.1 — 맛집 추천
- [ ] 네이버 검색 API 연동
- [ ] 메뉴 투표 카드
- [ ] 맛집 3곳 추천 + 투표

### v1.2 — 점심킹 & Kudos
- [ ] "점심값 내가 쏠게" 기능
- [ ] 중복 시 가위바위보 대결 (익명 유지)
- [ ] 익명 감사 메시지(Kudos) 시스템

### v1.3 — 관리자 & 통계
- [ ] 관리자 명령어 (일괄 추가/삭제, 강제 페어링)
- [ ] 통계 대시보드 (참여율, 인기 메뉴, 점심킹 랭킹)
- [ ] 공휴일 자동 감지 (선택)


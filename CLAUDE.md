# devops-bot

Slack 이벤트를 LLM으로 분석해 Jira 티켓 생성 및 모니터링 알람 자동 대응을 수행하는 DevOps 자동화 봇.

## 로드맵

| Phase | LLM | 인프라 | 핵심 기능 |
|-------|-----|--------|-----------|
| **Phase 1** | Claude API (Anthropic) | 로컬 + ngrok | Jira 티켓메이커 |
| **Phase 2** | AWS Bedrock (Claude) | AWS 클라우드 | Remote MCP 서버 전환 |
| **Phase 3** | AWS Bedrock (Claude) | AWS 클라우드 | 모니터링 알람 자동 대응 |

### Phase 1 (현재)
- Slack 이모지 리액션 → 스레드 분석 → Jira 티켓 자동 생성
- Claude API 직접 호출, 로컬 서버 + ngrok으로 동작 검증

### Phase 2
- LLM 백엔드를 Claude API → AWS Bedrock으로 전환
- Remote MCP 서버 구축 (Streamable HTTP transport)
- AWS 클라우드 인프라로 이전 (ECS Fargate 또는 Lambda + API Gateway)

### Phase 3
- Datadog / Prometheus / Loki MCP 연동
- Kubernetes MCP 연동
- 알람 수신 → 원인 분석 → 대응 액션 자동화 → Jira 인시던트 티켓 생성

---

## 아키텍처

```
Slack Thread
    ↓  🎫 이모지 리액션
Slack Bolt App (reaction_added 이벤트)
    ↓  conversations.replies API
스레드 전체 메시지 수집
    ↓  Claude API
{title, description, priority, labels} 구조화
    ↓  Jira REST API v3
Jira Cloud 티켓 생성
    ↓  chat.postMessage
Slack 스레드에 티켓 링크 회신
```

---

## 기술 스택

| 역할 | 라이브러리 | Phase |
|------|-----------|-------|
| Slack 이벤트 수신 | `slack-bolt` | 1+ |
| LLM 분석 | `anthropic` (Claude API) | 1 |
| Jira 연동 | `jira` (jira-python) | 1+ |
| 환경변수 | `python-dotenv` | 1+ |
| 로컬 터널 | `ngrok` (별도 설치) | 1 |
| LLM 백엔드 전환 | AWS Bedrock (boto3) | 2+ |
| 클라우드 인프라 | AWS ECS Fargate / Lambda | 2+ |
| MCP 서버 | `mcp` Python SDK | 2+ |
| 모니터링 MCP | datadog-mcp, prometheus-mcp, loki-mcp | 3 |
| 오케스트레이션 MCP | kubernetes-mcp | 3 |

---

## 디렉토리 구조

```
devops-bot/
├── CLAUDE.md
├── AGENTS.md
├── README.md
├── .env.example
├── requirements.txt
├── src/
│   ├── app.py                  # Slack Bolt 진입점 (포트 3000)
│   ├── config.py               # 환경변수 로딩
│   ├── handlers/
│   │   └── reaction.py         # reaction_added 핸들러
│   └── services/
│       ├── slack_reader.py     # 스레드 메시지 수집
│       ├── llm.py              # Claude API 호출 및 파싱
│       └── jira_client.py      # Jira 티켓 생성
├── tests/
│   └── test_services.py
└── mcp_server/                 # Phase 2+
    ├── jira_mcp.py             # Phase 2: Jira MCP tools
    ├── slack_mcp.py            # Phase 2: Slack MCP tools
    └── monitoring_mcp.py       # Phase 3: Datadog/Prometheus/Loki/K8s MCP tools
```

---

## 로컬 실행 커맨드

```bash
# 의존성 설치
pip install -r requirements.txt

# 환경변수 설정
cp .env.example .env
# .env 파일에 실제 값 입력

# 로컬 서버 실행
python src/app.py

# 별도 터미널 - ngrok 터널 (Slack 이벤트 수신용)
ngrok http 3000
```

---

## 환경변수

| 변수 | 설명 |
|------|------|
| `SLACK_BOT_TOKEN` | Slack Bot Token (`xoxb-...`) |
| `SLACK_SIGNING_SECRET` | Slack App Signing Secret |
| `ANTHROPIC_API_KEY` | Claude API Key (`sk-ant-...`) |
| `JIRA_URL` | Jira Cloud URL (`https://yourco.atlassian.net`) |
| `JIRA_EMAIL` | Jira 계정 이메일 |
| `JIRA_API_TOKEN` | Jira API Token |
| `JIRA_PROJECT_KEY` | 티켓 생성 대상 프로젝트 키 (예: `PROJ`) |
| `TRIGGER_EMOJI` | 트리거 이모지 이름 (기본값: `ticket`) |
| `PORT` | 서버 포트 (기본값: `3000`) |

---

## Slack App 설정

### Bot Token Scopes (OAuth & Permissions)
- `reactions:read` - 이모지 리액션 이벤트 수신
- `channels:history` - 공개 채널 메시지 읽기
- `groups:history` - 비공개 채널 메시지 읽기
- `chat:write` - 스레드 회신

### Event Subscriptions
- Request URL: `https://<ngrok-url>/slack/events`
- Subscribe: `reaction_added`

### 봇을 채널에 초대
```
/invite @devops-bot
```

---

## 검증 방법

```bash
# Jira 연결 확인
python -c "from src.services.jira_client import JiraClient; JiraClient().test_connection()"

# LLM 파싱 확인
python -c "from src.services.llm import analyze_thread; print(analyze_thread(['버그 수정 필요: 로그인 페이지 500 에러']))"

# End-to-End: Slack 채널에서 🎫 이모지 리액션 → Jira 티켓 생성 확인
```

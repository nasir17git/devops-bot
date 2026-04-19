# AGENTS.md — devops-bot 에이전트 워크플로우

## 개요

이 파일은 devops-bot의 자율 에이전트 동작 방식과 MCP 툴 확장 계획을 정의한다.
Claude Code 또는 Remote MCP 클라이언트가 이 프로젝트를 이해하고 작업할 때 참조한다.

---

## Phase 1: Jira 티켓메이커 워크플로우

### 트리거 조건
- Slack 메시지에 `TRIGGER_EMOJI` (기본: `ticket`) 이모지 리액션 추가
- 봇이 해당 채널에 초대되어 있어야 함

### 단계별 처리 흐름

```
[1] reaction_added 이벤트 수신
    - handlers/reaction.py
    - 이모지 이름이 TRIGGER_EMOJI와 일치하는지 확인

[2] 스레드 메시지 수집
    - services/slack_reader.py
    - Slack API: conversations.replies(channel, thread_ts)
    - 봇 메시지 제외, 사용자 메시지만 수집
    - 포맷: "작성자: 내용" 형식으로 리스트 구성

[3] LLM 분석 (Claude API)
    - services/llm.py
    - 스레드 내용을 시스템 프롬프트와 함께 전달
    - 응답: JSON {title, description, priority, labels}
    - 파싱 실패 시 Slack에 오류 메시지 회신

[4] Jira 티켓 생성
    - services/jira_client.py
    - Jira REST API v3: POST /rest/api/3/issue
    - issuetype: Task (기본값)
    - 생성 성공 시 issue key 반환 (예: PROJ-123)

[5] Slack 스레드 회신
    - handlers/reaction.py
    - "✅ Jira 티켓이 생성되었습니다: [PROJ-123](링크)" 형식
```

---

## LLM 프롬프트 설계

### System Prompt (llm.py)
```
You are a Jira ticket creator assistant.
Analyze the given Slack thread messages and extract the following fields:

- title: concise ticket title (max 100 characters)
- description: detailed description in Jira markdown format
- priority: one of [Highest, High, Medium, Low, Lowest]
- labels: list of relevant tags (max 5)

Respond ONLY in valid JSON format. No additional text.
```

### User Message 구성
```
다음 Slack 스레드 내용을 Jira 티켓으로 만들어줘:

---
[작성자1]: 메시지 내용
[작성자2]: 메시지 내용
...
---
```

---

## 에이전트 작업 가이드라인

### 코드 수정 시
- `src/services/` 내 각 서비스는 독립적으로 테스트 가능하게 유지
- 환경변수는 항상 `src/config.py`를 통해 접근 (하드코딩 금지)
- Jira/Slack API 호출 실패 시 예외를 상위로 전파하고 Slack에 오류 회신

### 새 핸들러 추가 시
- `src/handlers/` 에 새 파일 생성
- `src/app.py`에 핸들러 등록
- `AGENTS.md`의 트리거 조건 업데이트

### 테스트
- 각 서비스는 `tests/test_services.py`에서 단위 테스트
- LLM 호출은 mock 처리 (비용 절감)
- Jira/Slack은 실제 API로 통합 테스트

---

## Phase 2: Remote MCP 서버 + AWS 전환 (예정)

### 목표
- LLM 백엔드를 Claude API → **AWS Bedrock** (Claude 모델)으로 교체
- 인프라를 로컬 → **AWS 클라우드** (ECS Fargate 또는 Lambda + API Gateway)로 이전
- `mcp_server/` 에 MCP SDK 기반 서버 구현 (Streamable HTTP transport)

### LLM 백엔드 변경
- `src/services/llm.py` 에서 `anthropic` → `boto3` (Bedrock Runtime)
- 모델: `anthropic.claude-3-5-sonnet-20241022-v2:0` 등 Bedrock 지원 모델
- 환경변수 추가: `AWS_REGION`, `BEDROCK_MODEL_ID`

### 계획 중인 MCP Tools (`mcp_server/`)

| 파일 | Tool | 설명 |
|------|------|------|
| `jira_mcp.py` | `create_jira_ticket` | title, description, priority → Jira 티켓 생성 |
| `jira_mcp.py` | `search_jira_issues` | JQL 쿼리로 이슈 검색 |
| `jira_mcp.py` | `get_jira_ticket` | issue key로 티켓 상세 조회 |
| `slack_mcp.py` | `read_slack_thread` | channel, thread_ts → 스레드 메시지 반환 |
| `slack_mcp.py` | `post_slack_message` | 채널/스레드에 메시지 전송 |

### 구현 방식
- `mcp` Python SDK 사용
- `src/services/` 의 기존 서비스 재사용
- Streamable HTTP transport로 Remote MCP 클라이언트 지원

---

## Phase 3: 모니터링 알람 자동 대응 (예정)

### 목표
모니터링 알람을 수신해 LLM이 원인을 분석하고 대응 액션을 자동 실행.

### 트리거 조건
- Datadog 알람 → Slack 채널 메시지
- Prometheus AlertManager → Slack webhook
- Loki 로그 이상 감지 → Slack 알림

### 단계별 처리 흐름

```
[1] 알람 메시지 수신 (Slack 이벤트 또는 Webhook)
    - 알람 유형 분류 (Datadog / Prometheus / Loki)

[2] 관련 메트릭/로그 수집
    - Datadog MCP: get_metrics, get_events
    - Prometheus MCP: query, query_range
    - Loki MCP: query_logs

[3] Kubernetes 상태 수집
    - kubernetes-mcp: get_pods, get_events, describe_node

[4] LLM (Bedrock) 원인 분석 및 대응 방안 도출
    - 수집된 메트릭/로그/K8s 상태를 컨텍스트로 전달
    - 원인 추론 + 권장 액션 생성

[5] 자동 대응 실행 또는 인시던트 티켓 생성
    - 자동 실행: kubernetes-mcp로 Pod 재시작 등
    - 티켓 생성: Jira 인시던트 티켓 (create_incident_ticket)

[6] Slack 채널에 분석 결과 및 조치 내용 회신
```

### 계획 중인 MCP Tools (`mcp_server/monitoring_mcp.py`)

| Tool | 설명 |
|------|------|
| `get_datadog_alert` | 알람 상세 정보 조회 |
| `get_datadog_metrics` | 특정 메트릭 시계열 데이터 조회 |
| `query_prometheus` | PromQL 쿼리 실행 |
| `query_loki_logs` | LogQL 쿼리로 로그 조회 |
| `get_k8s_pods` | 네임스페이스 Pod 상태 조회 |
| `restart_k8s_deployment` | Deployment 롤링 재시작 |
| `create_incident_ticket` | Jira 인시던트 티켓 생성 |

# AI 챗봇 패널 설계 문서

## 목표

주간업무보고 앱 우측에 AI 어시스턴트 패널을 추가한다. 팀 전체 보고 데이터를 컨텍스트로 받아 요약·조언·수정 제안을 제공하며, 사용자 승인 시 state를 직접 업데이트한다.

## 아키텍처

단일 `index.html` 파일에 CSS + HTML + JS 추가. 외부 의존성 없음. Claude API는 브라우저에서 `fetch`로 직접 호출한다.

**모델:** `claude-haiku-4-5-20251001`
**API Key:** 사용자가 UI에서 입력 → `localStorage['weekly_chat_api_key']`에 저장 (state와 분리)

---

## 레이아웃

우측 **고정 오버레이 패널** (width: 360px, 메인 콘텐츠 위에 겹침).

```
[사이드바 240px] [메인 콘텐츠 flex:1] [AI 패널 360px fixed right, z-index:200]
```

- **기본:** 닫힌 상태. `state.chatOpen`은 저장하지 않음 (새로고침 시 항상 닫힘)
- **토글 버튼:** 사이드바 하단 footer (`AI 어시스턴트` 버튼)
- **패널 헤더:** "AI 어시스턴트" 제목 + [×] 닫기 버튼
- **API Key 미설정 시:** 패널 내부에 key 입력 UI 표시 (저장 후 대화 가능)

### 패널 구조

```
#chatPanel (fixed, right:0, top:0, height:100vh, width:360px, z-index:200)
├── .chat-header
│   ├── "AI 어시스턴트"
│   └── [×] 닫기
├── #chatApiKeySetup  ← API key 미설정 시에만 표시
│   ├── 안내 텍스트 + type="password" input
│   └── [저장] 버튼
├── #chatMessages (overflow-y:auto, flex:1)
│   └── .chat-msg (user | assistant | tool-card)
└── .chat-input-row
    ├── <textarea> (Enter=전송, Shift+Enter=줄바꿈)
    └── [전송] 버튼 — 요청 중 disabled + "..." 표시
```

---

## 상태 관리

### state에 추가 (저장되는 것)
```javascript
state.chatMessages = []
// 각 항목: { role: 'user'|'assistant', content: string, tools: [...] | null }
// tools: tool_use 블록 배열 (적용/취소 상태 포함)
```

### state에 추가 (저장 안 되는 것, 런타임 전용)
```javascript
state.chatOpen = false   // 새로고침 시 항상 닫힘
```

### 대화 기록 관리
- `state.chatMessages`는 `saveState()` / `loadState()`에 포함
- 최대 20개 메시지 유지. 초과 시 **가장 오래된 user+assistant 쌍**부터 제거
- API 호출 시에는 최근 20개만 전송 (전체 기록은 저장 유지)

---

## Claude API 호출

### 엔드포인트
```
POST https://api.anthropic.com/v1/messages
Headers:
  x-api-key: <key>
  anthropic-version: 2023-06-01
  anthropic-dangerous-direct-browser-access: true
  content-type: application/json
```

### Request Body
```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 1024,
  "system": "<system prompt>",
  "tools": [ ...5개 tool 정의 (아래 스키마 그대로 배열로 전달)... ],
  "messages": [ ...최근 20개... ]
}
```

`tools` 배열을 request body에 명시해야 Claude가 `tool_use` 블록을 반환한다. 생략 시 텍스트 응답만 반환됨.

### System Prompt
```
당신은 전략개발팀 주간업무보고 어시스턴트입니다.
현재 보고기간: {year}년 {month}월 {week}주차

아래는 팀 전체 데이터입니다:
{buildChatContext()}

내용 수정이 필요하면 반드시 tool을 사용해 제안하세요.
텍스트로만 답할 때는 tool을 사용하지 않아도 됩니다.
```

### buildChatContext()

팀원별 데이터를 마크다운 텍스트로 직렬화. JSON 대신 사람이 읽기 쉬운 형식으로 토큰 절약.

```
[조낙운] (ID: cho_nakwoon)
과제 1: "병원 시스템 고도화" (ID: task-xxx, 진척 60%, ~2026-04-30)
  금주활동:
    - API 연동 완료 (ID: item-aaa, 80%)
  차주계획:
    - 테스트 케이스 작성 (ID: item-bbb)
  이슈: (없음)
운영/요청: (없음)

[김유정] (ID: kim_yujeong)
...
```

- 빈 항목 → "(없음)"
- 배포는 제품명+차수+진척만 포함 (레드마인 항목 생략)
- ID 값을 명시해야 tool이 올바르게 타깃팅 가능

---

## Tool Use 스키마 (5개)

AI가 수정을 제안할 때 아래 tool을 사용한다. 모든 식별자는 state의 `id` 필드(UUID)를 사용한다.

### `add_activity`
금주활동 또는 차주계획 항목 추가
```json
{
  "name": "add_activity",
  "description": "팀원의 특정 과제에 금주활동(current) 또는 차주계획(next) 항목을 추가합니다",
  "input_schema": {
    "type": "object",
    "properties": {
      "memberId": { "type": "string", "description": "팀원 ID (예: kim_yujeong)" },
      "taskId":   { "type": "string", "description": "과제 ID" },
      "weekType": { "type": "string", "enum": ["current", "next"] },
      "text":     { "type": "string", "description": "활동 내용" },
      "progress": { "type": "number", "description": "진척률 0-100 (생략 가능)" }
    },
    "required": ["memberId", "taskId", "weekType", "text"]
  }
}
```

### `update_activity`
기존 활동 항목 텍스트/진척률 수정
```json
{
  "name": "update_activity",
  "description": "팀원의 기존 활동 항목을 수정합니다",
  "input_schema": {
    "type": "object",
    "properties": {
      "memberId": { "type": "string" },
      "taskId":   { "type": "string" },
      "weekType": { "type": "string", "enum": ["current", "next"] },
      "itemId":   { "type": "string", "description": "활동 항목 ID" },
      "text":     { "type": "string", "description": "새 활동 내용 (생략 시 유지)" },
      "progress": { "type": "number", "description": "새 진척률 0-100 (생략 시 유지)" }
    },
    "required": ["memberId", "taskId", "weekType", "itemId"]
  }
}
```

### `update_task_meta`
과제 메타정보 수정
```json
{
  "name": "update_task_meta",
  "description": "과제의 KPI/제품명, 마감일, 전체 진척률을 수정합니다",
  "input_schema": {
    "type": "object",
    "properties": {
      "memberId":        { "type": "string" },
      "taskId":          { "type": "string" },
      "kpiProduct":      { "type": "string", "description": "KPI/제품명 (생략 시 유지)" },
      "deadline":        { "type": "string", "description": "YYYY-MM-DD (생략 시 유지)" },
      "overallProgress": { "type": "number", "description": "0-100 (생략 시 유지)" }
    },
    "required": ["memberId", "taskId"]
  }
}
```

### `add_issue`
과제 이슈 항목 추가
```json
{
  "name": "add_issue",
  "description": "팀원의 특정 과제에 이슈 항목을 추가합니다",
  "input_schema": {
    "type": "object",
    "properties": {
      "memberId": { "type": "string" },
      "taskId":   { "type": "string" },
      "text":     { "type": "string", "description": "이슈 내용" }
    },
    "required": ["memberId", "taskId", "text"]
  }
}
```

### `add_operation`
운영/요청사항 추가
```json
{
  "name": "add_operation",
  "description": "팀원의 운영/요청사항을 추가합니다",
  "input_schema": {
    "type": "object",
    "properties": {
      "memberId": { "type": "string" },
      "text":     { "type": "string" },
      "weekType": { "type": "string", "enum": ["current", "next"] }
    },
    "required": ["memberId", "text", "weekType"]
  }
}
```

**이번 범위에서 제외 (추후):** 과제 신규 추가, 항목 삭제, 배포 수정

---

## 응답 렌더링

Claude API 응답의 `content` 배열을 순서대로 처리한다.

| content 블록 타입 | 렌더링 |
|---|---|
| `text` | 일반 어시스턴트 메시지 말풍선으로 표시 |
| `tool_use` | 제안 카드로 표시 (아래 참조) |

### 제안 카드 UI
```
┌─────────────────────────────────┐
│ 💡 AI 제안                      │
│ 김유정 › 과제명 › 금주활동 추가 │
│ " - API 연동 완료 (80%)"        │
│                   [적용] [취소] │
└─────────────────────────────────┘
```
- **[적용]** → tool input 파싱 → state 반영 → `saveState()` → `renderMain()` → 카드 "✓ 적용됨"으로 교체
- **[취소]** → 카드 회색 처리 + "취소됨" 표시
- 하나의 응답에 tool 여러 개 → 카드 여러 개, 각각 독립 적용
- `renderMain()` 호출로 스크롤 위치가 리셋되는 것은 이번 범위에서 허용

---

## Tool 적용 로직

```javascript
function applyToolUse(toolName, input) {
  const mdata = state.members[input.memberId];
  if (!mdata) return false;
  const task = mdata.tasks.find(t => t.id === input.taskId);  // ID로 찾기

  switch (toolName) {
    case 'add_activity': { /* 새 item 생성, uuid 부여 */ break; }
    case 'update_activity': { /* item.id로 찾아 text/progress 수정 */ break; }
    case 'update_task_meta': { /* task 필드 덮어쓰기 */ break; }
    case 'add_issue': { /* 새 issue 추가 */ break; }
    case 'add_operation': { /* 새 operation 추가 */ break; }
  }
  saveState();
  renderMain();
  return true;
}
```

ID를 찾지 못하면 `false` 반환 → 카드에 "해당 항목을 찾을 수 없습니다" 표시.

---

## 에러 처리

| 상황 | UI 처리 |
|------|---------|
| API Key 없음 | 패널 내 설정 UI 표시 (toast 아님) |
| 401 Unauthorized | 채팅창 인라인 에러 메시지 "API Key가 올바르지 않습니다" |
| 네트워크 오류 | 채팅창 인라인 "요청 실패, 다시 시도해주세요" |
| tool ID 불일치 | 제안 카드에 "해당 항목을 찾을 수 없습니다" |
| 요청 중 재전송 방지 | 전송 버튼 disabled + 텍스트 "..." + 채팅창 하단에 "응답 대기 중..." 메시지 표시 |
| API Key localStorage 노출 | 로컬 전용 도구이므로 허용. 입력란은 type="password"로 마스킹. 저장 시 "API Key가 로컬에 저장됩니다" 안내 텍스트 표시. |

---

## 구현 범위 (이번 Phase)

- [ ] 패널 HTML/CSS + 토글
- [ ] API Key 설정 UI (type="password", localStorage 저장)
- [ ] `buildChatContext()` — 팀 데이터 → 마크다운 텍스트
- [ ] Claude API fetch 호출
- [ ] 텍스트 응답 말풍선 렌더링
- [ ] tool_use 파싱 + 제안 카드 UI
- [ ] 5개 tool 적용 로직 (`applyToolUse`)
- [ ] 대화 기록 최대 20턴 관리
- [ ] 전송 중 버튼 disabled

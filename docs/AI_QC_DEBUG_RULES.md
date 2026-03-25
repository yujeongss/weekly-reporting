# AI 어시스턴트 QC / 디버그 규칙

> 최종 업데이트: 2026-03-24
> 테스트 환경: Chrome (Playwright), `index.html` file:// 로컬 실행
> 모델: `claude-haiku-4-5-20251001`

---

## 1. 테스트 케이스 목록 및 통과 기준

### 기존 TC (test_ai_assistant.mjs — 앱 로딩 / 채팅 기본 동작)

| TC | 항목 | 통과 기준 | 마지막 결과 |
|----|------|----------|------------|
| TC-01 | 앱 로딩 | 타이틀 "주간업무보고" 포함, 사이드바 렌더 | ✅ PASS |
| TC-02 | 팀원별 초기화 버튼 | `.member-reset-btn` 존재, 텍스트 "초기화" | ✅ PASS |
| TC-03 | AI FAB 버튼 | `#chatFab` 존재 및 visible | ✅ PASS |
| TC-04 | 채팅 패널 열기 | `#chatPanel.open` 존재, 환영 메시지 출력 | ✅ PASS |
| TC-05 | API Key 설정 | `window.CLAUDE_API_KEY` 존재, `sk-ant-` 형식 | ✅ PASS |
| TC-06 | 메시지 전송 및 AI 응답 | 20초 내 assistant 메시지 수신 | ✅ PASS |
| TC-07 | 도구 호출 및 적용 | `.chat-tool-card` 생성, 적용 버튼 동작 | ✅ PASS |
| TC-08 | 콘솔 에러 없음 | `console.error` / `pageerror` 0건 | ✅ PASS |

**기존 TC 최종 결과: ✅ PASS 15 / ❌ FAIL 0 / ⚠️ WARN 0** (2026-03-24)

---

### CRUD TC (test_ai_crud.mjs — 추가/수정/삭제 전체 검증)

| TC | 그룹 | 항목 | 통과 기준 | 마지막 결과 |
|----|------|------|----------|------------|
| TC-A1 | 운영사항 | 추가 | 도구 적용 + localStorage + DOM | ✅ PASS |
| TC-A2 | 운영사항 | 수정 | 도구 적용 + 새값/구값 + DOM | ✅ PASS |
| TC-A3 | 운영사항 | 삭제 | 도구 적용 + localStorage(0) + DOM 소거 | ✅ PASS |
| TC-B1 | 금주활동 | 추가 | 도구 적용 + localStorage + DOM | ✅ PASS |
| TC-B2 | 금주활동 | 수정 | 도구 적용 + 새값/구값 + DOM | ✅ PASS |
| TC-B3 | 금주활동 | 삭제 | 도구 적용 + localStorage(0) + DOM 소거 | ✅ PASS |
| TC-C1 | 이슈 | 추가 | 도구 적용 + localStorage + DOM | ✅ PASS |
| TC-C2 | 이슈 | 삭제 | 도구 적용 + localStorage(0) + DOM 소거 | ✅ PASS |
| TC-D1 | 배포 | 추가 | 도구 적용 + localStorage(id/date) + DOM | ✅ PASS |
| TC-D2 | 배포 | 수정 | 도구 적용 + 날짜/버그수 검증 | ✅ PASS |
| TC-D3 | 배포 | 삭제 | 도구 적용 + localStorage(0) + DOM 소거 | ✅ PASS |
| TC-E1 | 차주계획 | 추가 | 도구 적용 + localStorage + DOM | ✅ PASS |
| TC-E2 | 차주계획 | 수정 | 도구 적용 + 새값/구값 + DOM | ✅ PASS |
| TC-E3 | 차주계획 | 삭제 | 도구 적용 + localStorage(0) + DOM 소거 | ✅ PASS |
| TC-F1 | update_task_meta | 진척률 수정 | 도구 적용 + localStorage(overallProgress=75) | ✅ PASS |
| TC-F2 | update_task_meta | 과제명 수정 | 도구 적용 + localStorage(kpiProduct) + DOM | ✅ PASS |
| TC-F3 | update_task_meta | 기한 수정 | 도구 적용 + localStorage(deadline) + input[date] value | ✅ PASS |
| TC-G1 | 과제 | 추가 (add_task) | 도구 적용 + localStorage + DOM | ✅ PASS |
| TC-G2 | 과제 | 삭제 (delete_task) | 도구 적용 + localStorage 항목 소거 + DOM 소거 | ✅ PASS |
| TC-H1 | 금주활동 상세 | 추가 (add_detail) | 도구 적용 + localStorage + DOM | ✅ PASS |
| TC-H2 | 금주활동 상세 | 수정 (update_detail) | 도구 적용 + 새값/구값 + DOM | ✅ PASS |
| TC-H3 | 금주활동 상세 | 삭제 (delete_detail) | 도구 적용 + localStorage 항목 소거 + DOM 소거 | ✅ PASS |
| TC-I1 | 차주계획 상세 | 추가 (add_detail) | 도구 적용 + localStorage + DOM | ✅ PASS |
| TC-I2 | 차주계획 상세 | 삭제 (delete_detail) | 도구 적용 + localStorage(0) + DOM 소거 | ✅ PASS |
| TC-J1 | update_activity | 활동 진척률 수정 | 도구 적용 + localStorage(progress=60) | ✅ PASS |

**CRUD TC 최종 결과: ✅ PASS 78 / ❌ FAIL 0 / ⚠️ WARN 2** (2026-03-24)

> WARN: TC-F1 DOM 검증 — 진척률이 CSS progress bar로만 표시되어 텍스트로 검증 불가. localStorage는 정확히 75 확인 (기능 정상).
> WARN: TC-J1 DOM 검증 — 활동 항목 진척률도 동일 사유. localStorage 60 확인 (기능 정상).

---

## 2. 스크린샷 목록

### 기존 TC (docs/screenshots/)

| 파일명 | 내용 |
|--------|------|
| 01_app_loaded.png | 앱 초기 로딩 화면 |
| 02_member_reset_button.png | 팀원별 초기화 버튼 |
| 03_fab_visible.png | AI FAB 버튼 |
| 04_chat_panel_open.png | 채팅 패널 열린 상태 |
| 05_message_typed.png | 메시지 입력 상태 |
| 06_message_sending.png | 전송 직후 (로딩 중) |
| 07_ai_response.png | AI 응답 수신 후 |
| 08_tool_card.png | 도구 카드 표시 상태 |
| 09_tool_applied.png | 도구 적용 후 |
| 10_final_state.png | 테스트 종료 최종 화면 |

### CRUD TC (docs/screenshots/ — crud_ 접두사)

| 파일명 | 내용 |
|--------|------|
| crud_00_start.png | CRUD 테스트 시작 상태 |
| crud_A1_op_added.png | 운영사항 추가 후 |
| crud_A2_op_updated.png | 운영사항 수정 후 |
| crud_A3_op_deleted.png | 운영사항 삭제 후 |
| crud_B_chat_reset.png | Group B 채팅 초기화 후 |
| crud_B1_act_added.png | 금주활동 추가 후 |
| crud_B2_act_updated.png | 금주활동 수정 후 |
| crud_B3_act_deleted.png | 금주활동 삭제 후 |
| crud_C1_issue_added.png | 이슈 추가 후 |
| crud_C2_issue_deleted.png | 이슈 삭제 후 |
| crud_D1_deploy_added.png | 배포 추가 후 |
| crud_D2_deploy_updated.png | 배포 수정 후 |
| crud_D3_deploy_deleted.png | 배포 삭제 후 |
| crud_E1_next_added.png | 차주계획 추가 후 |
| crud_E2_next_updated.png | 차주계획 수정 후 |
| crud_E3_next_deleted.png | 차주계획 삭제 후 |
| crud_F1_task_meta_updated.png | 진척률 수정 후 |
| crud_F2_task_name_updated.png | 과제명 수정 후 |
| crud_F3_deadline_updated.png | 기한 수정 후 |
| crud_G1_task_added.png | 과제 추가 후 |
| crud_G2_task_deleted.png | 과제 삭제 후 |
| crud_H1_detail_added.png | 금주활동 상세 추가 후 |
| crud_H2_detail_updated.png | 금주활동 상세 수정 후 |
| crud_H3_detail_deleted.png | 금주활동 상세 삭제 후 |
| crud_I1_next_detail_added.png | 차주계획 상세 추가 후 |
| crud_I2_next_detail_deleted.png | 차주계획 상세 삭제 후 |
| crud_J1_activity_progress_updated.png | 활동 진척률 수정 후 |
| crud_ZZ_final.png | CRUD 테스트 최종 화면 |

---

## 3. 알려진 함정 (Known Pitfalls)

### 2-1. FAB 버튼 애니메이션 클릭 실패

**증상:** `elementHandle.click: Timeout 30000ms exceeded — element is not stable`
**원인:** `#chatFab`에 `fabBounce` CSS 애니메이션이 적용되어 Playwright가 요소를 "stable"로 판단하지 못함
**해결책:** Playwright 클릭 시 반드시 `{ force: true }` 사용
```js
await fab.click({ force: true });
```

---

### 2-2. 로딩 메시지를 AI 응답으로 오인

**증상:** TC-07에서 `.chat-msg` 수가 증가했지만 텍스트가 "응답 대기 중..."
**원인:** `waitForFunction`이 `.chat-loading` 메시지도 카운트해 조기 종료
**해결책:** 대기 조건에서 `.chat-loading` 요소가 없을 때까지 추가 대기
```js
await page.waitForFunction((before) => {
  const loading = document.querySelector('.chat-msg.chat-loading');
  if (loading) return false;
  return document.querySelectorAll('.chat-msg, .chat-tool-card').length > before;
}, countBefore, { timeout: 30000 });
```

---

### 2-3. 도구 적용 버튼 클릭 타임아웃

**증상:** 도구 카드가 생성됐으나 `.chat-tool-apply-btn` 클릭 실패
**원인:** 채팅 패널 스크롤 위치로 인해 버튼이 viewport 밖에 있을 수 있음
**해결책:** 클릭 전 `scrollIntoViewIfNeeded()` + `{ force: true }` 적용
```js
await applyBtn.scrollIntoViewIfNeeded();
await applyBtn.click({ force: true });
```

---

### 2-4. CORS 에러 (file:// 실행 시)

**증상:** `fetch: net::ERR_FAILED` 또는 CORS 에러
**원인:** `file://` 프로토콜에서 Anthropic API 직접 호출 시 일부 브라우저가 차단
**해결책:** API 호출에 `anthropic-dangerous-direct-browser-access: true` 헤더 포함 (이미 적용됨)
```js
headers: {
  'anthropic-dangerous-direct-browser-access': 'true',
  ...
}
```
> Chrome은 정상 동작. Firefox는 추가 설정 필요할 수 있음.

---

### 2-5. API Key 미설정

**증상:** "config.js에 CLAUDE_API_KEY를 입력해 주세요." 에러 메시지
**원인:** `config.js`의 `CLAUDE_API_KEY` 값이 기본 플레이스홀더 상태
**해결책:** `config.js`에 실제 API 키 입력
```js
window.CLAUDE_API_KEY = 'sk-ant-api03-...실제키...';
```

---

### 2-6. 도구 호출 없이 텍스트로만 응답

**증상:** `.chat-tool-card`가 생성되지 않고 텍스트 응답만 옴
**원인:**
  - 요청이 애매하거나 대상 팀원/과제를 특정하지 못함
  - 모델이 상황에 따라 도구 없이 설명만 응답할 수 있음
**해결책:**
  - 팀원 이름 + 작업을 명확히 지정: "조낙운의 이번주 운영사항에 X 추가해줘"
  - 도구 없이 텍스트 응답은 정상 동작 범위이므로 `WARN`으로 처리

---

### 2-7. 채팅 히스토리 누적으로 응답 품질 저하

**증상:** 대화가 길어질수록 AI가 이전 context를 혼동하거나 도구 호출 정확도가 떨어짐
**원인:** `state.chatMessages`가 최대 20개 메시지로 trim되며, trim 기준이 역할(role) 순서를 깨뜨릴 수 있음
**위치:** `index.html` `_trimChatHistory()` 함수
**해결책:** 테스트 시작 시 새 브라우저 컨텍스트 사용 (localStorage 초기화 상태)

---

### 2-8. ⭐ FAB 클릭 후 패널 미오픈 → 첫 메시지 35s timeout (CRUD 테스트 주요 함정)

**증상:** `resetChatAndReopen` 후 첫 메시지 전송이 35초 timeout
**원인:** `#chatFab`의 `fabBounce` 애니메이션으로 인해 Playwright `click({ force: true })`가 발화하지 않아
  채팅 패널이 닫힌 채로 `sendChatMessage()`가 호출되지 않음
**해결책:** FAB 클릭 없이 `openChat()` 직접 호출
```js
await page.evaluate(() => {
  if (window._chatOpen) closeChat();
  openChat();
});
await page.waitForFunction(
  () => document.getElementById('chatPanel').classList.contains('open'),
  { timeout: 5000 }
);
```

---

### 2-9. ⭐ document.body.innerText로 삭제 검증 시 오탐 (false PASS/false FAIL)

**증상:** 항목 삭제 후 DOM 소거 검증이 FAIL (채팅 카드에 삭제 대상 텍스트가 남아있음)
**원인:** 도구 카드의 설명 텍스트(e.g., "조낙운 › 운영/요청 삭제: QC_OP_UPDATED_항목")가
  `document.body.innerText`에 포함되어 삭제 후에도 텍스트가 감지됨
**해결책:** `#mainContent` 범위로 한정
```js
const domText = await page.evaluate((u) =>
  document.getElementById('mainContent')?.innerText.includes(u), TARGET) ?? false;
```

---

### 2-10. ⭐ STORAGE_KEY 불일치 (test vs app)

**증상:** `injectTestTask`에서 `Cannot read properties of null (reading 'members')`
**원인:** 앱의 `STORAGE_KEY = 'weekly_report_data_v7'`이지만 테스트가 `v1`을 사용
**해결책:** 테스트 파일 상단의 `STORAGE_KEY` 상수를 앱과 일치시킬 것.
  단, `loadState()`가 localStorage를 무시하고 항상 fresh start하는 경우 `window.state` 직접 주입 필요:
```js
await page.evaluate(() => {
  const mem = state.members['kim_yujeong'];
  mem.tasks.unshift({ id: 'qc-task-001', ... });
  saveState();
  renderMain();
});
```

---

### 2-11. ⭐ Rate limit 429 (50,000 input tokens/min)

**증상:** 연속 API 호출 시 `요청 실패 (429): This request would exceed the rate limit`
**원인:** 팀 전체 데이터를 system prompt에 포함하므로 호출당 토큰이 많음.
  6명 × 과제/운영/배포 데이터 = ~2000-3000 tokens/call
**해결책:**
- 테스트 그룹 간 4초 딜레이 추가 (최소)
- TC 간 딜레이 6초 기본 (`await page.waitForTimeout(6000)`)
- GROUP H→I처럼 연속 API 호출이 집중되는 구간은 10초 이상으로 늘릴 것
```js
// H3→I2 구간처럼 연속 호출이 많을 경우
await page.waitForTimeout(10000);
```

---

### 2-12. ⭐ 배포 삭제 시 AI가 ID를 잘못 복사

**증상:** `delete_deployment` 도구 적용 후 localStorage에 항목이 남아있음
**원인:** AI가 context의 자동생성 ID(e.g., `mn43obxdrtznz2u`)를 다른 값으로 복사하거나 혼동
**해결책:** `_applyToolUse('delete_deployment', ...)` 에 product 이름 fallback 추가
```js
let toDelete = mdata.deployments.find(d => d.id === inp.deploymentId);
if (!toDelete && inp.product) toDelete = mdata.deployments.find(d => d.product === inp.product);
```
  또한 `delete_deployment` tool schema의 `deploymentId` description에
  "context의 (ID: ...) 값을 그대로 복사하세요" 명시.

---

### 2-13. ⭐ update_activity / delete_activity / delete_issue 에서 AI가 잘못된 ID 전달

**증상:** 도구 카드가 표시되고 apply 클릭까지 성공하지만 실제 state가 변경되지 않음
**원인:** AI가 자동생성 ID(`mn46...` 형태)를 context에서 정확히 복사하지 않고 틀린 ID 전달
  → `_applyToolUse`에서 `arr.find(i => i.id === inp.itemId)` 가 null을 반환하고 `return false`

**해결책:**
1. `_applyToolUse`에 텍스트 기반 fallback 추가:
```js
// update_activity
let item = arr.find(i => i.id === inp.itemId);
if (!item && inp.originalText) item = arr.find(i => i.text === inp.originalText);

// delete_activity
const toDelAct = arr.find(i => i.id === inp.itemId) ||
  (inp.itemText ? arr.find(i => i.text === inp.itemText) : null);

// delete_issue
const toDelIssue = issues.find(i => i.id === inp.issueId) ||
  (inp.issueText ? issues.find(i => i.text === inp.issueText) : null);
```
2. tool schema에 fallback 파라미터 추가:
   - `update_activity`: `originalText` (수정 전 항목 텍스트)
   - `delete_activity`: `itemText` (삭제할 항목 텍스트)
   - `delete_issue`: `issueText` (삭제할 이슈 텍스트)
3. 시스템 프롬프트에 "originalText / itemText / issueText 함께 전달" 명시

---

### 2-14. ⭐ 연속 TC를 같은 채팅 세션에서 실행하면 AI가 혼동 (수정/삭제 주로 영향)

**증상:** B2(수정), E2(차주수정), C2(이슈삭제), D3(배포삭제)에서 도구 적용됐지만 state 불변
  또는 AI가 이전 대화 이력을 보고 잘못된 tool을 호출
**원인:** 같은 채팅 세션에서 add → update/delete 순서로 실행 시,
  이전 add의 대화 이력이 축적되어 AI가 itemId를 혼동

**해결책:** 수정/삭제 TC 직전에 `resetChatAndReopen` 호출 (독립 세션으로 분리)
```js
// 각 수정/삭제 TC 바로 앞
await page.waitForTimeout(6000);
await resetChatAndReopen(page);

// TC-B2, TC-B3, TC-C2, TC-D3, TC-E2, TC-E3 모두 적용
```
  단, `resetChatAndReopen`은 `state.chatMessages` 초기화 + `openChat()` 직접 호출이므로
  주입한 QC task 데이터는 유지됨.

---

## 4. 디버그 체크리스트

테스트 실패 시 아래 순서로 확인:

```
[ ] 1. 브라우저 개발자 도구 콘솔에서 [chat] 접두사 로그 확인
       - "[chat] API response:" → API 응답 원문
       - "[chat] tool call:" → 도구 호출 파라미터
       - "[chat] rendering N messages" → 렌더링 시점 확인

[ ] 2. Network 탭에서 api.anthropic.com 요청 상태 확인
       - 200: 정상
       - 401: API Key 오류
       - 429: Rate limit 초과 → 4초 이상 대기 후 재시도
       - 0 / ERR_FAILED: CORS 또는 네트워크 문제

[ ] 3. localStorage 확인 (Application 탭)
       - 키: weekly_report_data_v7
       - chatMessages 배열 길이 및 구조 확인

[ ] 4. window.CLAUDE_API_KEY 직접 확인
       - 콘솔: console.log(window.CLAUDE_API_KEY?.slice(0,20))

[ ] 5. 도구 카드가 안 뜨는 경우
       - 콘솔에서 "[chat] tool call:" 로그 확인
       - tool_use 블록이 있는데 카드가 없다면 _renderToolCard() 에러 확인
       - memberId 매핑 실패 여부: _resolveMemberId() 로그 추가
```

---

## 5. 테스트 실행 방법

```bash
# 프로젝트 루트에서
cd D:\work\claude-test\weekly-reporting

# 기존 TC (앱 로딩 / 채팅 기본 동작)
node --experimental-vm-modules docs/test_ai_assistant.mjs

# CRUD TC (추가/수정/삭제 전체 검증) ← 메인 QC
node --experimental-vm-modules docs/test_ai_crud.mjs
```

스크린샷 저장 위치: `docs/screenshots/`

---

## 6. AI 어시스턴트 기능 범위

### 지원하는 도구 (tools)

| 도구명 | 설명 | 필수 파라미터 | fallback |
|--------|------|--------------|---------|
| `add_activity` | 금주활동/차주계획 항목 추가 | memberId, taskId, weekType, text | — |
| `update_activity` | 활동 내용/진척률 수정 | memberId, taskId, weekType, itemId | originalText |
| `update_task_meta` | 과제명, 기한, 전체진척도 수정 | memberId, taskId | — |
| `add_issue` | 이슈사항 추가 | memberId, taskId, text | — |
| `add_operation` | 운영/요청사항 추가 | memberId, text, weekType | — |
| `add_detail` | 활동 하위 세부항목 추가 | memberId, taskId, itemId, weekType, text | — |
| `update_detail` | 활동 하위 세부항목 수정 | memberId, taskId, weekType, itemId, detailId, text | originalText |
| `update_operation` | 운영사항 수정 | memberId, operationId | — |
| `add_deployment` | 배포 정보 추가 | memberId, product | — |
| `update_deployment` | 배포 정보 수정 | memberId, deploymentId | — |
| `add_task` | 과제 추가 | memberId, kpiProduct | — |
| `delete_activity` | 활동 삭제 | memberId, taskId, weekType, itemId | itemText |
| `delete_detail` | 활동 하위 세부항목 삭제 | memberId, taskId, weekType, itemId, detailId | originalText |
| `delete_issue` | 이슈 삭제 | memberId, taskId, issueId | issueText |
| `delete_operation` | 운영사항 삭제 | memberId, operationId | — |
| `delete_deployment` | 배포 삭제 | memberId, deploymentId | product |
| `delete_task` | 과제 삭제 | memberId, taskId | kpiProduct |

### 지원하지 않는 기능 (현재 버전)

- 팀원 추가/삭제
- 보고 기간 변경
- Excel 내보내기 트리거

---

## 7. 환경 요구사항

| 항목 | 요구사항 |
|------|----------|
| 브라우저 | Chromium (Playwright) / Chrome 권장 |
| Node.js | 18+ (ESM 지원) |
| Playwright | 1.58+ |
| API 키 | Anthropic API Key (`sk-ant-api03-...`) |
| 네트워크 | `api.anthropic.com` 접근 가능 |
| 실행 위치 | `file://` 직접 실행 가능 (CORS 헤더 포함) |

# Phase 3 — UX 심화 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 주간업무보고 앱에 Undo, 복사/붙여넣기, 주간 이월, 드래그앤드롭, 진척율 select, 검색, 반응형, 온보딩, 주차별 비교 9가지 UX 기능 추가

**Architecture:** 단일 파일 `index.html` (~5607줄) 내 CSS + JS 수정. 모든 새 상태는 모듈 레벨 변수(`_undoStack`, `_clipboard`, `_dragSrc`, `_search`, `_onboarding`)로 관리. localStorage에는 `weekly_report_snapshots` key만 추가.

**Tech Stack:** Vanilla JS, inline CSS, HTML5 Drag & Drop API, ExcelJS 4.4.0 (기존, 변경 없음)

---

## 코드 현황 (Phase 2 완료 후)

- `index.html` ~5607줄
- `renderTaskCard(memberId, task)` — line ~2475, 과제 카드 HTML 반환
- `renderTasksSection(memberId, data)` — line ~2455, tasks 목록 + 추가 버튼
- `removeTask(memberId, taskId)` — line ~2754, confirm 후 즉시 삭제 + saveState
- `applyPeriod()` — line ~2383, 보고기간 변경 후 saveState + renderMain
- `showToast(msg)` — line ~4641, `_toastTimer` 사용, `el.textContent = msg`
- `generateId()` — line ~2250, 고유 ID 생성
- `getMemberData(memberId)` — member state 반환 (없으면 empty 반환)
- `getMember(memberId)` — member meta 반환
- `esc(str)` — XSS 방지
- `saveState()` / `loadState()` — localStorage `weekly_report_data_v7` 키
- 모듈 레벨 변수: `_teamSummaryCollapsed` (line ~4341), `_teamFilter` (line ~4342), `_toastTimer` (line ~4543)
- `#sidebar` (aside element, line ~1914), 너비 252px
- `<main id="main"><div id="mainContent"></div></main>` (line ~1966)

---

## 파일 구조

- **수정:** `D:/work/claude-test/weekly-reporting/index.html`
  - CSS: 각 태스크별 새 클래스 추가
  - JS 신규 함수: `undoRemoveTask`, `copyTask`, `pasteTask`, `carryOverTask`, `dragTaskStart`, `dragTaskOver`, `dragTaskDrop`, `toggleSidebar`, `searchTasks`, `showSearchResults`, `startOnboarding`, `nextOnboardingStep`, `endOnboarding`, `saveSnapshot`, `loadSnapshots`, `getPrevProgress`
  - JS 수정 함수: `removeTask`, `showToast`, `applyPeriod`, `renderTaskCard`, `renderTasksSection`

---

## Task 1: 진척율 입력 select로 교체 (3-5)

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - `renderTaskCard()` — `<input type="number">` → `<select>` (0/25/50/75/100)

- [ ] **Step 1: `renderTaskCard()`에서 진척율 input 찾기**

  Grep으로 `type="number" class="field-input"` 검색 후 Read로 해당 부분 확인.
  현재 코드:
  ```html
  <input type="number" class="field-input" min="0" max="100"
         value="${pct}"
         onchange="updateTaskField('${memberId}','${task.id}','overallProgress',+this.value)">
  ```

- [ ] **Step 2: select로 교체**

  위 `<input type="number">` 블록을 아래로 교체:
  ```javascript
  <select class="field-input progress-select"
          onchange="updateTaskField('${memberId}','${task.id}','overallProgress',+this.value)">
    ${[0,25,50,75,100].map(v =>
      `<option value="${v}"${pct===v?' selected':''}>${v}%</option>`
    ).join('')}
  </select>
  ```

  `pct === v` 비교: `pct`는 `task.overallProgress || 0` (숫자). 0/25/50/75/100 이외 값이 저장된 경우 어떤 옵션도 selected 안 됨 — 허용.

- [ ] **Step 3: CSS 추가**

  팀 관련 CSS 근처에 추가:
  ```css
  .progress-select { min-width: 80px; }
  ```

- [ ] **Step 4: 브라우저 확인**

  과제 카드에서 진척율이 숫자 입력 대신 select(0%/25%/50%/75%/100%)로 표시되는지 확인.

- [ ] **Step 5: 커밋**
  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 과제 카드 진척율 입력을 select(0/25/50/75/100%)로 교체"
  ```

---

## Task 2: 주간 항목 이월 confirm (3-3)

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - `applyPeriod()` — confirm 다이얼로그 + 이월 로직 추가

- [ ] **Step 1: `applyPeriod()` 읽기**

  Grep으로 `function applyPeriod` 찾아 Read로 전체 함수 확인.

- [ ] **Step 2: confirm + 이월 로직 추가**

  현재 `applyPeriod()` 내에서 유효성 검증 블록 끝, `saveState()` 호출 바로 전에 삽입:

  ```javascript
  // 유효성 검증 블록 현재:
  if (year >= 2020 && year <= 2035 && month >= 1 && month <= 12 && week >= 1 && week <= 5) {
    state.meta.year  = year;
    state.meta.month = month;
    state.meta.week  = week;
    saveState();
  }
  ```

  변경 후:
  ```javascript
  if (year >= 2020 && year <= 2035 && month >= 1 && month <= 12 && week >= 1 && week <= 5) {
    state.meta.year  = year;
    state.meta.month = month;
    state.meta.week  = week;
    if (confirm('차주 계획을 금주 활동으로 이월하시겠습니까?')) {
      for (const m of state.meta.members) {
        const data = getMemberData(m.id);
        for (const t of (data.tasks || [])) {
          t.currentWeekItems = (t.nextWeekItems || []).map(i => ({
            ...i, id: generateId(),
            details: (i.details || []).map(d => ({ ...d, id: generateId() }))
          }));
          t.nextWeekItems = [];
        }
      }
    }
    saveState();
  }
  ```

  **참고:** 이월 시 새 ID 생성(`generateId()`)으로 ID 충돌 방지. `details`도 새 ID 발급.

- [ ] **Step 3: 브라우저 확인**

  보고 기간 변경 후 '적용' 클릭 → confirm 다이얼로그 표시 → 확인 시 차주계획이 금주활동으로 이동되고 차주계획 비워짐.

- [ ] **Step 4: 커밋**
  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 보고기간 변경 시 차주계획 이월 confirm 다이얼로그 추가"
  ```

---

## Task 3: 주차별 비교 스냅샷 (3-9)

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - 신규 함수: `saveSnapshot()`, `loadSnapshots()`, `getPrevProgress(memberId, taskId)`
  - 수정: `applyPeriod()` — 주차 변경 전 스냅샷 저장
  - 수정: `renderTeamTableView()` — 진척율 옆 ▲▼ 변화량 표시

- [ ] **Step 1: localStorage key 상수 추가**

  Grep으로 `STORAGE_KEY` 가 있는 줄(~line 1997) 찾아 바로 아래에 추가:
  ```javascript
  const SNAPSHOT_KEY = 'weekly_report_snapshots';
  ```

- [ ] **Step 2: 스냅샷 함수 추가**

  `saveState()` 함수 바로 아래에 추가:
  ```javascript
  function saveSnapshot() {
    const { year, month, week } = state.meta;
    const weekKey = `${year}-${month}-${week}`;
    let snaps = loadSnapshots();
    // 같은 weekKey가 있으면 덮어쓰기
    snaps = snaps.filter(s => s.weekKey !== weekKey);
    snaps.push({ weekKey, timestamp: Date.now(), members: JSON.parse(JSON.stringify(state.members)) });
    // 최신 8주만 보관
    snaps.sort((a, b) => b.timestamp - a.timestamp);
    snaps = snaps.slice(0, 8);
    try { localStorage.setItem(SNAPSHOT_KEY, JSON.stringify(snaps)); } catch(e) {}
  }

  function loadSnapshots() {
    try {
      const raw = localStorage.getItem(SNAPSHOT_KEY);
      return raw ? JSON.parse(raw) : [];
    } catch(e) { return []; }
  }

  function getPrevProgress(memberId, taskId) {
    const snaps = loadSnapshots();
    if (!snaps.length) return null;
    // 현재 주차 제외, 가장 최근 스냅샷
    const { year, month, week } = state.meta;
    const curKey = `${year}-${month}-${week}`;
    const prev = snaps.filter(s => s.weekKey !== curKey)[0];
    if (!prev) return null;
    const tasks = (prev.members[memberId] || {}).tasks || [];
    const t = tasks.find(t => t.id === taskId);
    return t ? (t.overallProgress || 0) : null;
  }
  ```

- [ ] **Step 3: `applyPeriod()`에 스냅샷 저장 추가**

  현재 `applyPeriod()`의 유효성 검증 블록 (Task 2에서 추가된 confirm 포함):

  ```javascript
  if (year >= 2020 && year <= 2035 && month >= 1 && month <= 12 && week >= 1 && week <= 5) {
    state.meta.year  = year;
    ...
  ```

  `state.meta.year = year;` 라인 **바로 전에** (state를 변경하기 전 현재 상태 저장):
  ```javascript
  saveSnapshot(); // 현재 주차 상태를 스냅샷으로 저장
  state.meta.year  = year;
  state.meta.month = month;
  state.meta.week  = week;
  ```

- [ ] **Step 4: `renderTeamTableView()`에 delta 표시**

  Grep으로 `renderTeamTableView` 찾아 Read. 팀원별 tasks를 순회하며 `overallProgress`를 렌더하는 부분 찾기. 현재 진척율 셀은 Phase 1에서 읽기 전용으로 변경됨 (`${t.overallProgress || 0}%` 형태).

  delta 표시 추가:
  ```javascript
  // 기존 (Phase 1 변경 후 읽기 전용):
  <td class="team-table-pct">${t.overallProgress || 0}%</td>

  // 변경 후:
  // tasks forEach 안에서, 진척율 td를 렌더하기 직전에 delta 계산:
  const prevPct = getPrevProgress(m.id, t.id);
  const curPct = t.overallProgress || 0;
  const deltaHtml = prevPct !== null
    ? (() => {
        const d = curPct - prevPct;
        if (d === 0) return '';
        return d > 0
          ? `<span class="pct-delta up">▲${d}%</span>`
          : `<span class="pct-delta down">▼${Math.abs(d)}%</span>`;
      })()
    : '';
  ```

  td 내용:
  ```html
  <td class="team-table-pct">${curPct}%${deltaHtml}</td>
  ```

  **주의:** `getPrevProgress` 호출은 tasks.forEach 내부, 각 task당 한 번만.

- [ ] **Step 5: CSS 추가**
  ```css
  .pct-delta { font-size: 10px; margin-left: 4px; font-weight: 600; }
  .pct-delta.up { color: #16A34A; }
  .pct-delta.down { color: #DC2626; }
  ```

- [ ] **Step 6: 브라우저 확인**

  보고기간 변경 후 → 이전 주차 데이터와 비교해 팀현황 테이블의 진척율 옆에 ▲/▼ 표시 확인.

- [ ] **Step 7: 커밋**
  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 주차별 진척율 비교 스냅샷 + 팀현황 ▲▼ 변화량 표시"
  ```

---

## Task 4: 삭제 Undo (3-1)

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - 신규 모듈 변수: `_undoStack`
  - 신규 함수: `undoRemoveTask()`
  - 수정: `removeTask()` — confirm 제거, undo 방식으로 전환
  - 수정: `showToast()` — action 버튼 지원 추가

- [ ] **Step 1: `_undoStack` 모듈 변수 추가**

  `_toastTimer` 선언 (line ~4543) 바로 아래에:
  ```javascript
  let _undoStack = null; // { memberId, task, insertIndex, timerId }
  ```

- [ ] **Step 2: `showToast()` action 버튼 지원**

  현재 `showToast(msg)`:
  ```javascript
  function showToast(msg) {
    const existing = document.querySelector('.toast');
    if (existing) existing.remove();
    clearTimeout(_toastTimer);
    const el = document.createElement('div');
    el.className = 'toast';
    el.textContent = msg;
    document.body.appendChild(el);
    _toastTimer = setTimeout(() => el.remove(), 2800);
  }
  ```

  변경 후 (`action` 파라미터 추가, `duration` 옵션 지원):
  ```javascript
  function showToast(msg, opts) {
    const existing = document.querySelector('.toast');
    if (existing) existing.remove();
    clearTimeout(_toastTimer);
    const el = document.createElement('div');
    el.className = 'toast';
    if (opts && opts.actionLabel) {
      el.style.display = 'flex';
      el.style.alignItems = 'center';
      el.style.gap = '10px';
      const span = document.createElement('span');
      span.textContent = msg;
      const btn = document.createElement('button');
      btn.className = 'toast-action-btn';
      btn.textContent = opts.actionLabel;
      btn.onclick = () => { clearTimeout(_toastTimer); el.remove(); opts.action(); };
      el.appendChild(span);
      el.appendChild(btn);
    } else {
      el.textContent = msg;
    }
    document.body.appendChild(el);
    _toastTimer = setTimeout(() => el.remove(), (opts && opts.duration) || 2800);
  }
  ```

- [ ] **Step 3: CSS 추가**
  ```css
  .toast-action-btn {
    background: rgba(255,255,255,0.2); color: white; border: 1px solid rgba(255,255,255,0.4);
    border-radius: 4px; padding: 2px 8px; font-size: 12px; cursor: pointer; white-space: nowrap;
  }
  .toast-action-btn:hover { background: rgba(255,255,255,0.35); }
  ```

- [ ] **Step 4: `undoRemoveTask()` 함수 추가**

  `removeTask()` (line ~2754) 바로 아래에:
  ```javascript
  function undoRemoveTask() {
    if (!_undoStack) return;
    clearTimeout(_undoStack.timerId);
    const { memberId, task, insertIndex } = _undoStack;
    _undoStack = null;
    const data = getMemberData(memberId);
    data.tasks.splice(insertIndex, 0, task);
    saveState();
    renderMain();
    showToast('↩ 삭제가 취소되었습니다');
  }
  ```

- [ ] **Step 5: `removeTask()` 수정**

  현재:
  ```javascript
  function removeTask(memberId, taskId) {
    if (!confirm('이 과제를 삭제하시겠습니까?')) return;
    const data = getMemberData(memberId);
    const taskName = data.tasks.find(t => t.id === taskId)?.kpiProduct || '과제';
    data.tasks = data.tasks.filter(t => t.id !== taskId);
    saveState();
    pushNotification('task_delete', memberId, getMember(memberId)?.name || memberId, taskName);
    renderMain();
  }
  ```

  변경 후:
  ```javascript
  function removeTask(memberId, taskId) {
    const data = getMemberData(memberId);
    const insertIndex = data.tasks.findIndex(t => t.id === taskId);
    if (insertIndex === -1) return;
    const task = data.tasks[insertIndex];
    const taskName = task.kpiProduct || '과제';

    // 기존 undo 스택이 있으면 즉시 확정 (saveState)
    if (_undoStack) {
      clearTimeout(_undoStack.timerId);
      saveState();
      _undoStack = null;
    }

    data.tasks.splice(insertIndex, 1);
    renderMain();
    // pushNotification은 실제 삭제가 확정될 때(5초 후)에만 발생 — 되돌리기 시 알림 방지

    const timerId = setTimeout(() => {
      _undoStack = null;
      saveState();
      pushNotification('task_delete', memberId, getMember(memberId)?.name || memberId, taskName);
    }, 5000);

    _undoStack = { memberId, task, insertIndex, timerId };

    showToast(`"${taskName}" 삭제됨`, {
      actionLabel: '되돌리기',
      action: undoRemoveTask,
      duration: 5000
    });
  }
  ```

- [ ] **Step 6: 브라우저 확인**

  과제 삭제 버튼 클릭 → confirm 없이 즉시 삭제되고 "삭제됨 — 되돌리기" 토스트 5초 표시. '되돌리기' 클릭 시 복원. 5초 후 완전 삭제(localStorage에 반영).

- [ ] **Step 7: 커밋**
  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 과제 삭제 Undo — 5초 되돌리기 토스트"
  ```

---

## Task 5: 과제 복사/붙여넣기 (3-2)

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - 신규 모듈 변수: `_clipboard`
  - 신규 함수: `copyTask(memberId, taskId)`, `pasteTask(memberId)`, `carryOverTask(memberId, taskId)`
  - 수정: `renderTaskCard()` — 복사 버튼 + 이월 버튼 추가
  - 수정: `renderTasksSection()` — 붙여넣기 버튼 추가

- [ ] **Step 1: `_clipboard` 모듈 변수 추가**

  `_undoStack` 선언 바로 아래에:
  ```javascript
  let _clipboard = null; // deep-cloned task object
  ```

- [ ] **Step 2: `copyTask()` 함수 추가**

  `undoRemoveTask()` 아래에:
  ```javascript
  function copyTask(memberId, taskId) {
    const task = getMemberData(memberId).tasks.find(t => t.id === taskId);
    if (!task) return;
    _clipboard = JSON.parse(JSON.stringify(task));
    showToast(`"${task.kpiProduct || '과제'}" 복사됨`);
    renderMain(); // 붙여넣기 버튼 표시를 위해
  }

  function pasteTask(memberId) {
    if (!_clipboard) return;
    const newTask = JSON.parse(JSON.stringify(_clipboard));
    // 새 ID 발급
    newTask.id = generateId();
    newTask.currentWeekItems = (newTask.currentWeekItems || []).map(i => ({
      ...i, id: generateId(),
      details: (i.details || []).map(d => ({ ...d, id: generateId() }))
    }));
    newTask.nextWeekItems = (newTask.nextWeekItems || []).map(i => ({
      ...i, id: generateId(),
      details: (i.details || []).map(d => ({ ...d, id: generateId() }))
    }));
    newTask.issues = (newTask.issues || []).map(i => ({ ...i, id: generateId() }));
    getMemberData(memberId).tasks.push(newTask);
    saveState();
    renderMain();
    showToast('과제 붙여넣기 완료');
  }

  function carryOverTask(memberId, taskId) {
    const task = getMemberData(memberId).tasks.find(t => t.id === taskId);
    if (!task) return;
    task.currentWeekItems = (task.nextWeekItems || []).map(i => ({
      ...i, id: generateId(),
      details: (i.details || []).map(d => ({ ...d, id: generateId() }))
    }));
    task.nextWeekItems = [];
    saveState();
    renderMain();
    showToast('차주계획을 금주활동으로 이월했습니다');
  }
  ```

- [ ] **Step 3: `renderTaskCard()`에 복사 버튼 + 이월 버튼 추가**

  Read로 `renderTaskCard()` 전체 확인. `task-meta-header` div 안에 복사 버튼 추가:

  현재:
  ```javascript
  <div class="task-meta-header">
    <div contenteditable="true" class="task-kpi" ...>${esc(task.kpiProduct || '')}</div>
    <button class="task-remove-btn" onclick="removeTask('${memberId}','${task.id}')">🗑 과제 삭제</button>
  </div>
  ```

  변경 후:
  ```javascript
  <div class="task-meta-header">
    <div contenteditable="true" class="task-kpi" ...>${esc(task.kpiProduct || '')}</div>
    <div style="display:flex;gap:6px;align-items:center;">
      <button class="task-copy-btn" onclick="copyTask('${memberId}','${task.id}')" title="과제 복사">⧉</button>
      <button class="task-remove-btn" onclick="removeTask('${memberId}','${task.id}')">🗑 과제 삭제</button>
    </div>
  </div>
  ```

  그리고 차주계획 컬럼 헤더 아래 이월 버튼 추가:
  현재:
  ```javascript
  <div class="activity-col">
    <h4 class="next">차주계획</h4>
    ${nextItemsHtml}
  ```

  변경 후:
  ```javascript
  <div class="activity-col">
    <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:4px;">
      <h4 class="next" style="margin:0;">차주계획</h4>
      <button class="carryover-btn" onclick="carryOverTask('${memberId}','${task.id}')" title="차주계획을 금주활동으로 이월">↰ 이월</button>
    </div>
    ${nextItemsHtml}
  ```

- [ ] **Step 4: `renderTasksSection()`에 붙여넣기 버튼 추가**

  현재:
  ```javascript
  return `
    <div class="section">
      <div class="section-header">
        <span class="section-title">과제</span>
        <button class="add-btn" onclick="addTask('${memberId}')">+ 과제 추가</button>
      </div>
      ${inner}
    </div>`;
  ```

  변경 후:
  ```javascript
  const pasteBtn = _clipboard
    ? `<button class="paste-task-btn" onclick="pasteTask('${memberId}')">📋 붙여넣기 (${esc(_clipboard.kpiProduct || '과제')})</button>`
    : '';

  return `
    <div class="section">
      <div class="section-header">
        <span class="section-title">과제</span>
        <div style="display:flex;gap:6px;">
          ${pasteBtn}
          <button class="add-btn" onclick="addTask('${memberId}')">+ 과제 추가</button>
        </div>
      </div>
      ${inner}
    </div>`;
  ```

- [ ] **Step 5: CSS 추가**
  ```css
  .task-copy-btn {
    background: none; border: 1px solid var(--border-light); border-radius: 4px;
    padding: 2px 6px; font-size: 13px; cursor: pointer; color: var(--text-muted);
  }
  .task-copy-btn:hover { background: var(--bg); color: var(--text); }
  .carryover-btn {
    background: none; border: none; font-size: 11px; color: var(--primary);
    cursor: pointer; padding: 1px 4px;
  }
  .carryover-btn:hover { text-decoration: underline; }
  .paste-task-btn {
    background: var(--primary-light, #EEF2FF); color: var(--primary);
    border: 1px solid var(--primary); border-radius: 6px;
    padding: 4px 10px; font-size: 12px; cursor: pointer;
  }
  .paste-task-btn:hover { opacity: 0.85; }
  ```

- [ ] **Step 6: 브라우저 확인**

  - 과제 카드 우상단에 ⧉ 버튼 → 클릭 시 "복사됨" 토스트
  - 다른 팀원 탭 전환 시 section-header에 "📋 붙여넣기 (과제명)" 버튼 표시
  - 붙여넣기 클릭 시 새 ID로 과제 복제됨
  - 차주계획 헤더 옆 "↰ 이월" 버튼 → 클릭 시 nextWeekItems → currentWeekItems 이동

- [ ] **Step 7: 커밋**
  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 과제 복사/붙여넣기 및 이번 주로 이월 기능 추가"
  ```

---

## Task 6: 드래그앤드롭 순서 변경 (3-4)

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - 신규 모듈 변수: `_dragSrc`
  - 신규 함수: `dragTaskStart`, `dragTaskOver`, `dragTaskDrop`, `dragTaskEnd`
  - 수정: `renderTaskCard()` — draggable 속성 + 핸들 추가

- [ ] **Step 1: `_dragSrc` 모듈 변수 추가**

  `_clipboard` 선언 바로 아래에:
  ```javascript
  let _dragSrc = null; // { memberId, taskId }
  ```

- [ ] **Step 2: DnD 이벤트 함수 추가**

  `pasteTask()` 함수 아래에:
  ```javascript
  function dragTaskStart(memberId, taskId, event) {
    _dragSrc = { memberId, taskId };
    event.dataTransfer.effectAllowed = 'move';
  }

  function dragTaskOver(event) {
    event.preventDefault();
    event.dataTransfer.dropEffect = 'move';
  }

  function dragTaskDrop(memberId, taskId, event) {
    event.preventDefault();
    if (!_dragSrc || _dragSrc.taskId === taskId) return;
    if (_dragSrc.memberId !== memberId) return; // 다른 팀원 간 이동 불가
    const data = getMemberData(memberId);
    const tasks = data.tasks;
    const fromIdx = tasks.findIndex(t => t.id === _dragSrc.taskId);
    const toIdx = tasks.findIndex(t => t.id === taskId);
    if (fromIdx === -1 || toIdx === -1) return;
    const [moved] = tasks.splice(fromIdx, 1);
    tasks.splice(toIdx, 0, moved);
    _dragSrc = null;
    saveState();
    renderMain();
  }

  function dragTaskEnd() {
    _dragSrc = null;
  }
  ```

- [ ] **Step 3: `renderTaskCard()`에 draggable + 핸들 추가**

  현재:
  ```javascript
  return `
    <div class="task-card" data-task-id="${task.id}">
      <div class="task-meta">
  ```

  변경 후 (`draggable` 속성 + 이벤트 핸들러 + 핸들 아이콘):
  ```javascript
  return `
    <div class="task-card" data-task-id="${task.id}"
         draggable="true"
         ondragstart="dragTaskStart('${memberId}','${task.id}',event)"
         ondragover="dragTaskOver(event)"
         ondrop="dragTaskDrop('${memberId}','${task.id}',event)"
         ondragend="dragTaskEnd()">
      <div class="drag-handle" title="드래그하여 순서 변경">⠿</div>
      <div class="task-meta">
  ```

- [ ] **Step 4: CSS 추가**
  ```css
  .task-card[draggable] { cursor: default; }
  .drag-handle {
    position: absolute; left: 6px; top: 50%; transform: translateY(-50%);
    font-size: 16px; color: var(--text-muted); opacity: 0.4; cursor: grab;
    user-select: none; line-height: 1;
  }
  .task-card:hover .drag-handle { opacity: 0.8; }
  .task-card { position: relative; padding-left: 20px; }
  ```

  **주의:** `.task-card`에 이미 `position` 이 있을 수 있으므로 Read로 기존 `.task-card` CSS 확인 후 필요 시 `position: relative`만 추가.

- [ ] **Step 5: 브라우저 확인**

  과제 카드 좌측 ⠿ 핸들을 드래그하여 같은 팀원의 다른 과제 위에 드롭 → 순서 변경.
  다른 팀원 탭의 카드로는 드롭 불가 (같은 멤버 내에서만 동작).

- [ ] **Step 6: 커밋**
  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 과제 카드 드래그앤드롭 순서 변경 (HTML5 DnD)"
  ```

---

## Task 7: 반응형 사이드바 (3-7)

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - HTML: sidebar-logo 영역에 `☰` 버튼 추가
  - JS 신규: `toggleSidebar()`
  - CSS: 사이드바 collapsed 상태, media query, 테이블 overflow

- [ ] **Step 1: `toggleSidebar()` 함수 추가**

  `switchTab()` 함수 (line ~2416) 근처에 추가:
  ```javascript
  function toggleSidebar() {
    document.getElementById('sidebar').classList.toggle('collapsed');
  }
  ```

- [ ] **Step 2: 사이드바 HTML에 토글 버튼 추가**

  Read로 line 1914 근처 확인. `<aside id="sidebar">` 내 `<div class="sidebar-logo"...>` 안, 오른쪽 끝 `<div id="notifBellMount"></div>` 근처에 추가.

  현재:
  ```html
  <div id="notifBellMount"></div>
  ```

  변경 후:
  ```html
  <button id="sidebarCloseBtn" class="sidebar-close-btn" onclick="toggleSidebar()" title="사이드바 접기">✕</button>
  <div id="notifBellMount"></div>
  ```

  그리고 `<main id="main">` 시작 바로 뒤에 열기 버튼 추가:
  현재:
  ```html
  <main id="main">
    <div id="mainContent"></div>
  ```

  변경 후:
  ```html
  <main id="main">
    <button id="sidebarOpenBtn" class="sidebar-open-btn" onclick="toggleSidebar()" title="사이드바 열기">☰</button>
    <div id="mainContent"></div>
  ```

- [ ] **Step 3: CSS 추가**

  ```css
  /* 사이드바 접기/펼치기 */
  #sidebar.collapsed {
    width: 0;
    min-width: 0;
    overflow: hidden;
    border-right: none;
  }
  .sidebar-close-btn {
    background: none; border: none; font-size: 14px; cursor: pointer;
    color: rgba(255,255,255,0.7); padding: 2px 4px;
  }
  .sidebar-close-btn:hover { color: white; }
  #sidebarOpenBtn {
    display: none; /* 기본: 숨김 */
    position: fixed; top: 12px; left: 12px; z-index: 100;
    background: var(--primary); color: white; border: none;
    border-radius: 6px; padding: 6px 10px; font-size: 16px;
    cursor: pointer; box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  }
  #sidebar.collapsed ~ main #sidebarOpenBtn,
  #sidebar.collapsed + main #sidebarOpenBtn { display: block; }

  /* 통합 테이블 반응형 */
  .team-table-wrap { overflow-x: auto; }
  .team-table { min-width: 600px; }

  /* 768px 미만 자동 숨김 */
  @media (max-width: 768px) {
    #sidebar { width: 0; min-width: 0; overflow: hidden; border-right: none; }
    #sidebarOpenBtn { display: block; }
  }
  ```

  **주의:** `#sidebar.collapsed ~ main #sidebarOpenBtn`은 인접 형제 선택자. 실제 DOM 구조에서 `#main` 바로 안의 `#sidebarOpenBtn`을 대상으로 하므로 정확한 선택자 확인 필요. 동작하지 않으면 JS로 직접 제어:
  ```javascript
  function toggleSidebar() {
    const sidebar = document.getElementById('sidebar');
    sidebar.classList.toggle('collapsed');
    document.getElementById('sidebarOpenBtn').style.display =
      sidebar.classList.contains('collapsed') ? 'block' : 'none';
  }
  ```

- [ ] **Step 4: 브라우저 확인**

  - 사이드바 ✕ 버튼 클릭 → 사이드바 숨김, ☰ 버튼 표시
  - ☰ 클릭 → 사이드바 다시 표시
  - 브라우저 너비 768px 이하로 줄이면 자동 숨김

- [ ] **Step 5: 커밋**
  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 사이드바 접기/펼치기 토글 + 모바일 반응형"
  ```

---

## Task 8: 검색 (3-6)

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - 신규 모듈 변수: `_searchQuery`, `_searchOpen`
  - 신규 함수: `toggleSearch()`, `doSearch()`, `clearSearch()`
  - 수정: `renderMain()` — 검색 바 + 결과 하이라이트 추가
  - 수정: `renderTaskCard()` — 검색 매칭 시 하이라이트 클래스 추가

- [ ] **Step 1: 모듈 변수 추가**

  `_dragSrc` 선언 아래에:
  ```javascript
  let _searchQuery = '';
  let _searchOpen = false;
  ```

- [ ] **Step 2: 검색 함수 추가**

  `dragTaskEnd()` 아래에:
  ```javascript
  function toggleSearch() {
    _searchOpen = !_searchOpen;
    if (!_searchOpen) { _searchQuery = ''; }
    renderMain();
    if (_searchOpen) {
      requestAnimationFrame(() => document.getElementById('searchInput')?.focus());
    }
  }

  function doSearch(query) {
    _searchQuery = query.trim();
    renderMain();
  }

  function clearSearch() {
    _searchQuery = '';
    _searchOpen = false;
    renderMain();
  }

  function getSearchMatches() {
    if (!_searchQuery) return [];
    const q = _searchQuery.toLowerCase();
    const matches = [];
    for (const m of state.meta.members) {
      for (const t of (getMemberData(m.id).tasks || [])) {
        const inKpi = (t.kpiProduct || '').toLowerCase().includes(q);
        const inCurrent = (t.currentWeekItems || []).some(i => i.text.toLowerCase().includes(q));
        const inNext = (t.nextWeekItems || []).some(i => i.text.toLowerCase().includes(q));
        if (inKpi || inCurrent || inNext) {
          matches.push({ memberId: m.id, memberName: m.name, taskId: t.id, kpiProduct: t.kpiProduct || '' });
        }
      }
    }
    return matches;
  }
  ```

- [ ] **Step 3: `renderMain()`에 검색 바 + 결과 드롭다운 추가**

  Grep으로 `function renderMain` 찾아 Read로 전체 함수 확인. `renderMain()`은 두 가지 분기를 갖는다:
  - **팀현황 탭 분기**: `if (state.activeTab === '__team__')` 블록 — `document.getElementById('mainContent').innerHTML = renderTeamView();`
  - **팀원 탭 분기**: 그 외 — `document.getElementById('mainContent').innerHTML = \`<div class="member-header">...\``

  함수 맨 앞(두 분기 모두 시작 전)에 `searchBarHtml` 변수 선언 추가:
  ```javascript
  function renderMain() {
    const searchBarHtml = _searchOpen ? `
      <div class="search-bar">
        <input id="searchInput" type="text" class="search-input" placeholder="과제명, 금주활동, 차주계획 검색..."
               value="${esc(_searchQuery)}"
               oninput="doSearch(this.value)"
               onkeydown="if(event.key==='Escape')clearSearch()">
        <button class="search-clear-btn" onclick="clearSearch()">✕</button>
      </div>
      ${_searchQuery ? renderSearchResults(getSearchMatches()) : ''}
    ` : '';

    if (state.activeTab === '__team__') {
      document.getElementById('mainContent').innerHTML = searchBarHtml + renderTeamView();
      return;
    }
    // ... 팀원 탭 분기
    document.getElementById('mainContent').innerHTML = `
      ${searchBarHtml}
      <div class="member-header">
        ...
  ```

  **두 분기(팀현황 + 팀원) 모두 반드시 `searchBarHtml`을 포함해야 함.** 하나라도 빠지면 해당 탭에서 검색 바가 표시되지 않음.

- [ ] **Step 4: `renderSearchResults()` 함수 추가**

  `getSearchMatches()` 아래에:
  ```javascript
  function renderSearchResults(matches) {
    if (!matches.length) return '<div class="search-no-results">검색 결과 없음</div>';
    return `<div class="search-results">
      ${matches.map(m =>
        `<div class="search-result-item" onclick="switchTab('${m.memberId}');clearSearch();requestAnimationFrame(()=>{document.querySelector('.task-card[data-task-id=\\'${m.taskId}\\']')?.classList.add('search-highlight');})">
          <span class="search-result-member">${esc(m.memberName)}</span>
          <span class="search-result-task">${esc(m.kpiProduct)}</span>
        </div>`
      ).join('')}
    </div>`;
  }
  ```

  **XSS 주의:** `m.memberId`, `m.taskId`는 `generateId()` 결과라 안전하지만, `m.memberName`, `m.kpiProduct`는 반드시 `esc()` 통과.

- [ ] **Step 5: 검색 아이콘 버튼 추가 (두 곳)**

  검색 버튼은 두 곳에 추가해야 한다:

  **① 팀원 탭 — `renderMain()` 내 `member-status-bar` div:**

  현재 `<div class="member-status-bar">` 안 맨 오른쪽에 추가:
  ```javascript
  <button class="search-icon-btn" onclick="toggleSearch()" title="검색">🔍</button>
  ```

  **② 팀현황 탭 — `renderTeamView()` 함수 내 헤더 영역:**

  Grep으로 `function renderTeamView` 찾아 Read. 팀현황 탭에는 `member-header`가 없으므로, `renderTeamView()`가 반환하는 HTML의 최상단 `.team-header` 또는 stat-bar 위에:
  ```javascript
  <div style="display:flex;justify-content:flex-end;padding:8px 16px 0;">
    <button class="search-icon-btn" onclick="toggleSearch()" title="검색">🔍</button>
  </div>
  ${renderTeamStatBar()}
  ...
  ```

  혹은 기존 팀현황 `.team-header` 내 적절한 위치에 삽입. 정확한 위치는 Read로 확인.

- [ ] **Step 6: CSS 추가**
  ```css
  .search-bar {
    display: flex; align-items: center; gap: 8px;
    padding: 10px 16px; background: var(--card-bg);
    border-bottom: 1px solid var(--border-light);
  }
  .search-input {
    flex: 1; border: 1px solid var(--border-light); border-radius: 6px;
    padding: 6px 10px; font-size: 13px; background: var(--bg);
    color: var(--text);
  }
  .search-clear-btn {
    background: none; border: none; font-size: 14px; cursor: pointer; color: var(--text-muted);
  }
  .search-results {
    background: var(--card-bg); border: 1px solid var(--border-light);
    border-radius: 8px; margin: 8px 16px; overflow: hidden;
  }
  .search-result-item {
    display: flex; gap: 10px; padding: 8px 12px; cursor: pointer;
    border-bottom: 1px solid var(--border-light);
  }
  .search-result-item:last-child { border-bottom: none; }
  .search-result-item:hover { background: var(--bg); }
  .search-result-member { font-size: 12px; color: var(--text-muted); white-space: nowrap; }
  .search-result-task { font-size: 13px; color: var(--text); }
  .search-no-results { padding: 12px 16px; font-size: 13px; color: var(--text-muted); }
  .search-icon-btn { background: none; border: none; font-size: 16px; cursor: pointer; }
  .task-card.search-highlight { outline: 2px solid #FCD34D; outline-offset: 2px; }
  ```

- [ ] **Step 7: 브라우저 확인**

  🔍 클릭 → 검색 바 펼쳐짐. 텍스트 입력 시 매칭 결과 드롭다운. 결과 클릭 시 해당 팀원 탭으로 전환 + 카드 노란 하이라이트. ESC로 닫힘.

- [ ] **Step 8: 커밋**
  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 헤더 검색 기능 — 과제명/활동 검색 + 결과 드롭다운"
  ```

---

## Task 9: 온보딩 투어 (3-8)

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - 신규 모듈 변수: `_onboardingStep`
  - 신규 함수: `startOnboarding()`, `nextOnboarding()`, `endOnboarding()`
  - 수정: 팀 이름 입력 완료 후 온보딩 실행 (기존 `editTeamName()` → `confirmTeamName()` 흐름)

- [ ] **Step 1: 모듈 변수 추가**

  `_searchOpen` 선언 아래에:
  ```javascript
  let _onboardingStep = 0;
  ```

- [ ] **Step 2: 온보딩 함수 추가**

  `renderSearchResults()` 아래에:
  ```javascript
  const ONBOARDING_STEPS = [
    { targetSelector: '.sidebar-item:not(#teamTab)',
      title: '팀원 탭 선택',
      text: '팀원 이름을 클릭해 해당 팀원의 주간보고를 작성합니다.' },
    { targetSelector: '.add-btn',
      title: '과제 추가',
      text: '+ 과제 추가 버튼으로 KPI 과제를 등록하고 금주활동, 차주계획, 이슈를 입력합니다.' },
    { targetSelector: '#teamTab',
      title: '팀현황 확인',
      text: '팀현황 탭에서 전체 팀원의 진척현황을 한눈에 확인합니다.' },
    { targetSelector: '.export-btn',
      title: 'Excel 내보내기',
      text: 'Excel 내보내기 버튼으로 주간보고 양식을 자동 생성합니다.' }
  ];

  function startOnboarding() {
    _onboardingStep = 0;
    showOnboardingStep();
  }

  function showOnboardingStep() {
    document.querySelector('.onboarding-overlay')?.remove();
    if (_onboardingStep >= ONBOARDING_STEPS.length) { endOnboarding(); return; }
    const step = ONBOARDING_STEPS[_onboardingStep];
    const target = document.querySelector(step.targetSelector);
    if (!target) { nextOnboarding(); return; }

    const rect = target.getBoundingClientRect();
    const overlay = document.createElement('div');
    overlay.className = 'onboarding-overlay';
    overlay.innerHTML = `
      <div class="onboarding-backdrop"></div>
      <div class="onboarding-tooltip" style="top:${rect.bottom + 10}px;left:${Math.max(10, rect.left)}px;">
        <div class="onboarding-step-num">${_onboardingStep + 1} / ${ONBOARDING_STEPS.length}</div>
        <div class="onboarding-title">${esc(step.title)}</div>
        <div class="onboarding-text">${esc(step.text)}</div>
        <div class="onboarding-actions">
          <button class="onboarding-skip-btn" onclick="endOnboarding()">건너뛰기</button>
          <button class="onboarding-next-btn" onclick="nextOnboarding()">
            ${_onboardingStep < ONBOARDING_STEPS.length - 1 ? '다음 →' : '완료'}
          </button>
        </div>
      </div>
    `;
    document.body.appendChild(overlay);
  }

  function nextOnboarding() {
    _onboardingStep++;
    showOnboardingStep();
  }

  function endOnboarding() {
    document.querySelector('.onboarding-overlay')?.remove();
    _onboardingStep = 0;
    try { localStorage.setItem('weekly_report_onboarded', '1'); } catch(e) {}
  }
  ```

- [ ] **Step 3: 팀 이름 설정 완료 시 온보딩 자동 시작**

  Grep으로 `weekly_report_onboarded` 또는 `editTeamName`, `confirmTeamName`, `sidebarTeamName` 등을 검색해 팀 이름 확정 함수를 찾는다.

  팀 이름이 최초 설정될 때 (기존 코드에 팀 이름 입력 완료 후 저장하는 시점) 아래를 추가:
  ```javascript
  if (!localStorage.getItem('weekly_report_onboarded')) {
    setTimeout(startOnboarding, 300);
  }
  ```

  **정확한 위치:** Read로 팀 이름 저장 함수 확인 후 해당 위치에 삽입.

- [ ] **Step 4: 헤더에 "?" 버튼 추가**

  사이드바 footer 또는 sidebar-logo 근처에 "?" 버튼 추가 (static HTML):
  ```html
  <button onclick="startOnboarding()" title="사용 가이드" class="help-btn">?</button>
  ```

- [ ] **Step 5: CSS 추가**
  ```css
  .onboarding-backdrop {
    position: fixed; inset: 0; background: rgba(0,0,0,0.3); z-index: 999;
  }
  .onboarding-tooltip {
    position: fixed; z-index: 1000; background: white; border-radius: 10px;
    padding: 16px; max-width: 280px; box-shadow: 0 8px 32px rgba(0,0,0,0.18);
  }
  .onboarding-step-num { font-size: 11px; color: var(--text-muted); margin-bottom: 6px; }
  .onboarding-title { font-size: 14px; font-weight: 700; margin-bottom: 6px; }
  .onboarding-text { font-size: 13px; color: var(--text-secondary); line-height: 1.5; margin-bottom: 12px; }
  .onboarding-actions { display: flex; justify-content: space-between; }
  .onboarding-skip-btn {
    background: none; border: none; font-size: 12px; color: var(--text-muted); cursor: pointer;
  }
  .onboarding-next-btn {
    background: var(--primary); color: white; border: none; border-radius: 6px;
    padding: 5px 14px; font-size: 12px; cursor: pointer; font-weight: 600;
  }
  .help-btn {
    background: rgba(255,255,255,0.15); border: 1px solid rgba(255,255,255,0.3);
    color: white; border-radius: 50%; width: 20px; height: 20px;
    font-size: 11px; cursor: pointer; font-weight: 700;
    display: flex; align-items: center; justify-content: center;
  }
  ```

- [ ] **Step 6: 브라우저 확인**

  - localStorage에서 `weekly_report_onboarded` 키 제거 후 새로고침
  - 팀 이름 설정 완료 시 온보딩 투어 자동 시작
  - 4단계 툴팁 투어 진행
  - "건너뛰기" 또는 "완료" 클릭 시 종료, localStorage에 완료 표시
  - 재방문 시 투어 미표시
  - "?" 버튼 클릭 시 언제든 재실행 가능

- [ ] **Step 7: 커밋**
  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 최초 접속 온보딩 투어 (4단계) + '?' 버튼으로 재실행"
  ```

---

## Task 10: GitHub 배포

- [ ] **Step 1: 전체 변경 확인**
  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git log --oneline -10 && git status
  ```

- [ ] **Step 2: GitHub push**
  ```bash
  git push origin main
  ```

- [ ] **Step 3: Vercel 배포**
  ```bash
  NODE_TLS_REJECT_UNAUTHORIZED=0 vercel --prod
  ```

---

## 검증 체크리스트

- [ ] 과제 진척율이 select(0/25/50/75/100%)로 입력됨
- [ ] 보고기간 변경 시 이월 confirm 표시
- [ ] 과제 삭제 시 5초 Undo 토스트 (confirm 없음)
- [ ] ⧉ 복사 → 다른 팀원 탭에 붙여넣기 버튼 표시
- [ ] 차주계획 이월 버튼 동작
- [ ] 드래그핸들 ⠿ 드래그 → 순서 변경
- [ ] 사이드바 ✕ 버튼으로 접기/펼치기
- [ ] 🔍 검색 → 결과 드롭다운 → 카드 하이라이트
- [ ] 최초 접속 온보딩 투어 4단계
- [ ] 팀현황 테이블 진척율 옆 ▲▼ 표시 (스냅샷 있을 때)

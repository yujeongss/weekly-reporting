# 알림 벨 아이콘 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 사이드바 상단에 벨 아이콘을 추가하고 과제 추가/삭제, 작성완료/수정중 상태 변경 시 알림을 생성·표시한다.

**Architecture:** `index.html` 단일 파일에 CSS → 데이터 함수 → 렌더 함수 → HTML → 트리거 연결 순으로 추가한다. 알림은 `weekly_notifications` localStorage 키에 별도 저장한다. 드롭다운 패널은 `position: fixed`로 렌더링하여 `.sidebar-logo`의 `overflow: hidden` 제약을 우회한다.

**Tech Stack:** Vanilla JS, CSS, localStorage — 외부 라이브러리 없음.

---

## 수정 파일 맵

| 파일 | 위치 | 변경 내용 |
|------|------|----------|
| `index.html` | CSS `</style>` 직전 (~line 1418) | 알림 관련 CSS 클래스 추가 |
| `index.html` | `<aside id="sidebar">` 내 `.sidebar-logo` (~line 1424) | `.notif-wrap` + 벨 버튼 HTML 삽입 |
| `index.html` | TOAST 섹션 앞 (~line 3224) | 알림 데이터·렌더·토글 함수 추가 |
| `index.html` | `addTask()` (~line 2177) | `pushNotification` 호출 추가 |
| `index.html` | `removeTask()` (~line 2198) | task 이름 캡처 + `pushNotification` 추가 |
| `index.html` | `setMemberStatus()` (~line 1711) | `pushNotification` 호출 추가 |
| `index.html` | `renderSidebar()` (~line 1773) | 벨 뱃지 갱신 추가 |
| `index.html` | `init()` (~line 4151) | `loadNotifications()` + document click 리스너 추가 |

---

## Task 1: 알림 CSS 추가

**Files:**
- Modify: `index.html` — `</style>` 직전에 CSS 삽입

- [ ] **Step 1: `</style>` 직전 (line ~1418)에 아래 CSS 블록 추가**

```css
/* ===== NOTIFICATIONS ===== */
.notif-wrap {
  position: relative;
}
.notif-bell-btn {
  background: rgba(255,255,255,0.18);
  border: 1px solid rgba(255,255,255,0.28);
  border-radius: 8px;
  color: #fff;
  cursor: pointer;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 15px;
  position: relative;
  flex-shrink: 0;
  transition: background 0.15s;
}
.notif-bell-btn:hover { background: rgba(255,255,255,0.28); }
.notif-badge {
  position: absolute;
  top: -5px;
  right: -5px;
  background: #EF4444;
  color: #fff;
  font-size: 10px;
  font-weight: 700;
  min-width: 16px;
  height: 16px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0 3px;
  pointer-events: none;
}
.notif-panel {
  position: fixed;
  top: 58px;
  left: 0;
  width: 252px;
  background: var(--card-bg);
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-lg);
  z-index: 9999;
  overflow: hidden;
  display: none;
}
.notif-panel.open { display: block; }
.notif-panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  border-bottom: 1px solid var(--border);
  font-size: 13px;
  font-weight: 600;
  color: var(--text);
}
.notif-mark-read-btn {
  font-size: 11px;
  color: var(--accent);
  background: none;
  border: none;
  cursor: pointer;
  padding: 2px 4px;
  border-radius: 4px;
}
.notif-mark-read-btn:hover { background: var(--accent-light); }
.notif-list { max-height: 320px; overflow-y: auto; }
.notif-empty {
  padding: 24px 16px;
  text-align: center;
  font-size: 13px;
  color: var(--text-muted);
}
.notif-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 10px 16px;
  border-bottom: 1px solid var(--border-light);
  cursor: default;
  transition: background 0.1s;
}
.notif-item:last-child { border-bottom: none; }
.notif-item.unread { background: var(--accent-light); }
.notif-item:hover { background: var(--bg); }
.notif-item-icon { font-size: 14px; flex-shrink: 0; margin-top: 1px; }
.notif-item-body { flex: 1; min-width: 0; }
.notif-item-msg {
  font-size: 12px;
  color: var(--text);
  line-height: 1.4;
  word-break: break-word;
}
.notif-item-time {
  font-size: 11px;
  color: var(--text-muted);
  margin-top: 2px;
}
.notif-new-dot {
  width: 6px;
  height: 6px;
  background: var(--accent);
  border-radius: 50%;
  flex-shrink: 0;
  margin-top: 5px;
}
```

- [ ] **Step 2: 브라우저에서 `index.html` 열기 → 개발자 도구 콘솔에서 CSS 변수 확인**

```js
// 콘솔 실행 — 에러 없으면 OK
getComputedStyle(document.documentElement).getPropertyValue('--accent').trim()
// 기대값: "#4F46E5" 또는 유사한 값
```

- [ ] **Step 3: 저장 — 이 시점에서 아직 UI 변화 없음. CSS만 추가된 상태.**

---

## Task 2: 알림 데이터 함수 추가

**Files:**
- Modify: `index.html` — TOAST 섹션 바로 위 (~line 3224)에 NOTIFICATIONS 섹션 삽입

- [ ] **Step 1: `// TOAST` 주석 바로 위에 아래 JS 블록 추가**

```js
// ============================================================
// NOTIFICATIONS
// ============================================================
const NOTIF_KEY  = 'weekly_notifications';
const NOTIF_MAX  = 50;
const NOTIF_SHOW = 20;

let _notifications = [];

function loadNotifications() {
  try {
    _notifications = JSON.parse(localStorage.getItem(NOTIF_KEY) || '[]');
  } catch(e) {
    _notifications = [];
  }
}

function saveNotifications() {
  localStorage.setItem(NOTIF_KEY, JSON.stringify(_notifications));
}

const NOTIF_ICONS = {
  task_add:       '📋',
  task_delete:    '🗑',
  status_done:    '✅',
  status_editing: '✏️',
};

function pushNotification(type, memberId, memberName, extraLabel) {
  const icons = { task_add: '📋', task_delete: '🗑', status_done: '✅', status_editing: '✏️' };
  const labels = {
    task_add:       `${memberName} · 과제 추가${extraLabel ? ': ' + extraLabel : ''}`,
    task_delete:    `${memberName} · 과제 삭제${extraLabel ? ': ' + extraLabel : ''}`,
    status_done:    `${memberName} → 작성완료`,
    status_editing: `${memberName} → 수정 중`,
  };
  const notif = {
    id:         generateId(),
    type,
    memberId,
    memberName,
    message:    labels[type] || type,
    createdAt:  Date.now(),
    read:       false,
  };
  _notifications.unshift(notif);
  if (_notifications.length > NOTIF_MAX) _notifications = _notifications.slice(0, NOTIF_MAX);
  saveNotifications();
  updateNotifBadge();
}

function updateNotifBadge() {
  const badge = document.getElementById('notifBadge');
  if (!badge) return;
  const count = _notifications.filter(n => !n.read).length;
  badge.textContent = count > 0 ? (count > 9 ? '9+' : count) : '';
  badge.style.display = count > 0 ? 'flex' : 'none';
}

function markAllRead() {
  _notifications.forEach(n => n.read = true);
  saveNotifications();
  updateNotifBadge();
}

function formatRelativeTime(ts) {
  const diff = Date.now() - ts;
  const mins  = Math.floor(diff / 60000);
  const hours = Math.floor(diff / 3600000);
  const days  = Math.floor(diff / 86400000);
  if (diff < 60000)   return '방금 전';
  if (mins  < 60)     return `${mins}분 전`;
  if (hours < 24)     return `${hours}시간 전`;
  return `${days}일 전`;
}
```

- [ ] **Step 2: 브라우저 콘솔에서 데이터 함수 동작 검증**

```js
// 콘솔에서 실행
loadNotifications();
pushNotification('task_add', 'kim_yujeong', '김유정', '테스트과제');
pushNotification('status_done', 'cho_nakwoon', '조낙운');
console.log(_notifications.length);       // 기대: 2
console.log(_notifications[0].read);      // 기대: false
console.log(formatRelativeTime(Date.now() - 90000)); // 기대: "1분 전"
```

Expected: 에러 없이 위 기댓값 출력.

- [ ] **Step 3: 테스트 데이터 정리**

```js
// 콘솔 실행 — 방금 넣은 테스트 알림 초기화
localStorage.removeItem('weekly_notifications');
loadNotifications();
```

---

## Task 3: 렌더·토글 함수 추가

**Files:**
- Modify: `index.html` — Task 2에서 추가한 NOTIFICATIONS 섹션 내, `// TOAST` 바로 위

- [ ] **Step 1: `saveNotifications` 함수 아래에 아래 렌더·토글 함수 추가**

```js
function renderNotificationBell() {
  const count = _notifications.filter(n => !n.read).length;
  const badge = count > 0
    ? `<span id="notifBadge" class="notif-badge" style="display:flex">${count > 9 ? '9+' : count}</span>`
    : `<span id="notifBadge" class="notif-badge" style="display:none"></span>`;
  return `
    <div class="notif-wrap">
      <button class="notif-bell-btn" onclick="toggleNotificationPanel(event)" title="알림">
        🔔${badge}
      </button>
    </div>`;
}

function renderNotificationPanel() {
  const panel = document.getElementById('notifPanel');
  if (!panel) return;
  const items = _notifications.slice(0, NOTIF_SHOW);
  const unreadCount = _notifications.filter(n => !n.read).length;

  const itemsHtml = items.length === 0
    ? '<div class="notif-empty">최근 알림 없음</div>'
    : items.map(n => `
        <div class="notif-item${n.read ? '' : ' unread'}">
          <span class="notif-item-icon">${NOTIF_ICONS[n.type] || '🔔'}</span>
          <div class="notif-item-body">
            <div class="notif-item-msg">${esc(n.message)}</div>
            <div class="notif-item-time">${formatRelativeTime(n.createdAt)}</div>
          </div>
          ${n.read ? '' : '<div class="notif-new-dot"></div>'}
        </div>`).join('');

  panel.innerHTML = `
    <div class="notif-panel-header">
      <span>알림${unreadCount > 0 ? ` (${unreadCount})` : ''}</span>
      ${unreadCount > 0 ? '<button class="notif-mark-read-btn" onclick="markAllRead();closeNotifPanel()">모두 읽음</button>' : ''}
    </div>
    <div class="notif-list">${itemsHtml}</div>`;
}

let _notifOpen = false;

function toggleNotificationPanel(e) {
  e.stopPropagation();
  _notifOpen ? closeNotifPanel() : openNotifPanel();
}

function openNotifPanel() {
  const panel = document.getElementById('notifPanel');
  if (!panel) return;
  renderNotificationPanel();
  panel.classList.add('open');
  _notifOpen = true;
}

function closeNotifPanel() {
  const panel = document.getElementById('notifPanel');
  if (!panel) return;
  panel.classList.remove('open');
  _notifOpen = false;
  // 패널 닫힐 때 모두 읽음 처리
  markAllRead();
}
```

- [ ] **Step 2: 저장. 아직 HTML에 연결 전이므로 콘솔에서만 확인.**

```js
// 콘솔에서: 함수 정의 확인
typeof renderNotificationBell  // "function"
typeof toggleNotificationPanel // "function"
typeof closeNotifPanel         // "function"
```

---

## Task 4: 사이드바 정적 HTML에 벨 버튼 + 패널 삽입

**Files:**
- Modify: `index.html` — `<aside id="sidebar">` 내 `.sidebar-logo` 블록 (~line 1424)
- Modify: `index.html` — `init()` 함수 (~line 4151)

- [ ] **Step 1: `.sidebar-logo` 블록을 아래와 같이 교체**

현재:
```html
  <div class="sidebar-logo">
    <div class="sidebar-logo-icon">W</div>
    주간업무보고
  </div>
```

변경 후:
```html
  <div class="sidebar-logo" style="display:flex;align-items:center;justify-content:space-between;">
    <div style="display:flex;align-items:center;gap:10px;">
      <div class="sidebar-logo-icon">W</div>
      주간업무보고
    </div>
    <div id="notifBellMount"></div>
  </div>
  <div id="notifPanel" class="notif-panel"></div>
```

> **주의:** `.notif-panel`은 `.sidebar-logo` 밖, `<aside>` 바로 아래 형제 요소로 위치. `position: fixed`이므로 어디에 있어도 시각적으로는 동일하게 렌더링됨.

- [ ] **Step 2: `init()` 함수를 아래와 같이 수정**

현재:
```js
function init() {
  loadState();
  renderSidebar();
  renderMain();
}
```

변경 후:
```js
function init() {
  loadState();
  loadNotifications();
  renderSidebar();
  renderMain();

  // 벨 아이콘 마운트
  const bellMount = document.getElementById('notifBellMount');
  if (bellMount) bellMount.innerHTML = renderNotificationBell();

  // 패널 외부 클릭 시 닫기 (중복 등록 방지)
  if (!window._notifClickListenerAdded) {
    document.addEventListener('click', (e) => {
      if (_notifOpen && !e.target.closest('.notif-wrap') && !e.target.closest('#notifPanel')) {
        closeNotifPanel();
      }
    });
    window._notifClickListenerAdded = true;
  }
}
```

- [ ] **Step 3: 브라우저에서 `index.html` 새로고침 → 사이드바 상단에 🔔 버튼 확인**

기대: 사이드바 로고 우측에 반투명 🔔 버튼 보임, 알림 없으므로 뱃지 없음.

- [ ] **Step 4: 콘솔에서 알림 추가 후 벨 클릭 테스트**

```js
pushNotification('task_add', 'kim_yujeong', '김유정', '테스트과제');
// 기대: 🔔 옆에 빨간 뱃지 "1" 표시
```

그런 다음 🔔 클릭 → 패널에 "김유정 · 과제 추가: 테스트과제" 항목 확인.
패널 외부 클릭 → 패널 닫힘 + 뱃지 사라짐 확인.

- [ ] **Step 5: 테스트 데이터 정리**

```js
localStorage.removeItem('weekly_notifications');
location.reload();
```

---

## Task 5: renderSidebar에서 벨 뱃지 동기화

> `renderSidebar()`가 호출될 때 벨 아이콘을 재마운트하거나 뱃지만 갱신.

**Files:**
- Modify: `index.html` — `renderSidebar()` 함수 끝 (~line 1773)

- [ ] **Step 1: `renderSidebar()` 함수 맨 끝에 아래 줄 추가 (닫는 `}` 직전)**

현재 끝부분:
```js
function renderSidebar() {
  // ... 기존 코드 ...
  // (함수 끝)
}
```

추가:
```js
  // 벨 마운트 동기화 (renderSidebar가 DOM을 교체하지 않으므로 badge만 갱신)
  updateNotifBadge();
```

- [ ] **Step 2: 브라우저에서 확인 — 탭 전환 시 뱃지 유지 여부**

```js
pushNotification('status_done', 'cho_nakwoon', '조낙운');
// renderSidebar() 가 자동 호출됨 — 뱃지 "1" 유지 확인
```

- [ ] **Step 3: 테스트 데이터 정리**

```js
localStorage.removeItem('weekly_notifications'); location.reload();
```

---

## Task 6: 트리거 연결 — addTask, removeTask, setMemberStatus

**Files:**
- Modify: `index.html` — `addTask()` (~line 2177), `removeTask()` (~line 2198), `setMemberStatus()` (~line 1711)

- [ ] **Step 1: `addTask()` 에 알림 추가 — `saveState()` 호출 직후에 삽입**

현재:
```js
  getMemberData(memberId).tasks.push(task);
  saveState();
  renderMain();
```

변경 후:
```js
  getMemberData(memberId).tasks.push(task);
  saveState();
  pushNotification('task_add', memberId, getMember(memberId)?.name || memberId, task.kpiProduct || '새 과제');
  renderMain();
```

- [ ] **Step 2: `removeTask()` 에 알림 추가 — task 이름 캡처 후 filter 뒤에 삽입**

현재:
```js
function removeTask(memberId, taskId) {
  if (!confirm('이 과제를 삭제하시겠습니까?')) return;
  const data = getMemberData(memberId);
  data.tasks = data.tasks.filter(t => t.id !== taskId);
  saveState();
  renderMain();
}
```

변경 후:
```js
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

- [ ] **Step 3: `setMemberStatus()` 에 알림 추가 — `saveState()` 직후에 삽입**

현재:
```js
function setMemberStatus(memberId, status) {
  if (state.members[memberId]) {
    state.members[memberId].status = status;
    if (status === 'done') state.members[memberId].updatedAt = Date.now();
  }
  saveState();
  renderSidebar();
  renderMain();
}
```

변경 후:
```js
function setMemberStatus(memberId, status) {
  if (state.members[memberId]) {
    state.members[memberId].status = status;
    if (status === 'done') state.members[memberId].updatedAt = Date.now();
  }
  saveState();
  const notifType = status === 'done' ? 'status_done' : 'status_editing';
  pushNotification(notifType, memberId, getMember(memberId)?.name || memberId);
  renderSidebar();
  renderMain();
}
```

- [ ] **Step 4: 전체 기능 검증 — 브라우저에서 직접 테스트**

1. **과제 추가:** 아무 팀원 탭 → "과제 추가" 버튼 클릭 → 🔔 뱃지 "1" 확인
2. **작성완료:** 동일 탭에서 "작성완료" 버튼 클릭 → 뱃지 "2" 확인
3. **수정하기:** "수정하기" 버튼 클릭 → 뱃지 "3" 확인
4. **벨 클릭:** 패널에 3개 항목 확인, 아이콘·메시지·시간 모두 표시
5. **모두 읽음 버튼 클릭:** 패널 닫힘 + 뱃지 사라짐 확인
6. **과제 삭제:** 과제 추가 후 삭제 → 뱃지 재등장 확인
7. **외부 클릭:** 패널 열린 상태에서 메인 영역 클릭 → 패널 닫힘 확인
8. **새로고침 후:** 뱃지 및 패널 내역이 localStorage에서 복원됨 확인

- [ ] **Step 5: 콘솔 에러 없음 확인**

개발자 도구 Console 탭 — 빨간 에러 0건.

---

## 검증 체크리스트

```
[ ] 벨 아이콘이 사이드바 로고 우측에 표시됨
[ ] 미읽음 알림 수가 빨간 뱃지로 표시됨
[ ] 벨 클릭 시 드롭다운 패널 열림
[ ] 패널에 아이콘·메시지·상대시간 표시됨
[ ] 미읽음 항목에 파란 배경 + 파란 점(NEW) 표시됨
[ ] 패널 외부 클릭 시 패널 닫힘 + 뱃지 사라짐
[ ] "모두 읽음" 버튼 클릭 시 패널 닫힘 + 뱃지 초기화
[ ] 과제 추가 시 알림 생성됨 (📋 아이콘)
[ ] 과제 삭제 시 알림 생성됨 (🗑 아이콘)
[ ] 작성완료 시 알림 생성됨 (✅아이콘)
[ ] 수정하기 클릭 시 알림 생성됨 (✏️ 아이콘)
[ ] 새로고침 후 알림 내역 복원됨
[ ] 콘솔 에러 없음
```

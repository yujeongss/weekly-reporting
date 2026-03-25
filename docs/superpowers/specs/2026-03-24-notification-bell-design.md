# 알림 벨 아이콘 기능 설계

**날짜:** 2026-03-24
**프로젝트:** 전략개발팀 주간업무보고 (`index.html`)
**상태:** 승인됨

---

## 1. 개요

사이드바 상단 로고 영역에 벨(🔔) 아이콘을 추가하고, 특정 데이터 변경 이벤트 발생 시 알림을 생성하여 뱃지로 표시한다. 클릭 시 드롭다운 패널에서 최근 알림 목록을 확인할 수 있다.

---

## 2. UI 배치

### 벨 아이콘 위치
- `.sidebar-logo` 영역 우측에 벨 아이콘 버튼 배치
- 미읽음 알림이 있을 때 빨간색 뱃지(숫자) 표시
- **주의:** `.sidebar-logo`에 `overflow: hidden`이 있으므로, 드롭다운 패널은 `position: fixed`로 렌더링하여 부모 overflow 제약을 우회한다

```
┌─────────────────────────────┐
│  W  주간업무보고        🔔 3 │
└─────────────────────────────┘
```

### 드롭다운 패널
- 벨 클릭 시 토글 (`position: fixed`, 사이드바 너비에 맞춰 left/top 계산)
- 헤더: "알림 (N)" + "모두 읽음" 버튼
- 항목당: 아이콘 + 메시지 + 상대 시간(예: "방금 전", "3분 전")
- 빈 상태: "최근 알림 없음"
- 최대 20개 항목 표시 (저장은 최대 50개)

```
┌─────────────────────────────┐
│  알림 (3)        [모두 읽음] │
├─────────────────────────────┤
│ ✅  김유정 → 작성완료       │
│     방금 전                 │
├─────────────────────────────┤
│ 📋  조낙운 · QC테스트앱 추가 │
│     2분 전                  │
├─────────────────────────────┤
│ 🗑  이성대 · 과제 삭제      │
│     5분 전                  │
└─────────────────────────────┘
```

---

## 3. 데이터 모델

**localStorage 키:** `weekly_notifications`

```js
// Array of notification objects
[
  {
    id: string,           // generateId()
    type: "task_add" | "task_delete" | "status_done" | "status_editing",
    memberId: string,
    memberName: string,
    message: string,      // 표시 메시지
    createdAt: number,    // Date.now()
    read: boolean
  }
]
```

**보관 정책:** 최대 50개. 초과 시 오래된 항목부터 제거.

---

## 4. 알림 트리거

> `addTask()`, `removeTask()`, `setMemberStatus()`는 모두 기존 코드베이스에 존재하는 함수.
> `setMemberStatus('editing')` 호출 실재 확인 (수정하기 버튼 onclick).

| 이벤트 | 호출 함수 | type | 메시지 예시 |
|--------|----------|------|------------|
| 과제 추가 | `addTask()` | `task_add` | `"김유정 · 과제 추가: QC테스트과제"` |
| 과제 삭제 | `removeTask()` | `task_delete` | `"김유정 · 과제 삭제: QC테스트과제"` |
| 작성완료 | `setMemberStatus('done')` | `status_done` | `"김유정 → 작성완료"` |
| 수정 중 | `setMemberStatus('editing')` | `status_editing` | `"김유정 → 수정 중"` |

---

## 5. 동작 흐름

1. 트리거 함수 호출 → `pushNotification(type, data)` 실행
2. 알림 배열 맨 앞에 삽입 → localStorage 저장
3. 벨 아이콘 뱃지 숫자 갱신 (미읽음 수)
4. 벨 클릭 → 패널 토글 (열기/닫기)
5. **패널 외부 클릭 시 패널 닫힘** — `document` 레벨 click 리스너로 `event.target.closest('.notif-wrap')` 체크
6. 패널이 **닫힐 때** 모든 항목 `read: true` 처리 → 뱃지 사라짐 (패널이 열려있는 동안은 NEW 마커 유지)
7. "모두 읽음" 버튼 → 즉시 읽음 처리 + 패널 닫기

---

## 6. 구현 범위

### 추가할 함수
- `pushNotification(type, memberId, memberName, extraLabel?)` — 알림 생성 + 저장
- `loadNotifications()` — localStorage에서 로드
- `saveNotifications()` — localStorage에 저장
- `markAllRead()` — 전체 읽음 처리
- `renderNotificationBell()` — 벨 + 뱃지 HTML 생성
- `renderNotificationPanel()` — 드롭다운 패널 HTML 생성
- `toggleNotificationPanel()` — 패널 열기/닫기
- `formatRelativeTime(ts)` — 상대 시간 포맷

### 수정할 함수
- `renderSidebar()` — 벨 아이콘 포함하도록
- `addTask()` — `pushNotification('task_add', ...)` 호출 추가
- `deleteTask()` — `pushNotification('task_delete', ...)` 호출 추가
- `setMemberStatus()` — `pushNotification('status_done/editing', ...)` 호출 추가

### 추가할 CSS
- `.notif-bell-btn` — 벨 버튼 스타일 (sidebar-logo 내 우측 배치)
- `.notif-badge` — 빨간 뱃지
- `.notif-panel` — 드롭다운 패널
- `.notif-item` — 개별 알림 항목
- `.notif-item.unread` — 미읽음 강조

---

## 7. 구현 주의사항

1. **`removeTask` 내 `pushNotification` 삽입 위치:** `confirm()` 가드와 `.filter()` 실행 *이후*에 삽입. 취소된 삭제는 알림 생성 안 함. task 이름은 filter 전에 미리 캡처.
2. **`renderSidebar()`는 부분 업데이트 함수:** innerHTML 전체 교체가 아닌 `getElementById`로 특정 노드만 업데이트함. 벨 아이콘은 정적 HTML(`<aside id="sidebar">`)에 직접 삽입하거나, `renderSidebar` 내에서 `querySelector('.sidebar-logo')`로 타겟팅.
3. **`loadState()`는 항상 fresh reset:** `weekly_notifications` 는 별도 키이므로 영향 없음.

## 8. 제외 범위

- 과제 외 항목(금주활동, 배포 등) 변경 알림 — 트리거 이벤트 범위 밖
- AI 어시스턴트 변경 알림 — 별도 구현 가능하나 이번 범위 제외
- 푸시 알림(브라우저 Notification API) — 로컬 앱 특성상 불필요
- 알림 개별 삭제 — "모두 읽음" 만으로 충분

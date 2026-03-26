# Phase 1 — 빠른 수정 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 팀현황 진척율 읽기전용화, 이슈 강조 개선, 팀 전체 요약 수치 바 추가, CLAUDE.md 갱신

**Architecture:** 단일 파일(`index.html`) 내 CSS + JS 수정. 외부 의존성 없음. `renderTeamView()` → `renderTeamTableView()` / `renderFavCards()` / 신규 `renderTeamStatBar()` 함수 수정.

**Tech Stack:** Vanilla JS, inline CSS, 단일 HTML 파일

---

## 파일 구조

- **수정:** `D:/work/claude-test/weekly-reporting/index.html`
  - CSS 섹션 (~line 1409): `.team-table-row.has-issue` 스타일 변경
  - `renderTeamTableView()` (line 3591): 진척율 `<input>` → 텍스트
  - `renderFavCards()` (line 3758): task 카드에 이슈 dot 추가
  - `renderTeamView()` (line 3859): 요약 바 + 섹션 ID 추가
  - 신규 `renderTeamStatBar()` 함수 추가
  - `getTeamCounts()` (line 3513): 평균 진척율 계산 추가
- **수정:** `D:/work/claude-test/weekly-reporting/CLAUDE.md`

---

## Task 1: CLAUDE.md 갱신

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/CLAUDE.md`

- [ ] **Step 1: 현재 index.html 실제 상태 파악**

  다음을 확인:
  - 파일 총 줄 수: `wc -l index.html` (현재 약 5100줄)
  - STORAGE_KEY: `weekly_report_data_v7`
  - Excel export 방식: `buildMemberSheet()`, `buildTeamSheet()` (from-scratch, 템플릿 base64 없음)
  - state 구조: `favorites`, `chatMessages`, `teamViewMode`, `status`, `updatedAt`, `details` 포함

- [ ] **Step 2: CLAUDE.md 갱신**

  아래 내용으로 CLAUDE.md 업데이트:

  ```markdown
  # 전략개발팀 주간업무보고 반자동화 — Claude Code 가이드

  ## 프로젝트 개요

  팀원들이 매주 Excel로 작성하던 주간업무보고를 웹 UI로 관리하고 Excel로 자동 생성하는 도구.

  **핵심 파일:** `index.html` 단일 파일 (빌드 없음, 서버 없음, 약 5100줄)

  ---

  ## 아키텍처

  ```
  index.html (단일 파일, ~5100 lines)
  ├── CSS  (line ~8–1910)
  ├── HTML (line ~1910–1914)
  └── JS   (line ~1914–끝)
      ├── CONSTANTS / STATE         (line 1918)  — STORAGE_KEY, getDefaultState
      ├── STATE PERSISTENCE         — saveState/loadState (localStorage)
      ├── HELPERS / SIDEBAR / RENDER
      ├── TASKS / OPERATIONS / DEPLOYMENTS SECTIONS
      ├── CRUD — TASKS / OPERATIONS / DEPLOYMENTS
      ├── EXCEL EXPORT              — buildMemberSheet, buildTeamSheet (from-scratch)
      ├── TEAM VIEW                 — renderTeamView, renderTeamTableView, renderTeamCardView
      ├── FAVORITES                 — renderFavCards, toggleFav, isFav
      ├── AI ASSISTANT              — sendChatMessage, /api/claude proxy
      ├── NOTIFICATIONS             — pushNotification, renderNotifPanel
      └── INIT
  ```

  **외부 라이브러리:** ExcelJS 4.4.0 (CDN)

  ---

  ## 데이터 모델

  ```javascript
  localStorage key: 'weekly_report_data_v7'

  state = {
    meta: {
      teamName: "전략개발팀",
      year: 2026, month: 3, week: 2,
      members: [
        { id: "cho_nakwoon",   name: "조낙운",  division: "병원" },
        { id: "cho_minhee",   name: "조민희",  division: "인프라&공통" },
        { id: "lee_seongdae", name: "이성대",  division: "인프라&공통" },
        { id: "kim_yujeong",  name: "김유정",  division: "제약" },
        { id: "jung_jaeyoung",name: "정재영",  division: "제약" },
        { id: "oh_hyunmin",   name: "오현민",  division: "인프라&공통" }
      ]
    },
    activeTab: "kim_yujeong",   // 또는 '__team__'
    teamViewMode: 'table',       // 'table' | 'card'
    favorites: [{ type: 'task'|'deploy', memberId, id }],
    chatMessages: [{ role, content }],
    members: {
      "<memberId>": {
        status: 'done' | 'editing',
        updatedAt: null | ISO string,
        tasks: [{
          id, kpiProduct, division, deadline, overallProgress,
          currentWeekItems: [{ id, text, progress, details: [{id, text}] }],
          nextWeekItems:    [{ id, text, progress, details: [{id, text}] }],
          issues:           [{ id, text }]
        }],
        operations:  [{ id, text, weekType: "current"|"next" }],
        deployments: [{
          id, product, round, deployDate,
          counts: { feature, bug },
          progressText,
          redmineItems: [{ id, text }],
          issues: ""
        }]
      }
    }
  }
  ```

  ---

  ## Excel Export 방식

  **From-scratch 빌더 방식** (템플릿 base64 미사용):
  1. `exportToExcel()`: ExcelJS로 빈 workbook 생성
  2. `buildTeamSheet(ws)`: 팀 취합 시트 생성
  3. `buildMemberSheet(ws, memberId)`: 팀원별 개인 시트 생성
  4. 신규 팀원: `setupNewMemberSheet` 없이 동일 함수 재사용

  ---

  ## 핵심 함수 목록

  | 함수 | 역할 |
  |------|------|
  | `saveState()` / `loadState()` | localStorage 직렬화/역직렬화 |
  | `renderMain()` | 메인 영역 전체 재렌더 |
  | `renderTeamView()` | 팀현황 전체 |
  | `renderTeamTableView()` | 팀현황 통합 테이블 |
  | `renderTeamCardView()` | 팀현황 카드 뷰 |
  | `renderFavCards()` | 즐겨찾기 카드 |
  | `renderTaskCard(memberId, task)` | 과제 카드 |
  | `renderDeploymentsSection(memberId, data)` | 제품 배포 섹션 |
  | `addTask(memberId)` | 과제 추가 |
  | `exportToExcel()` | Excel 내보내기 |
  | `buildMemberSheet(ws, memberId)` | 개인 시트 from-scratch 생성 |
  | `buildTeamSheet(ws)` | 팀 시트 from-scratch 생성 |
  | `toggleFav(type, memberId, id)` | 즐겨찾기 토글 |
  | `isFav(type, memberId, id)` | 즐겨찾기 여부 확인 |
  | `sendChatMessage()` | AI 어시스턴트 메시지 전송 |
  | `showToast(msg)` | 하단 토스트 알림 |
  | `esc(str)` | XSS 방지 HTML 이스케이프 |

  ---

  ## 개발 규칙

  - **빌드 없음**: CDN만 사용, node_modules 커밋 안 함
  - **단일 파일**: 기능 추가 시 index.html 하나만 수정
  - **XSS 방지**: 사용자 입력은 반드시 `esc()` 통과 후 innerHTML
  - **저장 방식**: 모든 mutation 후 `saveState()` 호출
  - **재렌더**: `renderMain()` 호출로 전체 다시 그림
  ```

- [ ] **Step 3: 커밋**

  ```bash
  git add CLAUDE.md
  git commit -m "docs: CLAUDE.md를 현재 코드 상태에 맞게 갱신 (v7 key, from-scratch export, 전체 state 모델)"
  ```

---

## Task 2: 진척율 읽기전용화

**Files:**
- Modify: `index.html` — `renderTeamTableView()` (line 3626–3630)

**현재 코드 (line 3626–3630):**
```html
<td class="team-td team-td-pct">
  <input type="number" class="team-pct-input" min="0" max="100"
         value="${t.overallProgress || 0}"
         onchange="updateTaskField('${m.id}','${t.id}','overallProgress',+this.value);renderMain();">
</td>
```

- [ ] **Step 1: `<input>` → 텍스트로 교체**

  위 코드를 아래로 교체:

  ```html
  <td class="team-td team-td-pct" style="font-weight:600;color:var(--primary);">
    ${t.overallProgress || 0}%
  </td>
  ```

- [ ] **Step 2: 불필요해진 CSS 정리**

  `team-pct-input` CSS 클래스를 검색해 해당 스타일 블록 삭제.
  ```bash
  grep -n "team-pct-input" index.html
  ```

- [ ] **Step 3: 브라우저 확인**

  `index.html`을 브라우저에서 열고 팀현황 → 통합 테이블 진척율 컬럼이 텍스트로만 표시되는지 확인.
  클릭해도 편집 모드가 열리지 않아야 함.

- [ ] **Step 4: 커밋**

  ```bash
  git add index.html
  git commit -m "fix: 팀현황 통합 테이블 진척율을 읽기 전용으로 변경"
  ```

---

## Task 3: 이슈 강조 스타일 개선

**Files:**
- Modify: `index.html` — CSS `.team-table-row.has-issue` (line ~1409), `renderFavCards()` (line 3781)

**현재 CSS (line 1409):**
```css
.team-table-row.has-issue { background: #FFFBF0; }
```

- [ ] **Step 1: 이슈 행 CSS를 빨강 테마로 변경**

  기존 CSS를 아래로 교체:

  ```css
  .team-table-row.has-issue { background: rgba(220,38,38,0.04); border-left: 3px solid #DC2626; }
  .team-table-row:not(.has-issue) { border-left: 3px solid transparent; }
  ```

- [ ] **Step 2: 즐겨찾기 카드에 이슈 dot 추가**

  `renderFavCards()` 내 task 카드(line ~3781)에 이슈 dot 추가.
  `.fav-card`는 이미 `position:relative`이므로 dot을 `position:absolute`로 우측 상단에 배치.

  **현재:**
  ```javascript
  return `<div class="fav-card" data-fav="${k}">
  ```

  **변경 후** (issues 변수는 이미 line ~3777에서 선언됨):
  ```javascript
  const hasIssue = issues.length > 0;
  return `<div class="fav-card${hasIssue ? ' has-issue-fav' : ''}" data-fav="${k}">
    ${hasIssue ? '<span class="fav-issue-dot" title="이슈 있음"></span>' : ''}
  ```

  CSS 추가 (fav-card 스타일 근처):
  ```css
  .fav-issue-dot {
    position: absolute; top: 10px; right: 10px;
    width: 8px; height: 8px; border-radius: 50%;
    background: #DC2626;
  }
  .fav-card.has-issue-fav { border-top: 2px solid #DC2626; }
  ```

  deploy 타입 카드(line ~3815)에도 동일하게 추가:
  - `dep.issues` 문자열이 비어있지 않거나 redmineItems에 이슈 관련 내용이 있으면 hasIssue = true
  - 실제로는 `dep.issues && dep.issues.trim()` 으로 판단
  ```javascript
  const hasDeployIssue = !!(dep.issues && dep.issues.trim());
  return `<div class="fav-card${hasDeployIssue ? ' has-issue-fav' : ''}" data-fav="${k}">
    ${hasDeployIssue ? '<span class="fav-issue-dot" title="이슈 있음"></span>' : ''}
  ```

- [ ] **Step 3: 브라우저 확인**

  - 이슈 있는 행에 좌측 빨간 줄 + 연한 빨강 배경 확인
  - 이슈 있는 즐겨찾기 카드 우측 상단에 빨간 dot 확인

- [ ] **Step 4: 커밋**

  ```bash
  git add index.html
  git commit -m "feat: 팀현황 이슈 항목 빨간색 강조 + 즐겨찾기 카드 이슈 dot 추가"
  ```

---

## Task 4: 팀 전체 요약 수치 바 + 섹션 ID

**Files:**
- Modify: `index.html` — `getTeamCounts()` (line 3513), `renderTeamView()` (line 3859), 신규 `renderTeamStatBar()` 함수

- [ ] **Step 1: `getTeamCounts()`에 평균 진척율 추가**

  **현재 (line 3513–3522):**
  ```javascript
  function getTeamCounts() {
    let totalTasks = 0, issueTasks = 0, totalDeploys = 0;
    for (const m of state.meta.members) {
      const d = getMemberData(m.id);
      totalTasks  += (d.tasks || []).length;
      issueTasks  += (d.tasks || []).filter(t => (t.issues || []).some(i => i.text)).length;
      totalDeploys += (d.deployments || []).length;
    }
    return { totalTasks, issueTasks, totalDeploys };
  }
  ```

  **변경 후:**
  ```javascript
  function getTeamCounts() {
    let totalTasks = 0, issueTasks = 0, totalDeploys = 0;
    let progressSum = 0, progressCount = 0;
    for (const m of state.meta.members) {
      const d = getMemberData(m.id);
      const tasks = d.tasks || [];
      totalTasks   += tasks.length;
      issueTasks   += tasks.filter(t => (t.issues || []).some(i => i.text)).length;
      totalDeploys += (d.deployments || []).length;
      tasks.forEach(t => { progressSum += (t.overallProgress || 0); progressCount++; });
    }
    const avgProgress = progressCount > 0 ? Math.round(progressSum / progressCount) : null;
    return { totalTasks, issueTasks, totalDeploys, avgProgress };
  }
  ```

- [ ] **Step 2: `renderTeamStatBar()` 함수 신규 추가**

  `getTeamCounts()` 함수 바로 아래에 추가:

  ```javascript
  function renderTeamStatBar() {
    const { totalTasks, issueTasks, totalDeploys, avgProgress } = getTeamCounts();
    const avgText = avgProgress !== null ? avgProgress + '%' : '—';
    return `<div class="team-stat-bar">
      <div class="team-stat-item" onclick="document.getElementById('team-summary-section')?.scrollIntoView({behavior:'smooth'})">
        <span class="team-stat-label">전체 과제</span>
        <span class="team-stat-num">${totalTasks}</span>
      </div>
      <div class="team-stat-sep"></div>
      <div class="team-stat-item" onclick="document.getElementById('team-summary-section')?.scrollIntoView({behavior:'smooth'})">
        <span class="team-stat-label">평균 진척율</span>
        <span class="team-stat-num">${avgText}</span>
      </div>
      <div class="team-stat-sep"></div>
      <div class="team-stat-item${issueTasks > 0 ? ' warn' : ''}" onclick="document.getElementById('team-table-section')?.scrollIntoView({behavior:'smooth'})">
        <span class="team-stat-label">이슈</span>
        <span class="team-stat-num">${issueTasks}건</span>
      </div>
      <div class="team-stat-sep"></div>
      <div class="team-stat-item" onclick="document.getElementById('team-deploy-section')?.scrollIntoView({behavior:'smooth'})">
        <span class="team-stat-label">이번 주 배포</span>
        <span class="team-stat-num">${totalDeploys}건</span>
      </div>
    </div>`;
  }
  ```

- [ ] **Step 3: CSS 추가**

  CSS 섹션(팀 관련 스타일 근처)에 추가:

  ```css
  .team-stat-bar {
    display: flex; align-items: center; gap: 0;
    background: var(--card-bg); border: 1px solid var(--border-light);
    border-radius: 10px; padding: 10px 20px; margin-bottom: 12px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
  }
  .team-stat-item {
    flex: 1; display: flex; flex-direction: column; align-items: center;
    gap: 2px; cursor: pointer; padding: 4px 8px; border-radius: 6px;
    transition: background 0.15s;
  }
  .team-stat-item:hover { background: var(--bg); }
  .team-stat-item.warn .team-stat-num { color: #DC2626; }
  .team-stat-label { font-size: 11px; color: var(--text-muted); }
  .team-stat-num { font-size: 18px; font-weight: 700; color: var(--primary); }
  .team-stat-sep { width: 1px; height: 32px; background: var(--border-light); flex-shrink: 0; }
  ```

- [ ] **Step 4: `renderTeamView()`에 요약 바 + 섹션 ID 추가**

  **현재 `renderTeamView()` 반환 HTML (line 3865–3897):**
  ```javascript
  return `<div class="team-view">
    <div class="member-header">...</div>
    <div class="team-summary${...}">
      ...
    </div>
    ${renderFavCards()}
    <div class="section">
      <div class="section-header">
        <span class="section-title">팀원 현황</span>
        ...
      </div>
      ${content}
    </div>
    ${renderTeamOperationsSection()}
    ${renderTeamDeploySection()}
  </div>`;
  ```

  **변경 후:**
  ```javascript
  return `<div class="team-view">
    <div class="member-header">...</div>
    ${renderTeamStatBar()}
    <div id="team-summary-section" class="team-summary${_teamSummaryCollapsed ? ' collapsed' : ''}">
      ...
    </div>
    ${renderFavCards()}
    <div id="team-table-section" class="section">
      <div class="section-header">
        <span class="section-title">팀원 현황</span>
        ...
      </div>
      ${content}
    </div>
    ${renderTeamOperationsSection()}
    ${renderTeamDeploySection()}
  </div>`;
  ```

  주의사항:
  - `toggleTeamSummary()`는 `document.querySelector('.team-summary')` 클래스 셀렉터를 사용하므로 ID 추가 후에도 **변경 불필요**.
  - `renderTeamDeploySection()`은 이미 `<div class="section">` 래퍼로 감싸서 반환함. `id="team-deploy-section"`은 해당 함수 내부의 `<div class="section"` 태그에 직접 추가:
    ```javascript
    // renderTeamDeploySection() 내부 line ~3673
    return `<div id="team-deploy-section" class="section" style="margin-top:20px;">
    ```
    단, 배포가 없을 때 함수가 `''`를 반환하므로 스크롤 앵커 ID가 DOM에 없을 수 있음 — 허용 가능한 동작(배포 없으면 해당 섹션 자체가 없음).
  - 기존 `renderTeamCountCards()` (line 3539)는 `team-summary-body` 안에서 호출되어 "이슈 과제/배포건수" 카드를 보여줌. 새 `team-stat-bar`는 그 위에 별도로 표시되므로 **수치가 두 곳에 보임**. `renderTeamCountCards()` 호출을 `renderTeamView()`의 `team-summary-body`에서 **제거**하여 중복 방지.

- [ ] **Step 5: 브라우저 확인**

  - 팀현황 상단에 4개 수치 바 확인
  - 각 수치 클릭 시 해당 섹션으로 스크롤 확인
  - 이슈 건수 > 0이면 빨간색으로 표시 확인

- [ ] **Step 6: 커밋**

  ```bash
  git add index.html
  git commit -m "feat: 팀현황 상단 요약 수치 바 추가 (전체 과제, 평균 진척율, 이슈, 배포)"
  ```

---

## Task 5: GitHub 배포

- [ ] **Step 1: 전체 변경 확인**

  ```bash
  git log --oneline -5
  git status
  ```

- [ ] **Step 2: GitHub push**

  ```bash
  git push origin main
  ```

- [ ] **Step 3: Vercel 배포**

  ```bash
  NODE_TLS_REJECT_UNAUTHORIZED=0 vercel --prod
  ```

  또는 GitHub push 후 Vercel 자동 배포 확인.

---

## 검증 체크리스트

브라우저에서 `index.html` 열고 팀현황 탭 진입 후 확인:

- [ ] 통합 테이블 진척율 컬럼: 숫자 클릭해도 편집 불가 (텍스트만 표시)
- [ ] 이슈 있는 행: 좌측 빨간 줄 + 연한 빨강 배경
- [ ] 이슈 있는 즐겨찾기 카드: 우측 상단 빨간 dot
- [ ] 팀현황 상단: 전체 과제 / 평균 진척율 / 이슈 / 이번 주 배포 4개 수치 표시
- [ ] 각 수치 클릭 시 해당 섹션으로 스크롤
- [ ] CLAUDE.md: v7 key, 5100줄, from-scratch export 내용 반영

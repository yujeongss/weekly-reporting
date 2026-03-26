# Phase 2 — 팀현황 고도화 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 팀현황에 필터/정렬, 배포 타임라인, 팀 취합 Excel 내보내기 기능 추가

**Architecture:** 단일 파일(`index.html`) 내 CSS + JS 수정. 모든 새 기능은 모듈 레벨 변수(`_teamFilter`)와 순수 렌더 함수로 구현. Excel export는 ExcelJS from-scratch 빌더 방식 사용.

**Tech Stack:** Vanilla JS, inline CSS, ExcelJS 4.4.0 (CDN), 단일 HTML 파일

---

## 코드 현황 (Phase 1 완료 후)

- `index.html` ~5110줄
- `renderTeamTableView()` — line 3638, 팀원별 tasks를 테이블 행으로 렌더
- `renderTeamDeploySection()` — line 3691, 배포 테이블 반환 (`id="team-deploy-section"`)
- `renderTeamView()` — line 3908, 팀현황 전체 조합
- `exportToExcel()` — line 2944, 현재 개인용 Excel 내보내기
- `buildTeamSheet(ws)` — line 3278, 팀 취합 시트 (기존 포맷)
- `getMemberData(memberId)` — member state 반환
- `getMember(memberId)` — member meta 반환
- `esc(str)` — XSS 방지 이스케이프

---

## 파일 구조

- **수정:** `D:/work/claude-test/weekly-reporting/index.html`
  - CSS 섹션: 필터 바, 타임라인, 신규 버튼 스타일 추가
  - 신규 JS 함수: `renderTeamFilterBar()`, `applyTeamFilter()`, `renderDeployTimeline()`, `exportTeamSummary()`, `buildTeamSummarySheet(ws)`
  - 수정 JS: `renderTeamTableView()` — `_teamFilter` 적용, `renderTeamDeploySection()` — 타임라인 추가, `renderTeamView()` — 필터 바 + 버튼 추가

---

## Task 1: 팀원 현황 필터/정렬

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - 신규 모듈 변수: `_teamFilter`
  - 신규 함수: `renderTeamFilterBar()`, `applyTeamFilter()`
  - 수정: `renderTeamTableView()`, `renderTeamView()`
  - CSS: `.team-filter-bar` 관련

- [ ] **Step 1: `_teamFilter` 모듈 변수 추가**

  `_teamCardCollapsed`나 `_teamSummaryCollapsed` 같은 모듈 레벨 변수들 근처(line ~3895)에 추가:

  ```javascript
  let _teamFilter = { members: [], issuesOnly: false, progressRange: 'all', sort: 'default' };
  ```

  `members: []` = 빈 배열이면 전체 팀원 표시 (필터 없음).

- [ ] **Step 2: `renderTeamFilterBar()` 함수 추가**

  `renderTeamTableView()` 바로 위에 추가:

  ```javascript
  function renderTeamFilterBar() {
    const memberOptions = state.meta.members.map(m =>
      `<label class="team-filter-member-label">
        <input type="checkbox" value="${m.id}"
          ${_teamFilter.members.length === 0 || _teamFilter.members.includes(m.id) ? 'checked' : ''}
          onchange="applyTeamFilter('member', this.value, this.checked)">
        ${esc(m.name)}
      </label>`
    ).join('');

    return `<div class="team-filter-bar">
      <div class="team-filter-group">
        <span class="team-filter-label">팀원</span>
        <div class="team-filter-members">${memberOptions}</div>
      </div>
      <div class="team-filter-group">
        <label class="team-filter-toggle">
          <input type="checkbox" ${_teamFilter.issuesOnly ? 'checked' : ''}
            onchange="applyTeamFilter('issuesOnly', null, this.checked)">
          이슈만 보기
        </label>
      </div>
      <div class="team-filter-group">
        <span class="team-filter-label">진척율</span>
        <select class="team-filter-select" onchange="applyTeamFilter('progressRange', this.value)">
          <option value="all" ${_teamFilter.progressRange==='all'?'selected':''}>전체</option>
          <option value="low" ${_teamFilter.progressRange==='low'?'selected':''}>0–30%</option>
          <option value="mid" ${_teamFilter.progressRange==='mid'?'selected':''}>30–70%</option>
          <option value="high" ${_teamFilter.progressRange==='high'?'selected':''}>70–100%</option>
        </select>
      </div>
      <div class="team-filter-group">
        <span class="team-filter-label">정렬</span>
        <select class="team-filter-select" onchange="applyTeamFilter('sort', this.value)">
          <option value="default" ${_teamFilter.sort==='default'?'selected':''}>기본</option>
          <option value="pct_asc" ${_teamFilter.sort==='pct_asc'?'selected':''}>진척율 ↑</option>
          <option value="pct_desc" ${_teamFilter.sort==='pct_desc'?'selected':''}>진척율 ↓</option>
          <option value="deadline_asc" ${_teamFilter.sort==='deadline_asc'?'selected':''}>기한 ↑</option>
          <option value="deadline_desc" ${_teamFilter.sort==='deadline_desc'?'selected':''}>기한 ↓</option>
        </select>
      </div>
      <button class="team-filter-reset" onclick="resetTeamFilter()">초기화</button>
    </div>`;
  }
  ```

- [ ] **Step 3: `applyTeamFilter()` + `resetTeamFilter()` 함수 추가**

  `renderTeamFilterBar()` 바로 아래에 추가:

  ```javascript
  function applyTeamFilter(key, value, checked) {
    if (key === 'member') {
      // 첫 번째 uncheck 시 빈 배열(=전체표시) 상태에서 → 전체 ID로 bootstrap
      if (!checked && _teamFilter.members.length === 0) {
        _teamFilter.members = state.meta.members.map(m => m.id);
      }
      if (checked) {
        if (!_teamFilter.members.includes(value)) _teamFilter.members.push(value);
      } else {
        _teamFilter.members = _teamFilter.members.filter(id => id !== value);
      }
      // 전체 팀원이 모두 선택된 상태 → 빈 배열로 복귀 (canonical "no filter")
      if (_teamFilter.members.length === state.meta.members.length) {
        _teamFilter.members = [];
      }
    } else if (key === 'issuesOnly') {
      _teamFilter.issuesOnly = !!checked;
    } else if (key === 'progressRange') {
      _teamFilter.progressRange = value;
    } else if (key === 'sort') {
      _teamFilter.sort = value;
    }
    renderMain();
  }

  function resetTeamFilter() {
    _teamFilter = { members: [], issuesOnly: false, progressRange: 'all', sort: 'default' };
    renderMain();
  }
  ```

- [ ] **Step 4: `renderTeamTableView()` 에 필터/정렬 적용**

  `renderTeamTableView()` (line 3638) 함수 시작 부분에 필터·정렬 처리 로직 추가.

  현재 코드 첫 줄:
  ```javascript
  function renderTeamTableView() {
    const rows = [];
    for (const m of state.meta.members) {
  ```

  변경 후:
  ```javascript
  function renderTeamTableView() {
    const rows = [];

    // 1. 팀원 필터
    let members = state.meta.members;
    if (_teamFilter.members.length > 0) {
      members = members.filter(m => _teamFilter.members.includes(m.id));
    }

    // 2. 정렬 (팀원 단위)
    if (_teamFilter.sort === 'pct_asc') {
      members = [...members].sort((a, b) => (getMemberAvgProgress(a.id) || 0) - (getMemberAvgProgress(b.id) || 0));
    } else if (_teamFilter.sort === 'pct_desc') {
      members = [...members].sort((a, b) => (getMemberAvgProgress(b.id) || 0) - (getMemberAvgProgress(a.id) || 0));
    } else if (_teamFilter.sort === 'deadline_asc') {
      members = [...members].sort((a, b) => {
        const safeDate = t => t.deadline ? (new Date(t.deadline).getTime() || Infinity) : Infinity;
        const aMin = Math.min(...(getMemberData(a.id).tasks || []).map(safeDate).concat([Infinity]));
        const bMin = Math.min(...(getMemberData(b.id).tasks || []).map(safeDate).concat([Infinity]));
        return aMin - bMin;
      });
    } else if (_teamFilter.sort === 'deadline_desc') {
      members = [...members].sort((a, b) => {
        const safeDate = t => t.deadline ? (new Date(t.deadline).getTime() || 0) : 0;
        const aMax = Math.max(...(getMemberData(a.id).tasks || []).map(safeDate).concat([0]));
        const bMax = Math.max(...(getMemberData(b.id).tasks || []).map(safeDate).concat([0]));
        return bMax - aMax;
      });
    }

    for (const m of members) {
  ```

  그 다음, 각 팀원의 tasks를 필터링하는 로직을 `tasks.forEach` 이전에 추가:

  현재:
  ```javascript
    const data = getMemberData(m.id);
    const tasks = data.tasks || [];
    if (!tasks.length) {
  ```

  변경 후:
  ```javascript
    const data = getMemberData(m.id);
    let tasks = data.tasks || [];

    // 3. 이슈만 보기 필터
    if (_teamFilter.issuesOnly) {
      tasks = tasks.filter(t => (t.issues || []).some(i => i.text));
    }

    // 4. 진척율 구간 필터
    if (_teamFilter.progressRange !== 'all') {
      tasks = tasks.filter(t => {
        const p = t.overallProgress || 0;
        if (_teamFilter.progressRange === 'low')  return p <= 30;
        if (_teamFilter.progressRange === 'mid')  return p > 30 && p <= 70;
        if (_teamFilter.progressRange === 'high') return p > 70;
        return true;
      });
    }

    if (!tasks.length) {
  ```

- [ ] **Step 5: `renderTeamView()`에 필터 바 추가 (table 모드일 때만)**

  `renderTeamView()` (line 3908)에서 `${content}` 앞에 필터 바 삽입.

  현재:
  ```javascript
    <div id="team-table-section" class="section">
      <div class="section-header">
        <span class="section-title">팀원 현황</span>
        <div class="team-view-toggle">
  ```

  변경 후 (section-header 안의 toggle 다음, content 앞에):
  ```javascript
    <div id="team-table-section" class="section">
      <div class="section-header">
        <span class="section-title">팀원 현황</span>
        <div class="team-view-toggle">
          ...
        </div>
      </div>
      ${mode === 'table' ? renderTeamFilterBar() : ''}
      ${content}
    </div>
  ```

  **정확한 삽입 위치:** 실제 코드 line 3934–3942를 먼저 Read 도구로 읽고 삽입.
  필터 바는 `section-header` div가 닫힌 `</div>` 다음 줄, `${content}` 바로 앞에 위치해야 함. `section-header` 안에 넣으면 안 됨.

- [ ] **Step 6: CSS 추가**

  팀 관련 CSS 근처에 추가:

  ```css
  .team-filter-bar {
    display: flex; flex-wrap: wrap; align-items: center; gap: 12px;
    padding: 10px 14px; background: var(--bg); border: 1px solid var(--border-light);
    border-radius: 8px; margin-bottom: 10px; font-size: 13px;
  }
  .team-filter-group { display: flex; align-items: center; gap: 6px; }
  .team-filter-label { font-size: 12px; color: var(--text-muted); white-space: nowrap; }
  .team-filter-members { display: flex; flex-wrap: wrap; gap: 6px; }
  .team-filter-member-label { display: flex; align-items: center; gap: 3px; cursor: pointer; font-size: 12px; }
  .team-filter-toggle { display: flex; align-items: center; gap: 4px; cursor: pointer; }
  .team-filter-select {
    font-size: 12px; padding: 3px 6px; border: 1px solid var(--border-light);
    border-radius: 5px; background: var(--card-bg); color: var(--text);
  }
  .team-filter-reset {
    font-size: 12px; padding: 3px 8px; border: 1px solid var(--border-light);
    border-radius: 5px; background: var(--card-bg); color: var(--text-muted);
    cursor: pointer;
  }
  .team-filter-reset:hover { background: var(--bg); color: var(--text); }
  ```

- [ ] **Step 7: 브라우저 확인**

  팀현황 → 테이블 모드:
  - 필터 바가 표시됨
  - 팀원 체크박스 해제 시 해당 팀원 행 숨김
  - "이슈만 보기" 체크 시 이슈 없는 행 숨김
  - 진척율 구간 선택 시 해당 범위 행만 표시
  - 정렬 변경 시 팀원 순서 변경
  - "초기화" 클릭 시 전체 표시
  - 카드 모드에서는 필터 바 미표시

- [ ] **Step 8: 커밋**

  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 팀현황 통합 테이블 필터/정렬 기능 추가"
  ```

---

## Task 2: 배포 타임라인

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - 신규 함수: `renderDeployTimeline()`
  - 수정: `renderTeamDeploySection()`
  - CSS: `.deploy-timeline` 관련

- [ ] **Step 1: `renderDeployTimeline()` 함수 추가**

  `renderTeamDeploySection()` 바로 위에 추가:

  ```javascript
  function renderDeployTimeline() {
    const today = new Date();
    const year  = today.getFullYear();
    const month = today.getMonth(); // 0-indexed
    const daysInMonth = new Date(year, month + 1, 0).getDate();

    // 이번 달 배포만 수집
    const items = [];
    for (const m of state.meta.members) {
      for (const dep of (getMemberData(m.id).deployments || [])) {
        if (!dep.deployDate) continue;
        const d = new Date(dep.deployDate);
        if (d.getFullYear() !== year || d.getMonth() !== month) continue;
        items.push({ dep, member: m, day: d.getDate(), isPast: d < today });
      }
    }

    if (!items.length) return '';

    const dots = items.map(({ dep, member, day, isPast }) => {
      const pct = ((day - 0.5) / daysInMonth * 100).toFixed(1);
      const label = esc(dep.product || '') + ' ' + esc(dep.round || '');
      const feat = dep.counts ? (dep.counts.feature || 0) : 0;
      const bug  = dep.counts ? (dep.counts.bug  || 0) : 0;
      const tooltip = `${esc(member.name)} | ${label} | ${esc(dep.deployDate || '')} | ${feat+bug}건(${feat}/${bug}) | ${esc(dep.progressText || '—')}`;
      // isPast: 오늘 이전 날짜만 회색 처리 (오늘 배포는 예정=파랑으로 표시)
      return `<div class="tl-dot-wrap" style="left:${pct}%">
        <div class="tl-dot${isPast ? ' past' : ' future'}" title="${tooltip}"></div>
        <div class="tl-dot-label">${label}</div>
      </div>`;
    }).join('');

    // 오늘 마커
    const todayPct = ((today.getDate() - 0.5) / daysInMonth * 100).toFixed(1);

    return `<div class="deploy-timeline">
      <div class="tl-header">
        <span class="tl-title">${year}년 ${month + 1}월 배포 일정</span>
        <span class="tl-legend"><span class="tl-leg-dot past"></span>완료 <span class="tl-leg-dot future"></span>예정</span>
      </div>
      <div class="tl-track">
        <div class="tl-line"></div>
        <div class="tl-today" style="left:${todayPct}%" title="오늘"></div>
        ${dots}
      </div>
      <div class="tl-axis">
        <span>1일</span><span>${Math.round(daysInMonth/2)}일</span><span>${daysInMonth}일</span>
      </div>
    </div>`;
  }
  ```

- [ ] **Step 2: `renderTeamDeploySection()`에 타임라인 추가**

  `renderTeamDeploySection()` (line 3691)의 반환값 마지막에 타임라인 삽입.

  현재:
  ```javascript
  return `<div id="team-deploy-section" class="section" style="margin-top:20px;">
    <div class="section-header">
      <span class="section-title">제품 배포</span>
    </div>
    <div class="team-table-wrap">
      ...
    </div>
  </div>`;
  ```

  변경 후 (테이블 다음, `</div>` 닫기 전에 타임라인 추가):
  ```javascript
  return `<div id="team-deploy-section" class="section" style="margin-top:20px;">
    <div class="section-header">
      <span class="section-title">제품 배포</span>
    </div>
    <div class="team-table-wrap">
      ...
    </div>
    ${renderDeployTimeline()}
  </div>`;
  ```

  즉 `${renderDeployTimeline()}`를 `</div>` 닫기 직전에 삽입.

- [ ] **Step 3: CSS 추가**

  팀 관련 CSS 근처에 추가:

  ```css
  .deploy-timeline {
    margin-top: 16px; padding: 12px 16px;
    background: var(--bg); border-radius: 8px; border: 1px solid var(--border-light);
  }
  .tl-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; }
  .tl-title { font-size: 12px; font-weight: 600; color: var(--text-secondary); }
  .tl-legend { font-size: 11px; color: var(--text-muted); display: flex; align-items: center; gap: 6px; }
  .tl-leg-dot { display: inline-block; width: 8px; height: 8px; border-radius: 50%; }
  .tl-leg-dot.past { background: #9CA3AF; }
  .tl-leg-dot.future { background: var(--primary); }
  .tl-track { position: relative; height: 40px; margin-bottom: 20px; overflow: visible; }
  .tl-line {
    position: absolute; top: 50%; left: 0; right: 0;
    height: 2px; background: var(--border-light); transform: translateY(-50%);
  }
  .tl-today {
    position: absolute; top: 50%; width: 2px; height: 16px;
    background: #EF4444; transform: translate(-50%, -50%);
  }
  .tl-dot-wrap { position: absolute; top: 50%; transform: translate(-50%, -50%); }
  .tl-dot {
    width: 10px; height: 10px; border-radius: 50%; cursor: pointer;
    border: 2px solid white; box-shadow: 0 0 0 1px currentColor;
  }
  .tl-dot.past { background: #9CA3AF; color: #9CA3AF; }
  .tl-dot.future { background: var(--primary); color: var(--primary); }
  .tl-dot-label {
    position: absolute; top: 14px; left: 50%; transform: translateX(-50%);
    font-size: 10px; color: var(--text-muted); white-space: nowrap;
    max-width: 80px; overflow: hidden; text-overflow: ellipsis;
  }
  .tl-axis { display: flex; justify-content: space-between; font-size: 10px; color: var(--text-muted); }
  ```

- [ ] **Step 4: 브라우저 확인**

  팀현황 → 제품 배포 섹션 아래:
  - 이번 달 배포 타임라인이 표시됨
  - 각 배포를 점으로 표시, hover 시 상세 툴팁
  - 지난 배포: 회색, 예정 배포: 파랑
  - 오늘 날짜: 빨간 세로선
  - 이번 달 배포가 없으면 타임라인 미표시

- [ ] **Step 5: 커밋**

  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 팀현황 제품 배포 타임라인 추가"
  ```

---

## Task 3: 팀현황 Excel 내보내기 (팀명_주간보고 포맷)

**Files:**
- Modify: `D:/work/claude-test/weekly-reporting/index.html`
  - 신규 함수: `exportTeamSummary()`, `buildTeamSummarySheet(ws)`
  - 수정: `renderTeamView()` — "팀 보고서 내보내기" 버튼 추가
  - CSS: 버튼 스타일

팀명_주간보고.xlsx 분석 결과 (포맷):
```
B1: "N월 N주차"   D1: "팀명 주간 업무 보고"
Row 4: B4="구분", C4="내용"
Row 6: B6="금주실적", C6="■ 주요 활동"
Row 7: D7="구분" E-H7="주요활동" I7="일정"  (헤더)
Row 8+: D=과제명, E-H=금주활동(텍스트), I=기한  (전 팀원 순서대로)
Row (8+N+2): B="차주계획", C="■ 주요 계획"
Row (8+N+3): 헤더 반복
Row (8+N+4)+: 차주계획 데이터
Row (차주끝+3): B="주요이슈 & 타본부협조사항", C="■ 주요 이슈"
Row +1+: 이슈 내용 (D열)
Row (이슈끝+2): B="프로젝트 진척도"
Row +1:  D="프로젝트" E="프로젝트" F="단계" G="기한" H="진척%" I="비고"
Row +2+: D-E=과제명, G=deadline, H=overallProgress(소수 e.g. 0.75)
Row (진척끝+2): B="제품배포"
Row +1:  D="제품" E="차수" F="배포기한" G="건수(기능/오류)" H-I="진척"
Row +2+: 배포 데이터
```

- [ ] **Step 1: `buildTeamSummarySheet(ws)` 함수 작성**

  `buildTeamSheet(ws)` (line 3278) 바로 아래에 새 함수 추가:

  ```javascript
  function buildTeamSummarySheet(ws) {
    // 컬럼 너비
    ws.getColumn(1).width = 3;   // A spacer
    ws.getColumn(2).width = 14;  // B 구분
    ws.getColumn(3).width = 12;  // C 내용
    ws.getColumn(4).width = 20;  // D 구분/과제명
    ws.getColumn(5).width = 28;  // E 주요활동
    ws.getColumn(6).width = 8;   // F 단계
    ws.getColumn(7).width = 12;  // G 기한
    ws.getColumn(8).width = 10;  // H 진척%
    ws.getColumn(9).width = 16;  // I 비고/일정

    const HDR_FILL = { type:'pattern', pattern:'solid', fgColor:{argb:'FF334155'} };
    const HDR_FONT = { size:10, bold:true, color:{argb:'FFFFFFFF'} };
    const BODY_FONT = { size:10, name:'맑은 고딕' };
    const WRAP = { wrapText: true, vertical: 'top' };
    const BORDER_ALL = {
      top:{style:'thin'}, bottom:{style:'thin'},
      left:{style:'thin'}, right:{style:'thin'}
    };

    function hdrCell(cell, val) {
      cell.value = val; cell.fill = HDR_FILL; cell.font = HDR_FONT;
      cell.alignment = { horizontal:'center', vertical:'middle' };
      cell.border = BORDER_ALL;
    }
    function bodyCell(cell, val) {
      cell.value = val; cell.font = BODY_FONT;
      cell.alignment = WRAP; cell.border = BORDER_ALL;
    }
    function sectionLabel(row, label) {
      const c = ws.getCell(row, 2);
      c.value = label;
      c.font = { size:10, bold:true };
      c.fill = { type:'pattern', pattern:'solid', fgColor:{argb:'FFE2E8F0'} };
      c.border = BORDER_ALL;
    }

    // Row 1: 헤더
    ws.getCell(1,2).value = `${state.meta.month}월 ${state.meta.week}주차`;
    ws.getCell(1,4).value = `${state.meta.teamName} 주간 업무 보고`;
    ws.getCell(1,4).font = { size:13, bold:true };

    // Row 4: 구분/내용 헤더
    ws.getCell(4,2).value = '구분'; ws.getCell(4,3).value = '내용';

    // ── 금주실적 ──
    let row = 6;
    sectionLabel(row, '금주실적');
    ws.getCell(row, 3).value = '■ 주요 활동';
    row++;

    // 헤더행
    hdrCell(ws.getCell(row, 4), '구분');
    ['주요활동','주요활동','주요활동','주요활동'].forEach((v, i) => hdrCell(ws.getCell(row, 5+i), v));
    hdrCell(ws.getCell(row, 9), '일정');
    row++;

    // 데이터: 전 팀원 tasks → currentWeekItems
    for (const m of state.meta.members) {
      for (const t of (getMemberData(m.id).tasks || [])) {
        const items = (t.currentWeekItems || []).filter(i => i.text);
        const text = items.map(i => '- ' + i.text).join('\n');
        if (!text && !t.kpiProduct) continue;
        bodyCell(ws.getCell(row, 4), t.kpiProduct || '');
        ws.getCell(row, 5).value = text;
        ws.getCell(row, 5).font = BODY_FONT;
        ws.getCell(row, 5).alignment = WRAP;
        ws.getCell(row, 5).border = BORDER_ALL;
        ws.mergeCells(row, 5, row, 8); // E–H 병합
        bodyCell(ws.getCell(row, 9), t.deadline || '');
        ws.getRow(row).height = Math.max(20, items.length * 16);
        row++;
      }
    }

    row += 2;

    // ── 차주계획 ──
    sectionLabel(row, '차주계획');
    ws.getCell(row, 3).value = '■ 주요 계획';
    row++;

    hdrCell(ws.getCell(row, 4), '구분');
    ['주요활동','주요활동','주요활동','주요활동'].forEach((v, i) => hdrCell(ws.getCell(row, 5+i), v));
    hdrCell(ws.getCell(row, 9), '일정');
    row++;

    for (const m of state.meta.members) {
      for (const t of (getMemberData(m.id).tasks || [])) {
        const items = (t.nextWeekItems || []).filter(i => i.text);
        const text = items.map(i => '- ' + i.text).join('\n');
        if (!text && !t.kpiProduct) continue;
        bodyCell(ws.getCell(row, 4), t.kpiProduct || '');
        ws.getCell(row, 5).value = text;
        ws.getCell(row, 5).font = BODY_FONT;
        ws.getCell(row, 5).alignment = WRAP;
        ws.getCell(row, 5).border = BORDER_ALL;
        ws.mergeCells(row, 5, row, 8); // E–H 병합
        bodyCell(ws.getCell(row, 9), t.deadline || '');
        ws.getRow(row).height = Math.max(20, items.length * 16);
        row++;
      }
    }

    row += 2;

    // ── 주요이슈 ──
    sectionLabel(row, '주요이슈 &');
    ws.getCell(row, 3).value = '■ 주요 이슈';
    const issueRow = row;
    ws.getCell(row+1, 2).value = '타본부협조사항';
    row += 2;

    for (const m of state.meta.members) {
      for (const t of (getMemberData(m.id).tasks || [])) {
        const issues = (t.issues || []).filter(i => i.text);
        if (!issues.length) continue;
        const text = issues.map(i => '- ' + i.text).join('\n');
        bodyCell(ws.getCell(row, 4), t.kpiProduct || '');
        ws.getCell(row, 5).value = text;
        ws.getCell(row, 5).font = BODY_FONT;
        ws.getCell(row, 5).alignment = WRAP;
        ws.getCell(row, 5).border = BORDER_ALL;
        ws.mergeCells(row, 5, row, 8); // E–H 병합
        bodyCell(ws.getCell(row, 9), '');
        row++;
      }
    }

    row += 2;

    // ── 프로젝트 진척도 ──
    sectionLabel(row, '프로젝트 진척도');
    row++;

    hdrCell(ws.getCell(row, 4), '프로젝트');
    hdrCell(ws.getCell(row, 5), '프로젝트');
    hdrCell(ws.getCell(row, 6), '단계');
    hdrCell(ws.getCell(row, 7), '기한');
    hdrCell(ws.getCell(row, 8), '진척%');
    hdrCell(ws.getCell(row, 9), '비고');
    row++;

    for (const m of state.meta.members) {
      for (const t of (getMemberData(m.id).tasks || [])) {
        if (!t.kpiProduct) continue;
        bodyCell(ws.getCell(row, 4), t.kpiProduct);
        bodyCell(ws.getCell(row, 5), t.kpiProduct);
        bodyCell(ws.getCell(row, 6), '');
        bodyCell(ws.getCell(row, 7), t.deadline || '');
        const pctCell = ws.getCell(row, 8);
        pctCell.value = (t.overallProgress || 0) / 100;
        pctCell.numFmt = '0%';
        pctCell.font = BODY_FONT;
        pctCell.border = BORDER_ALL;
        bodyCell(ws.getCell(row, 9), '');
        row++;
      }
    }

    row += 2;

    // ── 제품배포 ──
    sectionLabel(row, '제품배포');
    row++;

    hdrCell(ws.getCell(row, 4), '제품');
    hdrCell(ws.getCell(row, 5), '차수');
    hdrCell(ws.getCell(row, 6), '배포기한');
    hdrCell(ws.getCell(row, 7), '건수(기능/오류)');
    hdrCell(ws.getCell(row, 8), '진척');
    hdrCell(ws.getCell(row, 9), '진척');
    row++;

    for (const m of state.meta.members) {
      for (const dep of (getMemberData(m.id).deployments || [])) {
        if (!dep.product) continue;
        const feat = dep.counts ? (dep.counts.feature || 0) : 0;
        const bug  = dep.counts ? (dep.counts.bug  || 0) : 0;
        bodyCell(ws.getCell(row, 4), dep.product);
        bodyCell(ws.getCell(row, 5), dep.round || '');
        bodyCell(ws.getCell(row, 6), dep.deployDate || '');
        bodyCell(ws.getCell(row, 7), `${feat+bug}건(${feat}/${bug})`);
        bodyCell(ws.getCell(row, 8), dep.progressText || '');
        bodyCell(ws.getCell(row, 9), dep.progressText || '');
        row++;
      }
    }
  }
  ```

- [ ] **Step 2: `exportTeamSummary()` 함수 추가**

  `exportToExcel()` 함수 바로 아래에 추가:

  ```javascript
  async function exportTeamSummary() {
    showToast('팀 보고서 생성 중...');
    try {
      const wb = new ExcelJS.Workbook();
      const sheetName = `${state.meta.month}월${state.meta.week}주차`;
      buildTeamSummarySheet(wb.addWorksheet(sheetName));
      const buffer = await wb.xlsx.writeBuffer();
      const blob = new Blob([buffer], { type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `${state.meta.teamName}_주간보고_${state.meta.month}월_${state.meta.week}주차.xlsx`;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
      showToast('✓ 팀 보고서 내보내기 완료!');
    } catch (err) {
      console.error('Team summary export error:', err);
      showToast('오류: ' + err.message);
    }
  }
  ```

- [ ] **Step 3: "팀 보고서 내보내기" 버튼 추가**

  `renderTeamView()` (line 3908)의 `member-status-bar` 영역에 버튼 추가.

  현재 (line 3918–3921):
  ```javascript
      <div class="member-status-bar">
        <!--<span class="team-done-count...">...</span>-->
        <!--<button ... >취합하기</button>-->
      </div>
  ```

  변경 후:
  ```javascript
      <div class="member-status-bar">
        <!--<span class="team-done-count...">...</span>-->
        <!--<button ... >취합하기</button>-->
        <button class="status-btn team-summary-export-btn" onclick="exportTeamSummary()">📊 팀 보고서 내보내기</button>
      </div>
  ```

- [ ] **Step 4: CSS 추가**

  ```css
  .team-summary-export-btn {
    background: var(--primary); color: white;
    border: none; border-radius: 6px; padding: 5px 12px;
    font-size: 12px; cursor: pointer; font-weight: 600;
  }
  .team-summary-export-btn:hover { opacity: 0.88; }
  ```

- [ ] **Step 5: 브라우저 확인**

  팀현황 → "팀 보고서 내보내기" 버튼 클릭:
  - `팀명_주간보고_N월_N주차.xlsx` 파일이 다운로드됨
  - 파일 열면: 금주실적 / 차주계획 / 주요이슈 / 프로젝트 진척도 / 제품배포 섹션이 각각 데이터로 채워짐
  - 진척도 H열 값이 %로 표시됨 (0.75 → 75%)

- [ ] **Step 6: 커밋**

  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git add index.html && git commit -m "feat: 팀현황 Excel 내보내기 추가 (팀명_주간보고 포맷)"
  ```

---

## Task 4: GitHub 배포

- [ ] **Step 1: 전체 변경 확인**

  ```bash
  cd "D:/work/claude-test/weekly-reporting" && git log --oneline -5 && git status
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

- [ ] 팀현황 테이블 모드에서 필터 바 표시, 카드 모드에서 미표시
- [ ] 팀원 필터 동작 (체크박스)
- [ ] 이슈만 보기 토글 동작
- [ ] 진척율 구간 필터 동작
- [ ] 정렬 변경 시 팀원 순서 변경
- [ ] 초기화 버튼 동작
- [ ] 이번 달 배포 타임라인 표시 (hover tooltip)
- [ ] 팀 보고서 내보내기 Excel 다운로드 및 내용 확인

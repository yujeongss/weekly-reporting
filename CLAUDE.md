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

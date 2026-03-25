# 전략개발팀 주간업무보고 반자동화 — Claude Code 가이드

## 프로젝트 개요

팀원 6명이 매주 Excel로 작성하던 주간업무보고를 웹 UI로 관리하고 Excel로 자동 생성하는 도구.

**핵심 파일:** `index.html` 단일 파일 (빌드 없음, 서버 없음)

---

## 아키텍처

```
index.html (단일 파일, 1884 lines)
├── CSS  (line ~8–825)
├── HTML (line ~826–825)
└── JS   (line ~827–1884)
    ├── CONSTANTS         (line 827)
    ├── STATE             (line 833)
    ├── STATE PERSISTENCE (line 869) — localStorage
    ├── HELPERS           (line 908)
    ├── SIDEBAR           (line 924)
    ├── MAIN RENDER       (line 999)
    ├── TASKS SECTION     (line 1019)
    ├── OPERATIONS        (line 1148)
    ├── DEPLOYMENTS       (line 1180)
    ├── CRUD — TASKS      (line ~1258)
    ├── CRUD — OPERATIONS (line ~1385)
    ├── CRUD — DEPLOYMENTS(line ~1400)
    ├── TEMPLATE BASE64   (line 1519) — 예시.xlsx base64 내장
    ├── EXPORT TO EXCEL   (line 1524)
    ├── EXCEL HELPERS     (line 1567)
    ├── ADD MEMBER        (line 1824) — 팀원 동적 추가
    ├── TOAST             (line 1796)
    ├── KEYBOARD SHORTCUTS(line 1812)
    └── INIT              (line 1872)
```

**외부 라이브러리:** ExcelJS 4.4.0 (CDN) — 셀 스타일링 지원

---

## 데이터 모델

```javascript
localStorage key: 'weekly_report_data_v1'

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
  activeTab: "kim_yujeong",
  members: {
    "<memberId>": {
      tasks: [{
        id, kpiProduct, division, deadline, overallProgress,
        currentWeekItems: [{ id, text, progress }],
        nextWeekItems:    [{ id, text, progress }],
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

**템플릿 기반 방식** (포맷 완벽 보존):
1. `전략개발팀_주간업무보고_예시.xlsx` 를 base64로 HTML에 내장 (`TEMPLATE_B64` const)
2. export 시 ExcelJS로 템플릿 로드 → 각 시트 데이터만 덮어씀
3. `clearSheetData(ws, startRow)`: 기존 병합 해제 + 셀 값 초기화 → 새 데이터 작성
4. 템플릿에 없는 신규 팀원: `setupNewMemberSheet(ws, member)` 로 시트 생성 후 `fillIndividualSheet` 실행

**시트 구성:**
- `전략개발팀` — 팀 취합 (금주실적, clearSheetData row 12+)
- `전략개발팀_통합` — 작성예시 역할 (수정 안 함)
- 팀원 6명 개인 시트 — 각자 데이터로 덮어씀
- 신규 팀원(동적 추가) — `addWorksheet` + `setupNewMemberSheet`로 빈 시트 생성

**개인 시트 레이아웃:**
```
Row 4: B4:G4 merged — "주간보고" (다크 배경, 흰 글씨)
Row 5: B5 "KPI/제품" | C5:E5 "금주활동" | F5:G5 "차주계획" (다크 배경)
Row 6+: 과제 데이터 (B=과제명/기한, C=금주활동, F=차주계획)
Row ?: "운영/요청 사항"
Row ?: B:G merged "제품 배포" (다크 배경)
Row ?+: 3개 배포 블록 (항상 표시, 데이터 없어도)
```

---

## 핵심 함수 목록

| 함수 | 역할 |
|------|------|
| `saveState()` / `loadState()` | localStorage 직렬화/역직렬화 |
| `renderSidebar()` | 좌측 사이드바 (팀원 목록, 보고기간) |
| `renderMain()` | 메인 영역 전체 재렌더 |
| `renderTaskCard(memberId, task)` | 과제 카드 HTML 생성 |
| `renderDeploymentsSection(memberId, data)` | 제품 배포 섹션 |
| `addTask(memberId)` | 과제 추가 + 녹색 배너 + 토스트 + 스크롤 |
| `showAddMemberModal()` | 팀원 추가 모달 (이름 입력, 사업부문 기본값 '기타') |
| `confirmAddMember()` | 팀원 state 추가 + 사이드바 반영 |
| `setupNewMemberSheet(ws, member)` | 신규 팀원 Excel 시트 헤더 세팅 |
| `applyPeriod()` | 보고기간 변경 + 녹색 플래시 피드백 |
| `exportToExcel()` | 템플릿 로드 → 전 시트 채우기 → 다운로드 |
| `clearSheetData(ws, startRow)` | 기존 병합 해제 + 셀 값 초기화 |
| `fillIndividualSheet(ws, memberId)` | 개인 시트 데이터 작성 |
| `fillTeamSheet(ws)` | 팀 시트 데이터 작성 |
| `esc(str)` | XSS 방지 HTML 이스케이프 |
| `showToast(msg)` | 하단 토스트 알림 |

---

## Phase 계획

| Phase | 상태 | 내용 |
|-------|------|------|
| Phase 1 | ✅ 완료 | 개인 탭 CRUD, Excel 내보내기 (예시.xlsx 템플릿), 팀원 동적 추가, 인터랙션 피드백 |
| Phase 2 | 예정 | 팀 시트 자동 취합, 프로젝트 진척도, 이슈/협조 |
| Phase 3 | 예정 | 우측 AI 챗봇 패널 연동 |

---

## 개발 규칙

- **빌드 없음**: CDN만 사용, node_modules 커밋 안 함
- **단일 파일**: 기능 추가 시 index.html 하나만 수정
- **XSS 방지**: 사용자 입력은 반드시 `esc()` 통과 후 innerHTML
- **저장 방식**: 모든 mutation 후 `saveState()` 호출
- **재렌더**: `renderMain()` 호출로 전체 다시 그림 (가상 DOM 없음)
- **Excel 병합셀**: master cell에만 값/스타일 설정, slave cell 직접 접근 금지

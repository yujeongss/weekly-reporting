# 주간업무보고 고도화 설계 문서

**날짜:** 2026-03-25
**프로젝트:** 전략개발팀 주간업무보고 반자동화 도구
**범위:** 팀현황 개선 + UX 심화 + 팀 취합 Excel 내보내기

---

## 개요

단일 파일(`index.html`, 현재 약 5100줄) 웹앱의 3단계 고도화 계획.
빌드 없음, 서버 없음, 외부 라이브러리 신규 추가 없음 원칙 유지.

**현재 상태:**
- localStorage key: `weekly_report_data_v7`
- Excel export: from-scratch 빌더 방식 (`buildMemberSheet`, `buildTeamSheet`)
- 템플릿 base64 방식 미사용 (코드에 `TEMPLATE_B64` 없음)

---

## Phase 1 — 빠른 수정

### 1-0. CLAUDE.md 업데이트

현재 CLAUDE.md가 구버전 기준(1884줄, v1 key, 템플릿 방식)으로 작성되어 있음.
Phase 1 착수 전 CLAUDE.md를 현재 코드 상태에 맞게 갱신:
- 파일 줄 수, localStorage key, export 방식, state 모델(favorites, chatMessages, teamViewMode 포함) 반영

### 1-1. 진척율 편집 불가

**배경:** 사용자 요청. 팀현황 통합 테이블에서 진척율을 수정할 수 없어야 함.
진척율은 개인 탭에서만 입력하고 팀현황은 읽기 전용으로 표시.

**변경:** `renderTeamTableView()`에서 진척율 컬럼의 `<input type="number">` → 순수 텍스트(`%` 표시).
개인 탭에서 수정하면 팀현황에 반영되는 단방향 흐름 유지.

### 1-2. 이슈 강조

이슈가 있는 항목을 시각적으로 구분:
- 통합 테이블: 이슈 있는 행에 좌측 3px 빨간 border + 연한 빨강 배경(`rgba(220,38,38,0.05)`)
- 진척현황 카드: 이슈 있는 과제 수를 "⚠ N개 이슈" 뱃지로 표시
- 즐겨찾기 카드: 이슈 있으면 우측 상단에 빨간 dot(8px 원) 표시

### 1-3. 팀 전체 요약 수치

진척현황 섹션 위에 한 줄 요약 바 추가:

```
전체 과제 N개  |  평균 진척율 N%  |  이슈 N건  |  이번 주 배포 N건
```

- state에서 실시간 계산 (렌더 시마다)
- 각 숫자 클릭 시 스크롤 앵커:
  - "전체 과제" → `#team-summary-section` (진척현황 카드 영역)
  - "평균 진척율" → `#team-summary-section`
  - "이슈" → `#team-table-section` (통합 테이블, 이슈 강조 행으로)
  - "이번 주 배포" → `#team-deploy-section`
- 위 4개 ID를 해당 섹션 wrapper에 부여

---

## Phase 2 — 팀현황 고도화

### 2-1. 필터/정렬

통합 테이블 상단에 필터 바 추가.

**필터 옵션:**
- 팀원 선택 (멀티셀렉트 드롭다운)
- 이슈 있는 항목만 보기 (토글 버튼)
- 진척율 구간: 전체 / 0–30% / 30–70% / 70–100%

**정렬 옵션:**
- 기본(입력순) / 진척율 오름 / 진척율 내림 / 기한 오름 / 기한 내림

**상태 관리:**
- 필터 상태는 모듈 레벨 변수(`_teamFilter`)에만 유지, localStorage 저장 안 함
- 팀원 탭 전환 후 팀현황으로 돌아오면 필터 초기화됨 (의도적 동작)
- 필터 적용 시 `renderTeamTableView()` 재호출로 행만 갱신

### 2-2. 배포 타임라인

팀현황 제품배포 섹션 아래에 가로 타임라인 추가:

- 이번 달 기준 가로 선 위에 배포 날짜를 점으로 표시
- `deployDate` 없는 배포 항목은 타임라인에서 제외
- 이번 달 외 배포: 제외 (현재 월 기준으로만 표시)
- 각 점: 제품명 + 차수 레이블
- hover 시 상세 툴팁 (제품, 차수, 기한, 건수, 진척)
- 지난 배포(오늘 이전): 회색 / 예정 배포(오늘 이후): 파랑
- 라이브러리 없이 순수 CSS + JS (`getBoundingClientRect` 불필요, 퍼센트 포지션으로 계산)
- 날짜 파싱: `new Date(deployDate)` → `(day / daysInMonth) * 100%`

### 2-3. 팀현황 Excel 내보내기 (새 양식)

`팀명_주간보고.xlsx` 포맷으로 현재 주차 데이터를 새 workbook으로 내보내기.

**구현 방식:**
- 현재 `buildMemberSheet` / `buildTeamSheet` 와 동일한 from-scratch 빌더 패턴 사용
- 새 함수 `buildTeamSummarySheet(ws)` 추가
- 팀현황 탭에 "팀 보고서 내보내기" 버튼 별도 추가 (기존 개인용 내보내기와 별개)

**시트 레이아웃 (`팀명_주간보고.xlsx` 참조):**

| 위치 | 내용 | 데이터 소스 |
|------|------|------------|
| B1 | "N월 N주차" | meta.month, week |
| D1 | 팀명 + " 주간 업무 보고" | meta.teamName |
| B4 | "구분" | 고정 |
| B6 | "금주실적" | 고정 |
| D7-I7 | 헤더행 (구분/주요활동/일정) | 고정 |
| D8+ | 금주실적 데이터: D=과제명, E-H=금주활동(merged), I=기한 | tasks → currentWeekItems |
| B23 | "차주계획" | 고정 |
| D25+ | 차주계획 데이터: 동일 구조 | tasks → nextWeekItems |
| B37 | "주요이슈 & 타본부협조사항" | 고정 |
| D38+ | 이슈 내용 | tasks → issues |
| B41 | "프로젝트 진척도" | 고정 |
| D42+ | D=프로젝트명, G=기한, H=진척%(소수: 0.75 = 75%) | tasks → kpiProduct, deadline, overallProgress |
| B49 | "제품배포" | 고정 |
| D50+ | D=제품, E=차수, F=기한, G=건수(기능/오류), H-I=진척 | deployments |

- 전 팀원 데이터를 순서대로 행 추가 (팀원 구분 없이 과제 통합)
- 기존 `buildTeamSheet()`는 현재 팀현황 탭 내보내기용으로 유지 (변경 없음)

---

## Phase 3 — UX 심화

### 3-1. 실행 취소 (Undo)

- 삭제 즉시 처리하지 않고 "삭제됨 — 되돌리기" 토스트를 5초간 표시
- 5초 내 "되돌리기" 클릭 시 복원, 이후 완전 삭제
- 구현: 삭제 시 항목을 state에서 제거 + `saveState()` 대신 타이머 시작, 만료 시 `saveState()`
- 단일 레벨 undo (마지막 삭제만)

### 3-2. 복사/붙여넣기

- 과제 카드 우측 상단에 복사 아이콘(⧉) 추가
- 클릭 시 해당 과제 데이터를 모듈 레벨 변수 `_clipboard`에 **deep clone** 저장
- 붙여넣기 시 `generateId()`로 task id 및 모든 child item id(`currentWeekItems`, `nextWeekItems`, `issues`) 새로 생성 (ID 충돌 방지)
- 다른 팀원 탭 상단에 "붙여넣기 (과제명)" 버튼 표시 (clipboard 있을 때만)
- "이번 주로 이월" 버튼: 해당 과제의 `nextWeekItems` → `currentWeekItems`로 복사, `nextWeekItems` 초기화

### 3-3. 주간 항목 이월

- 보고기간 주차 변경(`applyPeriod()`) 시 네이티브 `confirm()` 다이얼로그:
  `"차주 계획을 금주 활동으로 이월하시겠습니까?"`
- 확인 시: 전 팀원의 `nextWeekItems` → `currentWeekItems` 복사, `nextWeekItems` 초기화 후 `saveState()`
- 취소 시: 이월 없이 주차만 변경
- `applyPeriod()` 함수 내 기존 저장/렌더 로직 앞에 confirm 분기 삽입 (async 변환 불필요)

### 3-4. 드래그앤드롭 순서변경

- 과제 카드 간 드래그앤드롭으로 `tasks` 배열 순서 변경
- HTML5 Drag and Drop API 사용 (라이브러리 없음)
- 카드 좌측에 드래그 핸들(⠿) 아이콘 표시
- **제약:** HTML5 DnD는 iOS Safari / Android Chrome 터치에서 미지원 — 터치 사용자는 이 기능 미작동 (알려진 한계로 문서화)

### 3-5. 진척율 입력 개선

- 과제 카드의 `overallProgress` 입력: `<input type="number">` → `<select>` (0 / 25 / 50 / 75 / 100%)
- 또는 숫자 입력 + 우측 슬라이더 병행 제공 (구현 시 단순한 쪽 선택)

### 3-6. 검색

- 헤더 우측에 검색 아이콘(🔍), 클릭 시 검색 바 펼쳐짐
- 검색 대상: 과제명(`kpiProduct`), 금주활동(`currentWeekItems.text`), 차주계획(`nextWeekItems.text`)
- 결과: 매칭된 팀원 탭으로 전환 + 해당 카드에 노란 하이라이트 테두리
- 여러 팀원에 매칭 시 결과 드롭다운으로 선택

### 3-7. 반응형

- 사이드바에 접기/펼치기 토글 버튼 추가 (`☰`)
- 화면 너비 768px 미만 시 사이드바 자동 숨김 (CSS media query)
- 통합 테이블: 좁은 화면에서 `overflow-x: auto` + 최소 너비 유지

### 3-8. 온보딩

- 최초 접속(팀 이름 입력 완료) 시 툴팁 투어 시작 (4단계)
  1. 팀원 탭 선택 → 2. 과제 추가 버튼 → 3. 팀현황 탭 → 4. Excel 내보내기 버튼
- 툴팁 위치: 대상 요소의 `getBoundingClientRect()`로 절대좌표 계산 후 `position: fixed` 오버레이
- `localStorage`에 `weekly_report_onboarded` 키 저장 → 재방문 시 미표시
- 헤더 "?" 버튼으로 언제든 재실행 가능

### 3-9. 주차별 비교

**데이터 모델:**
- 별도 localStorage key `weekly_report_snapshots` 사용 (메인 state key와 분리)
- 구조: `Array<{ weekKey: string, timestamp: number, members: {...} }>` (최근 8주 보관)
- `weekKey` 형식: `"YYYY-M-W"` (예: `"2026-3-2"`)

**스냅샷 저장 시점:** `applyPeriod()`에서 주차가 변경될 때 현재 state.members를 스냅샷으로 저장.

**보관 정책:** 최신 8주만 유지, 초과 시 가장 오래된 항목 제거.

**마이그레이션:** 별도 key 사용이므로 기존 `v7` state와 충돌 없음. 첫 로드 시 key 없으면 빈 배열로 초기화.

**표시:** 팀현황 진척율 옆에 이전 주 대비 변화량 표시 (▲+5%, ▼-10%, 데이터 없으면 미표시).

---

## 기술 제약

- 단일 파일(`index.html`) 유지
- 빌드 도구 없음, CDN만 허용
- 외부 라이브러리 신규 추가 없음 (ExcelJS 4.4.0 기존 사용 유지)
- XSS 방지: 사용자 입력은 반드시 `esc()` 통과
- 모든 변경 후 `saveState()` 호출

---

## 구현 순서 요약

| Phase | 항목 수 | 핵심 변경 |
|-------|---------|----------|
| Phase 1 | 4개 | CLAUDE.md 갱신, 진척율 읽기전용, 이슈 강조, 팀 요약 수치 |
| Phase 2 | 3개 | 필터/정렬, 배포 타임라인, 팀 취합 Excel (from-scratch 빌더) |
| Phase 3 | 9개 | Undo, 복사, 이월, 드래그앤드롭, 검색, 반응형, 온보딩, 주차비교 |

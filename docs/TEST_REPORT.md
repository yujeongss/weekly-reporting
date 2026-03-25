# Test Report — 주간업무보고 반자동화

## 실행 환경

- Node.js + ExcelJS 4.4.0 (CDN과 동일 버전)
- 테스트 대상: `index.html` JS 로직 + Excel export 파이프라인

---

## 테스트 결과 (2026-03-17) — Phase 1 v1

| # | 테스트 | 결과 | 상세 |
|---|--------|------|------|
| 1 | Template loads with 8 sheets | ✅ PASS | 전략개발팀, 작성예시, 조낙운, 조민희, 이성대, 김유정, 정재영, 오현민 |
| 2 | clearSheetData unmerges data rows (>=6) | ✅ PASS | before:7 / unmerged:4 / after:3 |
| 3 | Headers row 4-5 preserved after clear | ✅ PASS | B4=주간보고, B5=KPI/제품 |
| 4 | Data cells cleared (B8 null) | ✅ PASS | B8=null |
| 5 | Output buffer generated | ✅ PASS | 35,616 bytes |
| 6 | Output re-loadable, data correct | ✅ PASS | B6=테스트 과제 |
| 7 | index.html JS syntax valid | ✅ PASS | 문법 오류 없음 |

**7 / 7 통과**

---

## 테스트 결과 (2026-03-17) — 예시.xlsx 템플릿 교체 + 팀원 추가 기능

| # | 테스트 | 결과 | 상세 |
|---|--------|------|------|
| 1 | 예시.xlsx 템플릿 로드 (8 sheets) | ✅ PASS | 전략개발팀, 전략개발팀_통합, 조낙운, 조민희, 이성대, 김유정, 정재영, 오현민 |
| 2 | clearSheetData (김유정 row 6+) | ✅ PASS | before:7 merges / after:3 / B6=null |
| 3 | 헤더 보존 (B4=주간보고) | ✅ PASS | 기존 merge 유지 |
| 4 | setupNewMemberSheet 신규 시트 생성 | ✅ PASS | B4=주간보고, B6=테스트과제 |
| 5 | 신규 시트 포함 버퍼 생성 | ✅ PASS | 36,287 bytes |
| 6 | 신규 시트 재로드 확인 | ✅ PASS | 테스트팀원 시트 포함 |
| 7 | JS syntax valid (1884 lines) | ✅ PASS | 문법 오류 없음 |

**7 / 7 통과**

---

## 디버그 히스토리

### 버그 1: `Identifier 'TEMPLATE_B64' has already been declared`

**증상:** index.html 열면 "로딩 중..." 에서 멈춤. 아무것도 렌더되지 않음.

**원인:** `update_html.js`가 두 번 실행되면서 `const TEMPLATE_B64`가 중복 선언됨.
- 1차 실행: TEMPLATE_B64 + exportToExcel + Excel helpers 전체 교체
- 2차 실행: exportToExcel 마커를 찾아 다시 교체하면서 TEMPLATE_B64를 새 섹션 앞에 또 삽입

```
[1차 실행 결과]  line 1398: const TEMPLATE_B64 = '...'   ← 1st
                 line 1401: // EXPORT TO EXCEL
                 line 1403: async function exportToExcel() {
[2차 실행 결과]  line 1405: const TEMPLATE_B64 = '...'   ← 2nd (중복!)
                 line 1408: async function exportToExcel() {
```

**진단:** `node -e "new Function(jsCode)"` → `Identifier 'TEMPLATE_B64' has already been declared`

**수정:** `fix_dup.js` - EXCEL EXPORT 헤더 이후부터 2번째 TEMPLATE_B64 섹션 시작점까지의 중복 블록 제거

```javascript
// fix_dup.js 핵심 로직
const keepBefore = html.substring(0, excelExportStart + excelExportHeader.length);
const keepAfter  = '\n' + html.substring(secondSectionStart);
```

**검증:** `new Function(js)` → syntax OK + 7/7 테스트 통과

---

### 버그 2: ExcelJS xborder() 병합셀 slave cell 접근 문제

**증상:** Excel export 시 전략개발팀 시트 표 깨짐. 개인 시트 헤더 배경색 없음.

**원인:**
- `xborder()` 함수가 병합 범위 내 slave cell에 `.border` 설정 시도 → silent fail
- `HEADER_FILL`이 `{argb: 'FF1E293B'}` 등 임의 색상 → 원본과 불일치

**수정:**
1. `xborder()` 함수 완전 제거
2. merged cell: master cell에만 값/스타일 설정
3. HEADER_FILL 값을 원본 xlsx 분석으로 확인한 정확한 theme 값으로 변경:
   ```javascript
   const HEADER_FILL = {
     type: 'pattern', pattern: 'solid',
     fgColor: { theme: 0, tint: -0.1499984740745262 },
     bgColor: { indexed: 64 }
   };
   ```

---

### 버그 3: spliceRows() 로드된 파일에서 동작 안 함

**증상:** 템플릿 기반 방식에서 기존 데이터가 지워지지 않고 새 데이터와 혼재됨.

**원인:** `ws.spliceRows(6, ws.rowCount)` 호출해도 기존 셀 값이 유지됨.
ExcelJS의 spliceRows는 새로 생성한 worksheet에서만 정상 동작.

**수정:** `clearSheetData(ws, startRow)` 패턴:
```javascript
function clearSheetData(ws, startRow) {
  // 1. 기존 병합 해제 (data 영역만)
  const merges = ws._merges || {};
  for (const [key, merge] of Object.entries(merges)) {
    if (merge?.model?.top >= startRow) {
      const col = n => String.fromCharCode(64 + n);
      try {
        ws.unMergeCells(`${col(merge.model.left)}${merge.model.top}:${col(merge.model.right)}${merge.model.bottom}`);
      } catch(e) {}
    }
  }
  // 2. 셀 값 초기화
  for (let r = startRow; r <= ws.rowCount; r++) {
    ws.getRow(r).eachCell({ includeEmpty: true }, cell => { cell.value = null; });
  }
}
```

---

## TDD 체크리스트 (Phase 2 작업 전)

Phase 2 시작 전 아래 항목을 node 스크립트로 검증해야 함:

- [ ] `clearSheetData(ws, 12)` — 전략개발팀 시트 row 12+ 초기화
- [ ] 팀원별 금주실적 취합 로직 — 사업부문 그룹핑 정확성
- [ ] 차주계획 취합 — 동일 구조
- [ ] 빈 tasks 처리 — 과제 없는 팀원 skip
- [ ] 병합셀 — `D:D`, `E:E`, `F:J`, `K:K` 각 task row에서 올바른 범위

---

## 테스트 실행 방법

```bash
cd D:/work/claude-test/weekly-reporting
npm install exceljs  # 최초 1회 (CDN과 동일 버전 테스트용)
node -e "
  const ExcelJS = require('./node_modules/exceljs');
  // ... 테스트 코드
"
```

브라우저 테스트:
1. `index.html` 파일을 Chrome/Edge에서 열기
2. DevTools Console (F12) 에러 없는지 확인
3. 팀원 탭 전환 동작 확인
4. 과제 추가 → 황색 플래시 + 스크롤 확인
5. Excel 내보내기 → xlsx 다운로드 → 원본과 포맷 비교

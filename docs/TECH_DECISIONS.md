# 기술 결정 로그

프로젝트 진행 중 내린 주요 기술적 결정 사항을 기록합니다.

---

## 1. 단일 HTML 파일 아키텍처

**결정**: 모든 CSS, HTML, JavaScript를 `index.html` 하나에 인라인으로 작성.

**이유**
- 별도 빌드 시스템(webpack, vite 등) 불필요
- 서버 없이 `file://` 프로토콜로 직접 실행 가능
- 팀원에게 파일 하나만 전달하면 즉시 사용 가능
- 의존성 설치 과정 없음 (node_modules 불필요)

**트레이드오프**
- 파일이 길어질수록 유지보수 난이도 증가
- 코드 분할, 트리쉐이킹 등 최적화 불가

---

## 2. ExcelJS vs SheetJS

**결정**: ExcelJS 4.4.0 선택.

**비교**

| 항목 | ExcelJS | SheetJS (community) |
|------|---------|---------------------|
| 셀 스타일링 (폰트/색상/테두리) | 지원 | 미지원 |
| 셀 병합 | 지원 | 지원 |
| 기존 xlsx 로드 후 수정 | 지원 | 지원 |
| CDN 제공 | 지원 | 지원 |
| 라이선스 | MIT | Apache 2.0 |

**결정 이유**

기존 보고서 양식은 헤더 배경색, 폰트 볼드, 병합 셀 등 복잡한 스타일을 포함합니다. SheetJS community 버전은 셀 스타일 읽기/쓰기를 지원하지 않아, 원본 포맷을 재현하려면 ExcelJS가 필수였습니다.

---

## 3. 데이터 저장: localStorage

**결정**: 별도 서버나 파일 시스템 없이 브라우저 localStorage 사용.

**이유**
- 서버 구축 및 운영 비용 없음
- 팀원 각자의 브라우저에서 독립적으로 데이터 관리
- 6명 팀 데이터 예상 용량 약 50KB로 localStorage 5MB 한도 내 충분
- 구현 단순성 (서버 API, 인증 등 불필요)

**키 이름**: `weekly_report_data_v1`

**주의 사항**
- 브라우저 캐시/사이트 데이터 삭제 시 데이터 소실
- 다른 기기 또는 다른 브라우저에서는 데이터 공유 불가
- 중요 데이터는 주기적으로 Excel 내보내기 후 별도 보관 권장

---

## 4. 템플릿 기반 Excel export 방식

**결정**: from-scratch 방식 포기, 원본 xlsx를 base64로 내장 후 로드→덮어쓰기 방식 채택.

**경위**

초기에는 ExcelJS로 빈 워크북에서 모든 셀을 직접 생성하는 from-scratch 방식을 시도했습니다. 그러나 OOXML 표준의 다양한 속성(테마 색상, 조건부 서식, 인쇄 설정 등)을 수동으로 재현하는 것이 현실적으로 불가능하여 생성된 파일의 포맷이 원본과 달랐습니다.

**채택 방식**

1. 원본 `.xlsx` 파일을 base64 문자열로 인코딩하여 `index.html`에 직접 내장
2. Excel export 시 base64를 ArrayBuffer로 디코딩하여 ExcelJS로 로드
3. 기존 데이터/병합을 `clearSheetData()`로 초기화
4. 현재 state 데이터를 정해진 셀 위치에 덮어쓰기
5. 완성된 워크북을 Blob으로 변환하여 다운로드

**장점**: 원본 양식의 모든 스타일, 인쇄 설정, 시트 구조가 완벽하게 보존됨.

---

## 5. ExcelJS 병합 셀(merged cell) 규칙

**결정**: 병합된 영역의 마스터 셀(좌상단)에만 값/스타일을 설정.

**문제**

ExcelJS에서 병합된 셀 영역의 슬레이브 셀(마스터 외 나머지 셀)에 값이나 스타일을 설정하면 오류 없이 **silent fail**이 발생합니다. 설정한 값이 반영되지 않으며 디버깅이 어렵습니다.

**규칙**

```js
// 잘못된 예: B3:D3 병합 시 슬레이브 셀에 접근
ws.getCell('C3').value = '데이터';  // silent fail

// 올바른 예: 마스터 셀(좌상단)에만 접근
ws.getCell('B3').value = '데이터';  // 정상 동작
```

병합 범위를 수정하려면 반드시 기존 병합을 해제(`unMergeCells`)한 후 재병합해야 합니다.

---

## 6. spliceRows() 동작 불가 문제

**결정**: 기존 파일 로드 후 행 삽입/삭제에 `spliceRows()` 사용 금지. `clearSheetData(ws, startRow)` 패턴으로 대체.

**문제**

ExcelJS의 `spliceRows()`는 새로 생성한 워크북에서는 동작하지만, **기존 xlsx 파일을 로드한 워크시트에서는 효과가 없습니다.** 행이 추가되거나 삭제되지 않고 무시됩니다.

**대체 패턴: clearSheetData()**

```js
function clearSheetData(ws, startRow) {
  // startRow부터 마지막 행까지 셀 값을 모두 null로 초기화
  for (let r = startRow; r <= ws.rowCount; r++) {
    ws.getRow(r).eachCell({ includeEmpty: true }, (cell) => {
      cell.value = null;
    });
  }
}
```

데이터 영역을 먼저 전부 초기화한 후, 현재 state 데이터를 처음 행부터 순서대로 덮어씁니다. 행 수가 가변적인 경우 최대 예상 행 수까지 미리 초기화합니다.

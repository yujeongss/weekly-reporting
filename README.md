# 주간업무보고 반자동화 도구

팀원들이 브라우저에서 주간업무보고를 작성하고 Excel로 내보내는 단일 파일 웹 도구입니다.
서버 없이 `index.html` 하나만으로 동작합니다.

---

## 빠른 시작

### 1. 저장소 클론

```bash
git clone https://github.com/yujeongss/weekly-reporting.git
cd weekly-reporting
```

### 2. `index.html`을 브라우저에서 열기

```bash
# 파일을 더블클릭하거나 브라우저에 드래그&드롭
# 또는 VS Code Live Server, Python 간단 서버 사용
python -m http.server 3000
```

### 3. 랜딩 페이지에서 팀 이름 입력 후 시작

---

## AI 어시스턴트 사용 (선택)

우측 하단 🤖 버튼으로 Claude AI 어시스턴트를 사용할 수 있습니다.
사용하려면 Anthropic API 키가 필요합니다.

**설정 방법:**

1. [Anthropic Console](https://console.anthropic.com/settings/keys)에서 API 키 발급
2. `config.example.js`를 복사해서 `config.js`로 이름 변경
3. `config.js` 안의 API 키 값을 본인 키로 교체
4. `index.html`과 같은 폴더에 `config.js` 저장

```bash
cp config.example.js config.js
# config.js 파일을 열어 API 키 입력
```

> `config.js`는 `.gitignore`에 포함되어 있어 git에 커밋되지 않습니다.

---

## 주요 기능

| 기능 | 설명 |
|------|------|
| 팀원별 탭 | 팀원별 독립 탭에서 과제 / 운영 / 배포 CRUD |
| 팀 현황 | 전체 팀원 진척율 · 즐겨찾기 카드 · 통합 테이블 |
| Excel 내보내기 | 기존 양식 템플릿에 데이터 자동 채워 `.xlsx` 생성 |
| 즐겨찾기 | KPI/제품 · 배포 항목 별표 북마크 및 요약 카드 |
| AI 어시스턴트 | Claude API 연동, 자연어로 데이터 입력/수정 |
| 자동 저장 | 모든 변경사항 브라우저 localStorage에 실시간 저장 |

---

## 프로젝트 구조

```
weekly-reporting/
├── index.html            # 전체 애플리케이션 (CSS + HTML + JS)
├── img/
│   └── 0.GCMediAI.png    # 로고 이미지
├── config.example.js     # API 키 설정 예시 파일
├── config.js             # API 키 실제 설정 (gitignore, 직접 생성)
├── api/
│   └── claude.js         # Vercel 서버리스 프록시 (배포용)
└── docs/                 # 개발 문서
```

---

## 기술 스택

- **Vanilla JavaScript** — 프레임워크 없음
- **ExcelJS 4.4.0** (CDN) — Excel 생성
- **localStorage** — 데이터 저장
- **Claude API** — AI 어시스턴트 (선택)

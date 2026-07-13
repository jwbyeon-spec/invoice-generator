# L10N 프리랜서 관리 (Localization Freelancer Manager)

게임 로컬라이제이션(번역·검수·LQA) **프리랜서 관리 웹앱**입니다. 번역 요청 발송, 정산(거래명세서), 계약서 생성, 슬랙 커뮤니케이션, AI 번역 검수를 한 페이지에서 처리합니다.

> 이 문서는 **처음 보는 AI/개발자가 README만 읽고 프로젝트 전체를 이해**할 수 있도록 작성됐습니다. 기능이 추가/변경될 때마다 이 README도 함께 갱신하는 것을 원칙으로 합니다(맨 아래 [README 유지 기준](#readme-유지-기준) 참고).

---

## 프로젝트 개요

- **한 명의 관리자(변지우 / jwbyeon@cookapps.com)**가 여러 프리랜서에게 게임 번역 작업을 요청하고, 정산·계약·검수를 관리하는 **사내 어드민 도구**입니다.
- **정적 단일 파일(`index.html`)**로 구현되어 **GitHub Pages**로 배포됩니다. 별도 서버·빌드가 없습니다.
- 데이터 저장/연동은 외부 서비스로 위임합니다: **Supabase**(요청 현황 DB), **Slack API**(메시지), **Google Apps Script**(관리시트 연동·TM 갱신·AI 검수), **Claude API**(AI 검수).
- 민감정보(주민번호·계좌 등)는 **브라우저 메모리에서만** 다루며, 클라우드/localStorage에는 저장하지 않습니다.

---

## 기술 스택

| 구분 | 사용 기술 | 비고 |
|---|---|---|
| 프론트엔드 | **Vanilla JS + HTML/CSS 단일 파일** (`index.html`) | 프레임워크·번들러 없음 |
| 엑셀 파싱 | **SheetJS (XLSX)** | 관리시트(엑셀)에서 프리랜서/작업 데이터 로드 |
| PDF 생성 | **html2canvas + jsPDF** | 거래명세서·계약서를 이미지 캡처 → 다페이지 A4 PDF |
| 클라우드 DB | **Supabase (REST, publishable key + RLS)** | `translation_requests` 테이블 (요청 현황) |
| 메시징 | **Slack Web API** (form-urlencoded, user OAuth token) | CORS 회피 위해 form 본문 사용, `<@U...>` 자동 태그 |
| 서버리스 백엔드 | **Google Apps Script 웹앱** | 관리시트 fetch, TM 갱신, AI 검수 (도메인 제한 배포) |
| AI 검수 | **Claude API (`claude-haiku-4-5`)** | Apps Script `UrlFetchApp`로 호출, 키는 Script Properties |
| 호스팅 | **GitHub Pages** | repo `jwbyeon-spec/invoice-generator`, 브랜치 `main` |
| 캐시 | **localStorage** | 민감정보 제외(sanitized)본만 저장 |

---

## 폴더 구조

```
거래명세서_자동생성/               # git 저장소 루트 (= GitHub Pages 소스)
├─ index.html                    # 앱 전체 (마크업 + CSS + JS 단일 파일, ~3,100줄)
├─ README.md                     # 이 문서
├─ .gitignore                    # *.xlsx/*.csv 등 민감·임시 파일 제외
└─ .claude/
   └─ launch.json                # 로컬 미리보기 설정 (python http.server 3000)
```

- **저장소에는 `index.html` 하나만 배포**됩니다. 엑셀/CSV 등 데이터 파일은 `.gitignore`로 커밋을 막습니다(민감정보 유출 방지).
- **Google Apps Script 코드는 이 저장소에 없습니다.** 별도의 스탠드얼론 Apps Script 프로젝트(웹앱)로 관리되며, `index.html`의 `TM_WEBAPP_URL` 상수로 호출합니다. (아래 [Apps Script 백엔드](#apps-script-백엔드-별도-프로젝트) 참고)

### `index.html` 내부 구성(논리적)

- **탭 뷰**: 대시보드 / 상세 내역 / 번역 요청 / AI 검수 / TM 갱신 / 전체 메세지 발송 / 계약서 생성
- **핵심 상수**: `SUPA_URL`·`SUPA_KEY`(Supabase), `TM_WEBAPP_URL`(Apps Script), `SHEET_TO_INFO`(프리랜서 로스터 매핑), `COMPANY`(갑 회사 정보), `TM_LINKS`(게임별 TM 시트)
- **데이터 로드**: `loadWorkbookFromArrayBuffer()` → `loadListInfo()`가 `g_listInfo`(프리랜서 명단) 채움. 월 선택은 **거래명세서 집계(`g_results`)에만** 필요, 그 외 탭은 로드만으로 동작.

### Apps Script 백엔드 (별도 프로젝트)

`doGet(e)`의 `action` 파라미터로 분기하는 단일 웹앱:

| `action` | 기능 | 반환 |
|---|---|---|
| `updateTM` | 개발팀 시트 → 프리랜서 시트 TM 갱신 | HTML |
| `getMgmt` | 관리 구글시트를 xlsx(base64)로 반환 | JSONP |
| `sheetCols` | 작업 시트 헤더(도착어 열 목록) 반환 | JSONP |
| `review` | 시트 특정 도착어 열을 Claude로 검수 → 결과 | JSONP |

- 배포 형태: **cookapps.com 도메인 제한** + `assertAdmin_()`(jwbyeon@cookapps.com만 허용).
- Claude API 키는 **Script Properties `CLAUDE_API_KEY`**에 저장(코드/깃 노출 없음).

---

## 구현 완료 기능

### 대시보드
- 월별 프리랜서 카드: 언어·상태 칩, **실지급액(net pay)** 강조 표시
- **클릭 복사**: 프리랜서 이름 클릭 → 이름 복사 / 실지급액 클릭 → 금액 복사
- 요약 카드: 총 프리랜서·총 작업건수·**총 실지급액(KRW/USD)** (금액 클릭 복사)
- 확인/확인취소, 체크박스 선택 → **선택자 거래명세서 PDF 생성**(html2canvas+jsPDF), 전체 슬랙 전송

### 정산 / 거래명세서
- 원천세 분리 계산: **소득세 `ROUNDDOWN(지급액×3%, -1)` + 지방소득세 `ROUNDDOWN(소득세×10%, -1)`**
- USD 프리랜서는 공제 없음(소수점 처리만)
- 거래명세서는 **대시보드에서** 생성 (별도 탭 없음)

### 번역 요청
- 게임명·품명·마감·단어수(신규/중복)·작업시트 링크·특이사항 입력
- 프리랜서 개별 선택 / 전체 발송, **슬랙 자동 태그** 발송
- 발송 시 **Supabase `translation_requests`에 기록**(work_sheet 포함)
- **요청 현황 보드**: 요청 묶음별 구분선, 작업시트 `바로가기` 링크, **완료(🎉)행에 `AI 검수` 버튼**

### AI 검수
- 작업 시트 링크 입력(또는 요청 현황에서 진입) → **도착어 열 선택** → Claude(Haiku) 검수
- 검수 항목(우선순위): ① 숫자·변수·태그 보존 ② 미번역 ③ 오번역 (톤·용어일관성 제외)
- 결과: **Token 기준** 표(원문/번역/AI 제안) + 유형별 요약 카드
- 헤더가 약자(EN/TC/JP…)든 언어명이든 열을 직접 골라 대응 (매핑: KR/EN/RU/TC/SC/JP/TH/PT/FR/ID)

### 계약서 생성
- 프리랜서 선택 → **을(乙) 정보(성명·주민번호/여권·주소·연락처·계좌)와 단가 3종 자동 채움**
- 계약기간·계약일·계약명 입력, 단가 수정 가능
- **㈜쿡앱스 프리랜서 용역 계약서 전문**(제1~15조 + 주요 계약내용 표 + 갑/을 서명란) 재현, 담당자 변지우 고정
- 미리보기 + **`💾 PDF 저장`** (다페이지 A4 분할)

### TM 갱신
- 게임별 버튼 → Apps Script 웹앱이 개발팀 시트 → 프리랜서 공유 시트로 갱신(프리랜서 링크 유지)

### 전체 메세지 발송
- 엑셀 등록 프리랜서에게 자유 메시지 슬랙 발송(자동 태그), 개별/전체 선택

### 데이터 연동 / 보안
- **구글시트에서 불러오기**: Apps Script JSONP(`getMgmt`)로 관리시트를 xlsx로 받아 로드 (엑셀 업로드 대안)
- 비밀번호 잠금 화면(클라이언트 사이드, 관리자 전용)
- 민감정보는 브라우저 메모리에서만 사용 — `stripSensitiveSheet()`로 localStorage 캐시에서 제거, `safeInfo()`로 Supabase 전송에서 제외
- Apps Script 도메인 제한 + `assertAdmin_()` 이메일 검증

---

## 현재 진행 중인 기능

### AI 검수 백엔드 연결 (프론트 완료 / 백엔드 설정 남음)
- 프론트(탭·열 선택·결과 표)는 완료·배포됨.
- 남은 작업:
  1. Apps Script에 `sheetCols` / `review` 액션 코드 추가 후 **재배포**
  2. **Claude API 키를 Script Properties `CLAUDE_API_KEY`에 등록**
  3. `UrlFetchApp` 외부요청 권한 승인
- 키 등록 전까지는 `검수` 버튼이 "열 정보 실패"로 표시됨(정상).

---

## 향후 개발 예정 기능

- **AI 검수 정확도 옵션**: 오탐/미탐이 신경 쓰이면 모델을 `claude-haiku-4-5` → `claude-sonnet-5`로 상향(코드 한 줄). 1차 Haiku 전체 → 의심 행만 2차 Sonnet 재검수하는 하이브리드도 후보.
- **요청 종료 처리(worklist)**: `translation_requests.closed_at` 컬럼·권한은 준비됨(현재 미사용). 완료 후 "종료 처리"로 목록에서 제거하는 UI 재도입 검토.
- **이모지 트리거 자동 검수**: 프리랜서가 완료 이모지(🎉)를 누르면 해당 언어만 자동 검수(현재는 관리자 버튼 방식).
- **계약서 양식 다양화**: NDA 등 추가 양식.

---

## 프로젝트 실행 방법

### 로컬 미리보기
```bash
# 저장소 루트에서
python -m http.server 3000
# 브라우저에서 http://localhost:3000 접속
```
- `.claude/launch.json`에 동일 설정이 있어 지원 도구에서 바로 실행 가능.
- 정적 파일이라 서버 없이 `index.html`을 브라우저로 열어도 대부분 동작(단, 일부 외부 연동은 https/도메인 필요).

### 실제 사용
1. **GitHub Pages URL** 접속 (`https://jwbyeon-spec.github.io/invoice-generator/`)
2. 관리자 비밀번호로 잠금 해제(비밀번호는 코드 내 상수)
3. 상단에서 **`📥 구글시트에서 불러오기`** 또는 **엑셀 업로드**로 데이터 로드
   - 거래명세서를 뽑을 때만 **월 선택 + 불러오기** 필요. 계약서·번역요청·검수·발송 탭은 로드만으로 동작.

### 배포 (⚠️ 매우 중요 — [주의사항](#주의사항) 참고)
```bash
git add index.html
git commit -m "..."
git push origin main
# push 후 반드시 라이브 반영 검증(아래 주의사항)
```

---

## 주의사항

1. **저장소는 반드시 Public 유지** — GitHub Pages 무료 플랜 조건. Private로 바꾸면 Pages가 꺼짐(Enterprise 필요).
2. **배포 = commit + push, 그리고 "라이브 검증"까지가 배포다.** push 성공 ≠ 반영 완료.
   - 반영 확인: `curl -s https://jwbyeon-spec.github.io/invoice-generator/index.html | grep -c "<추가한 마커>"`
   - **Pages 빌드가 누락/지연**되는 사고가 있었음. raw(`raw.githubusercontent.com/.../main/index.html`)엔 있는데 Pages엔 없으면 **빈 커밋으로 재빌드**: `git commit --allow-empty -m "재빌드" && git push`
   - 반영 후 브라우저는 **`Ctrl+Shift+R`**(강력 새로고침)로 캐시 무효화.
3. **민감정보는 절대 커밋/클라우드 저장 금지** — 주민번호·계좌·연락처·주소는 브라우저 메모리에서만. `.gitignore`가 `*.xlsx/*.csv`를 막고, `stripSensitiveSheet()`/`safeInfo()`가 캐시·DB 전송에서 제외.
4. **비밀 키를 코드/깃에 넣지 말 것**:
   - Slack 토큰 → 페이지에서 관리자가 입력(코드에 하드코딩 X)
   - Claude API 키 → Apps Script **Script Properties `CLAUDE_API_KEY`** (코드/깃 노출 X)
   - Supabase는 **publishable key만** 사용(RLS로 보호). service_role 키는 절대 프론트에 두지 않음.
5. **Apps Script 웹앱은 cookapps.com 도메인 제한 + `assertAdmin_()`** — 관리자(jwbyeon@cookapps.com)만 관리시트/검수 호출 가능. 재배포 시 **"배포 관리 → 편집 → 새 버전"**으로 해야 URL이 유지됨(새 배포는 URL이 바뀜).
6. **세션 환경정보의 `Is a git repository: false`는 오탐일 수 있음** — 실제로는 git 저장소이니 `git status`로 확인.
7. **프리랜서 명단 드롭다운(계약서 탭)**은 관리시트 `List` 탭의 **`이름` 또는 `영문 이름`** 칸이 채워진 사람만 표시. 안 보이면 데이터를 다시 불러오거나 해당 칸 확인.
8. **프리랜서 로스터는 시트에서 자동 생성** — 번역 요청 대상 목록·대시보드·이모지 워크플로는 `SHEET_TO_INFO` 맵(`탭이름 → {영문명, 언어}`) 기준인데, 이 맵은 **관리시트 `List` 탭의 `탭 이름` 열에서 자동 구성**됨(하드코딩 `ROSTER_DEFAULT`와 병합). **프리랜서가 늘면 `List`에 행을 추가하고 `탭 이름`(작업 탭 이름 그대로, 예 `Anthony(EN)`)·`이름`(=영문명)·`슬랙 채널`을 채우면 코드 수정 없이 자동 반영**. `탭 이름`이 비면 하드코딩 기본값만 적용됨. `영문명`은 `List`의 이름 칸 값과 일치해야 `g_listInfo`에서 매칭. 언어는 `언어` 열 또는 탭이름 접미사 코드(EN/TH/TW…)로 추론.

---

## TODO

- [ ] **AI 검수 백엔드 연결**: Apps Script `sheetCols`/`review` 배포 + `CLAUDE_API_KEY` 등록 + 외부요청 권한 승인
- [ ] AI 검수 실사용 후 **오탐/미탐 점검** → 필요 시 모델 상향(Sonnet) 또는 하이브리드 검수
- [ ] 요청 **종료 처리(worklist)** UI 도입 여부 결정 (`closed_at` 활용)
- [ ] 이모지 트리거 **자동 검수** 검토
- [ ] 계약서 **추가 양식**(NDA 등)
- [ ] (선택) Apps Script 코드도 버전관리(clasp) 편입 검토 — 현재 저장소 밖

---

## README 유지 기준

기능을 추가/변경/삭제할 때 **이 README를 같은 커밋 흐름에서 함께 갱신**합니다.

- **새 기능 추가** → `구현 완료 기능`에 항목 추가, 관련 시 `폴더 구조`/`기술 스택` 갱신, `TODO`/`향후 개발 예정`에서 해당 항목 제거·이동
- **작업 중 기능** → `현재 진행 중인 기능`에 남은 단계 명시, 완료되면 `구현 완료`로 이동
- **주의점/함정 발견** → `주의사항`에 반영(특히 배포·보안·데이터)
- **상수/URL/키 위치 변경** → `폴더 구조` 및 `주의사항`의 관련 서술 갱신
- 원칙: **이 README만 읽고도 현재 상태와 다음 할 일을 알 수 있게** 유지.

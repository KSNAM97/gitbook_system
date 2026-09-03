# HTML - 통합 정리

> **Tag:** #HTML #통합정리 #요약
> **핵심 요약:** HTML 파일 1~7의 전체 내용을 태그 종류·구조·입력까지 흐름 순서대로 연결한 통합 요약 문서이다.

> 파일 1~7의 전체 내용을 흐름 순서대로 연결한 요약 문서

---

## 1. 핵심 기술 개념 (Concept)

## 1. 전체 흐름도

```
HTML 기초 (파일 1)
  └─ HTML = HyperText + Markup + Language
  └─ 렌더링 = 브라우저가 HTML 코드를 화면으로 변환
  └─ 태그 = 시작 태그 + 콘텐츠 + 닫는 태그
  └─ 단일 태그 = <br />, <img />, <meta />, <input />
  └─ 속성 = 태그 내부에 key="value" 형태로 작성
  └─ 기본 구조: DOCTYPE → html → head → body

텍스트 표시 (파일 2)
  └─ 제목: h1 ~ h6 (블록)
  └─ 문단: p + &nbsp; (블록)
  └─ 줄바꿈: <br /> (단일 태그)
  └─ 구분선: <hr />
  └─ 인라인 텍스트: b, strong, em, u, s, mark, big, small, sup, sub

태그 구분 (파일 3)
  └─ 블록 요소: 한 줄 전체 차지, width/height/margin 자유
  └─ 인라인 요소: 내용만큼만, width/height 제한

이미지 (파일 4)
  └─ <img src="경로" alt="설명" width="n" height="n">
  └─ src = 이미지 경로 (로컬/URL)
  └─ alt = 대체 텍스트 (접근성 + SEO)

컨테이너 (파일 5)
  └─ div = 블록 컨테이너 (영역 구분)
  └─ span = 인라인 컨테이너 (글자 일부 묶음)
  └─ 전역 속성: id, class, style, title

링크 (파일 6)
  └─ <a href="URL" target="_blank">텍스트</a>
  └─ target="_self": 현재 탭 / target="_blank": 새 탭
  └─ 이미지에 링크: <a><img></a>
  └─ 앵커 링크: href="#id값"

입력태그 (파일 7)
  └─ input type: text, password, button, color, date, range, number...
  └─ select + option: 드롭다운 메뉴
  └─ form: 서버 전송 컨테이너 (action, method)
  └─ name 속성: 각 input 데이터 식별자
```

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** 표준 HTML5 마크업, 최신 브라우저 공통.

## 2. `<head>` 컨피그 전체 정리 (MDN 보완)

```html
<head>
    <meta charset="UTF-8">                          <!-- 문자 인코딩 (필수) -->
    <meta name="viewport"
          content="width=device-width, initial-scale=1.0"> <!-- 반응형 (필수) -->
    <meta name="description" content="페이지 설명">  <!-- SEO -->
    <meta name="author" content="작성자">
    <title>탭에 표시되는 제목</title>
    <link rel="icon" href="favicon.ico">            <!-- 파비콘 -->
    <link rel="stylesheet" href="main.css">         <!-- CSS 연결 -->
    <meta property="og:title"       content="제목"> <!-- SNS 공유 -->
    <meta property="og:description" content="설명">
    <meta property="og:image"       content="URL">
</head>
<body>
    <!-- 콘텐츠 -->
    <script src="app.js"></script>  <!-- JS는 </body> 앞 배치 권장 -->
</body>
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

## 3. 태그 종합 비교표

### 3-1. 텍스트 관련 태그

| 태그 | 요소 종류 | 단일 태그 | 주요 역할 |
|---|---|---|---|
| `<h1>`~`<h6>` | 블록 |  | 제목 (h1이 가장 큼) |
| `<p>` | 블록 |  | 문단 (앞뒤 여백 자동) |
| `<br>` | — |  | 줄바꿈 |
| `<hr>` | 블록 |  | 수평 구분선 |
| `<strong>` | 인라인 |  | 굵게 + 의미 강조 |
| `<b>` | 인라인 |  | 굵게 (표현용) |
| `<em>` | 인라인 |  | 이탤릭 |
| `<u>` | 인라인 |  | 밑줄 |
| `<s>` / `<del>` | 인라인 |  | 취소선 |
| `<mark>` | 인라인 |  | 형광펜 |
| `<sup>` | 인라인 |  | 윗 첨자 |
| `<sub>` | 인라인 |  | 아랫 첨자 |

### 3-2. 구조·컨테이너 태그

| 태그 | 요소 종류 | 역할 |
|---|---|---|
| `<div>` | 블록 | 큰 영역 묶음 |
| `<span>` | 인라인 | 글자 일부 묶음 |
| `<img>` | 인라인 | 이미지 표시 (단일 태그) |
| `<a>` | 인라인 | 하이퍼링크 |

### 3-3. 입력 태그

| 태그 | 역할 |
|---|---|
| `<input type="text">` | 텍스트 입력 |
| `<input type="password">` | 비밀번호 입력 |
| `<input type="button">` | 클릭 버튼 |
| `<input type="color">` | 색상 선택 |
| `<input type="date">` | 날짜 선택 |
| `<input type="range">` | 슬라이더 |
| `<input type="number">` | 숫자 입력 |
| `<input type="checkbox">` | 체크박스 |
| `<input type="radio">` | 라디오 버튼 |
| `<input type="email">` | 이메일 형식 검증 |
| `<input type="file">` | 파일 선택 |
| `<select>` + `<option>` | 드롭다운 메뉴 |
| `<button>` | 버튼 |
| `<form>` | 서버 전송 양식 컨테이너 |

---

## 4. 블록 vs 인라인 핵심 정리

| 항목 | 블록 요소 | 인라인 요소 |
|---|---|---|
| 차지 공간 | 한 줄 전체 | 내용만큼만 |
| 자동 줄바꿈 | O | X |
| width/height | 설정 가능 | 일반적으로 안 됨 |
| 대표 태그 | h1~h6, p, div, hr | span, a, img, strong, em |
| 주요 용도 | 페이지 구조·영역 | 문장 안 부분 스타일 |

---

## 관련 문서

- 1.  HTML - HTML 기초와 기본구조
- 2.  HTML - 텍스트 표시 방법
- 3.  HTML - 태그의 구분·인라인 텍스트 요소
- 4.  HTML - 이미지 태그
- 5.  HTML - 컨테이너 태그
- 6.  HTML - 링크
- 7. ⌨ HTML - 입력태그
- 9.  HTML - 트러블슈팅 치트시트
- 10.  HTML - SQL 퀵 레퍼런스

>  **핵심 요약**
> - HTML: 웹브라우저가 읽는 구조 언어 — 문법은 태그 하나뿐
> - 기본 구조: DOCTYPE → html → head(설정) → body(콘텐츠)
> - 텍스트 태그: h1~h6(제목), p(문단), br(줄바꿈), 인라인 텍스트 태그들
> - 블록 vs 인라인: 블록=한 줄 전체 / 인라인=내용만큼만
> - 이미지: `<img src alt width height>` — 단일 태그
> - 컨테이너: div(블록 묶음) / span(인라인 묶음)
> - 링크: `<a href target>` — 텍스트·이미지 모두 감쌀 수 있음
> - 입력: input(type별), select(드롭다운), form(서버 전송)
> - 관련: 1.  HTML - HTML 기초와 기본구조 · 9.  HTML - 트러블슈팅 치트시트

# 🌐 HTML - HTML 기초와 기본구조

> **Tag:** #HTML #기초 #기본구조 #태그 #부트캠프
> **핵심 요약:** HTML은 태그를 유일한 문법으로 사용하는 웹페이지 구조 언어로, DOCTYPE → html → head → body 기본 구조를 가진다.

---

## 1. 📖 개요 (Overview)

HTML은 HyperText Markup Language의 약자로, 웹브라우저와 사람 사이에서 사용하는 언어이다. **HyperText**는 하이퍼링크를 통해 문서와 문서를 연결하는 기능이며 `<a>` 태그로 구현한다. **Markup**은 텍스트에 구조를 부여하기 위한 표시 방법으로 `<p>`, `<h1>` 등이 해당한다. **Language**는 고유의 문법과 규칙을 가진 언어라는 의미이며, HTML의 문법은 곧 태그이다. 개발자가 HTML 코드로 웹페이지 내용을 작성하면 브라우저가 이를 로딩하여 웹페이지가 완성된다.

이렇게 작성된 HTML 코드가 웹브라우저를 통해 해석되고 화면에 표현되는 과정을 **렌더링(Rendering)**이라고 한다.

HTML 문서를 만들기 위해서는 텍스트 편집기(코드 에디터)와 웹브라우저가 필요하다.

| 필요한 것 | 예시 |
|---|---|
| 텍스트 편집기 (코드 에디터) | VSCode, Brackets, Notepad++ |
| 웹브라우저 | Chrome, Firefox, Edge, Safari |

**VSCode**는 다양한 플러그인과 높은 확장성을 가지고, **Brackets**는 초보자에게 적합한 간단한 인터페이스를 제공하며, **Notepad++**는 가볍고 빠른 실행 속도가 특징이다. 코드 에디터는 자동완성·하이라이팅 기능이 추가된 메모장이라고 이해하면 된다.

**태그(tag)**는 HTML코드에서 정보(콘텐츠)를 정의하는 형식이며, HTML의 유일한 문법이다.

```html
<!-- 기본 태그: 시작과 끝 -->
<태그명>이곳에 콘텐츠를 기입한다.</태그명>

<!-- 단일 태그: 닫는 태그 없음 -->
<태그명 />

<!-- 속성 포함 -->
<태그명 속성명="속성값"></태그명>
<태그명 속성명="속성값" />
```

**속성**은 태그의 부가적인 기능을 정의하며 선택사항이다. 속성은 시작 태그 내부에 작성하고 개수 제한이 없으며, 속성값은 반드시 큰따옴표(`""`) 안에 작성해야 한다.

HTML 기본 구조는 다음과 같다.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>페이지 제목</title>
</head>
<body>
    <!-- 화면에 보이는 내용 -->
</body>
</html>
```

| 태그 | 역할 |
|---|---|
| `<!DOCTYPE html>` | 문서 유형 선언 (HTML5 표준) — 문서 맨 첫 줄 |
| `<html lang="ko">` | HTML 문서의 루트(root) 요소, 문서 시작~끝 |
| `<head>` | 브라우저에 보이지 않는 메타 정보 (인코딩, 제목 등) |
| `<meta charset="UTF-8">` | 문자 인코딩 지정 (한글+영문 모두 지원) |
| `<title>` | 브라우저 탭에 표시되는 문서 제목 |
| `<body>` | 실제 화면에 표시될 콘텐츠 영역 |

`<head>` 안에는 부트캠프 수업에서 배운 `charset`, `title` 외에도 실무에서 자주 쓰는 설정 태그들이 있다(MDN 기준 보완).

```html
<head>
    <!-- 1. 문자 인코딩 (필수) -->
    <meta charset="UTF-8">

    <!-- 2. 반응형 디자인 — 모바일 화면 대응 (실무 필수) -->
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <!-- 3. 페이지 설명 — 검색 결과(SEO)에 표시됨 -->
    <meta name="description" content="이 페이지는 HTML 기초를 설명합니다.">

    <!-- 4. 작성자 정보 -->
    <meta name="author" content="홍길동">

    <!-- 5. 브라우저 탭 제목 -->
    <title>페이지 제목</title>

    <!-- 6. Favicon (탭 아이콘) -->
    <link rel="icon" href="favicon.ico" type="image/x-icon">

    <!-- 7. 외부 CSS 연결 -->
    <link rel="stylesheet" href="main.css">

    <!-- 8. Open Graph — SNS 공유 시 미리보기 제목·설명·이미지 -->
    <meta property="og:title"       content="페이지 제목">
    <meta property="og:description" content="페이지 설명">
    <meta property="og:image"       content="https://example.com/image.png">
    <meta property="og:url"         content="https://example.com">
</head>

<!-- JavaScript는 보통 </body> 바로 앞에 배치 (HTML 로드 후 실행) -->
<script src="app.js"></script>
```

| 태그 | 역할 | 언제 쓰나 |
|---|---|---|
| `<meta charset>` | 문자 인코딩 | 항상 |
| `<meta viewport>` | 모바일 화면 대응 | 반응형 페이지 |
| `<meta description>` | SEO 페이지 설명 | 검색 노출 |
| `<title>` | 탭 제목 | 항상 |
| `<link rel="icon">` | 파비콘 | 탭 아이콘 필요 시 |
| `<link rel="stylesheet">` | 외부 CSS 연결 | CSS 분리 시 |
| `<meta og:*>` | SNS 미리보기 | 카카오·Facebook 공유 시 |
| `<script>` | JS 연결 | JS 사용 시 (`</body>` 앞 권장) |

> 💡 `keywords` 메타 태그(`<meta name="keywords">`)는 스팸 남용으로 현재 대부분의 검색엔진이 **무시**한다.

### Q7. 주석은 어떻게 쓰나?
👉 **핵심 답변:**

```html
<!-- 이 안의 내용은 브라우저에서 처리되지 않는다 -->
<!-- 주로 코드에 대한 메모를 남길 때 사용 -->
```

> 💡 사람에게는 보이지만 웹브라우저에게는 보이지 않는 코드

---

## 2. 🛠️ 표준 설정 템플릿 (Configuration)

> **적용 환경:** 표준 HTML5 마크업, 최신 브라우저 공통.

## ⚠️ 자주 하는 실수

| 실수 | 올바른 방법 |
|---|---|
| 닫는 태그 누락 | 시작 태그가 있으면 반드시 `</태그명>` 닫기 |
| 속성값에 따옴표 없음 | `속성명="값"` — 큰따옴표 필수 |
| `<!DOCTYPE>` 누락 | 문서 최상단 첫 줄에 반드시 작성 |
| `lang="en"` 한국어 페이지에 적용 | 한국어면 `lang="ko"` 로 변경 |

---

## 🔗 관련 문서

- 2. 📝 HTML - 텍스트 표시 방법
- 3. 🔤 HTML - 태그의 구분·인라인 텍스트 요소
- 4. 🖼️ HTML - 이미지 태그
- 5. 📦 HTML - 컨테이너 태그
- 6. 🔗 HTML - 링크
- 7. ⌨️ HTML - 입력태그

> 📌 **핵심 요약**
> - HTML = HyperText Markup Language, 웹페이지 구조를 만드는 언어
> - 렌더링 = HTML 코드를 웹브라우저가 해석하여 화면에 표시하는 과정
> - 태그 = HTML의 유일한 문법 단위 (`<태그명>콘텐츠</태그명>`)
> - 기본 구조: `<!DOCTYPE html>` → `<html>` → `<head>` → `<body>`
> - 관련: 2. 📝 HTML - 텍스트 표시 방법 · 3. 🔤 HTML - 태그의 구분·인라인 텍스트 요소

# ⚡ HTML - 퀵 레퍼런스

> **Tag:** #HTML #퀵레퍼런스 #치트시트 #레퍼런스 #부트캠프
> **핵심 요약:** 코딩 중 바로 찾아보는 HTML 태그·속성 전체 패턴 모음으로, 기본 구조부터 텍스트·이미지·링크·입력 태그까지 한 곳에 정리했다.

> 코딩 중 바로 찾아보는 HTML 태그·속성 전체 패턴 모음

---

## 1. 기본 구조 템플릿

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="페이지 설명 (SEO)">
    <title>페이지 제목</title>
    <link rel="icon" href="favicon.ico">
    <link rel="stylesheet" href="style.css">
</head>
<body>

    <!-- 콘텐츠 작성 -->

    <script src="app.js"></script>
</body>
</html>
```

---

## 2. 텍스트 태그 패턴

```html
<!-- 제목 -->
<h1>제목 1 (가장 큼)</h1>
<h2>제목 2</h2>
<h3>제목 3</h3>
<h4>제목 4</h4>
<h5>제목 5</h5>
<h6>제목 6 (가장 작음)</h6>

<!-- 문단 / 줄바꿈 / 공백 / 구분선 -->
<p>문단 텍스트</p>
<p>공백 &nbsp;&nbsp;&nbsp; 여러 칸</p>
줄바꿈<br>다음 줄
<hr>

<!-- 인라인 텍스트 꾸미기 -->
<b>굵게 (표현용)</b>
<strong>굵게 + 의미 강조</strong>
<em>이탤릭 (기울임)</em>
<u>밑줄</u>
<s>취소선</s>
<del>취소선 (의미 포함)</del>
<mark>형광펜</mark>
<big>약간 크게</big>
<small>약간 작게</small>
100<sup>2</sup>    <!-- 100² 윗 첨자 -->
H<sub>2</sub>O     <!-- 아랫 첨자 -->
```

---

## 3. 이미지 패턴

```html
<!-- 기본 -->
<img src="경로/파일명.jpg" alt="이미지 설명">

<!-- 크기 지정 (너비만 → 비율 유지) -->
<img src="photo.jpg" alt="사진" width="300">

<!-- 너비 + 높이 (비율 강제 변경) -->
<img src="photo.jpg" alt="사진" width="500" height="300">

<!-- 상대경로 예시 -->
<img src="../images/sample.jpg" alt="샘플">     <!-- 상위 폴더 -->
<img src="images/sample.jpg"   alt="샘플">     <!-- 하위 폴더 -->
<img src="sample.jpg"          alt="샘플">     <!-- 같은 폴더 -->
```

---

## 4. 컨테이너·전역 속성 패턴

```html
<!-- div: 블록 컨테이너 -->
<div>
    <h1>제목</h1>
    <p>내용</p>
</div>

<!-- span: 인라인 컨테이너 -->
<p>오늘 날씨는 <span style="color: red;">맑음</span>입니다.</p>

<!-- 전역 속성 -->
<div id="header" class="container" style="color: blue;" title="헤더">
    내용
</div>

<!-- id: 페이지 내 유일 식별자 -->
<!-- class: 여러 요소 공유 가능 -->
<!-- style: 인라인 CSS 직접 적용 -->
<!-- title: 마우스 오버 툴팁 -->
```

---

## 5. 링크 패턴

```html
<!-- 기본 링크 -->
<a href="https://www.naver.com">네이버</a>

<!-- 새 탭에서 열기 -->
<a href="https://www.google.com" target="_blank">구글 (새 탭)</a>

<!-- 현재 탭에서 열기 (기본값) -->
<a href="https://www.naver.com" target="_self">네이버 (현재 탭)</a>

<!-- 이미지 링크 -->
<a href="https://www.naver.com" target="_blank">
    <img src="naver.jpg" alt="네이버" title="네이버로 이동" width="200">
</a>

<!-- 페이지 내 앵커 링크 -->
<a href="#section2">섹션 2로 점프</a>
<h2 id="section2">섹션 2 제목</h2>

<!-- 이메일 링크 -->
<a href="mailto:email@example.com">이메일 보내기</a>

<!-- 전화 링크 (모바일) -->
<a href="tel:010-1234-5678">전화 걸기</a>
```

---

## 6. 입력 태그 패턴

### 6-1. input type 전체

```html
<input type="text"     name="id"     placeholder="아이디"   autofocus>
<input type="password" name="passwd" placeholder="비밀번호">
<input type="email"    name="email"  placeholder="이메일">
<input type="number"   name="age"    min="0" max="150">
<input type="range"    name="vol"    min="0" max="100" step="5">
<input type="color"    name="color">
<input type="date"     name="birth">
<input type="checkbox" name="agree"  value="yes"> 동의
<input type="radio"    name="gender" value="M"> 남성
<input type="radio"    name="gender" value="F"> 여성
<input type="file"     name="upload">
<input type="button"   value="버튼">
<input type="submit"   value="전송">
<input type="reset"    value="초기화">
<button>버튼 태그</button>

<!-- 읽기 전용 + 기본값 -->
<input type="number" value="010" readonly>
```

### 6-2. select (드롭다운)

```html
<!-- 단일 선택 -->
<select name="menu">
    <option value="" selected disabled>선택하세요</option>
    <option value="hamburger">햄버거</option>
    <option value="pasta">파스타</option>
    <option value="pizza">피자</option>
</select>

<!-- 다중 선택 -->
<select name="menu" multiple>
    <option value="hamburger">햄버거</option>
    <option value="pasta">파스타</option>
</select>
```

### 6-3. form (서버 전송)

```html
<form action="/login" method="post">
    <input name="id"     type="text"     placeholder="아이디"> <br>
    <input name="passwd" type="password" placeholder="비밀번호"> <br>
    <input type="submit" value="로그인">
</form>

<!-- method="get"  : URL에 데이터 노출 (검색 등) -->
<!-- method="post" : 본문에 데이터 전송 (로그인·회원가입 등) -->
```

---

## 7. 특수문자 (HTML Entity)

| 표시 | HTML 코드 | 설명 |
|---|---|---|
| (공백) | `&nbsp;` | Non-Breaking Space |
| `<` | `&lt;` | Less Than |
| `>` | `&gt;` | Greater Than |
| `&` | `&amp;` | Ampersand |
| `"` | `&quot;` | 큰따옴표 |
| `©` | `&copy;` | 저작권 |
| `®` | `&reg;` | 등록상표 |

---

## 8. 헷갈리는 항목 정리

### 8-1. `<b>` vs `<strong>`

| 태그 | 화면 | 스크린 리더 | 사용 목적 |
|---|---|---|---|
| `<b>` | 굵게 | 특별 강조 없음 | 시각적 표현 |
| `<strong>` | 굵게 | 강하게 발음 | 의미적 강조 |

### 8-2. `<div>` vs `<span>`

| 태그 | 요소 | 줄바꿈 | 용도 |
|---|---|---|---|
| `<div>` | 블록 | O | 영역 구분 |
| `<span>` | 인라인 | X | 글자 일부 묶음 |

### 8-3. `target="_self"` vs `target="_blank"`

| 값 | 동작 |
|---|---|
| `_self` | 현재 탭에서 열기 (기본값) |
| `_blank` | 새 탭에서 열기 |

### 8-4. `type="button"` vs `type="submit"`

| 값 | 동작 |
|---|---|
| `button` | 클릭 이벤트만 발생 (JS 연결용) |
| `submit` | form 데이터를 action URL로 전송 |
| `reset` | form 입력값 전체 초기화 |

### 8-5. 상대경로 기준

| 경로 패턴 | 의미 |
|---|---|
| `파일명.jpg` | 현재 파일과 **같은 폴더** |
| `images/파일명.jpg` | 현재 폴더의 **하위 폴더** images |
| `../파일명.jpg` | **상위 폴더** |
| `../../파일명.jpg` | 상위의 상위 폴더 |

---

## 🔗 관련 문서

- 1. 🌐 HTML - HTML 기초와 기본구조
- 2. 📝 HTML - 텍스트 표시 방법
- 3. 🔤 HTML - 태그의 구분·인라인 텍스트 요소
- 4. 🖼️ HTML - 이미지 태그
- 5. 📦 HTML - 컨테이너 태그
- 6. 🔗 HTML - 링크
- 7. ⌨️ HTML - 입력태그
- 8. 🧩 HTML - 통합 정리
- 9. 🚑 HTML - 트러블슈팅 치트시트

> 📌 **핵심 요약**
> - 기본 구조: DOCTYPE → html → head(charset·viewport·title·CSS) → body → script
> - 텍스트: h1~h6, p, br, hr, strong/b, em, u, s, mark, sup, sub
> - 이미지: `<img src alt width>` — 너비만 지정하면 비율 유지
> - 링크: `<a href target>` — `_blank`(새 탭), `#id`(앵커 링크)
> - 입력: input type 전체·select·form (action/method)
> - 특수문자: `&nbsp;`(공백), `&lt;`(<), `&gt;`(>), `&amp;`(&)
> - 관련: 8. 🧩 HTML - 통합 정리 · 9. 🚑 HTML - 트러블슈팅 치트시트

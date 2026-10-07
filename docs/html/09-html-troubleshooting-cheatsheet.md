# HTML - 트러블슈팅 치트시트

> **Tag:** #HTML #트러블슈팅 #치트시트 #오류
> **핵심 요약:** HTML 작성 시 자주 발생하는 오류(이미지 깨짐, 한글 깨짐, 줄바꿈 등)와 그 해결법을 시나리오별로 정리한 치트시트이다.

> HTML 작성 시 자주 발생하는 오류와 해결법 모음

---

## 1. 핵심 기술 개념 (Concept)

## 시나리오별 트러블슈팅

### 시나리오 1. 이미지가 깨져서 alt 텍스트만 보임

```html
<!--  경로 오류 -->
<img src="images/photo.jpg" alt="사진">

<!--  상대경로 확인 — 현재 파일 기준으로 경로 작성 -->
<img src="../images/photo.jpg" alt="사진">
<!-- 또는 파일이 같은 폴더에 있다면 -->
<img src="photo.jpg" alt="사진">
```

**체크리스트:**
- 파일명 대소문자 일치 여부 (`Photo.jpg` ≠ `photo.jpg`)
- 경로 구분자: HTML에서는 `\` 대신 `/` 사용
- 파일이 실제로 해당 경로에 존재하는지 확인

### 시나리오 2. 한글이 깨져서 이상한 문자로 표시됨

```html
<!--  charset 없음 -->
<head>
    <title>페이지</title>
</head>

<!--  charset을 head 맨 위에 추가 -->
<head>
    <meta charset="UTF-8">
    <title>페이지</title>
</head>
```

>  `<meta charset>` 은 `<head>` 태그 내부 **가장 첫 번째** 줄에 작성하는 것을 권장

### 시나리오 3. Enter를 눌렀는데 브라우저에서 줄바꿈이 안 됨

```html
<!--  Enter로 줄바꿈 시도 — 브라우저는 무시함 -->
<p>첫 번째 줄
두 번째 줄</p>

<!--  <br> 태그 사용 -->
<p>첫 번째 줄<br>
두 번째 줄</p>

<!-- 또는 각각 <p> 태그로 분리 -->
<p>첫 번째 문단</p>
<p>두 번째 문단</p>
```

### 시나리오 4. 공백(스페이스)을 여러 개 넣었는데 하나만 표시됨

```html
<!--  스페이스 여러 개 입력 — 브라우저가 하나로 합침 -->
<p>여기는     솔데스크 학원입니다.</p>

<!--  &nbsp; 사용 -->
<p>여기는 &nbsp;&nbsp;&nbsp;&nbsp; 솔데스크 학원입니다.</p>
```

### 시나리오 5. 태그를 닫지 않아서 이후 내용 전체가 스타일 적용됨

```html
<!--  닫는 태그 누락 — 이후 모든 내용이 굵게 됨 -->
<p><b>중요한 내용 입력합니다.
<p>다음 문단도 굵게 보임</p>

<!--  반드시 닫기 -->
<p><b>중요한 내용</b> 입력합니다.</p>
<p>다음 문단은 정상</p>
```

### 시나리오 6. 링크를 클릭해도 이동하지 않음

```html
<!--  href가 비어 있음 -->
<a href="">네이버</a>

<!--  URL 작성 -->
<a href="https://www.naver.com" target="_blank">네이버</a>

<!-- 현재 페이지 내 이동 (앵커) -->
<a href="#section1">섹션 1로 이동</a>
<h2 id="section1">섹션 1</h2>
```

### 시나리오 7. 모바일에서 텍스트가 너무 작거나 레이아웃이 깨짐

```html
<!--  viewport 없음 -->
<head>
    <meta charset="UTF-8">
    <title>페이지</title>
</head>

<!--  viewport 추가 -->
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>페이지</title>
</head>
```

### 시나리오 8. form 데이터가 서버에 구분 없이 전달됨

```html
<!--  name 없음 — 서버에서 어떤 필드인지 알 수 없음 -->
<input type="text" placeholder="아이디">
<input type="password" placeholder="비밀번호">

<!--  name 속성으로 식별자 부여 -->
<input name="user_id"  type="text"     placeholder="아이디">
<input name="password" type="password" placeholder="비밀번호">
```

### 시나리오 9. `<span>` 에 width/height를 줬는데 적용 안 됨

```html
<!--  인라인 요소에 width/height 직접 적용 — 무시됨 -->
<span style="width: 200px; height: 50px;">텍스트</span>

<!--  display: inline-block 으로 전환 -->
<span style="display: inline-block; width: 200px; height: 50px;">텍스트</span>

<!-- 또는 블록 컨테이너인 div 사용 -->
<div style="width: 200px; height: 50px;">텍스트</div>
```

### 시나리오 10. 이미지가 비율이 이상하게 늘어남

```html
<!--  width/height 둘 다 지정 → 비율 강제 변경 -->
<img src="photo.jpg" alt="사진" width="500" height="100">

<!--  한 쪽만 지정 → 비율 자동 유지 -->
<img src="photo.jpg" alt="사진" width="300">
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

## 태그별 자주 하는 실수 요약

### `<head>` 설정

| 실수 | 올바른 방법 |
|---|---|
| `charset` 누락 | `<meta charset="UTF-8">` 첫 번째 줄에 추가 |
| `viewport` 누락 | `<meta name="viewport" content="width=device-width, initial-scale=1.0">` |
| JS를 `<head>` 안에 넣음 | `<script>` 는 `</body>` 바로 앞에 배치 |

### 텍스트

| 실수 | 올바른 방법 |
|---|---|
| Enter로 줄바꿈 | `<br>` 사용 |
| 공백 여러 칸 | `&nbsp;` 반복 |
| `<b>` 와 `<strong>` 혼동 | 의미 강조면 `<strong>`, 표현만이면 `<b>` |

### 이미지

| 실수 | 올바른 방법 |
|---|---|
| `alt` 생략 | 항상 `alt` 작성 (빈 문자열이라도) |
| 경로에 `\` 사용 | HTML 경로는 `/` |
| 양쪽 크기 고정 | width 또는 height 하나만 지정해서 비율 유지 |

### 링크

| 실수 | 올바른 방법 |
|---|---|
| `href=""` 비어 있음 | URL 또는 `#` 작성 |
| 외부 링크 `_blank` 누락 | 외부 사이트는 `target="_blank"` |
| `<a>` 안에 `<a>` 중첩 | 링크 안에 링크 불가 |

### 입력

| 실수 | 올바른 방법 |
|---|---|
| `name` 없이 `input` 사용 | 모든 `input` 에 `name` 속성 추가 |
| `value=""` 없이 `option` 작성 | `value`가 실제 전송값 |
| `type="button"` 으로 form 전송 시도 | form 전송은 `type="submit"` |

---

## 관련 문서

- 1.  HTML - HTML 기초와 기본구조
- 2.  HTML - 텍스트 표시 방법
- 3.  HTML - 태그의 구분·인라인 텍스트 요소
- 4.  HTML - 이미지 태그
- 5.  HTML - 컨테이너 태그
- 6.  HTML - 링크
- 7. ⌨ HTML - 입력태그
- 8.  HTML - 통합 정리
- 10.  HTML - 퀵 레퍼런스

>  **핵심 요약**
> - 이미지 안 보임: src 경로·파일명 대소문자·구분자(`/`) 확인
> - 한글 깨짐: `<meta charset="UTF-8">` head 첫 줄에 추가
> - 줄바꿈 안 됨: `<br>` 태그 사용 (Enter는 HTML에서 무시)
> - 공백 하나만 표시: `&nbsp;` 반복 사용
> - 인라인 width/height 안 됨: `display: inline-block` 적용
> - 관련: 8.  HTML - 통합 정리 · 10.  HTML - 퀵 레퍼런스

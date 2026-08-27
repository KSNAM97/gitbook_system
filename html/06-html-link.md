# 🔗 HTML - 링크

> **Tag:** #HTML #링크 #a태그 #href #anchor #부트캠프
> **핵심 요약:** `<a>` 태그는 href로 이동 경로를, target으로 열기 방식을 지정하며 텍스트와 이미지 모두 링크로 만들 수 있다.

---

## 1. 📖 개요 (Overview)

`<a>` 태그는 anchor(닻)의 약자로, **현재 문서에서 다른 문서로 이동하는 링크**를 만드는 태그이다.

```html
<a href="이동할URL">링크 텍스트</a>
```

`href`는 hyperlink Reference의 약자로, 쌍따옴표 안에 이동할 경로를 입력한다. 브라우저에서 마우스를 올리면 손가락 모양(☞)으로 표시된다.

`target` 속성은 값에 따라 동작이 달라진다.

| target 값 | 동작 |
|---|---|
| `_self` | 현재 탭에서 열기 (기본값, 생략 가능) |
| `_blank` | **새 탭**에서 열기 |

```html
<!-- 현재 탭에서 열기 -->
<a href="https://www.naver.com" target="_self">네이버 (현재 탭)</a>

<!-- 새 탭에서 열기 -->
<a href="https://www.google.com" target="_blank">구글 (새 탭)</a>
```

이미지에 링크를 거는 방법은 `<a>` 태그로 `<img>` 태그를 감싸는 것이다.

```html
<a href="https://www.naver.com" target="_blank">
    <img src="../image/naver.jpg" alt="네이버로 이동" title="네이버" width="200">
</a>

<a href="https://www.google.com" target="_blank">
    <img src="../image/google.jpg" alt="구글로 이동" title="구글" width="200">
</a>
```

페이지 내부 특정 위치로 이동하려면 `id` 속성과 `#` 을 사용한 **앵커 링크**를 사용한다.

```html
<!-- 이동 목적지에 id 부여 -->
<h2 id="section2">섹션 2 제목</h2>

<!-- 해당 id로 점프 -->
<a href="#section2">섹션 2로 이동</a>
```

---

## 🖥️ 실습 예제

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>링크 실습</title>
    <link rel="stylesheet" href="main.css">
</head>
<body>
    <div>
        <h1>네이버로 이동하려면
            <a href="https://www.naver.com" target="_self">클릭</a>
        </h1>

        <h1>구글로 이동하려면
            <a href="https://www.google.com" target="_blank">클릭</a>
        </h1>
    </div>

    <div>
        <a href="https://www.naver.com" target="_blank">
            <img src="../image/naver.jpg" alt="네이버 이미지" title="네이버 이동" width="200">
        </a><br>
        <a href="https://www.google.com" target="_blank">
            <img src="../image/google.jpg" alt="구글 이미지" title="구글 이동" width="200">
        </a>
    </div>
</body>
</html>
```

---

## ⚠️ 자주 하는 실수

| 실수 | 올바른 방법 |
|---|---|
| `href` 없이 `<a>` 사용 | `href=""` 또는 `href="#"` 입력 |
| 외부 링크에 `target="_blank"` 미설정 | 외부 링크는 보통 `_blank` 로 새 탭에서 열기 |
| `<a>` 안에 `<a>` 중첩 | 링크 안에 링크 불가 |

---

## 🔗 관련 문서

- 1. 🌐 HTML - HTML 기초와 기본구조
- 4. 🖼️ HTML - 이미지 태그
- 5. 📦 HTML - 컨테이너 태그
- 7. ⌨️ HTML - 입력태그

> 📌 **핵심 요약**
> - `<a>` 태그: 인라인 요소, 텍스트·이미지 모두 링크로 만들 수 있음
> - `href`: 이동할 URL 또는 경로 (필수)
> - `target="_self"`: 현재 탭 (기본값) / `target="_blank"`: 새 탭에서 열기
> - 앵커 링크: `href="#id값"` 으로 페이지 내 특정 위치 이동
> - 관련: 4. 🖼️ HTML - 이미지 태그 · 5. 📦 HTML - 컨테이너 태그

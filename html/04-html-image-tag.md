# 🖼️ HTML - 이미지 태그

> **Tag:** #HTML #img태그 #이미지 #src #alt #부트캠프
> **핵심 요약:** `<img>` 는 단일 태그로, src(경로)와 alt(대체텍스트)를 속성으로 사용해 이미지를 표시한다.

---

## 1. 📖 개요 (Overview)

img 태그의 기본 형태는 다음과 같다.

```html
<img src="이미지경로" alt="이미지 설명" />

<!-- 크기 지정 -->
<img src="이미지경로" alt="이미지 설명" width="500" height="300" />
```

`src` 속성은 "source"의 약자로, 표시할 이미지의 **위치(경로)와 파일명**을 입력받는 속성이다. 서버 또는 자신의 컴퓨터에 저장된 이미지 경로를 사용한다.

```html
<!-- 로컬 경로 (상대경로) -->
<img src="../sample_image/sample.jpg" alt="샘플 이미지" width="500" height="300">

<!-- 절대경로 -->
<img src="C:/images/photo.jpg" alt="사진">

<!-- 외부 URL -->
<img src="https://example.com/image.jpg" alt="외부 이미지">
```

`alt` 속성은 "Alternative(대체)"의 약자로 세 가지 역할을 한다: 첫째, 이미지 로딩 실패 시 대체 텍스트를 표시한다. 둘째, 네트워크 지연으로 이미지가 늦게 로딩될 때 대체 텍스트를 표시한다. 셋째, 시각 장애인용 스크린 리더가 이미지 대신 alt 텍스트를 읽어준다(접근성).

```html
<!-- alt가 없으면 이미지가 깨질 때 빈 영역만 보임 -->
<img src="photo.jpg" alt="강아지가 뛰어노는 사진">
```

width와 height를 둘 다 지정하면 이미지 비율이 강제로 변경될 수 있다. 한쪽만 지정하면 비율이 유지된다.

```html
<!-- 너비만 지정 → 높이는 비율에 맞게 자동 조정 -->
<img src="sample.jpg" alt="샘플" width="300">

<!-- 둘 다 지정 → 비율 무시하고 강제 크기 적용 -->
<img src="sample.jpg" alt="샘플" width="500" height="300">
```

---

## 🖥️ 실습 예제

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>이미지 태그 실습</title>
</head>
<body>
    <!-- 로컬 이미지 -->
    <img src="../sample_image/sample.jpg" alt="이미지 로딩 실패" width="500" height="300">
    <img src="../sample_image/sample1.jpg" alt="이미지 로딩 실패" width="500" height="300">

    <!-- 링크와 이미지 결합 -->
    <a href="https://www.naver.com" target="_blank">
        <img src="../image/naver.jpg" alt="네이버로 이동" title="네이버" width="200">
    </a>
</body>
</html>
```

---

## ⚠️ 자주 하는 실수

| 실수 | 올바른 방법 |
|---|---|
| `alt` 속성 생략 | 항상 작성 (접근성 + SEO) |
| 경로 구분자를 `\` 로 작성 | HTML에서는 `/` 사용 |
| 이미지가 안 보일 때 alt 확인 안 함 | alt 텍스트로 경로 오류 파악 |

---

## 🔗 관련 문서

- 1. 🌐 HTML - HTML 기초와 기본구조
- 5. 📦 HTML - 컨테이너 태그
- 6. 🔗 HTML - 링크

> 📌 **핵심 요약**
> - `<img>` 는 단일 태그 (닫는 태그 없음)
> - `src`: 이미지 경로 (필수), `alt`: 대체 텍스트 (접근성·SEO용, 권장)
> - width/height 하나만 지정 시 비율 유지, 둘 다 지정 시 비율 강제 변경
> - 경로 구분자는 `\` 대신 `/` 사용
> - 관련: 5. 📦 HTML - 컨테이너 태그 · 6. 🔗 HTML - 링크

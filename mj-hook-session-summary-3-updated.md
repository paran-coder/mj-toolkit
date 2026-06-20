# MJ Hook — 세션 요약 3 (2026-06-20)

## 현재 상태

### 확장프로그램

- 현재 기준 버전: `v1.1.13`
- 파일 구성:
  - `manifest.json`
  - `content.js`
  - `background.js`
  - `icons/`
- Midjourney 페이지에서 Hook 버튼 표시 성공
- Explore 페이지에서 이미지 감지 성공
- Hook 클릭 시 Toolkit 페이지 자동 오픈 성공
- Midjourney → content.js → background.js → Toolkit 전달 성공
- Toolkit으로 이미지 payload 전달 성공
- Hook으로 가져온 이미지가 Toolkit 갤러리에 추가되는 것 확인

### MJ Toolkit

- 단일 `index.html` 기반 정적 웹앱
- IndexedDB에 이미지 저장
- Hook payload 수신 receiver 추가됨
- `mj-hook-save` CustomEvent 수신 성공
- `postMessage` fallback 수신 성공
- `window.__MJ_HOOK_LAST_PAYLOAD__` 회수 처리 추가됨
- 중복 payload 무시 처리 추가됨
- `addFiles([file])` 연결 성공
- Hook 이미지가 수동 업로드 이미지와 같은 갤러리 시스템에 들어감

---

## 현재 성공한 기능

- 이미지 전달 ✅
- Toolkit 자동 오픈 ✅
- Job ID 전달 ✅
- 생성 시각 전달 ✅
- 파일명 생성 ✅
- 프롬프트 추출 ✅
- 비율(ar) 추출 ✅
- sref 추출 ✅
- profile 추출 ✅
- Hook 이미지 Toolkit 반영 ✅
- PNG tEXt 청크 주입 ✅
- Hook payload JSON 메타 전달 ✅
- 수동 업로드와 Hook 업로드 비교 테스트 진행 ✅

---

## 현재 미해결 핵심 이슈

### 버전 추출 ❌

수동으로 업로드한 Midjourney Save Image 파일은 Toolkit에서 `v8.1` 같은 버전 메타가 정상 표시된다.

반면 Hook으로 가져온 이미지는:

```txt
이미지: 성공
프롬프트: 성공
ar: 성공
sref: 성공
profile: 성공
버전: 실패
상단 필터: 기타로 분류됨
```

### 중요한 원칙

버전을 못 찾는다고 기본값을 넣으면 안 된다.

```txt
금지:
- v8.1 기본값 삽입
- v15 같은 오탐 저장
- 페이지 전체 텍스트에서 아무 v숫자나 잡기
- Midjourney 내부 API 후보 무차별 fetch
- CSP를 깨는 inline script 주입
```

정확한 원칙:

```txt
실제 버전을 찾으면 저장
못 찾으면 빈 값 유지
가짜 버전은 절대 저장하지 않음
```

---

## 주요 시행착오

### 1. chrome.storage.local 방식

초기 구조:

```txt
content.js
  → chrome.runtime.sendMessage
  → background.js
  → chrome.storage.local에 payload 저장
  → Toolkit 탭에서 storage 읽기
  → CustomEvent 발송
```

문제:

- 이미지 buffer가 커지면 storage 경유가 불안정함
- isolated world에서 Toolkit 앱의 `addFiles()`에 직접 접근하기 어려움

현재는 이 구조를 버림.

---

### 2. background.js에서 Toolkit MAIN world 직접 주입

현재 구조:

```txt
content.js
  → background.js
  → chrome.scripting.executeScript({ world: 'MAIN' })
  → Toolkit window에 payload 주입
  → mj-hook-save / postMessage
  → index.html receiver
```

결과:

- Toolkit 자동 오픈 성공
- 이미지 전달 성공
- Hook 이미지 갤러리 추가 성공

---

### 3. v8.1 fallback 시도

한때 버전을 못 찾으면 `v8.1`을 기본값으로 넣는 방향을 검토했으나 폐기.

이유:

- 실제 이미지가 v8.1이라는 보장이 없음
- 메타 툴킷의 분류/검색/통계가 오염됨
- 사용자가 명확히 반대함

결론:

```txt
v8.1 fallback 금지
```

---

### 4. v15 오탐

버전 탐색 범위를 너무 넓게 잡았을 때 `v15`가 버전으로 표시됨.

판단:

- `v15`는 Midjourney 모델 버전이 아니라 UI/번들/내부 번호 오탐 가능성이 높음

대응:

```txt
v 계열 허용 범위: v1 ~ v10
niji 계열 허용 범위: niji 1 ~ niji 7
v15 차단
```

---

### 5. inline script / API 후보 fetch 문제

React 내부 데이터를 찾기 위해 inline script를 시도했으나 Midjourney CSP에 막힘.

오류:

```txt
Executing inline script violates Content Security Policy
```

또한 Midjourney 내부 API 후보를 직접 fetch하면서 401 오류 발생.

대응:

```txt
inline script 직접 주입 금지
무차별 API 후보 fetch 금지
chrome.scripting.executeScript({ world: 'MAIN' }) 방식으로 제한
```

---

## 현재 통신 구조

```txt
Midjourney 페이지
  → content.js가 CDN 이미지 감지
  → 이미지 host에 Hook 버튼 삽입
  → Hook 클릭
  → PNG 원본 URL 생성
  → CDN PNG fetch
  → PNG tEXt 청크 주입
  → payload 생성
  → chrome.runtime.sendMessage
  → background.js
  → Toolkit 탭 탐색 또는 생성
  → Toolkit MAIN world에 payload 주입
  → window.dispatchEvent(CustomEvent('mj-hook-save'))
  → window.postMessage fallback
  → index.html receiver
  → imageBuffer를 File 객체로 복원
  → addFiles([file])
  → IndexedDB 저장
```

---

## 파일 구조

권장 구조:

```txt
mj-hook-project/
  toolkit/
    index.html

  extension/
    manifest.json
    background.js
    content.js
    icons/
      icon16.png
      icon48.png
      icon128.png

  checklist.md
  context-notes.md
  CHANGELOG.md
  README.md
```

---

## 배포 상황

### 기존

- 운영 Netlify:
  - `https://mj-toolkit.netlify.app/`
- 테스트 Netlify:
  - `https://paranserver.netlify.app/`

### 현재 문제

- Netlify 무료 사용량이 소진되어 추가 배포가 어려운 상태

### 신규 방향

별도 도메인 구매 없이 무료 서비스만 사용한다.

```txt
1순위: Vercel 무료 배포
2순위: Netlify 기존 주소 유지
3순위: GitHub Pages 무료 백업
```

### 필요한 변경

`background.js`를 단일 `TOOLKIT_URL` 구조에서 URL fallback 구조로 변경해야 한다.

예상 구조:

```js
const TOOLKIT_URLS = [
  'https://<vercel-project>.vercel.app/',
  'https://paranserver.netlify.app/',
  'https://mj-toolkit.netlify.app/',
  'https://castle1945.github.io/mj-toolkit/'
];
```

`manifest.json`에도 Vercel/GitHub Pages host permission 추가 필요.

---

## 미드저니 DOM / 메타 구조

확인된 구조:

```txt
이미지:
img[src*="cdn.midjourney.com"]

CDN URL:
cdn.midjourney.com/{job-id}/{index}.png 또는 webp 변형

프롬프트:
div.break-word.shrink-0.text-sm 계열
단, 이미지 카드 밖에 있을 수 있음

파라미터:
ar
sref
profile
v / niji
```

주의:

- Explore 페이지는 카드 구조가 Create와 다름
- 이미지 host를 잘못 잡으면 주변 카드의 profile/sref가 섞일 수 있음
- 큰 ancestor 전체 텍스트를 긁으면 오염값이 들어올 수 있음
- 버전은 DOM에 보이지 않거나 React/Next 내부 데이터에 숨어 있을 수 있음

---

## Toolkit 메타 표시 기준

수동 업로드 이미지 기대값:

```txt
프롬프트
photo of Korean actress

파라미터
버전 v8.1
비율 4:5
sref 2169211740
profile 878tcq4
profile 1olw3g6
```

Hook 이미지 현재값:

```txt
프롬프트 표시됨
비율 표시됨
sref 표시됨
profile 표시됨
버전 없음
상단 필터에서 기타로 분류
```

목표:

```txt
Hook 이미지도 수동 업로드 이미지와 같은 방식으로 버전/메타 분류
```

---

## Console 로그 상태

### 무시 가능

```txt
favicon.ico 404
duplicate MJ Hook payload ignored
```

설명:

- favicon 오류는 기능과 무관
- duplicate 로그는 background가 안정성을 위해 여러 번 payload를 쏘고, Toolkit receiver가 중복 저장을 막는 정상 동작

### 해결 필요

```txt
Executing inline script violates Content Security Policy
401 Unauthorized
과도한 debug 로그
```

현재 대응:

- inline script 직접 삽입 제거
- 내부 API 후보 무차별 fetch 제거
- 추후 debug mode 플래그 필요

---

## 다음 세션에서 할 일

### 1. 무료 배포 구조 전환

- GitHub 저장소 생성
- `toolkit/index.html` 업로드
- Vercel 프로젝트 생성
- Vercel 배포 URL 확정
- GitHub Pages 백업 배포 설정
- `manifest.json` host permissions 업데이트
- `background.js` URL fallback 구조 적용

### 2. 버전 추출 해결

- Midjourney 화면에서 실제 버전이 어디에 있는지 재확인
- 수동 Save Image 파일과 Hook fetch 파일의 메타 차이 재확인
- Hook 클릭 시 `[MJ Hook] 추출 결과` 로그 분석
- `pageProbe`, `jobMeta`, `versionSource` 확인
- React/Next 내부 데이터에 version/modelVersion/model/versionName 계열 필드가 있는지 확인
- 정확한 버전 필드가 확인될 때만 저장

### 3. 안정화

- background payload 중복 발송 횟수 줄이기
- Console 로그 정리
- Create 페이지 회귀 테스트
- Explore 페이지 무한 스크롤 테스트
- 수동 업로드와 Hook 업로드 비교 테스트
- 저장 순서 문제 확인

### 4. 문서 정리

- `CHANGELOG.md` 작성
- `README.md` 작성
- 설치/업데이트 가이드 작성
- 기존 사용자 안내 문구 작성

---

## 프로젝트 규칙

1. `CHANGELOG.md` 작성 + 파일명에 버전 표기
2. `checklist.md` + `context-notes.md` 작성 및 유지
3. 작업마다 기록 업데이트
4. 파일은 요청할 때만 전달
5. 매 단계마다 대화로 결정 후 진행
6. 혼자 앞서가지 않음
7. 추가 규칙은 그때그때 반영

---

## 다음 세션 시작 문구

```txt
MJ Hook 프로젝트 이어받기.
현재 버전은 확장프로그램 v1.1.13 / Toolkit index v1.1.13.
이미지 전달, 프롬프트, ar, sref, profile은 성공했고 버전 추출만 미완료.
Netlify 무료 사용량이 소진되어 GitHub + Vercel + GitHub Pages 무료 배포 구조로 전환하려고 한다.
checklist.md, context-notes.md, session-summary 파일 기준으로 이어서 작업하자.
```

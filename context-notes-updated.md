# MJ Hook 컨텍스트 노트

업데이트: 2026-06-20  
현재 기준 버전: 확장프로그램 v1.1.13 / Toolkit index v1.1.13  
현재 상태: 이미지 전달 성공, sref/profile/ar 추출 성공, 버전 추출 미완료  
현재 배포 이슈: Netlify 무료 사용량 소진으로 Vercel + GitHub Pages 무료 이전 준비

---

## 프로젝트 개요

미드저니 웹페이지에서 이미지를 클릭 한 번으로 MJ Toolkit에 저장하는 Chrome 확장프로그램 프로젝트.

목표는 단순 이미지 저장이 아니라, **Midjourney Save Image로 받은 파일처럼 프롬프트, 버전, 비율, sref, profile, Job ID, 생성 시각 등 메타정보를 Toolkit에서 분류 가능하게 보존하는 것**이다.

MJ Toolkit은 단일 `index.html` 기반 정적 웹앱이며, 브라우저 IndexedDB에 이미지를 저장한다. 데이터는 외부 서버를 거치지 않는 구조를 목표로 한다.

---

## 현재 전체 흐름

```txt
Midjourney 페이지
  → content.js가 이미지 위 Hook 버튼 표시
  → Hook 클릭
  → CDN 이미지 PNG fetch
  → PNG tEXt 청크에 Hook 메타 주입
  → chrome.runtime.sendMessage
  → background.js
  → Toolkit 탭 찾기 또는 열기
  → Toolkit MAIN world에 payload 주입
  → index.html의 MJ Hook receiver
  → File 객체 생성
  → addFiles([file])
  → IndexedDB 저장
  → 갤러리/상세 패널에 표시
```

---

## 현재 성공한 부분

- Midjourney 페이지에서 Hook 버튼 표시 성공
- 이미지 hover 시 Hook 버튼 표시 구조 적용
- Explore 페이지에서 이미지 감지 성공
- Hook 클릭 시 Toolkit 페이지 자동 오픈 성공
- 이미지 payload가 Toolkit으로 전달됨
- Toolkit에서 Hook 이미지가 갤러리에 추가됨
- 프롬프트 표시 성공
- Job ID 전달 성공
- 생성 시각 전달 성공
- 파일명 생성 성공
- 비율(ar) 추출 성공
- sref 추출 성공
- profile 추출 성공
- profile 오염값 일부 차단 성공
- `v15` 같은 오탐 버전 차단 필요성 확인
- `niji 7` 허용 기준 반영 필요성 확인
- 수동 업로드와 Hook 업로드 비교 테스트 진행 중

---

## 현재 미해결 핵심 이슈

### 1. 버전 추출 미완료

수동으로 업로드한 Midjourney Save Image 파일은 Toolkit에서 `v8.1` 같은 버전 메타가 정상 표시된다.

반면 Hook으로 가져온 이미지는 다음 상태다.

```txt
이미지 전달: 성공
프롬프트: 성공
ar: 성공
sref: 성공
profile: 성공
버전: 실패
상단 필터: 기타로 분류됨
```

원인은 Hook 방식이 Midjourney CDN PNG를 직접 fetch하는 구조라서, 수동 Save Image 파일에 들어 있는 원본 메타데이터가 CDN PNG에는 없기 때문이다.

따라서 버전은 다음 중 하나에서 실제로 찾아야 한다.

```txt
1. 이미지 카드 주변 DOM
2. 상세 모달/Job 페이지 DOM
3. React/Next 내부 데이터
4. Midjourney가 Save Image 버튼에서 사용하는 실제 다운로드 경로 또는 데이터
```

단, 다음 방식은 금지한다.

```txt
- 버전을 못 찾는다고 v8.1을 기본값으로 넣기
- v15 같은 UI/번들 번호를 모델 버전으로 저장하기
- 내부 API 후보를 무차별 fetch해서 401 오류를 만드는 방식
- CSP를 위반하는 inline script 주입 방식
```

정확한 원칙은 다음이다.

```txt
버전을 실제로 찾으면 저장
못 찾으면 빈 값 유지
가짜 버전은 절대 저장하지 않음
```

---

## 통신 방식 변경 이력

### 초기 결정: Chrome Storage API + CustomEvent

초기에는 다음 구조를 사용했다.

```txt
content.js
  → chrome.runtime.sendMessage
  → background.js
  → chrome.storage.local에 payload 저장
  → Toolkit 탭에서 storage 읽기
  → CustomEvent('mj-hook-save')
  → index.html receiver
```

이 방식은 작은 payload에는 가능했지만, 이미지 buffer가 커지면서 불안정했다.

### 현재 구조: background.js에서 Toolkit MAIN world 직접 주입

현재는 `chrome.storage.local` 경유를 버리고, `background.js`가 Toolkit 탭의 MAIN world에 직접 payload를 주입한다.

이유:

```txt
- imageBuffer가 커서 storage/local 또는 executeScript 인자 제한 문제 가능
- isolated world에서 addFiles() 접근 불가
- Toolkit 앱이 듣는 window 이벤트는 MAIN world 기준이 안정적
```

현재는 Toolkit 탭이 여러 개 열려 있을 수 있으므로, 열린 Toolkit 탭 전체에 payload를 쏘고, Toolkit receiver에서 중복 payload를 무시한다.

---

## Midjourney 메타데이터 관련 정리

### 수동 Save Image 파일

- Midjourney가 내려주는 원본 저장 파일
- PNG 내부에 프롬프트/버전/파라미터 메타가 포함됨
- Toolkit에서 버전이 정상 표시됨

### CDN 직접 fetch 파일

- Hook이 현재 fetch하는 파일
- 같은 이미지라도 Save Image 파일과 메타 구성이 다름
- 원본 버전 메타가 없을 수 있음
- 따라서 Hook이 직접 tEXt chunk를 주입하고 있음

### 현재 채택 방식

```txt
CDN PNG fetch
→ 프롬프트/ar/sref/profile/jobId/creationTime을 수집
→ PNG tEXt chunk에 Description / Creation Time / Author / MJ Hook Metadata 주입
→ Toolkit으로 전달
```

### 남은 과제

Save Image와 동일한 수준의 버전 메타를 확보하려면, Midjourney가 실제 Save Image에서 사용하는 데이터 경로나 카드 내부 job data를 더 정확히 찾아야 한다.

---

## content.js 현재 방향

역할:

```txt
- Midjourney 이미지 감지
- 이미지 위 Hook 버튼 삽입
- 이미지 URL 파싱
- PNG 원본 URL 생성
- 프롬프트 추출
- ar/sref/profile 추출
- 잘못된 profile/UI 텍스트 제거
- PNG tEXt chunk 주입
- payload 생성
- background.js로 전송
```

중요 원칙:

```txt
- 버튼은 이미지 hover 시에만 표시
- 버튼 위치는 이미지 host 기준 absolute
- Explore 우선 지원
- Create 회귀 테스트 필요
- 버전은 실제로 찾은 값만 저장
- v8.1 기본값 삽입 금지
- v15 오탐 저장 금지
```

---

## background.js 현재 방향

역할:

```txt
- content.js에서 SAVE_TO_TOOLKIT 메시지 수신
- Toolkit URL 후보 중 열린 탭 탐색
- 없으면 Toolkit 탭 열기
- Toolkit MAIN world에 payload 주입
- 필요 시 Midjourney MAIN world에서 page probe 수행
```

현재까지 확인된 것:

```txt
- Toolkit 자동 오픈 성공
- payload 주입 성공
- duplicate payload ignored 로그는 정상
- Console 로그가 많아 추후 정리 필요
```

다음 구조로 변경 예정:

```js
const TOOLKIT_URLS = [
  'https://<vercel-project>.vercel.app/',
  'https://paranserver.netlify.app/',
  'https://mj-toolkit.netlify.app/',
  'https://castle1945.github.io/mj-toolkit/'
];
```

목표:

```txt
Vercel 우선
Netlify 백업
GitHub Pages 비상 백업
```

---

## index.html 현재 방향

역할:

```txt
- MJ Hook payload 수신
- imageBuffer → File 객체 변환
- file.mjHookMeta / file.mjMeta / PNG tEXt metadata 파싱
- addFiles([file]) 호출
- IndexedDB 저장
- 갤러리 카드 렌더링
- 상세 패널 메타 렌더링
- 상단 필터 분류
```

현재 성공:

```txt
- mj-hook-save CustomEvent 수신
- postMessage fallback 수신
- window.__MJ_HOOK_LAST_PAYLOAD__ 회수
- duplicate payload 무시
- Hook 이미지 갤러리 표시
- Hook 메타 일부 표시
```

현재 문제:

```txt
- Hook 이미지의 version 값이 비어 있음
- version이 없으면 상단 필터에서 기타로 분류됨
- 수동 업로드와 Hook 업로드의 메타 표시가 완전히 일치하지 않음
```

---

## 배포 전략 변경

### 기존

```txt
Netlify:
- https://mj-toolkit.netlify.app/
- https://paranserver.netlify.app/
```

문제:

```txt
Netlify 무료 사용량 소진으로 추가 배포가 어려움
```

### 신규 무료 전략

별도 도메인은 구매하지 않는다.

```txt
1순위: Vercel 무료 배포
2순위: Netlify 기존 주소 유지
3순위: GitHub Pages 무료 백업
```

권장 저장소 구조:

```txt
mj-hook-project/
  toolkit/
    index.html

  extension/
    manifest.json
    background.js
    content.js
    icons/
```

진행할 일:

```txt
- GitHub 저장소 생성
- toolkit/index.html 업로드
- Vercel에서 GitHub 저장소 Import
- Vercel 배포 URL 확보
- GitHub Pages 활성화
- extension 코드의 Toolkit URL fallback 적용
- manifest host_permissions에 Vercel/GitHub Pages 추가
```

---

## 기존 사용자 처리

기존 사용자는 확장프로그램 코드에 박힌 Netlify 주소를 계속 사용한다.

따라서 Vercel로 새로 배포해도 기존 사용자가 자동으로 이동하지 않는다.

현실적인 대응:

```txt
- 확장프로그램 v1.2.0 업데이트
- TOOLKIT_URL 단일 상수 제거
- TOOLKIT_URLS fallback 구조 적용
- 기존 Netlify 주소는 가능한 한 레거시/백업으로 유지
```

---

## Console 오류 정리

### 무시 가능

```txt
favicon.ico 404
duplicate MJ Hook payload ignored
```

설명:

```txt
favicon.ico 404는 기능과 무관
duplicate payload ignored는 background가 여러 번 쏘고 index가 중복 저장을 막는 정상 동작
```

### 해결 필요

```txt
Executing inline script violates Content Security Policy
401 Unauthorized
```

대응:

```txt
- inline script 직접 주입 금지
- chrome.scripting.executeScript({ world: 'MAIN' }) 사용
- Midjourney 내부 API 후보 무차별 fetch 금지
```

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

## 다음 작업 우선순위

### 1. 무료 배포 전환

```txt
GitHub 저장소 생성
Vercel 배포
GitHub Pages 백업
확장프로그램 URL fallback 적용
```

### 2. 버전 추출 해결

```txt
Midjourney 실제 버전 데이터 위치 확인
pageProbe 로그 분석
React/Next 데이터 필드 확인
Save Image 경로 또는 job data 확인
실제 버전만 Hook 메타에 저장
```

### 3. 안정화

```txt
Console 로그 정리
payload 중복 발송 횟수 축소
Create/Explore 회귀 테스트
수동 업로드와 Hook 업로드 결과 비교
```

### 4. 릴리즈 문서 정리

```txt
CHANGELOG.md 작성
README.md 작성
설치/업데이트 가이드 작성
기존 사용자 안내 문구 작성
```

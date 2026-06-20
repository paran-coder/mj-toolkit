# MJ Hook 체크리스트

업데이트: 2026-06-20  
현재 기준 버전: 확장프로그램 v1.1.13 / Toolkit index v1.1.13  
현재 테스트 서버: Netlify 사용량 소진으로 Vercel/GitHub Pages 이전 준비

---

## 현재 상태 요약

- [x] Midjourney → 확장프로그램 content.js → background.js 통신 성공
- [x] Toolkit 페이지 자동 오픈 성공
- [x] Toolkit 페이지로 이미지 payload 전달 성공
- [x] Toolkit `mj-hook-save` receiver 설치 및 수신 성공
- [x] Hook 이미지가 Toolkit 갤러리에 추가되는 것 확인
- [x] 프롬프트 원문 정제 성공
- [x] Job ID 전달 성공
- [x] 생성 시각 전달 성공
- [x] 파일명 생성 성공
- [x] sref 추출 성공
- [x] profile 추출 성공
- [x] 비율(ar) 추출 성공
- [ ] 버전(v / niji) 추출 미완료 — 현재 최우선 이슈
- [ ] Hook 이미지와 수동 업로드 이미지의 메타 표시 완전 일치 필요
- [ ] Netlify 무료 사용량 소진으로 배포 전략 변경 필요

---

## 준비

- [x] 워크플로우 작성 (2026-06-19-mj-hook-plan.md)
- [x] 프로젝트 규칙 확정
- [x] checklist.md 작성
- [ ] context-notes.md 작성
- [ ] CHANGELOG.md 작성
- [x] 기존 Netlify URL 확정: https://mj-toolkit.netlify.app/
- [x] 테스트 Netlify URL 확정: https://paranserver.netlify.app/
- [x] Midjourney Create/Explore 이미지 DOM 기본 구조 확인
- [x] CDN URL → 원본 PNG URL 변환 방식 확인
- [x] CDN PNG에 원본 메타가 없을 수 있어 tEXt 청크 직접 주입 방식 채택
- [ ] Vercel 배포 URL 확정
- [ ] GitHub 저장소 구조 확정
- [ ] GitHub Pages 백업 배포 여부 결정

---

## 배포 전략

### 기존 상태

- [x] Netlify에 MJ Toolkit 배포 완료
- [x] Netlify 테스트 서버 사용
- [ ] Netlify 무료 사용량 소진으로 추가 배포 불가 상태 확인

### 신규 전략

- [ ] GitHub 저장소 생성
- [ ] `toolkit/index.html` 업로드
- [ ] `extension/manifest.json`, `extension/content.js`, `extension/background.js`, `extension/icons/` 구조 정리
- [ ] Vercel에서 GitHub 저장소 Import
- [ ] Vercel 배포 URL 확보
- [ ] GitHub Pages 백업 배포 설정
- [ ] 확장프로그램에 Toolkit URL fallback 구조 적용
- [ ] 기존 Netlify 주소는 레거시/백업으로 유지

### 권장 URL 우선순위

```txt
1. Vercel 최신 운영 주소
2. Netlify 기존/테스트 주소
3. GitHub Pages 백업 주소
```

---

## Task 1: 확장프로그램 기본 구조

- [x] `manifest.json` 작성
- [x] MV3 service worker 구조 적용
- [x] `storage`, `tabs`, `scripting` 권한 적용
- [x] Midjourney / CDN / Toolkit host permissions 설정
- [ ] Vercel URL host permission 추가
- [ ] GitHub Pages URL host permission 추가
- [ ] URL fallback 구조에 맞춰 host permissions 정리
- [ ] 아이콘 준비 (16/48/128px) — 임시 스킵, 나중에 교체

---

## Task 2: Content Script — 버튼 오버레이

- [x] MutationObserver로 이미지 감지
- [x] `img[src*="cdn.midjourney.com"]` 기준으로 감지 방식 변경
- [x] 🪝 Hook 버튼 주입
- [x] 이미지 hover 시 버튼 표시 방식 적용
- [x] Explore 페이지 우선 대응
- [x] 스크롤 시 버튼 위치가 이미지 기준으로 따라가도록 host 탐색 로직 개선
- [ ] Explore 카드별 host 탐색 안정성 추가 검증
- [ ] Create 페이지 회귀 테스트
- [ ] Explore 페이지 무한 스크롤 회귀 테스트

---

## Task 3: Content Script — 이미지 캡처 + 메타 추출

- [x] CDN 이미지 URL 파싱
- [x] webp 썸네일 URL → PNG 원본 URL 변환
- [x] 이미지 fetch → Blob 변환
- [x] PNG tEXt 청크 주입
- [x] Description / Creation Time / Author 주입
- [x] MJ Hook Metadata JSON chunk 주입
- [x] 프롬프트 텍스트 추출
- [x] ar 비율 추출
- [x] sref 추출
- [x] profile 추출
- [x] profile 오염값 필터링 (`Subtle`, `Strong`, `CreatorRun`, `batch`, `as` 등)
- [x] `v15` 같은 오탐 버전 차단
- [x] `niji 7` 허용 기준 반영
- [ ] 실제 버전(v8.1 등) 추출 미완료
- [ ] 버전 추출 소스 재설계 필요
- [ ] React/Next 내부 데이터 탐색 방식 안정화 필요
- [ ] CSP 위반 없는 버전 탐색 방식 확정 필요
- [ ] Midjourney 내부 API 직접 호출 방식 제거 유지
- [ ] `pageProbe`, `jobMeta`, `versionSource` 로그 기반 추가 분석

---

## Task 4: Background Script — 탭 통신 중계

- [x] MJ Toolkit 탭 감지
- [x] 탭 없으면 새로 열기
- [x] 기존 Toolkit 탭이 여러 개일 때 전체 탭 주입 방식 적용
- [x] Toolkit MAIN world에 payload 주입 성공
- [x] 중복 발송 후 Toolkit에서 duplicate ignore 처리 확인
- [x] Midjourney → background → Toolkit 전달 성공
- [ ] Console 로그 과다 출력 정리
- [ ] payload 중복 발송 횟수 축소
- [ ] URL fallback 구조 적용
- [ ] Vercel / Netlify / GitHub Pages 순차 fallback 적용
- [ ] Toolkit URL을 단일 상수에서 배열 구조로 변경

---

## Task 5: MJ Toolkit index.html 수정

- [x] `mj-hook-save` CustomEvent 수신 코드 추가
- [x] `postMessage` fallback 수신 코드 추가
- [x] `window.__MJ_HOOK_LAST_PAYLOAD__` 회수 처리 추가
- [x] Hook payload → File 객체 변환
- [x] `addFiles([file])` 연결 성공
- [x] 중복 payload 무시 처리
- [x] Hook 이미지 갤러리 반영 확인
- [x] Hook 배지/메타 표시 구조 추가
- [x] 프롬프트 박스에서 Job ID/Page 오염 제거
- [x] sref/profile/ar 칩 표시 성공
- [x] 수동 업로드 이미지와 Hook 이미지의 상세 패널 구조 유사화
- [ ] Hook 이미지 버전 칩 표시 미완료
- [ ] Hook 이미지 상단 필터에서 버전 분류 미완료
- [ ] 수동 업로드와 Hook 업로드의 메타 파싱 결과 완전 일치 필요
- [ ] 배포 대상을 Netlify에서 Vercel로 이전

---

## Task 6: 무료 배포 구조 전환

- [ ] GitHub 저장소 생성
- [ ] 저장소 구조 결정

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

- [ ] Vercel 프로젝트 생성
- [ ] Vercel GitHub 연동
- [ ] Vercel 배포 URL 확정
- [ ] GitHub Pages 활성화
- [ ] GitHub Pages URL 확정
- [ ] Netlify 기존 주소 유지 여부 결정
- [ ] 확장프로그램 v1.2.0 URL fallback 업데이트
- [ ] 기존 사용자 안내 문구 작성

---

## 테스트

### 완료

- [x] 크롬 확장프로그램 로컬 설치 (`chrome://extensions`)
- [x] Midjourney 페이지에서 Hook 버튼 노출 확인
- [x] Hook 클릭 시 Toolkit 페이지 오픈 확인
- [x] 이미지 저장 → Toolkit 반영 확인
- [x] Hook 이미지 프롬프트 표시 확인
- [x] Hook 이미지 Job ID 표시 확인
- [x] Hook 이미지 생성 시각 표시 확인
- [x] Hook 이미지 sref/profile/ar 표시 확인
- [x] duplicate payload ignored 로그 확인 — 정상 동작으로 판단
- [x] `favicon.ico 404`는 기능과 무관한 오류로 분류

### 진행 중

- [ ] 메타데이터 정상 추출 확인
- [ ] Hook 이미지 버전 추출 확인
- [ ] 수동 드래그&드롭 이미지와 Hook 이미지 메타 비교
- [ ] 수동 업로드와 Hook 업로드의 상단 필터 분류 비교
- [ ] Explore 페이지에서 여러 카드 연속 Hook 테스트
- [ ] Create 페이지 회귀 테스트

### 해결 필요

- [ ] Hook 이미지에서 버전이 빠지는 문제
- [ ] CSP 오류 재발 방지
- [ ] Midjourney API 401 오류 재발 방지
- [ ] Console 로그 정리
- [ ] Netlify 배포 한도 문제 해결
- [ ] Vercel 이전 후 확장프로그램 URL 업데이트
- [ ] 기존 사용자 업데이트 경로 정리

---

## 현재 핵심 이슈: 버전 추출

### 증상

- 수동 업로드 이미지:
  - `v8.1` 표시됨
  - 상단 필터에도 `v8.1` 분류됨

- Hook 이미지:
  - 이미지, 프롬프트, ar, sref, profile은 전달됨
  - 버전이 비어 있음
  - 상단 필터에서 `기타`로 분류됨

### 금지한 방식

- [x] 버전을 못 찾을 때 `v8.1` 기본값 삽입 금지
- [x] `v15` 같은 오탐값 저장 금지
- [x] Midjourney 내부 API 후보를 무차별 fetch하는 방식 금지
- [x] inline script 주입 방식 금지

### 다음 접근 방향

- [ ] Midjourney 페이지 내 실제 버전 표시 위치 확인
- [ ] Hook 클릭 시점의 `[MJ Hook] 추출 결과` 객체 상세 확인
- [ ] `pageProbe.hits`, `pageProbe.scanned`, `targetCount` 확인
- [ ] React/Next 데이터에서 `version`, `modelVersion`, `model`, `versionName` 계열 필드 탐색
- [ ] 버전이 DOM에 없는 경우 Job 상세 모달/페이지에서 가져오는 방식 검토
- [ ] 정확한 버전 필드가 확인되기 전까지 버전은 빈 값 유지

---

## 다음 작업 우선순위

1. **무료 배포 전환**
   - GitHub 저장소 생성
   - Vercel 배포
   - GitHub Pages 백업
   - 확장프로그램 URL fallback 적용

2. **버전 추출 해결**
   - Midjourney 실제 데이터 위치 확인
   - CSP/401 없이 version source 확보
   - Hook 메타에 실제 버전만 저장

3. **로그 정리**
   - duplicate 발송 횟수 축소
   - debug mode 플래그 추가
   - production console 최소화

4. **릴리즈 정리**
   - `CHANGELOG.md` 작성
   - `context-notes.md` 작성
   - 확장프로그램 v1.2.0 정리

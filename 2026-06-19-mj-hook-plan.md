# MJ Hook 구현 플랜

> **Goal:** 미드저니 웹에서 버튼 클릭 한 번으로 이미지 Blob + 프롬프트를 MJ Toolkit IndexedDB에 저장하는 크롬 확장프로그램(MJ Hook)을 만들고, MJ Toolkit에 수신 코드를 추가한다.

**Architecture:** 확장프로그램 Content Script가 미드저니 페이지에서 이미지와 프롬프트를 감지하고 저장 버튼을 오버레이한다. 버튼 클릭 시 이미지를 Blob으로 즉시 다운받아 Chrome Storage에 임시 저장하고, MJ Toolkit 탭에 postMessage로 알린다. MJ Toolkit은 메시지를 수신해 IndexedDB에 저장한다.

**Tech Stack:** Chrome Extension Manifest V3, Content Script, chrome.storage.local, postMessage, Fetch API(Blob), 기존 MJ Toolkit IndexedDB

**확장프로그램 이름:** MJ Hook
**툴킷 내 배지:** 🪝 Hook

---

## 전체 흐름

```
미드저니 페이지
  └── Content Script 동작
        ├── 이미지 카드 감지
        ├── "🪝 Hook" 버튼 오버레이
        └── 클릭 시:
              ① 이미지 fetch → Blob 변환
              ② 프롬프트 텍스트 추출
              ③ chrome.storage.local에 임시 저장
              └── MJ Toolkit 탭에 postMessage 전송
                    └── 툴킷이 메시지 수신
                          └── IndexedDB에 저장 (배지: 🪝 Hook)
```

---

## 파일 구조

```
mj-hook/
  manifest.json        ← 확장프로그램 설정
  content.js           ← 미드저니 페이지에서 동작
  background.js        ← 탭 통신 중계
  icon.png             ← 아이콘 (16/48/128px)
```

---

## Task 1: 확장프로그램 기본 구조 생성

**파일:** `manifest.json`

```json
{
  "manifest_version": 3,
  "name": "MJ Hook",
  "version": "1.0",
  "description": "미드저니 이미지를 MJ Toolkit에 바로 저장합니다.",
  "permissions": ["storage", "tabs"],
  "host_permissions": [
    "https://www.midjourney.com/*",
    "https://cdn.midjourney.com/*"
  ],
  "content_scripts": [{
    "matches": ["https://www.midjourney.com/*"],
    "js": ["content.js"]
  }],
  "background": {
    "service_worker": "background.js"
  }
}
```

---

## Task 2: Content Script — 버튼 오버레이

**파일:** `content.js`

미드저니 이미지 카드를 감지하고 저장 버튼을 주입한다.

```javascript
// 이미지 카드 감지 (MutationObserver로 동적 로딩 대응)
const observer = new MutationObserver(() => injectButtons());
observer.observe(document.body, { childList: true, subtree: true });

function injectButtons() {
  // 미드저니 이미지 카드 셀렉터 (실제 DOM 확인 후 조정 필요)
  document.querySelectorAll('img[src*="cdn.midjourney.com"]').forEach(img => {
    if (img.dataset.mjHookInjected) return;
    img.dataset.mjHookInjected = 'true';

    const btn = document.createElement('button');
    btn.textContent = '🪝 Hook';
    btn.className = 'mj-hook-btn';
    btn.onclick = () => saveToToolkit(img, btn);

    img.parentElement.style.position = 'relative';
    img.parentElement.appendChild(btn);
  });
}
```

---

## Task 3: Content Script — 이미지 Blob 캡처 + 전송

**파일:** `content.js` (이어서)

```javascript
async function saveToToolkit(img, btn) {
  btn.textContent = '⏳ 저장 중...';
  btn.disabled = true;

  try {
    const imageUrl = img.src;
    const prompt = extractPrompt(img); // 주변 DOM에서 프롬프트 텍스트 추출

    // ① 이미지를 Blob으로 즉시 다운로드 (URL 만료 전에)
    const response = await fetch(imageUrl);
    const blob = await response.blob();
    const arrayBuffer = await blob.arrayBuffer();

    // ② 전송 payload 구성
    const payload = {
      imageBuffer: Array.from(new Uint8Array(arrayBuffer)),
      mimeType: blob.type,
      prompt: prompt,
      source: 'mj-hook',  // 툴킷 배지용 식별자
      savedAt: Date.now()
    };

    chrome.runtime.sendMessage({ type: 'SAVE_TO_TOOLKIT', payload });
    btn.textContent = '✅ 저장됨';
  } catch (e) {
    btn.textContent = '❌ 실패';
    btn.disabled = false;
  }
}

function extractPrompt(img) {
  // 미드저니 DOM 구조 확인 후 실제 셀렉터로 교체 필요
  const card = img.closest('[class*="card"], [class*="item"]');
  return card?.querySelector('[class*="prompt"]')?.textContent?.trim() || '';
}
```

---

## Task 4: Background Script — 탭 통신 중계

**파일:** `background.js`

```javascript
chrome.runtime.onMessage.addListener((msg, sender) => {
  if (msg.type !== 'SAVE_TO_TOOLKIT') return;

  // MJ Toolkit 탭을 찾아서 메시지 전달
  chrome.tabs.query({ url: '*://mj-toolkit.netlify.app/*' }, (tabs) => {
    if (tabs.length === 0) {
      // 툴킷 탭이 없으면 새로 열기
      chrome.tabs.create({ url: 'https://mj-toolkit.netlify.app' }, (tab) => {
        chrome.tabs.onUpdated.addListener(function listener(tabId, info) {
          if (tabId === tab.id && info.status === 'complete') {
            chrome.tabs.sendMessage(tabId, msg);
            chrome.tabs.onUpdated.removeListener(listener);
          }
        });
      });
    } else {
      chrome.tabs.sendMessage(tabs[0].id, msg);
    }
  });
});
```

---

## Task 5: MJ Toolkit 수정 — 메시지 수신 코드 추가

**파일:** `mj-toolkit.html` (기존 파일 수정)

기존 `addFiles()` 함수를 재사용해서 드래그&드롭과 동일한 경로로 저장한다.

```javascript
// 확장프로그램(MJ Hook) 메시지 수신
chrome.runtime?.onMessage?.addListener((msg) => {
  if (msg.type !== 'SAVE_TO_TOOLKIT') return;
  const { imageBuffer, mimeType, prompt, source } = msg.payload;

  // Blob 복원
  const blob = new Blob([new Uint8Array(imageBuffer)], { type: mimeType });
  const file = new File([blob], 'mj-hook-image.png', { type: mimeType });

  // 기존 addFiles()에 넘기기 (드래그&드롭과 동일한 저장 경로)
  // source: 'mj-hook' → 카드 배지에 🪝 Hook 표시
  addFiles([file], { prompt, source });
});
```

---

## 주의사항

1. **미드저니 DOM 셀렉터는 현장 확인 필요.**
   미드저니 UI가 수시로 바뀌므로 실제 `img` 태그 구조와 프롬프트 텍스트 위치는 개발 시 DevTools로 직접 확인해야 한다.

2. **Netlify URL 확정 필요.**
   background.js의 탭 검색 URL이 실제 배포 URL과 일치해야 한다. 현재 추정 URL: `mj-toolkit.netlify.app`

3. **저장 출처 구분.**
   `source: 'mj-hook'` 값을 툴킷 카드 렌더링 시 배지로 표시하려면 MJ Toolkit의 카드 렌더 함수도 소폭 수정 필요.

---

## 체크리스트

- [ ] Task 1: manifest.json 작성
- [ ] Task 2: content.js — 버튼 오버레이
- [ ] Task 3: content.js — Blob 캡처 + 전송
- [ ] Task 4: background.js — 탭 통신 중계
- [ ] Task 5: mj-toolkit.html — 메시지 수신 코드 추가
- [ ] 미드저니 실제 DOM 셀렉터 확인 및 교체
- [ ] Netlify 배포 URL 확정
- [ ] 크롬 확장프로그램 로컬 테스트 (chrome://extensions → 개발자 모드)
- [ ] MJ Toolkit Netlify 재배포

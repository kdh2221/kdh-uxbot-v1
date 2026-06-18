# UXBot 챗봇 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** inswave 진입화면(`index-immersive.html`)에 UX 컨설팅팀 문의를 응대하는 목업 챗봇(UXBot)을 추가한다.

**Architecture:** 단일 자기완결형 HTML 파일에 챗 위젯 UI + 목업 응답 엔진을 인라인으로 추가한다. LLM 호출은 `askUXBot()` 단일 어댑터로 추상화하며, 지금은 키워드/의도 매칭 기반 목업으로 구현하고, 추후 어댑터 본문만 Claude `claude-opus-4-8` 스트리밍 호출로 교체한다.

**Tech Stack:** 순수 HTML/CSS/JavaScript (의존성/빌드 없음). 기존 `index-immersive.html`의 다크+시안 브랜드 톤과 CSS 변수 재사용.

## Global Constraints

- 대상 파일: `c:\Users\user\.claude\kdh-trend-web-v1\index-immersive.html` (단일 파일, 인라인 CSS/JS)
- 외부 라이브러리/CDN 추가 금지 (기존 fullPage.js·구글폰트 외 신규 의존성 없음)
- 브랜드 토큰 재사용: `--accent:#6ce0ff`, `--bg:#06070b`, `--ink:#f3f4f7`, `--ink-soft:#9aa0ad`, `--line`, `--ease`
- `lang="ko"` 유지, 모든 UI 텍스트·더미 데이터 한국어
- 접근성: 키보드 포커스 가능, `:focus-visible` 표시, 본문 대비 확보, `prefers-reduced-motion: reduce` 시 모션 비활성
- 브라우저에서 `file://` 직접 열기로 동작해야 함 (빌드 단계 없음)
- 실제 Claude API 호출/키 처리/백엔드는 이번 범위 아님 (어댑터는 mock 모드만)
- 테스트 하니스 부재: 로직 검증은 `?selftest=1` 진입 시 콘솔 어서션으로, UI 검증은 브라우저 육안 확인으로 한다

---

### Task 1: 지식 베이스 + 의도 매칭 엔진

UX팀 지식 데이터와, 사용자 메시지를 의도(intent)로 분류해 답변 텍스트를 고르는 순수 함수를 만든다. 순수 로직이라 콘솔 어서션으로 검증한다.

**Files:**
- Modify: `c:\Users\user\.claude\kdh-trend-web-v1\index-immersive.html` (본문 끝 `</script>` 들 사이, 마지막 `<script>` 블록 앞에 새 `<script>` 추가)

**Interfaces:**
- Consumes: 없음
- Produces:
  - 전역 `window.UXBOT_KB` — `{ welcome: string, quickReplies: string[], fallback: string, intents: Array<{ id:string, keywords:string[], answer:string }> }`
  - 전역 함수 `matchUXIntent(message: string): { id: string, answer: string }` — 매칭 실패 시 `{ id: "fallback", answer: UXBOT_KB.fallback }`

- [ ] **Step 1: 자가 테스트 하니스(실패 상태)를 먼저 추가한다**

`index-immersive.html`에서 플로팅 챗 버튼(`<a class="chat" ...>`) 바로 아래, fullPage 스크립트(`<script src="https://cdn.jsdelivr.net/npm/fullpage.js@4/...">`) **앞에** 다음 블록을 추가한다:

```html
  <!-- ============ UXBOT: knowledge base + intent matcher ============ -->
  <script>
  (function(){
    "use strict";

    window.UXBOT_KB = {
      welcome: "안녕하세요! inswave UX 컨설팅팀 담당 챗봇이에요. 서비스 소개, 진행 절차, 기간·비용, 자주 묻는 질문까지 도와드릴게요. 무엇이 궁금하세요?",
      quickReplies: [
        "UX 컨설팅이 뭔가요?",
        "진행 절차가 궁금해요",
        "기간과 비용은 어떻게 되나요?",
        "원격으로도 진행되나요?"
      ],
      fallback: "정확한 답변을 위해 담당 컨설턴트에게 연결해 드리는 게 좋겠어요. 상단의 ‘도입 문의’ 또는 우측 하단 문의 버튼으로 남겨 주시면 빠르게 회신드릴게요.",
      intents: [
        {
          id: "service",
          keywords: ["뭐", "무엇", "소개", "서비스", "어떤", "ux 컨설팅", "컨설팅이"],
          answer: "inswave UX 컨설팅팀은 사용성 진단, UX 전략 수립, UI 설계, 디자인 시스템 구축까지 제품의 화면 경험 전반을 함께 설계합니다. 금융·공공·기업 서비스의 복잡한 화면을 더 쉽고 일관되게 만드는 데 강점이 있어요."
        },
        {
          id: "process",
          keywords: ["절차", "프로세스", "과정", "단계", "어떻게 진행", "진행 방식", "흐름"],
          answer: "보통 다음 5단계로 진행돼요. ① 문의·범위 협의 → ② 사용성 진단/리서치 → ③ UX 전략·UI 설계 → ④ 프로토타입 검증 → ⑤ 산출물 핸드오프. 프로젝트 성격에 따라 단계는 조정될 수 있습니다."
        },
        {
          id: "cost",
          keywords: ["비용", "가격", "견적", "기간", "얼마", "예산", "기간은"],
          answer: "기간과 비용은 범위에 따라 달라집니다. 소규모 진단형은 수 주, 설계까지 포함한 정규 프로젝트는 보통 1~3개월 범위예요. 정확한 견적은 화면 수·요구사항을 함께 검토해야 하니, ‘도입 문의’로 남겨 주시면 맞춤 견적을 드립니다."
        },
        {
          id: "remote",
          keywords: ["원격", "비대면", "온라인", "재택", "화상"],
          answer: "네, 원격으로도 진행 가능합니다. 킥오프와 주요 리뷰는 화상으로, 산출물과 진행 현황은 온라인으로 공유드려요. 필요 시 오프라인 워크숍도 병행합니다."
        },
        {
          id: "deliverable",
          keywords: ["산출물", "결과물", "납품", "무엇을 받", "받게"],
          answer: "주요 산출물은 사용성 진단 리포트, UX 전략 문서, 화면 설계(와이어프레임·프로토타입), 디자인 시스템/가이드입니다. 프로젝트 범위에 맞춰 구성해 드려요."
        },
        {
          id: "contact",
          keywords: ["문의", "상담", "연락", "신청", "도입"],
          answer: "도입 상담은 상단의 ‘도입 문의’ 버튼 또는 우측 하단 문의 버튼으로 신청하실 수 있어요. 남겨 주시면 담당 컨설턴트가 빠르게 회신드립니다."
        }
      ]
    };

    window.matchUXIntent = function(message){
      var text = String(message || "").toLowerCase().replace(/\s+/g, " ").trim();
      var kb = window.UXBOT_KB;
      for (var i = 0; i < kb.intents.length; i++) {
        var intent = kb.intents[i];
        for (var k = 0; k < intent.keywords.length; k++) {
          if (text.indexOf(intent.keywords[k].toLowerCase()) !== -1) {
            return { id: intent.id, answer: intent.answer };
          }
        }
      }
      return { id: "fallback", answer: kb.fallback };
    };

    // Self-test harness — runs only with ?selftest=1
    if (window.location.search.indexOf("selftest=1") !== -1) {
      var cases = [
        ["UX 컨설팅이 뭔가요?", "service"],
        ["진행 절차가 궁금해요", "process"],
        ["기간과 비용은 얼마인가요?", "cost"],
        ["원격으로도 진행되나요?", "remote"],
        ["어떤 산출물을 받게 되나요?", "deliverable"],
        ["상담 신청하고 싶어요", "contact"],
        ["오늘 점심 뭐 먹지?", "service"],
        ["asdfqwer 1234", "fallback"]
      ];
      var pass = 0;
      cases.forEach(function(c){
        var got = window.matchUXIntent(c[0]).id;
        var ok = got === c[1];
        if (ok) pass++;
        console.log((ok ? "PASS" : "FAIL") + " | \"" + c[0] + "\" → " + got + " (expected " + c[1] + ")");
      });
      console.log("UXBOT self-test: " + pass + "/" + cases.length + " passed");
    }
  })();
  </script>
```

- [ ] **Step 2: 자가 테스트를 실행해 결과를 확인한다**

브라우저에서 `index-immersive.html?selftest=1` 로 열고 콘솔(F12)을 확인한다.
Expected: `UXBOT self-test: 8/8 passed` (모든 케이스 PASS). "오늘 점심..." 케이스는 "뭐" 키워드로 `service`에 매칭되는 것이 의도된 동작이다.

만약 실패가 있으면 해당 intent의 `keywords`를 조정한 뒤 다시 실행한다.

- [ ] **Step 3: 커밋**

(git 미초기화 상태면 이 단계는 건너뛴다. 초기화돼 있으면 아래 실행.)

```bash
git add index-immersive.html docs/superpowers
git commit -m "feat(uxbot): add UX knowledge base and intent matcher with self-test"
```

---

### Task 2: 스트리밍 어댑터 `askUXBot()`

UI가 의존할 단일 진입점을 만든다. 응답 텍스트를 청크 단위로 흘려보내 LLM 스트리밍과 동일한 UX를 제공하고, 추후 교체 지점을 한 곳으로 고정한다.

**Files:**
- Modify: `c:\Users\user\.claude\kdh-trend-web-v1\index-immersive.html` (Task 1에서 추가한 `(function(){...})()` 내부, self-test 블록 바로 앞)

**Interfaces:**
- Consumes: `window.matchUXIntent` (Task 1)
- Produces:
  - 전역 `window.askUXBot(message: string, history: Array<{role,content}>, onChunk: (text:string)=>void): Promise<string>`
    - 동작(mock): `matchUXIntent`로 답변을 고른 뒤, 어절 단위로 `onChunk`를 순차 호출하며 점진 출력하고, 완료 시 전체 답변 문자열로 resolve 한다. `history`는 현재 mock에서 미사용(추후 LLM 멀티턴용 시그니처 보존).

- [ ] **Step 1: 어댑터 함수를 추가한다 (self-test 블록 앞)**

Task 1에서 추가한 IIFE 안, `// Self-test harness` 주석 줄 **바로 위**에 다음을 추가한다:

```javascript
    // Single swap point. mock now; later: Claude claude-opus-4-8 streaming via backend proxy.
    window.askUXBot = function(message, history, onChunk){
      var reduce = window.matchMedia("(prefers-reduced-motion: reduce)").matches;
      var full = window.matchUXIntent(message).answer;
      return new Promise(function(resolve){
        if (reduce) {
          if (typeof onChunk === "function") onChunk(full);
          resolve(full);
          return;
        }
        var words = full.split(" ");
        var i = 0, shown = "";
        var tick = function(){
          if (i >= words.length) { resolve(full); return; }
          shown += (i === 0 ? "" : " ") + words[i];
          i++;
          if (typeof onChunk === "function") onChunk(shown);
          setTimeout(tick, 28);
        };
        setTimeout(tick, 220); // brief "thinking" delay
      });
    };
```

- [ ] **Step 2: 어댑터 동작을 콘솔에서 검증한다**

브라우저에서 `index-immersive.html` 를 열고 콘솔에서 실행:

```javascript
askUXBot("진행 절차가 궁금해요", [], function(s){ console.log("chunk:", s); })
  .then(function(full){ console.log("DONE:", full); });
```

Expected: `chunk:` 로그가 누적되며 점점 길어지고(점진 출력), 마지막에 `DONE:` 으로 ② 사용성 진단... 이 포함된 process 답변 전문이 출력된다.

- [ ] **Step 3: 커밋**

```bash
git add index-immersive.html
git commit -m "feat(uxbot): add streaming askUXBot adapter (mock mode)"
```

---

### Task 3: 챗 위젯 UI (마크업 + 스타일)

플로팅 토글 버튼과 챗 패널의 정적 마크업·스타일을 추가한다. 이 단계에서는 열고 닫기만 동작시키고, 메시지 송수신은 Task 4에서 연결한다.

**Files:**
- Modify: `c:\Users\user\.claude\kdh-trend-web-v1\index-immersive.html`
  - CSS: `</style>` 직전에 위젯 스타일 추가
  - 마크업: 기존 `<a class="chat" ...>` 요소를 챗 위젯 마크업으로 교체

**Interfaces:**
- Consumes: 브랜드 CSS 변수 (Global Constraints)
- Produces:
  - DOM: `#uxbot-toggle`(버튼), `#uxbot-panel`(패널), `#uxbot-log`(메시지 리스트), `#uxbot-form`/`#uxbot-input`/`#uxbot-quick`(입력 영역), `#uxbot-close`(닫기)
  - 전역 함수 없음 (열고 닫기는 Task 3 인라인 스크립트, 송수신은 Task 4에서 확장)

- [ ] **Step 1: 위젯 CSS를 추가한다 (`</style>` 직전)**

`index-immersive.html`의 메인 `<style>` 블록 마지막, `</style>` 바로 위에 추가:

```css
  /* ============ UXBOT widget ============ */
  #uxbot-toggle{
    position:fixed;right:clamp(18px,3vw,34px);bottom:clamp(18px,3vw,34px);z-index:300;
    display:inline-flex;align-items:center;gap:10px;
    background:var(--accent);color:#05121a;font-weight:600;font-size:14px;font-family:"Inter",sans-serif;
    padding:13px 20px;border:none;border-radius:999px;cursor:pointer;
    box-shadow:0 14px 40px rgba(108,224,255,.4);
    transition:transform .2s var(--ease),box-shadow .25s;
  }
  #uxbot-toggle:hover{transform:translateY(-3px);box-shadow:0 18px 50px rgba(108,224,255,.55)}
  #uxbot-toggle .ping{width:9px;height:9px;border-radius:50%;background:#05121a;position:relative}
  #uxbot-toggle .ping::after{content:"";position:absolute;inset:0;border-radius:50%;background:#05121a;animation:ping 1.8s var(--ease) infinite}

  #uxbot-panel{
    position:fixed;right:clamp(18px,3vw,34px);bottom:calc(clamp(18px,3vw,34px) + 60px);z-index:300;
    width:min(380px,calc(100vw - 36px));height:min(560px,calc(100vh - 140px));
    display:flex;flex-direction:column;overflow:hidden;
    background:#0c0e14;border:1px solid var(--line);border-radius:18px;
    box-shadow:0 24px 70px rgba(0,0,0,.6);
    opacity:0;visibility:hidden;transform:translateY(16px) scale(.98);
    transition:opacity .3s var(--ease),transform .3s var(--ease),visibility .3s var(--ease);
  }
  #uxbot-panel.open{opacity:1;visibility:visible;transform:none}
  .uxbot-head{
    display:flex;align-items:center;justify-content:space-between;gap:10px;
    padding:16px 18px;border-bottom:1px solid var(--line);background:rgba(108,224,255,.06);
  }
  .uxbot-head .title{font-family:"Space Grotesk",sans-serif;font-weight:700;font-size:15px;display:flex;align-items:center;gap:9px}
  .uxbot-head .title .dot{width:8px;height:8px;border-radius:50%;background:var(--accent);box-shadow:0 0 12px var(--accent)}
  .uxbot-head .sub{font-size:11.5px;color:var(--ink-soft);margin-top:2px}
  #uxbot-close{background:none;border:none;color:var(--ink-soft);font-size:20px;line-height:1;cursor:pointer;padding:4px;border-radius:8px}
  #uxbot-close:hover{color:var(--ink)}

  #uxbot-log{flex:1;overflow-y:auto;padding:18px;display:flex;flex-direction:column;gap:12px}
  .uxbot-msg{max-width:84%;padding:11px 14px;border-radius:14px;font-size:14px;line-height:1.55;white-space:pre-wrap;word-break:break-word}
  .uxbot-msg.bot{align-self:flex-start;background:rgba(255,255,255,.05);color:var(--ink);border:1px solid var(--line);border-bottom-left-radius:4px}
  .uxbot-msg.user{align-self:flex-end;background:var(--accent);color:#05121a;font-weight:500;border-bottom-right-radius:4px}
  .uxbot-typing{align-self:flex-start;color:var(--ink-soft);font-size:13px;padding:4px 6px}
  .uxbot-typing span{display:inline-block;width:6px;height:6px;margin:0 1px;border-radius:50%;background:var(--ink-soft);animation:uxblink 1.2s infinite}
  .uxbot-typing span:nth-child(2){animation-delay:.2s}
  .uxbot-typing span:nth-child(3){animation-delay:.4s}
  @keyframes uxblink{0%,60%,100%{opacity:.3}30%{opacity:1}}

  #uxbot-quick{display:flex;flex-wrap:wrap;gap:8px;padding:0 18px 12px}
  .uxbot-chip{
    font-size:12.5px;color:var(--ink);background:rgba(255,255,255,.04);
    border:1px solid var(--line);border-radius:999px;padding:8px 13px;cursor:pointer;
    transition:border-color .2s,background .2s;
  }
  .uxbot-chip:hover{border-color:var(--accent);background:rgba(108,224,255,.08)}

  #uxbot-form{display:flex;gap:9px;padding:14px 16px;border-top:1px solid var(--line)}
  #uxbot-input{
    flex:1;background:rgba(255,255,255,.05);border:1px solid var(--line);border-radius:12px;
    color:var(--ink);font-size:14px;font-family:"Inter",sans-serif;padding:11px 14px;outline:none;
  }
  #uxbot-input:focus-visible{border-color:var(--accent)}
  #uxbot-send{
    background:var(--accent);color:#05121a;border:none;border-radius:12px;
    font-weight:600;font-size:14px;padding:0 16px;cursor:pointer;
  }
  #uxbot-send:disabled{opacity:.5;cursor:default}

  #uxbot-toggle:focus-visible,#uxbot-close:focus-visible,.uxbot-chip:focus-visible,#uxbot-send:focus-visible{
    outline:2px solid var(--accent);outline-offset:3px;
  }

  @media (prefers-reduced-motion:reduce){
    #uxbot-toggle .ping::after{display:none}
    #uxbot-panel{transition:opacity .01s,visibility .01s}
    .uxbot-typing span{animation:none}
  }
```

- [ ] **Step 2: 기존 플로팅 버튼을 위젯 마크업으로 교체한다**

`index-immersive.html`에서 다음 한 줄을 찾는다:

```html
  <a class="chat" href="#contact" aria-label="상담 문의"><span class="ping"></span>상담 문의</a>
```

이 줄을 아래 마크업으로 **교체**한다:

```html
  <!-- ============ UXBOT widget ============ -->
  <button id="uxbot-toggle" type="button" aria-expanded="false" aria-controls="uxbot-panel" aria-label="UX 담당 챗봇 열기">
    <span class="ping" aria-hidden="true"></span>UX 상담봇
  </button>
  <section id="uxbot-panel" role="dialog" aria-modal="false" aria-label="UX 담당 챗봇" aria-hidden="true">
    <header class="uxbot-head">
      <div>
        <div class="title"><span class="dot"></span>UX 담당봇</div>
        <div class="sub">UX 컨설팅 문의를 도와드려요</div>
      </div>
      <button id="uxbot-close" type="button" aria-label="챗봇 닫기">&times;</button>
    </header>
    <div id="uxbot-log" aria-live="polite"></div>
    <div id="uxbot-quick"></div>
    <form id="uxbot-form" autocomplete="off">
      <input id="uxbot-input" type="text" placeholder="궁금한 점을 입력하세요…" aria-label="메시지 입력" />
      <button id="uxbot-send" type="submit">전송</button>
    </form>
  </section>
```

- [ ] **Step 3: 열고 닫기 인라인 스크립트를 추가한다**

Task 1/2의 UXBOT IIFE `</script>` **다음**(여전히 fullPage 스크립트 앞)에 새 `<script>`를 추가한다:

```html
  <!-- ============ UXBOT: open/close wiring ============ -->
  <script>
  (function(){
    "use strict";
    var toggle = document.getElementById("uxbot-toggle");
    var panel = document.getElementById("uxbot-panel");
    var closeBtn = document.getElementById("uxbot-close");
    var input = document.getElementById("uxbot-input");

    function openPanel(){
      panel.classList.add("open");
      panel.setAttribute("aria-hidden", "false");
      toggle.setAttribute("aria-expanded", "true");
      if (input) input.focus();
    }
    function closePanel(){
      panel.classList.remove("open");
      panel.setAttribute("aria-hidden", "true");
      toggle.setAttribute("aria-expanded", "false");
      toggle.focus();
    }
    toggle.addEventListener("click", function(){
      if (panel.classList.contains("open")) closePanel(); else openPanel();
    });
    closeBtn.addEventListener("click", closePanel);
    document.addEventListener("keydown", function(e){
      if (e.key === "Escape" && panel.classList.contains("open")) closePanel();
    });

    // expose for Task 4
    window.__uxbotOpen = openPanel;
  })();
  </script>
```

- [ ] **Step 4: 열고 닫기를 브라우저에서 확인한다**

`index-immersive.html` 를 연다.
Expected: 우하단 "UX 상담봇" 버튼이 보인다 → 클릭하면 패널이 슬라이드업으로 열리고 입력창에 포커스가 간다 → 닫기(×) 또는 Esc로 닫힌다. (아직 메시지 전송/추천 질문은 동작하지 않아도 정상.)

- [ ] **Step 5: 커밋**

```bash
git add index-immersive.html
git commit -m "feat(uxbot): add chat widget markup, styles, and open/close wiring"
```

---

### Task 4: 대화 연결 (송수신 + 환영 + 추천 질문 + 스트리밍 렌더)

위젯을 `askUXBot()` 어댑터에 연결해 실제 대화가 흐르게 한다. 첫 오픈 시 환영 메시지와 추천 질문 칩을 렌더하고, 전송 시 사용자 말풍선 → 타이핑 인디케이터 → 스트리밍 봇 응답 순으로 처리한다.

**Files:**
- Modify: `c:\Users\user\.claude\kdh-trend-web-v1\index-immersive.html` (Task 3의 "open/close wiring" `<script>` 내부를 확장)

**Interfaces:**
- Consumes: `window.UXBOT_KB`, `window.askUXBot` (Task 1·2), Task 3의 DOM 요소들
- Produces: 동작하는 대화 UI (신규 전역 없음)

- [ ] **Step 1: wiring 스크립트를 대화 로직으로 확장한다**

Task 3에서 추가한 "open/close wiring" IIFE 전체를 아래로 **교체**한다:

```html
  <!-- ============ UXBOT: conversation wiring ============ -->
  <script>
  (function(){
    "use strict";
    var toggle = document.getElementById("uxbot-toggle");
    var panel = document.getElementById("uxbot-panel");
    var closeBtn = document.getElementById("uxbot-close");
    var log = document.getElementById("uxbot-log");
    var quick = document.getElementById("uxbot-quick");
    var form = document.getElementById("uxbot-form");
    var input = document.getElementById("uxbot-input");
    var sendBtn = document.getElementById("uxbot-send");

    var kb = window.UXBOT_KB;
    var history = [];
    var greeted = false;
    var busy = false;

    function scrollDown(){ log.scrollTop = log.scrollHeight; }

    function addMsg(role, text){
      var el = document.createElement("div");
      el.className = "uxbot-msg " + role;
      el.textContent = text;
      log.appendChild(el);
      scrollDown();
      return el;
    }

    function showTyping(){
      var el = document.createElement("div");
      el.className = "uxbot-typing";
      el.setAttribute("aria-hidden", "true");
      el.innerHTML = "<span></span><span></span><span></span>";
      log.appendChild(el);
      scrollDown();
      return el;
    }

    function renderQuickReplies(){
      quick.innerHTML = "";
      kb.quickReplies.forEach(function(q){
        var chip = document.createElement("button");
        chip.type = "button";
        chip.className = "uxbot-chip";
        chip.textContent = q;
        chip.addEventListener("click", function(){ if (!busy) send(q); });
        quick.appendChild(chip);
      });
    }

    function greet(){
      if (greeted) return;
      greeted = true;
      addMsg("bot", kb.welcome);
      renderQuickReplies();
    }

    function send(message){
      var text = String(message || "").trim();
      if (!text || busy) return;
      busy = true;
      sendBtn.disabled = true;
      quick.innerHTML = "";           // hide chips during a turn
      addMsg("user", text);
      history.push({ role: "user", content: text });
      input.value = "";

      var typing = showTyping();
      var botEl = null;

      window.askUXBot(text, history, function(partial){
        if (typing && typing.parentNode) { typing.parentNode.removeChild(typing); typing = null; }
        if (!botEl) botEl = addMsg("bot", "");
        botEl.textContent = partial;
        scrollDown();
      }).then(function(full){
        if (typing && typing.parentNode) { typing.parentNode.removeChild(typing); typing = null; }
        if (!botEl) botEl = addMsg("bot", full); else botEl.textContent = full;
        history.push({ role: "assistant", content: full });
        busy = false;
        sendBtn.disabled = false;
        renderQuickReplies();         // restore chips for next turn
        input.focus();
      });
    }

    function openPanel(){
      panel.classList.add("open");
      panel.setAttribute("aria-hidden", "false");
      toggle.setAttribute("aria-expanded", "true");
      greet();
      input.focus();
    }
    function closePanel(){
      panel.classList.remove("open");
      panel.setAttribute("aria-hidden", "true");
      toggle.setAttribute("aria-expanded", "false");
      toggle.focus();
    }

    toggle.addEventListener("click", function(){
      if (panel.classList.contains("open")) closePanel(); else openPanel();
    });
    closeBtn.addEventListener("click", closePanel);
    document.addEventListener("keydown", function(e){
      if (e.key === "Escape" && panel.classList.contains("open")) closePanel();
    });
    form.addEventListener("submit", function(e){
      e.preventDefault();
      send(input.value);
    });
  })();
  </script>
```

- [ ] **Step 2: 전체 대화 흐름을 브라우저에서 확인한다**

`index-immersive.html` 를 연다.
Expected:
1. "UX 상담봇" 클릭 → 패널 오픈, **환영 메시지** + **추천 질문 칩 4개** 표시
2. 추천 칩 "진행 절차가 궁금해요" 클릭 → 사용자 말풍선(우측, 시안) 생성 → 타이핑 인디케이터(…) → 봇 답변이 어절 단위로 점진 출력(process 답변)
3. 응답 중에는 입력 전송 버튼이 비활성, 완료 후 다시 활성 + 칩 복원
4. 입력창에 "비용 얼마인가요" 직접 입력 후 전송 → cost 답변 출력
5. "안녕 반가워" 같은 범위 밖 입력 → fallback 안내(문의 유도) 출력

- [ ] **Step 3: 커밋**

```bash
git add index-immersive.html
git commit -m "feat(uxbot): wire conversation flow with welcome, quick replies, streaming render"
```

---

### Task 5: 접근성·모션·최종 점검

위젯의 접근성과 reduced-motion 동작을 마무리하고, 기존 페이지(로더·fullPage 스내핑)와의 충돌이 없는지 회귀 점검한다.

**Files:**
- Modify: `c:\Users\user\.claude\kdh-trend-web-v1\index-immersive.html` (필요 시 미세 조정만)

**Interfaces:**
- Consumes: Task 1~4 산출물
- Produces: 없음 (검증·미세 조정 단계)

- [ ] **Step 1: reduced-motion 동작을 확인한다**

OS 또는 브라우저에서 "동작 줄이기(prefers-reduced-motion: reduce)"를 켠 상태로 `index-immersive.html` 를 연다.
Expected: 봇 응답이 어절 스트리밍 없이 **한 번에 전체 출력**(Task 2 어댑터의 reduce 분기), 토글 핑 애니메이션·타이핑 점멸 비활성, 패널 트랜지션 사실상 즉시.

- [ ] **Step 2: 키보드 접근성을 확인한다**

마우스 없이 Tab/Shift+Tab/Enter/Esc 로만 조작한다.
Expected: Tab으로 토글 버튼 도달 → Enter로 열기 → 입력창·전송·칩·닫기 버튼에 `:focus-visible` 외곽선이 보이며 순회 가능 → Enter로 전송 → Esc로 닫기 후 포커스가 토글 버튼으로 복귀.

- [ ] **Step 3: 기존 페이지 회귀 점검**

`index-immersive.html` 를 일반 모드로 연다.
Expected: 0→100% 로더 정상, fullPage 휠 스내핑 정상, 우측 섹션 도트 정상, 통계 카운트업 정상 — 챗 위젯 추가로 인한 깨짐이 없다. 챗 패널이 열려 있어도 배경 페이지 스크롤/네비가 정상이며, 패널은 페이지 위(z-index 300)에 올바르게 표시된다.

- [ ] **Step 4: self-test 재확인**

`index-immersive.html?selftest=1` 로 열고 콘솔에서 `UXBOT self-test: 8/8 passed` 를 재확인한다.

- [ ] **Step 5: 커밋**

```bash
git add index-immersive.html
git commit -m "chore(uxbot): verify a11y, reduced-motion, and page regression"
```

---

## 향후 작업 (이번 범위 밖)

LLM 전환 시 `askUXBot()` 본문만 교체: 모델 `claude-opus-4-8`, `/v1/messages` 스트리밍, UX팀 시스템 프롬프트(페르소나+지식+가드레일), 멀티턴(`history` 사용), 시스템 프롬프트 prompt caching. 키 노출 방지를 위해 **백엔드 프록시 경유**로 호출한다. (프론트 직접 호출은 데모 한정이며 `anthropic-dangerous-direct-browser-access` 헤더가 필요.)

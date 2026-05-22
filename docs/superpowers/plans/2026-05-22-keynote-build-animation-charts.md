# 슬라이드덱 개선 구현 계획 — 단계별 등장 · 차트 데이터화 · 시각 효과

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 단일 HTML 슬라이드덱(`index.html`)에 reveal.js식 클릭 단계별 등장(fragment), 데이터 기반 차트, 절제된 애니메이션을 더한다. 외부 JS 의존성 없이, 기존 내레이션·SFX·접근성 동작을 보존하면서.

**Architecture:** 기존 바닐라 JS `goToSlide(n)` 위에 `fragIndex` 상태와 `advance()`/`retreat()`를 얹어 슬라이드 내부 단계 진행을 처리한다. 차트는 `SURVEY` 데이터 객체 + 순수 기하 함수 + 렌더 함수로 분리해 숫자만 바꾸면 자동 재계산되게 한다. 모든 모션은 `prefers-reduced-motion`을 존중하고, 시각장애 발표자를 위해 fragment 등장 시 Web Audio 틱으로 청각 피드백을 준다.

**Tech Stack:** HTML5, 바닐라 JS(ES5 스타일 — 기존 코드 관례 유지: `var`, function 선언), CSS3, SVG, Web Audio API. 빌드 도구·외부 라이브러리 없음.

**기준 커밋:** `33996f8` (4차: 내레이션 추가). 작업 브랜치: `feat/build-animation-and-charts` (이미 생성됨, origin=aro-deeply fork, upstream=Engccer).

---

## 테스트 전략 (프로젝트 제약 반영)

이 프로젝트는 단일 HTML이며 테스트 프레임워크가 없다. Engccer 원본에 PR로 전달하므로 npm/jsdom 등 테스트 의존성을 추가하면 단일 HTML·무외부의존 원칙과 충돌한다. 따라서 검증은 두 가지로 한다:

1. **순수 기하 함수 콘솔 단언** — `scoreBarWidth`, `helpedBarWidth`, `donutSegments` 등 부작용 없는 계산 함수는 브라우저 콘솔에 단언 스니펫을 붙여 기존 손계산값과 일치하는지 객관적으로 확인한다(동치 검증).
2. **수동 체크리스트** — 상호작용(fragment 진행·키보드·스와이프·오디오)은 로컬 서버(`python -m http.server 8765`)에서 명시된 기대 동작으로 확인한다.

각 태스크는 "변경 → 검증 스니펫/수동 확인 → 커밋" 흐름을 따른다.

---

## File Structure

- **Modify:** `C:\Users\user\Desktop\projects\260523-keynote\index.html` (유일한 변경 파일)
  - `<style>` 블록(상단): fragment·차트 애니메이션·강조 효과·reduced-motion CSS 추가
  - 슬라이드 마크업(S8·S9·S11·S13·S14·S16·S19·S20): `class="fragment"` 추가
  - 차트 마크업(S17·S18): JS 렌더 대상 컨테이너로 단순화
  - `<script>` 블록(하단): `fragIndex` 상태, `advance`/`retreat`, `goToSlide` 확장, `playTick`, `SURVEY` + 차트 함수, 차트 애니메이션, 입력 재배선

단일 파일이므로 논리 영역(CSS / 마크업 / JS)별로 태스크를 나눈다.

---

## Task 1: Fragment·애니메이션 기반 CSS 추가

**Files:**
- Modify: `index.html` (`<style>` 블록 끝, `.progress-bar` 규칙 뒤 — 약 line 42 이후)

- [ ] **Step 1: CSS 규칙 추가**

`index.html`의 `<style>` 안, `.progress-bar { ... }` 규칙 바로 다음 줄에 추가:

```css
/* ===== Fragment (단계별 등장) ===== */
.fragment {
  opacity: 0;
  transform: translateY(12px);
  transition: opacity 0.45s ease, transform 0.45s ease;
}
.fragment.frag-visible {
  opacity: 1;
  transform: none;
}
/* 카드류는 살짝 scale 추가 */
.pillar.fragment, .closing-card.fragment, .student-card.fragment, .info-card.fragment {
  transform: translateY(12px) scale(0.96);
}
.pillar.frag-visible, .closing-card.frag-visible, .student-card.frag-visible, .info-card.frag-visible {
  transform: none;
}
/* key-chip 그룹: 컨테이너가 fragment, 칩은 내부 stagger */
.game-keys.fragment .key-chip {
  opacity: 0;
  transform: translateY(8px);
  transition: opacity 0.4s ease, transform 0.4s ease;
}
.game-keys.frag-visible .key-chip { opacity: 1; transform: none; }
.game-keys.frag-visible .key-chip:nth-child(1) { transition-delay: 0.00s; }
.game-keys.frag-visible .key-chip:nth-child(2) { transition-delay: 0.06s; }
.game-keys.frag-visible .key-chip:nth-child(3) { transition-delay: 0.12s; }
.game-keys.frag-visible .key-chip:nth-child(4) { transition-delay: 0.18s; }
.game-keys.frag-visible .key-chip:nth-child(5) { transition-delay: 0.24s; }
.game-keys.frag-visible .key-chip:nth-child(6) { transition-delay: 0.30s; }

/* ===== 강조·포컬 효과 ===== */
@keyframes frag-glow-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(245, 200, 66, 0); }
  35% { box-shadow: 0 0 22px 2px rgba(245, 200, 66, 0.55); }
}
.closing-card.highlight.frag-visible {
  animation: frag-glow-pulse 1.6s ease-in-out 1;
}

/* ===== prefers-reduced-motion: 모든 모션 제거, 최종 상태 즉시 ===== */
@media (prefers-reduced-motion: reduce) {
  .fragment, .fragment.frag-visible,
  .pillar.fragment, .pillar.frag-visible,
  .closing-card.fragment, .closing-card.frag-visible,
  .student-card.fragment, .student-card.frag-visible,
  .info-card.fragment, .info-card.frag-visible,
  .game-keys.fragment .key-chip, .game-keys.frag-visible .key-chip {
    transition: none !important;
    transform: none !important;
  }
  .game-keys.frag-visible .key-chip { transition-delay: 0s !important; }
  .closing-card.highlight.frag-visible { animation: none !important; }
}
```

- [ ] **Step 2: 수동 확인 (서버 실행)**

Run: `cd "C:\Users\user\Desktop\projects\260523-keynote"; python -m http.server 8765`
브라우저에서 `http://localhost:8765/` 열기.
Expected: 아직 `.fragment` 클래스를 단 요소가 없으므로 **화면은 기존과 100% 동일**. 콘솔 에러 없음. (CSS만 추가, 적용 대상 없음)

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "feat(css): fragment 등장·강조·reduced-motion 스타일 추가"
```

---

## Task 2: Fragment 상태 + advance/retreat + goToSlide 확장 + 입력 재배선

**Files:**
- Modify: `index.html` `<script>` 블록 — `var currentSlide = -1;` 인근(약 line 1238), `goToSlide`(약 line 1311), keydown/스와이프/버튼 핸들러(약 line 1346~1396)

이 태스크가 핵심이다. 기존 내레이션 호출 흐름(`stopNarration`/`playNarration`)을 깨지 않는다.

- [ ] **Step 1: 상태 변수 추가**

`var currentSlide = -1;` 다음 줄에 추가:

```js
var fragIndex = 0; // 현재 슬라이드에서 드러난 fragment 개수
```

- [ ] **Step 2: fragment 헬퍼 추가** (`goToSlide` 함수 정의 바로 위에 추가)

```js
function slideFragments(slide) {
  return slide.querySelectorAll('.fragment');
}

function setFragmentsVisible(slide, count) {
  var frags = slideFragments(slide);
  for (var i = 0; i < frags.length; i++) {
    if (i < count) frags[i].classList.add('frag-visible');
    else frags[i].classList.remove('frag-visible');
  }
}
```

- [ ] **Step 3: `goToSlide` 시그니처에 opts 추가**

기존:
```js
function goToSlide(n) {
  if (n < 0 || n >= TOTAL_SLIDES || n === currentSlide) return;
```
변경:
```js
function goToSlide(n, opts) {
  if (n < 0 || n >= TOTAL_SLIDES || n === currentSlide) return;
  opts = opts || {};
```

- [ ] **Step 4: `goToSlide` 안에서 fragment 초기화**

`newSlide.classList.add('active');` 다음, `newSlide.setAttribute('aria-hidden', 'false');` 뒤에 추가:

```js
  var fragCount = slideFragments(newSlide).length;
  if (opts.revealAll) {
    setFragmentsVisible(newSlide, fragCount);
    fragIndex = fragCount;
  } else {
    setFragmentsVisible(newSlide, 0);
    fragIndex = 0;
  }
```

- [ ] **Step 5: `goToSlide`의 버튼 disabled 로직 교체**

기존:
```js
  prevBtn.disabled = (n === 0);
  nextBtn.disabled = (n === TOTAL_SLIDES - 1);
```
변경(우선 단순화 — 매 진입 시 재계산은 advance/retreat가 담당):
```js
  updateNavDisabled();
```

그리고 `goToSlide` 함수 **뒤에** 추가:
```js
function updateNavDisabled() {
  var fragCount = slideFragments(allSlides[currentSlide]).length;
  prevBtn.disabled = (currentSlide === 0 && fragIndex === 0);
  nextBtn.disabled = (currentSlide === TOTAL_SLIDES - 1 && fragIndex >= fragCount);
}
```

- [ ] **Step 6: `advance` / `retreat` 추가** (`updateNavDisabled` 뒤)

```js
function advance() {
  var slide = allSlides[currentSlide];
  var fragCount = slideFragments(slide).length;
  if (fragIndex < fragCount) {
    var frag = slideFragments(slide)[fragIndex];
    frag.classList.add('frag-visible');
    fragIndex++;
    playTick();
    onFragmentRevealed(frag); // 차트/포커스 처리 (Task 5·6에서 구현, 우선 빈 함수)
    updateNavDisabled();
  } else {
    goToSlide(currentSlide + 1);
  }
}

function retreat() {
  if (fragIndex > 0) {
    fragIndex--;
    slideFragments(allSlides[currentSlide])[fragIndex].classList.remove('frag-visible');
    updateNavDisabled();
  } else {
    goToSlide(currentSlide - 1, { revealAll: true });
  }
}
```

- [ ] **Step 7: 임시 스텁 함수 추가** (`playTick`·`onFragmentRevealed`는 이후 태스크에서 채움 — 지금은 no-op로 두어 ReferenceError 방지)

`advance` 함수 위에 추가:
```js
function playTick() {} // Task 3에서 구현
function onFragmentRevealed(el) { // Task 6에서 차트 애니메이션 연결
  if (!el) return;
  el.setAttribute('tabindex', '-1');
  try { el.focus({ preventScroll: true }); } catch (e) {}
}
```

- [ ] **Step 8: 입력 핸들러 재배선 (keydown)**

기존 keydown switch 문에서 `goToSlide(currentSlide + 1)` / `goToSlide(currentSlide - 1)` 호출을 다음으로 교체:
- `ArrowRight`, `PageDown` → `e.preventDefault(); advance(); break;`
- `ArrowLeft`, `Backspace`, `PageUp` → `e.preventDefault(); retreat(); break;`
- `Home` → `e.preventDefault(); goToSlide(0); break;` (그대로)
- `End` → `e.preventDefault(); goToSlide(TOTAL_SLIDES - 1, { revealAll: true }); break;`

- [ ] **Step 9: 버튼·스와이프 재배선**

```js
prevBtn.addEventListener('click', function() { retreat(); });
nextBtn.addEventListener('click', function() { advance(); });
```
스와이프 핸들러 안:
```js
    if (dx < 0) advance();
    else retreat();
```

- [ ] **Step 10: 수동 확인**

서버에서 `http://localhost:8765/` 열기. (아직 `.fragment` 마크업이 없으므로 모든 슬라이드는 fragment 0개)
Expected:
- →/다음: 슬라이드가 한 장씩 넘어감(기존과 동일, fragment 없으니 즉시 다음 슬라이드)
- ←/이전: 이전 슬라이드로
- End: 마지막 슬라이드로, Home: 첫 슬라이드로
- 콘솔 에러 없음. 내레이션·page-turn 정상 재생.

- [ ] **Step 11: 커밋**

```bash
git add index.html
git commit -m "feat(js): fragment 상태·advance/retreat·goToSlide 확장·입력 재배선"
```

---

## Task 3: Web Audio 틱 효과음 (시각장애 발표자 청각 피드백)

**Files:**
- Modify: `index.html` `<script>` — Task 2에서 만든 `function playTick() {}` 스텁 교체

- [ ] **Step 1: AudioContext lazy 초기화 + playTick 구현**

`function playTick() {}` 를 다음으로 교체:

```js
var _audioCtx = null;
function getAudioCtx() {
  if (_audioCtx) return _audioCtx;
  var AC = window.AudioContext || window.webkitAudioContext;
  if (!AC) return null;
  try { _audioCtx = new AC(); } catch (e) { _audioCtx = null; }
  return _audioCtx;
}
function playTick() {
  if (!audioEnabled) return; // 오디오 토글과 연동
  var ctx = getAudioCtx();
  if (!ctx) return;
  if (ctx.state === 'suspended') { ctx.resume().catch(function(){}); }
  try {
    var osc = ctx.createOscillator();
    var gain = ctx.createGain();
    osc.type = 'sine';
    osc.frequency.setValueAtTime(880, ctx.currentTime); // page-turn·내레이션과 구별되는 짧고 높은 톤
    gain.gain.setValueAtTime(0.0001, ctx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.12, ctx.currentTime + 0.01);
    gain.gain.exponentialRampToValueAtTime(0.0001, ctx.currentTime + 0.12);
    osc.connect(gain); gain.connect(ctx.destination);
    osc.start();
    osc.stop(ctx.currentTime + 0.13);
  } catch (e) {}
}
```

- [ ] **Step 2: 수동 확인**

임시로 검증을 위해, 콘솔에서 직접 `playTick()` 호출 → 짧은 "틱" 소리 1회.
오디오 토글을 "소리 끔"으로 한 뒤 `playTick()` → 무음.
Expected: 토글 on일 때만 짧은 틱. 콘솔 에러 없음.
(자동재생 정책상 첫 사용자 클릭 이후 소리가 나는 것이 정상)

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "feat(js): fragment 등장 틱 효과음(Web Audio) 추가"
```

---

## Task 4: 다중 항목 슬라이드에 `.fragment` 마크업 추가

**Files:**
- Modify: `index.html` 슬라이드 마크업 S8·S9·S11·S13·S14·S16·S19·S20

규칙: "항목 하나 = fragment 하나". key-chip 묶음은 그룹 1개(내부 stagger). 헤딩·eyebrow·인용·링크는 슬라이드 진입 시 함께 표시(fragment 아님). S17·S18 차트는 Task 5·6에서 별도 처리하므로 여기서 건드리지 않는다.

- [ ] **Step 1: S8 — pillar 3장**

`<div class="pillar cyan">` → `<div class="pillar cyan fragment">`
`<div class="pillar gold">` → `<div class="pillar gold fragment">`
`<div class="pillar purple">` → `<div class="pillar purple fragment">`

- [ ] **Step 2: S9 — sense 5종**

S9의 5개 `<div class="sense">` 를 각각 `<div class="sense fragment">` 로.

- [ ] **Step 3: S11 — key-chip 그룹 + byline**

`<div class="game-keys">` → `<div class="game-keys fragment">`
`<p class="game-byline">` → `<p class="game-byline fragment">`

- [ ] **Step 4: S13 — S11과 동일 패턴**

S13의 `<div class="game-keys">` → `class="game-keys fragment"`, `<p class="game-byline">` → `class="game-byline fragment"`.
(주의: S13에 game-keys/game-byline이 있는지 먼저 Read로 확인 후 적용. 없으면 해당 슬라이드의 항목성 요소에 맞춰 fragment 부여하고 PR 노트에 기록.)

- [ ] **Step 5: S14 — S11과 동일 패턴**

S14의 `game-keys` → `+fragment`, `game-byline` → `+fragment` (Read로 구조 확인 후).

- [ ] **Step 6: S16 — key-chip 그룹 + tagline**

`<div class="game-keys" style="...">` → `<div class="game-keys fragment" style="...">`
`<p class="tagline" ...>` (S16의 것) → `class="tagline fragment"`.
(big-quote·라이브 링크는 진입 시 표시)

- [ ] **Step 7: S19 — student-card 2장**

두 `<div class="student-card">` (두 번째는 `style="border-left-color:#f5c842;"` 포함) 각각에 `fragment` 추가:
`<div class="student-card fragment">`, `<div class="student-card fragment" style="border-left-color:#f5c842;">`

- [ ] **Step 8: S20 — closing-card 4장**

4개 `<div class="closing-card">` (세 번째는 `class="closing-card highlight"`) 각각에 `fragment` 추가:
`<div class="closing-card fragment">` ×3, `<div class="closing-card highlight fragment">` ×1.
(closing-links·closing-thanks는 진입 시 표시)

- [ ] **Step 9: 수동 확인 (핵심)**

서버에서 각 슬라이드 점검:
- S8 진입 → pillar 안 보임. → 누를 때마다 pillar 하나씩 등장(아래에서 떠오르며 살짝 확대), 매번 틱 소리. 3번째 후 → 다음 슬라이드.
- S9 → sense 5개 하나씩.
- S11 → 진입 시 일러스트·헤드라인 보임, → 누르면 key-chip 묶음이 stagger로 등장, 다시 → byline.
- S16 → key-chip 묶음, tagline 순.
- S19 → student-card 2장 하나씩.
- S20 → closing-card 4장 하나씩, ③ highlight 카드 등장 시 glow 펄스 1회.
- 뒤로(←): 같은 슬라이드 내 마지막 항목부터 사라짐(무음). 항목 0개에서 ← → 이전 슬라이드를 **전부 드러난 상태**로.
- 내레이션: 슬라이드 넘어갈 때만 재생, fragment 진행 중엔 안 끊김.

- [ ] **Step 10: 커밋**

```bash
git add index.html
git commit -m "feat(markup): 다중 항목 슬라이드에 fragment 클래스 추가 (S8·9·11·13·14·16·19·20)"
```

---

## Task 5: 차트 데이터 객체 + 순수 기하 함수 + 렌더 (동치 검증)

**Files:**
- Modify: `index.html` — `<script>` 상단(`var SFX = {...}` 인근)에 `SURVEY` + 함수 추가; S17·S18 SVG 마크업을 렌더 대상으로 단순화

목표: 현재 수치로 렌더한 결과가 기존과 동일(발음 막대 1개 제외 — 아래 교정 노트). 애니메이션은 Task 6에서.

> **교정 노트:** 기존 S18 "발음 듣기·익히기" 막대는 다른 막대(`x=0`)와 달리 `x=180`에서 시작하고 행 간격이 좁다(손편집 흔적). 데이터 기반 렌더는 5개 막대를 균일하게(`x=0`, 동일 행 간격) 그리므로 이 막대가 정렬 교정된다. 이는 의도된 개선이며 핸드오프에서 사용자 확인을 받는다.

- [ ] **Step 1: `SURVEY` 데이터 객체 추가** (`var SFX = {...};` 다음)

```js
var SURVEY = {
  scores: [
    { label: '전체 만족도', avg: 4.35, positive: 85.4 },
    { label: '집중도',     avg: 4.17, positive: 80.8 },
    { label: '학습 도움',   avg: 3.94, positive: 72.3 }
  ],
  usage: [
    { label: '더 자주',    pct: 57.7, color: '#f5c842' },
    { label: '지금 정도',   pct: 36.9, color: '#5de6c8' },
    { label: '잘 모르겠음', pct: 5.4,  color: '#8b949e' },
    { label: '줄였으면',    pct: 0,    color: '#ff6b8a' }
  ],
  helped: [
    { label: '수업이 재미있어지는 데', pct: 53.8, color: '#f5c842' },
    { label: '단어 외우기',           pct: 50.8, color: '#5de6c8' },
    { label: '본문 독해',             pct: 47.7, color: '#5de6c8' },
    { label: '문법 이해',             pct: 37.7, color: '#a78bfa' },
    { label: '발음 듣기·익히기',       pct: 31.5, color: '#a78bfa' }
  ],
  meta: { n: 130, school: '신명중학교 1학년', when: '2026년 5월' }
};
```

- [ ] **Step 2: 순수 기하 함수 추가** (SURVEY 다음)

```js
var SVGNS = 'http://www.w3.org/2000/svg';

// S17 점수 막대: 0..5점 → 0..600px (x 시작 180)
function scoreBarWidth(avg) { return avg / 5 * 600; }
// S18 도움 막대: 0..100% → 0..290px (x 시작 0)
function helpedBarWidth(pct) { return pct / 100 * 290; }
// 도넛: 반지름 90, 둘레 2πr
var DONUT_R = 90;
var DONUT_C = 2 * Math.PI * DONUT_R; // 565.4866...
function donutSegments(segments) {
  var out = [], cum = 0;
  for (var i = 0; i < segments.length; i++) {
    var dash = segments[i].pct / 100 * DONUT_C;
    out.push({ color: segments[i].color, dash: dash, offset: -cum });
    cum += dash;
  }
  return out;
}
```

- [ ] **Step 2b: 동치 검증 (콘솔 단언 스니펫)**

브라우저 콘솔에 붙여넣기:
```js
console.assert(Math.round(scoreBarWidth(4.35)) === 522, 'score 4.35');
console.assert(Math.round(scoreBarWidth(4.17)) === 500, 'score 4.17');
console.assert(Math.round(scoreBarWidth(3.94)) === 473, 'score 3.94');
console.assert(Math.round(helpedBarWidth(53.8)) === 156, 'helped 53.8');
console.assert(Math.round(helpedBarWidth(31.5)) === 91, 'helped 31.5');
var ds = donutSegments(SURVEY.usage);
console.assert(Math.abs(ds[0].dash - 326.286) < 0.1, 'donut seg0');
console.assert(Math.abs(ds[1].dash - 208.665) < 0.1, 'donut seg1');
console.assert(Math.abs(ds[1].offset - (-326.286)) < 0.1, 'donut off1');
console.assert(Math.abs(ds[2].offset - (-534.951)) < 0.1, 'donut off2');
console.log('OK if no assertion failures above');
```
Expected: assertion 실패 메시지 없음.

- [ ] **Step 3: S17 SVG를 렌더 대상으로 단순화**

S17의 `<svg class="info-svg" ...>` 내부에서 **막대·수치 `<text>`·`<desc>`를 제거**하고 축(0~5 눈금선·숫자)과 각주만 남긴 뒤, `<svg ...>`에 `id="chart-scores"` 추가. 막대 그룹이 들어갈 빈 `<g id="chart-scores-bars"></g>`를 축 다음에 둔다. (라벨 텍스트·각주는 JS가 채우므로 비워도 됨 — Step 5의 renderBarChart가 라벨까지 그림)

구체: S17 svg를 다음으로 교체:
```html
<svg class="info-svg" id="chart-scores" viewBox="0 0 800 220" role="img"
     aria-label="" >
  <title>5점 만점 평균 점수</title>
  <desc id="chart-scores-desc"></desc>
  <line x1="180" y1="22" x2="780" y2="22" stroke="#30363d" stroke-width="1"/>
  <text x="180" y="15" fill="#8b949e" font-size="11" font-family="Noto Sans KR,sans-serif">0</text>
  <text x="300" y="15" fill="#8b949e" font-size="11" font-family="Noto Sans KR,sans-serif">1</text>
  <text x="420" y="15" fill="#8b949e" font-size="11" font-family="Noto Sans KR,sans-serif">2</text>
  <text x="540" y="15" fill="#8b949e" font-size="11" font-family="Noto Sans KR,sans-serif">3</text>
  <text x="660" y="15" fill="#8b949e" font-size="11" font-family="Noto Sans KR,sans-serif">4</text>
  <text x="773" y="15" fill="#8b949e" font-size="11" font-family="Noto Sans KR,sans-serif">5</text>
  <g id="chart-scores-bars"></g>
  <text x="400" y="210" text-anchor="middle" fill="#8b949e" font-size="12" font-family="Noto Sans KR,sans-serif" id="chart-scores-foot"></text>
</svg>
```

- [ ] **Step 4: S18 SVG 2개를 렌더 대상으로 단순화**

도넛 svg에 `id="chart-usage"`, 내부 동적 부분(회전 그룹의 색 호 circle들, 중앙 %텍스트, 범례, 강조박스)을 비우고 배경 트랙 circle과 빈 그룹만 남긴다:
```html
<svg class="info-svg" id="chart-usage" viewBox="0 0 420 230" role="img" aria-label="">
  <title>활용 희망 분포</title>
  <desc id="chart-usage-desc"></desc>
  <circle cx="110" cy="115" r="90" fill="none" stroke="#21262d" stroke-width="32"/>
  <g id="chart-usage-arcs" transform="rotate(-90 110 115)"></g>
  <text id="chart-usage-center" x="110" y="108" text-anchor="middle" fill="#5de6c8" font-size="28" font-weight="900" font-family="Noto Sans KR,sans-serif"></text>
  <text x="110" y="130" text-anchor="middle" fill="#8b949e" font-size="11" font-family="Noto Sans KR,sans-serif">유지 또는 확대</text>
  <g id="chart-usage-legend" font-family="Noto Sans KR,sans-serif" font-size="12"></g>
  <g id="chart-usage-zero"></g>
</svg>
```
도움 막대 svg에 `id="chart-helped"`, 내부를 빈 그룹으로:
```html
<svg class="info-svg" id="chart-helped" viewBox="0 0 420 230" role="img" aria-label="">
  <title>가장 도움 받은 영역</title>
  <desc id="chart-helped-desc"></desc>
  <g id="chart-helped-bars"></g>
</svg>
```

- [ ] **Step 5: 렌더 함수 구현** (기하 함수 다음)

```js
function svgEl(tag, attrs, text) {
  var el = document.createElementNS(SVGNS, tag);
  for (var k in attrs) if (attrs.hasOwnProperty(k)) el.setAttribute(k, attrs[k]);
  if (text != null) el.textContent = text;
  return el;
}
var FONT = 'Noto Sans KR,sans-serif';

// S17: 점수 가로 막대 (avg 기반)
function renderScores(animate) {
  var g = document.getElementById('chart-scores-bars');
  g.innerHTML = '';
  var rows = [62, 118, 174]; // 라벨/수치 baseline y
  var barY = [42, 98, 154];
  SURVEY.scores.forEach(function(d, i) {
    var w = scoreBarWidth(d.avg);
    g.appendChild(svgEl('text', { x:0, y:rows[i], fill:'#e6edf3', 'font-size':18, 'font-family':FONT, 'font-weight':500 }, d.label));
    g.appendChild(svgEl('rect', { x:180, y:barY[i], width:600, height:28, fill:'#21262d', rx:5 }));
    var bar = svgEl('rect', { x:180, y:barY[i], width: animate ? 0 : w, height:28, fill:'#f5c842', rx:5 });
    bar.setAttribute('data-w', w);
    bar.className.baseVal = 'chart-bar';
    g.appendChild(bar);
    var vx = 180 + w + 8;
    var valText = svgEl('text', { x:vx, y:rows[i], fill:'#f5c842', 'font-size':18, 'font-family':FONT, 'font-weight':900 }, animate ? '0.00' : d.avg.toFixed(2));
    valText.setAttribute('data-count', d.avg); valText.setAttribute('data-decimals', 2);
    g.appendChild(valText);
    g.appendChild(svgEl('text', { x:vx+45, y:rows[i], fill:'#5de6c8', 'font-size':13, 'font-family':FONT }, '('+d.positive.toFixed(1)+'%)'));
  });
  document.getElementById('chart-scores-foot').textContent = SURVEY.meta.school + ' ' + SURVEY.meta.n + '명 · ' + SURVEY.meta.when;
  var desc = SURVEY.scores.map(function(d){ return d.label+' '+d.avg.toFixed(2)+'점 긍정 '+d.positive.toFixed(1)+'퍼센트'; }).join(', ');
  document.getElementById('chart-scores-desc').textContent = desc;
  document.getElementById('chart-scores').setAttribute('aria-label', desc);
}

// S18 도넛
function renderUsage(animate) {
  var arcs = document.getElementById('chart-usage-arcs');
  var legend = document.getElementById('chart-usage-legend');
  var zero = document.getElementById('chart-usage-zero');
  arcs.innerHTML = ''; legend.innerHTML = ''; zero.innerHTML = '';
  var segs = donutSegments(SURVEY.usage);
  SURVEY.usage.forEach(function(d, i) {
    if (d.pct <= 0) return;
    var c = svgEl('circle', { cx:110, cy:115, r:DONUT_R, fill:'none', stroke:d.color, 'stroke-width':32,
      'stroke-dasharray': (animate ? 0 : segs[i].dash) + ' ' + DONUT_C, 'stroke-dashoffset': segs[i].offset });
    c.setAttribute('data-dash', segs[i].dash);
    arcs.appendChild(c);
  });
  var keep = SURVEY.usage[0].pct + SURVEY.usage[1].pct;
  document.getElementById('chart-usage-center').textContent = keep.toFixed(1) + '%';
  // 범례 (pct>0 항목만)
  var ly = 52;
  SURVEY.usage.forEach(function(d) {
    if (d.pct <= 0) return;
    legend.appendChild(svgEl('rect', { x:225, y:ly-12, width:14, height:14, fill:d.color, rx:2 }));
    var t = svgEl('text', { x:245, y:ly, fill:'#e6edf3' }, d.label + ' ');
    var ts = svgEl('tspan', { fill:d.color, 'font-weight':700 }, d.pct.toFixed(1) + '%');
    t.appendChild(ts); legend.appendChild(t);
    ly += 24;
  });
  // 줄였으면 0명 강조 박스 (마지막 usage 항목)
  var last = SURVEY.usage[SURVEY.usage.length - 1];
  zero.appendChild(svgEl('rect', { x:225, y:125, width:180, height:60, rx:8, fill:'rgba(255,107,138,0.12)', stroke:'#ff6b8a', 'stroke-width':1.5 }));
  zero.appendChild(svgEl('text', { x:315, y:148, 'text-anchor':'middle', fill:'#ff6b8a', 'font-size':13, 'font-weight':700, 'font-family':FONT }, last.label + ' 좋겠어요'));
  zero.appendChild(svgEl('text', { x:315, y:170, 'text-anchor':'middle', fill:'#ff6b8a', 'font-size':22, 'font-weight':900, 'font-family':FONT }, '0명'));
  var desc = SURVEY.usage.map(function(d){ return d.label+' '+d.pct.toFixed(1)+'%'; }).join(', ');
  document.getElementById('chart-usage-desc').textContent = desc;
  document.getElementById('chart-usage').setAttribute('aria-label', '활용 희망 분포 · ' + last.label + ' 0퍼센트');
}

// S18 도움 막대
function renderHelped(animate) {
  var g = document.getElementById('chart-helped-bars');
  g.innerHTML = '';
  var startY = 28, rowH = 48; // 균일 행 간격 (교정: 발음 막대 포함 전부 x=0 정렬)
  SURVEY.helped.forEach(function(d, i) {
    var y = startY + i * rowH;
    var w = helpedBarWidth(d.pct);
    g.appendChild(svgEl('text', { x:0, y:y, fill:'#e6edf3', 'font-size':13, 'font-family':FONT }, d.label));
    g.appendChild(svgEl('rect', { x:0, y:y+8, width:290, height:16, fill:'#21262d', rx:3 }));
    var bar = svgEl('rect', { x:0, y:y+8, width: animate ? 0 : w, height:16, fill:d.color, rx:3 });
    bar.setAttribute('data-w', w); bar.className.baseVal = 'chart-bar';
    g.appendChild(bar);
    g.appendChild(svgEl('text', { x:296, y:y+20, fill:d.color, 'font-size':13, 'font-weight':700, 'font-family':FONT }, d.pct.toFixed(1) + '%'));
  });
  var desc = SURVEY.helped.map(function(d){ return d.label+' '+d.pct.toFixed(1)+'%'; }).join(', ');
  document.getElementById('chart-helped-desc').textContent = desc;
  document.getElementById('chart-helped').setAttribute('aria-label', '도움 받은 영역 가로 막대 그래프');
}
```

> 주의: `startY=28, rowH=48`로 5행이 28,76,124,172,220 → viewBox 높이 230 안에 들어감. 기존 1~4행(36,84,132,180)과 약간 다르나 균일 정렬이 목적. 핸드오프에서 시각 확인.

- [ ] **Step 6: 초기 렌더 호출** (스크립트 맨 끝 `goToSlide(0);` **앞**에 추가)

```js
renderScores(false);
renderUsage(false);
renderHelped(false);
```

- [ ] **Step 7: 동치/시각 확인**

서버에서 S17·S18로 이동. Step 2b 콘솔 단언 통과.
Expected: S17 막대 3개·수치·각주가 기존과 동일. S18 도넛(94.6% 중앙·범례·0명 박스) 동일. 도움 막대 5개는 균일 정렬(발음 막대가 x=0으로 교정됨).

- [ ] **Step 8: 커밋**

```bash
git add index.html
git commit -m "refactor(charts): SURVEY 데이터 객체 + 렌더 함수로 S17·S18 차트 데이터화"
```

---

## Task 6: 차트 진입 애니메이션 (막대 grow · 도넛 arc · count-up) + fragment 연동

**Files:**
- Modify: `index.html` `<script>` — 애니메이션 헬퍼 추가, `onFragmentRevealed`·`goToSlide`에서 차트 트리거

- [ ] **Step 1: 애니메이션 헬퍼 추가** (렌더 함수 다음)

```js
function prefersReducedMotion() {
  return window.matchMedia && window.matchMedia('(prefers-reduced-motion: reduce)').matches;
}

function animateCountUp(textEl) {
  var target = parseFloat(textEl.getAttribute('data-count'));
  var decimals = parseInt(textEl.getAttribute('data-decimals') || '0', 10);
  if (prefersReducedMotion()) { textEl.textContent = target.toFixed(decimals); return; }
  var dur = 800, start = null;
  function frame(ts) {
    if (start === null) start = ts;
    var p = Math.min((ts - start) / dur, 1);
    textEl.textContent = (target * p).toFixed(decimals);
    if (p < 1) requestAnimationFrame(frame);
    else textEl.textContent = target.toFixed(decimals);
  }
  requestAnimationFrame(frame);
}

function animateBars(containerId) {
  var bars = document.querySelectorAll('#' + containerId + ' .chart-bar');
  for (var i = 0; i < bars.length; i++) {
    (function(bar) {
      var w = parseFloat(bar.getAttribute('data-w'));
      if (prefersReducedMotion()) { bar.setAttribute('width', w); return; }
      bar.style.transition = 'none';
      bar.setAttribute('width', 0);
      requestAnimationFrame(function() {
        bar.style.transition = 'width 0.6s ease-out';
        bar.setAttribute('width', w);
      });
    })(bars[i]);
  }
}

function animateDonut(arcsId) {
  var arcs = document.querySelectorAll('#' + arcsId + ' circle');
  for (var i = 0; i < arcs.length; i++) {
    (function(c) {
      var dash = parseFloat(c.getAttribute('data-dash'));
      if (prefersReducedMotion()) { c.setAttribute('stroke-dasharray', dash + ' ' + DONUT_C); return; }
      c.style.transition = 'none';
      c.setAttribute('stroke-dasharray', '0 ' + DONUT_C);
      requestAnimationFrame(function() {
        c.style.transition = 'stroke-dasharray 0.7s ease-out';
        c.setAttribute('stroke-dasharray', dash + ' ' + DONUT_C);
      });
    })(arcs[i]);
  }
}

// 슬라이드/카드의 차트를 (재렌더 없이) 애니메이트
function animateChartsIn(root) {
  if (root.querySelector('#chart-scores-bars')) {
    animateBars('chart-scores-bars');
    var sv = root.querySelectorAll('#chart-scores-bars text[data-count]');
    for (var i=0;i<sv.length;i++) animateCountUp(sv[i]);
  }
  if (root.querySelector('#chart-usage-arcs')) animateDonut('chart-usage-arcs');
  if (root.querySelector('#chart-helped-bars')) animateBars('chart-helped-bars');
}
```

- [ ] **Step 2: S17 — 슬라이드 진입 시 차트 애니메이트**

`goToSlide` 안, fragment 초기화 직후에 추가:
```js
  // S17은 fragment가 없으니 진입 시 차트 애니메이트
  if (newSlide.id === 'slide-17') animateChartsIn(newSlide);
```

- [ ] **Step 3: S18 — fragment(info-card) 등장 시 해당 카드 차트 애니메이트**

`onFragmentRevealed`를 확장:
```js
function onFragmentRevealed(el) {
  if (!el) return;
  if (el.querySelector && (el.querySelector('#chart-usage-arcs') || el.querySelector('#chart-helped-bars'))) {
    animateChartsIn(el);
  }
  el.setAttribute('tabindex', '-1');
  try { el.focus({ preventScroll: true }); } catch (e) {}
}
```

- [ ] **Step 4: S18 info-card를 fragment로 마크업**

S18의 두 `<div class="info-card">` 에 `fragment` 추가, 그리고 하단 `<p class="tagline" ...>` 에도 `fragment` 추가:
- `<div class="info-card fragment">` (도넛 카드)
- `<div class="info-card fragment">` (도움 막대 카드)
- `<p class="tagline fragment" style="margin-top:2vh; max-width:900px;">`

(S18 차트는 진입 시 0폭으로 렌더되어 있다가 카드 fragment가 드러날 때 grow. 단, 진입 시 `renderUsage(false)`로 이미 최종폭일 수 있으므로 Step 5에서 진입 시 0폭 초기화 처리.)

- [ ] **Step 5: S18 진입 시 차트 0폭 초기화**

`goToSlide` 안에 추가(S17 처리 근처):
```js
  if (newSlide.id === 'slide-18' && !prefersReducedMotion() && !opts.revealAll) {
    // 카드가 드러날 때 grow하도록 0폭으로 리셋
    var ub = newSlide.querySelectorAll('.chart-bar');
    for (var bi=0; bi<ub.length; bi++) ub[bi].setAttribute('width', 0);
    var ua = newSlide.querySelectorAll('#chart-usage-arcs circle');
    for (var ai=0; ai<ua.length; ai++) ua[ai].setAttribute('stroke-dasharray', '0 ' + DONUT_C);
  }
```
(`revealAll`로 뒤에서 진입한 경우엔 최종 상태 유지 — Task 5의 render가 최종폭이므로 OK)

- [ ] **Step 6: 수동 확인**

- S17 진입: 막대 3개가 0→목표로 grow, 수치 0.00→4.35 등 count-up.
- S18 진입: 차트 비어있다가 첫 → 에 도넛 카드 등장+호 그려짐, 다음 → 막대 카드 등장+막대 grow, 다음 → tagline.
- 뒤로 S18 재진입(revealAll): 차트 최종 상태로 바로 보임.
- reduced-motion on(OS 설정 또는 DevTools rendering 탭): 애니메이션 없이 최종값 즉시.

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "feat(charts): 막대 grow·도넛 arc·count-up 애니메이션 + fragment 연동"
```

---

## Task 7: 강조·포컬 효과 (키워드 glow)

**Files:**
- Modify: `index.html` `<style>` (Task 1 블록에 이미 S20 ③ 펄스 있음 — 여기선 키워드 glow 추가)

- [ ] **Step 1: 키워드 등장 glow CSS 추가** (Task 1 강조 블록에 이어서)

```css
@keyframes kw-pop {
  0% { text-shadow: 0 0 0 rgba(245,200,66,0); }
  40% { text-shadow: 0 0 14px rgba(245,200,66,0.6); }
  100% { text-shadow: 0 0 0 rgba(245,200,66,0); }
}
.frag-visible .gold, .pillar.frag-visible .pillar-name {
  animation: kw-pop 1.2s ease-out 1;
}
@media (prefers-reduced-motion: reduce) {
  .frag-visible .gold, .pillar.frag-visible .pillar-name { animation: none !important; }
}
```

- [ ] **Step 2: 수동 확인**

fragment가 등장할 때 그 안의 `.gold` 키워드/원칙명에 은은한 glow가 1회 번쩍이고 사라짐. 무한 반복 없음. reduced-motion에서 없음.
Expected: 과하지 않은 1회 강조. 기존 색상 유지.

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "feat(css): fragment 등장 시 키워드 glow 강조(1회) 추가"
```

---

## Task 8: 전체 검증 패스 + push + 데이터 수령 대기

**Files:** 없음(검증·푸시)

- [ ] **Step 1: 전체 수동 체크리스트** (스펙 §9)

서버에서 처음부터 끝까지:
- [ ] 키보드 →/←/PageUp/Down/Home/End 정상, 뒤로 시 이전 슬라이드 전부 표시
- [ ] fragment 진행 중 내레이션 안 끊김(슬라이드 전환에서만 재생)
- [ ] fragment 등장 시 틱, 오디오 토글 off면 틱·내레이션 함께 정지, on 복원 시 현재 슬라이드 내레이션 재생
- [ ] S17·S18 차트 애니메이션, 콘솔 단언 통과
- [ ] 모바일 뷰포트(DevTools 360×640): 좌우 스와이프 fragment 진행, 세로 스크롤, 44px 터치 타깃
- [ ] `prefers-reduced-motion: reduce`: 전 애니메이션 즉시(틱·포커스 이동 유지)
- [ ] 콘솔 에러 0

- [ ] **Step 2: 원본 대비 diff 확인**

```bash
git diff upstream/main --stat
```
Expected: `index.html` + `docs/` 만 변경. narration/·sfx/·_generate_narration.py 무변경.

- [ ] **Step 3: fork로 push**

```bash
git push -u origin feat/build-animation-and-charts
```

- [ ] **Step 4: 데이터 수령 대기 (15:30 이후)**

새 설문 데이터 수령 시: `SURVEY` 객체의 `scores`/`usage`/`helped` 숫자와 `meta.n`만 교체 → 저장 → 콘솔 단언 재실행 → 시각 확인. 마크업·함수 변경 불필요.

- [ ] **Step 5: 데이터 반영 커밋 + PR**

```bash
git add index.html
git commit -m "data: 설문 응답자 추가분 반영 (수치 갱신)"
git push origin feat/build-animation-and-charts
gh pr create --repo Engccer/260523-keynote --base main --head aro-deeply:feat/build-animation-and-charts \
  --title "단계별 등장·차트 데이터화·시각 효과 추가" \
  --body "reveal.js식 fragment 단계 등장, 차트 데이터 기반 리팩터링(+애니메이션), 절제된 시각 효과. 내레이션·접근성 보존, prefers-reduced-motion 존중, 시각장애 발표자용 틱 피드백."
```

> docs/ 폴더(스펙·계획)를 PR에 포함할지 여부는 push 전 사용자와 결정. 제외하려면 별도 브랜치에서 docs 커밋을 drop하거나 `.gitignore` 처리.

---

## Self-Review 결과

- **스펙 커버리지:** §4 fragment→Task 2·4, 틱→Task 3, §5 차트 데이터화→Task 5, §6 애니메이션→Task 6·7, reduced-motion→전 태스크, §7 워크플로→Task 8. 누락 없음.
- **타입 일관성:** `slideFragments`/`setFragmentsVisible`/`advance`/`retreat`/`updateNavDisabled`/`playTick`/`onFragmentRevealed`/`renderScores`/`renderUsage`/`renderHelped`/`scoreBarWidth`/`helpedBarWidth`/`donutSegments`/`animateChartsIn` — 정의·호출 명칭 일치 확인.
- **플레이스홀더:** 없음. 모든 코드 단계에 실제 코드 포함.
- **알려진 교정:** S18 발음 막대 정렬 정규화(Task 5) — 핸드오프에서 사용자 확인 필요.
- **선/후 작업 분리:** Task 1~7 + Task 8 Step 1~3은 데이터 전 선작업. Step 4~5는 데이터 수령 후.

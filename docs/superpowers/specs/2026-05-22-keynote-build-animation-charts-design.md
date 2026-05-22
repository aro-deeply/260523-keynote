# 슬라이드덱 개선 설계 — 단계별 등장 · 차트 데이터화 · 시각 효과

- **작성일**: 2026-05-22
- **대상**: `index.html` (2026 교사 해커톤 기조강연 슬라이드덱, 단일 HTML)
- **기준 커밋**: `33996f8` (4차 수정: 슬라이드별 핵심 키워드 음성 내레이션 추가)
- **원본**: `Engccer/260523-keynote` (원본 보존 · fork → PR 전달)
- **작업자**: aro-deeply

## 1. 목적 · 배경

현재 덱은 슬라이드 전환 시 내부 콘텐츠가 한꺼번에 나타나 가독성·몰입도가 떨어진다.
또한 차트(S17/S18)는 막대 너비·도넛 `stroke-dasharray` 값이 SVG에 손으로 박혀 있어,
설문 응답자가 늘어 숫자가 바뀌면 수작업으로 다시 계산해야 한다.

본 작업은 (1) reveal.js식 클릭 단계별 등장, (2) 차트 데이터 기반 리팩터링,
(3) 절제된 시각 효과를 더해 발표 효과를 높이되, 외부 의존 없는 단일 HTML 원칙과
원본 대비 작은 diff(PR 검토 용이)를 유지한다.

설문 수정 데이터는 2026-05-22 15:30 이후 수령 예정. 그 전까지 데이터 외 모든 작업을
선행하고, 데이터 수령 후 `SURVEY` 객체 숫자만 교체해 마무리한다.

### 기준 베이스라인 (4차 커밋 반영)
- `index.html`에 **슬라이드별 핵심 키워드 내레이션** 로직이 이미 존재한다:
  `currentNarration`, `narrationDelayTimer`, `NARRATION_DELAY_MS(450)`, `pad2()`,
  `stopNarration()`, `playNarration(slideIndex)`. `goToSlide(n)`이 진입 시
  `stopNarration()` 후 `playNarration(n)`을 호출(page-turn SFX 후 ~450ms 딜레이).
- 내레이션은 **슬라이드 단위**(`narration/slide-NN.mp3`, 슬라이드당 1개)이며,
  **시각장애 발표자가 현재 슬라이드 위치를 청각으로 확인하는 핵심 수단**이다.
  데스크 스피커로 청중에게도 들려 "슬라이드 표제"처럼 작동한다.
- 내레이션 자산(`narration/`, `_generate_narration.py`)은 본 작업에서 **수정/재생성하지 않는다.**

## 2. 성공 기준

- 다중 항목 슬라이드에서 →/다음 클릭 시 항목이 하나씩 등장하고, 다 등장하면 다음 슬라이드로 넘어간다.
- 뒤로 이동 시 이전 슬라이드는 전부 드러난 상태로 표시된다.
- 슬라이드 안에서 항목 하나가 드러날 때 **은은한 틱 효과음**으로 청각 피드백을 준다(오디오 토글 연동).
- fragment를 드러내는 동작은 **슬라이드 내레이션을 중단·재시작하지 않는다**(내레이션은 슬라이드 진입 시에만 재생).
- 차트는 `SURVEY` 데이터 객체에서 자동 렌더되며, 숫자만 바꾸면 막대·도넛·수치·desc가 자동 갱신된다.
- 리팩터링 직후(데이터 변경 전) 차트 시각 결과는 기존과 동일하다(애니메이션 종료 상태 기준).
- `prefers-reduced-motion: reduce` 환경에서 모든 애니메이션이 최종 상태로 즉시 표시된다.
- 외부 JS 의존성을 추가하지 않는다(단일 HTML 유지).

## 3. 비목표 (YAGNI)

- 슬라이드 간 전환 효과 고도화(현재 0.5s fade+scale 유지).
- reveal.js 등 외부 프레임워크 도입.
- 설문 항목 자체의 추가/삭제(항목 수 고정, 숫자만 변동).
- 내레이션 mp3·`_generate_narration.py` 수정/재생성(기존 자산 보존, 재생 로직만 fragment와 통합).

## 4. Fragment(단계별 등장) 시스템

### 상태
- 기존 `currentSlide`에 `fragIndex`(현재 슬라이드에서 드러난 fragment 개수) 추가.

### 진행 함수
- `advance()`
  - 현재 슬라이드에 안 드러난 `.fragment`가 있으면 다음 하나에 `.frag-visible` 부여, `fragIndex++`,
    **틱 효과음 1회 재생**(`playTick()`), 해당 요소로 포커스 이동. **`goToSlide` 미호출 → 내레이션 유지.**
  - 모두 드러났으면 `goToSlide(currentSlide + 1)` (새 슬라이드는 fragment 0개로 진입, 내레이션 재생).
- `retreat()`
  - 드러난 fragment가 있으면 마지막 하나의 `.frag-visible` 제거, `fragIndex--`. (뒤로 가기는 무음 — 틱 없음)
  - 없으면 `goToSlide(currentSlide - 1, { revealAll: true })` (이전 슬라이드는 전부 드러난 채로, 내레이션 재생).

### goToSlide 변경 (기존 함수 확장)
- 시그니처에 `opts.revealAll` 옵션 추가.
- 진입 시 해당 슬라이드의 `.fragment`를 모두 숨김(기본) 또는 모두 표시(`revealAll`).
- `fragIndex`를 0(기본) 또는 fragment 총개수(`revealAll`)로 설정.
- 기존 `stopCurrentAudio()` / `stopNarration()` / `playNarration(n)` 호출 흐름은 그대로 유지.
- Home → `goToSlide(0)`, End → `goToSlide(TOTAL-1, { revealAll: true })`.

### 입력 연결
- 앞으로: ArrowRight · PageDown · 다음 버튼 · 왼쪽 스와이프 → `advance()`
- 뒤로: ArrowLeft · Backspace · PageUp · 이전 버튼 · 오른쪽 스와이프 → `retreat()`
- `nextBtn.disabled`: "마지막 슬라이드의 마지막 fragment까지 드러난 상태"에서만 true.
- `prevBtn.disabled`: "첫 슬라이드 fragment 0개" 상태에서만 true.

### 틱 효과음 (청각 피드백)
- 시각장애 발표자가 "항목이 추가됐음"을 청각으로 확인하도록, fragment 등장 시 짧은 틱 1회.
- **Web Audio API로 즉석 생성**(별도 음원 파일 없음 → 단일 HTML 유지). page-turn·내레이션과 구별되는 짧고 낮은 볼륨의 톤.
- `audioEnabled`가 true일 때만 재생(오디오 토글과 연동). `prefers-reduced-motion`과는 무관하게 동작(모션이 아니라 소리이므로, 발표자 위치 확인용으로 유지).

### 마크업
- 다중 항목 슬라이드의 각 항목에 `class="fragment"` 추가. 적용 후보:
  - S8 pillar 카드 3장 / S9 sense 5종 / S11·S13·S14·S16 key-chip
  - S17 막대 3개 / S18 도넛 범례·강조박스·막대 / S19 student-card 2장 / S20 closing 카드 4장
- 표지(S1)·한 문장/인용 슬라이드(S6·S7·S10 등)는 손대지 않음(한 번에 등장).
- 최종 적용 대상은 구현 중 슬라이드별로 확정하고 PR 설명에 목록화.

### 접근성
- fragment 등장 시 해당 요소에 `tabindex=-1` 부여 후 포커스 이동(스크린리더가 새 내용 인지).
- 틱 효과음으로 스피커 청취 발표자에게도 즉각 피드백.
- `prefers-reduced-motion: reduce`이면 이동 모션 없이 즉시 표시(포커스 이동·틱은 유지).
- 진행률 바는 슬라이드 단위 유지(fragment 미반영) — 단순·예측 가능.

### CSS
```css
.fragment { opacity: 0; transform: translateY(12px); transition: opacity .45s ease, transform .45s ease; }
.fragment.frag-visible { opacity: 1; transform: none; }
@media (prefers-reduced-motion: reduce) {
  .fragment { transition: none; transform: none; }
}
```

## 5. 차트 데이터 기반 리팩터링

### 데이터 객체 (스크립트 상단)
```js
var SURVEY = {
  scores: [ // S17 가로 막대 (5점 만점 평균, 괄호=긍정 4~5점 비율)
    { label:'전체 만족도', avg:4.35, positive:85.4 },
    { label:'집중도',     avg:4.17, positive:80.8 },
    { label:'학습 도움',   avg:3.94, positive:72.3 }
  ],
  usage: [ // S18 도넛 (앞으로도 활용했으면?)
    { label:'더 자주',    pct:57.7, color:'#f5c842' },
    { label:'지금 정도',   pct:36.9, color:'#5de6c8' },
    { label:'잘 모르겠음', pct:5.4,  color:'#8b949e' },
    { label:'줄였으면',    pct:0,    color:'#ff6b8a' }
  ],
  helped: [ // S18 가로 막대 (가장 도움 받은 영역, 복수 응답)
    { label:'수업이 재미있어지는 데', pct:53.8 },
    { label:'단어 외우기',           pct:50.8 },
    { label:'본문 독해',             pct:47.7 },
    { label:'문법 이해',             pct:37.7 },
    { label:'발음 듣기·익히기',       pct:31.5 }
  ],
  meta: { n:130, school:'신명중학교 1학년', when:'2026년 5월' }
};
```
- 항목 수는 고정(레이아웃 안정). 응답자 증가 시 숫자만 변동.

### 렌더 함수
- `renderBarChart(svgEl, items, opts)` — pct(또는 avg/5)를 막대 픽셀 너비로 자동 계산, 라벨·수치 텍스트 배치.
- `renderDonut(svgEl, segments)` — 둘레 `2πr`에서 각 조각의 `stroke-dasharray`/`dashoffset` 자동 계산(현재 손계산값 326.286 등 대체). 중앙 "유지 또는 확대 %"와 "줄였으면 0명" 강조 박스 포함.
- `meta`에서 표본 수·각주 텍스트, desc(스크린리더 설명) 자동 생성.

### SVG 구조 처리
- viewBox·축 눈금·title은 마크업 유지.
- 데이터로 변하는 `<rect>`(막대)·`<circle>`(도넛 호)·`<text>`(수치)·`<desc>`만 JS가 생성/갱신.

### 동치 검증
- 리팩터링 직후 현재 수치로 렌더한 결과가 기존 SVG와 시각적으로 동일한지 확인(애니메이션 종료 상태).

## 6. 시각 효과

### ① 차트 애니메이션 (슬라이드/fragment 진입 시)
- 막대: 너비 0 → 목표 (0.6s ease-out).
- 도넛: 호 0 → 목표 (`stroke-dashoffset` 트랜지션).
- 수치: 0 → 목표 count-up (`requestAnimationFrame`, ~0.8s). 소수점 표기 유지(4.35, 85.4%).
- "줄였으면 0명" 강조 박스는 마지막 단계에 등장.

### ② 빌드 등장 모션 (fragment 공통)
- 기본: opacity 0→1 + translateY(12px)→0, 0.45s ease.
- 카드류: scale(0.96)→1 추가.

### ③ 강조·포컬 효과 (절제)
- 핵심 키워드·수치: 등장 시 1회 glow(text-shadow)/색상 pop. 상시 반복 없음.
- S20 핵심 가치 ③ 하이라이트 카드: 등장 시 테두리 glow 펄스 1~2회 후 정지.

### 공통
- `prefers-reduced-motion: reduce`: 모든 grow/count-up/slide-up/glow 비활성, 최종 상태 즉시 표시.
- 기존 particles · 0.5s fade+scale 전환 · page-turn 효과음 · **슬라이드 내레이션 자동 재생** 보존.

## 7. Fork / PR 워크플로

1. `Engccer/260523-keynote`를 aro-deeply 계정으로 fork.
2. 로컬 리모트 재구성: `upstream` = Engccer 원본(읽기), `origin` = aro-deeply fork.
3. 작업 브랜치 `feat/build-animation-and-charts` 생성(기준 커밋 `33996f8`).
4. 선작업 커밋 → 데이터 수령 후 마무리 커밋 → `origin` push → Engccer로 PR.
- gh CLI는 aro-deeply로 인증 완료됨.

## 8. 작업 순서

선작업(데이터 수령 전):
1. Fork · 리모트 · 브랜치 정리
2. Fragment 시스템(JS advance/retreat + CSS), 내레이션 비간섭 + 틱 효과음 통합
3. 다중 항목 슬라이드 `.fragment` 마크업
4. 차트 데이터 객체 + 렌더 함수 3종 (현재 수치로 동치 검증)
5. 차트 애니메이션 + 빌드 모션 + 강조 효과 + reduced-motion
6. 로컬 검증 · 논리 단위 커밋

데이터 수령 후(15:30 이후):
7. `SURVEY` 숫자 교체 → 차트 자동 반영
8. 최종 검증 · push · PR

## 9. 검증 체크리스트

- 키보드 →/←/PageUp/Down/Home/End: fragment·슬라이드 진행, 뒤로 시 이전 슬라이드 전부 표시
- fragment 진행 중 슬라이드 내레이션이 끊기지 않음(슬라이드 전환에서만 재생)
- fragment 등장 시 틱 효과음 재생, 오디오 토글 off면 틱·내레이션 함께 정지
- 차트 리팩터링 전/후 시각 동일(애니메이션 종료 상태)
- 모바일: 좌우 스와이프 fragment 진행, 44px 터치 타깃, 세로 스크롤 보존, iOS 회전 레이아웃 유지
- 스크린리더: fragment 등장 시 포커스 이동, 차트 desc 자동 생성
- `prefers-reduced-motion: reduce`: 전 애니메이션 즉시 표시(틱·포커스 이동은 유지)
- 효과음 · 오디오 토글 · 내레이션 정상
- SFX 자동재생 차단 환경 무음 fallback

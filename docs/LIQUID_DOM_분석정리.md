# Liquid DOM 전수조사 & 활용 전략 정리

> 작성일: 2026-09-27
> 대상 저장소: [bmshin94/liquid-dom](https://github.com/bmshin94/liquid-dom)
> 원본(업스트림): [AndrewPrifer/liquid-dom](https://github.com/AndrewPrifer/liquid-dom)
> 라이브 데모: https://liquid-dom-showcase.vercel.app
> 라이선스: MIT

---

## 목차

1. [프로젝트 정체](#1-프로젝트-정체)
2. [저장소 구조 전수조사](#2-저장소-구조-전수조사)
3. [핵심 기술 원리 (쉬운 설명)](#3-핵심-기술-원리-쉬운-설명)
4. [설치 및 사용법](#4-설치-및-사용법)
5. [자주 묻는 질문 정리](#5-자주-묻는-질문-정리)
6. [수익화 아이디어 10선](#6-수익화-아이디어-10선)
7. [실행 로드맵 및 주의사항](#7-실행-로드맵-및-주의사항)

---

## 1. 프로젝트 정체

### 한 줄 정의

**애플의 Liquid Glass UI를 웹에서 WebGPU로 물리 기반 렌더링하는 TypeScript 모노레포.**

CSS `backdrop-filter: blur()` 같은 시각적 흉내가 아니라, SDF(Signed Distance Field)
기반으로 유리의 굴절·반사·분산·융합을 GPU 셰이더에서 실제로 계산한다.

### 분류

| 항목 | 값 |
| --- | --- |
| 형태 | npm 패키지 5개 (pnpm workspace 모노레포) |
| 종류 | 프론트엔드 UI 렌더링 라이브러리 (SDK) |
| 비교 대상 | Three.js, Framer Motion, React Spring |
| 런타임 | 100% 브라우저 클라이언트 사이드 |
| 백엔드 | 없음 |
| 언어 | TypeScript + WGSL(WebGPU 셰이딩 언어) |

### 저장소 메타데이터 (원본 기준)

| 항목 | 값 |
| --- | --- |
| Stars | 2,518 |
| Forks | 124 |
| 생성일 | 2026-04-18 |
| 설명 | "Liquid Glass for the Web" |
| 열린 이슈 | 2 |
| 기본 브랜치 | master |
| 주 기여자 | Andrew Prifer (커밋 49개) |

---

## 2. 저장소 구조 전수조사

```
liquid-dom/
├── packages/            # 배포용 npm 패키지 5개 (약 13,600줄)
│   ├── core/    8,840줄  💎 WebGPU 렌더러 + 씬그래프 (심장부)
│   ├── layout/  2,632줄  📐 렌더러 무관 레이아웃 엔진 (독립 사용 가능)
│   ├── react/   1,525줄  ⚛️ React 19 바인딩
│   ├── three/     252줄  🎮 Three.js WebGPU 어댑터
│   └── r3f/       396줄  🌉 React Three Fiber 브릿지
├── demo/                # 실행 가능한 예제 7종 (약 9,600줄)
├── tools/               # macOS 네이티브 런타임 분석 도구
├── ADAPTIVE_TINT.md     # 배경 밝기 기반 자동 색조 조절 레시피
├── ADAPTIVE_BLUR_PERF.md# 블러 커널 비용 수식 분석
├── RELEASING.md         # changesets 릴리스 절차
└── CLAUDE.md            # Claude Code 페르소나 지침 (라이브러리와 무관)
```

### 2.1 `@liquid-dom/core` (v0.1.1) — 엔진 본체

| 파일 | 줄 수 | 역할 |
| --- | ---: | --- |
| `src/layout.ts` | 1,419 | 레이아웃 노드 ↔ 씬그래프 동기화 |
| `src/scene.ts` | 1,204 | 씬그래프 (Scene / Container / Glass / Html / Group) |
| `src/renderer/dom-content-sync.ts` | 1,065 | 살아있는 DOM → GPU 텍스처 복사 |
| `src/renderer/core.ts` | 1,058 | WebGPU 렌더러 본체 |
| `src/shaders.ts` | 951 | WGSL 셰이더 전량 |
| `src/sdf.ts` | 456 | SDF 수학 (CPU 측 복제본, 히트테스트용) |
| `src/renderer/adaptive-blur.ts` | 229 | 적응형 다운샘플 블러 파이프라인 |
| `src/renderer/pointer-controller.ts` | 343 | SDF 기반 포인터 히트테스트 |

주요 셰이더 함수:

```wgsl
fn sdSmoothRoundRect(...)      // 스퀘어클(iOS 둥근 모서리) 거리
fn superellipseLength(...)     // p-norm 초타원
fn smoothUnion(...)            // 유리 융합 (메타볼 효과)
fn shapeSubmergedArea(...)     // 잠김 영역 기반 융합 게이팅
fn evaluateHeightProfile(...)  // convex / concave / lip 표면 프로파일
fn normalGateForSamples(...)   // 법선 각도 게이팅
```

씬그래프 계층 규칙:

```
Scene    → Container, Html, Group 만 허용
Container→ Glass, Group 만 허용
Glass    → Html, Group 만 허용   (Glass 안에 Glass 불가)
Group / StackingContext 는 어디든 삽입 가능하나 위 규칙은 그대로 적용
```

`Container` 광학 파라미터 그룹:

| 그룹 | 속성 |
| --- | --- |
| 형태/융합 | `spacing`, `normalGating`, `blendSupportGating` |
| 블러/변위 | `blur`, `bezelWidth`, `displacementFactor`, `displacementBlur` |
| 굴절 | `thickness`, `ior`, `contentIor`, `contentDepth`, `dispersion`, `surfaceProfile` |
| 정반사/반사 | `lightDirection`, `specularStrength`, `specularWidth`, `specularFalloff`, `specularSharpness`, `specularOpacity`, `reflectionOffset` |
| 색/그림자 | `tint`, `shadowColor`, `shadowOffsetX/Y`, `shadowBlur`, `shadowSpread` |
| 합성 | `opacity`, `zIndex` |

### 2.2 `@liquid-dom/layout` (v0.2.0) — 숨은 보석

유리와 **완전히 독립적으로** 사용 가능한 렌더러 무관 레이아웃 엔진.
SwiftUI의 2단계 레이아웃 모델을 JS로 이식했다.

```
1. 부모가 크기를 제안 (ProposedSize)
2. 자식이 실제 크기를 보고 (Size)
3. 부모가 자식을 배치 (Rect)
```

제공 노드: `HStack`, `VStack`, `ZStack`, `Frame`, `Padding`, `Spacer`,
`Background`, `Overlay`, `Noop`, `Leaf`, 커스텀 `Layout`

```ts
import { LayoutEngine, Frame, HStack, Spacer } from '@liquid-dom/layout'

const row  = new HStack({ spacing: 12, alignment: 'center' })
              .append(label, new Spacer(), button)
const root = new Frame({ width: 320, height: 56 }).append(row)
const engine = new LayoutEngine({ root })

engine.layout({ width: 320, height: 56 })
console.log(label.layout.rect)   // { x, y, width, height }
```

특징: 측정 캐싱, 안정적 노드 ID, `layout.rect`(부모 기준) / `layout.absoluteRect`(루트 기준),
`@liquid-dom/layout/dom` 서브패스의 `DomLeaf`로 실제 HTML 요소 측정 지원.

### 2.3 `@liquid-dom/react` (v0.1.1)

React 19 전용 선언형 바인딩. `LiquidCanvas`(캔버스 소유) 와 `LiquidScene`(헤드리스) 두 루트 제공.

```tsx
<LiquidCanvas style={{ width: '100vw', height: '100vh' }}>
  <GlassContainer blur={12} spacing={28}>
    <Frame width={280} height={160}>
      <Glass cornerRadius={44} pointerEvents>
        <Html sizing="fill"><button>Native content</button></Html>
      </Glass>
    </Frame>
  </GlassContainer>
</LiquidCanvas>
```

애니메이션 시스템 내장:

```tsx
transition={{ width: spring({ stiffness: 360, damping: 34 }) }}
transition={{ blur: easing({ duration: 0.25, ease: Easing.easeOut }) }}
```

훅: `useFrame`, `useAnimate`, `useTimeline`, `useLiquidScene`, `useRenderer`,
`useInvalidateFrame`, `useInvalidateLayout`, `AnimationConfigProvider`(timeScale)

### 2.4 `@liquid-dom/three` / `@liquid-dom/r3f`

이미 Three.js **WebGPURenderer** 씬을 돌리는 경우, 유리 UI를 후처리 합성 레이어로 얹는 어댑터.
`WebGpuGlassCore`가 외부 소유 `GPUDevice`를 받아 출력 텍스처에 렌더한다.
(WebGLRenderer는 지원하지 않음)

### 2.5 데모 7종

| 데모 | 줄 수 | 내용 |
| --- | ---: | --- |
| `demo/showcase` | 2,710 | iOS 제어센터 / 알림센터 / 알림 팝업 / 음악 사이드바 / 메뉴 / 비디오 컨트롤 / R3F 통합 |
| `demo/minimal` | 2,651 | 포인터 이벤트, 애니메이션, DOM 측정, SDF 겹침 등 기능별 최소 예제 |
| `demo/blending` | 1,814 | 유리 융합 실험실 (Leva 컨트롤) |
| `demo/three-layout` | 1,516 | Three.js + 레이아웃 엔진만 |
| `demo/three` / `three-react` / `three-r3f` | 989 | 통합 레벨별 샘플 |

별도로 `packages/layout/playground/`에 레이아웃 엔진 전용 놀이터
(레이아웃 케이스 / DOM 리프 / 애니메이션 / 스트레스 테스트 / 성능 프로파일 탭) 존재.

### 2.6 `tools/glass_runtime_probe.m`

Objective-C 파일. macOS Objective-C 런타임을 리플렉션으로 순회하며
애플 Liquid Glass 관련 내부 클래스/셀렉터를 탐색한다.
즉 **애플 구현을 역분석해서 파라미터를 맞춘 흔적**이다.

### 2.7 개발 인프라

```
패키지 매니저 : pnpm 10.33.0 (workspace)
번들러        : tsup
테스트        : vitest (+ jsdom)
데모 번들러   : vite 8
버전 관리     : changesets
타입          : TypeScript 6.0.2, @webgpu/types
```

---

## 3. 핵심 기술 원리 (쉬운 설명)

### 3.1 CSS와 무엇이 다른가

| | CSS `backdrop-filter` | Liquid DOM |
| --- | --- | --- |
| 방식 | 뒷배경을 뿌옇게만 처리 | 뒷배경을 실제로 휘어서 재샘플링 |
| 비유 | 안개 낀 창문 | 진짜 유리 렌즈 |
| 모양 | 사각형 / 둥근 사각형 | 물방울처럼 합체·분리·변형 |
| 빛 반사 | 없음 | 광원 방향 기반 자동 계산 |
| 처리 | CPU / 합성기 | GPU 셰이더 (120fps 목표) |
| 내부 요소 | 평범한 HTML | HTML이 유리에 굴절되어 보임 |

### 3.2 SDF(Signed Distance Field)가 핵심

> 화면의 모든 픽셀에 대해 "여기서 도형 표면까지의 거리"를 수식으로 답하는 함수.
> 음수 = 안쪽, 0 = 표면, 양수 = 바깥.

거리 하나로 아래를 전부 해결한다.

| 얻는 것 | 방법 |
| --- | --- |
| 물방울 합체 | 두 거리값을 부드럽게 섞음 (`smoothUnion`) |
| 표면 법선 | 거리값의 기울기(gradient) |
| 굴절 방향 | 법선 + 굴절률(IOR) → 배경 샘플 좌표 결정 |
| 클릭 판정 | 거리 < 0 이면 내부 |
| 애플식 모서리 | 초타원(p-norm) 수식 |

### 3.3 HTML을 유리 안에 넣는 파이프라인

```
1. <canvas layoutsubtree> 생성 (실험적 HTML-in-Canvas API)
2. 실제 살아있는 DOM 엘리먼트를 그 안에 마운트
3. 브라우저가 매 프레임 그 DOM을 페인트
4. 페인트 결과를 GPU 텍스처로 복사  ← dom-content-sync.ts
5. 셰이더가 그 텍스처를 유리로 굴절시켜 합성
6. 클릭 시 SDF로 역계산하여 원래 DOM에 이벤트 전달
```

결과적으로 유리 안의 진짜 `<input>`에 타이핑이 되고, 유리 뒤에서 영상이 재생된다.

### 3.4 적응형 블러

블러 반경에 따라 다운샘플 레벨을 `ceil(log2(radiusPx / denseRadiusPx))`로 선택해
비용을 낮춘다. `ADAPTIVE_BLUR_PERF.md`에 5/9/13/17탭 커널의 비용 모델이 수식으로 정리되어 있다.

```
downsample(L) = 4/3 * (1 - 4^-L)
upsample(L)   = 4/3 * (1 - 4^-L)
blur(L)       = 2 * S * 4^-L
total(L)      = downsample(L) + blur(L) + upsample(L)
```

### 3.5 적응형 색조

`renderer.setBackdropMetricsTracking(container, true)` 로 배경 밝기 통계를 수집한 뒤
`luminanceP50`(중앙값)을 기준으로 `container.tint`를 매 프레임 보간한다.
평균 대신 중앙값을 쓰는 이유는 작은 밝은/어두운 이상치에 덜 민감하기 때문.

---

## 4. 설치 및 사용법

### 4.1 내 프로젝트에 설치

```sh
pnpm add @liquid-dom/react react react-dom         # React (권장)
pnpm add @liquid-dom/core                           # 바닐라 JS
pnpm add @liquid-dom/three @liquid-dom/core three   # Three.js 보유 시
pnpm add @liquid-dom/r3f @liquid-dom/react @react-three/fiber react react-dom three
pnpm add @liquid-dom/layout                         # 레이아웃 엔진만
```

### 4.2 실행 전 필수 환경 설정

```
1. Chrome 사용 (Safari / Firefox 미지원)
2. chrome://flags/#canvas-draw-element → Enabled
3. 브라우저 재시작
4. 콘솔에서 navigator.gpu 가 undefined 가 아닌지 확인
```

### 4.3 이 저장소 직접 실행

```sh
pnpm install
pnpm -r build

pnpm dev:minimal      # 가장 가벼운 기능별 데모 (시작 추천)
pnpm dev:blending     # 유리 융합 실험실
pnpm dev:three-r3f    # 3D 씬 위 유리 UI
pnpm dev              # showcase (iOS 제어센터 재현)

pnpm --filter @liquid-dom/layout test
pnpm --filter @liquid-dom/core test
pnpm --filter @liquid-dom/react test
```

### 4.4 제약사항 요약

| 제약 | 내용 |
| --- | --- |
| WebGPU 필수 | `navigator.gpu` 없으면 렌더링 불가 |
| HTML-in-Canvas | `<Html>` 콘텐츠는 Chrome 실험 플래그 필요 (기본 OFF) |
| React 19 | react / react-dom 19 이상 |
| Three | WebGPURenderer 전용, WebGLRenderer 불가 |
| SSR | 불가. Next.js는 `'use client'` 또는 `dynamic(..., { ssr:false })` |

---

## 5. 자주 묻는 질문 정리

### Q. 플러그인인가, 스킬인가, MCP인가?

**셋 다 아님. 평범한 npm 라이브러리(SDK).**

| 종류 | 정의 | 해당? |
| --- | --- | --- |
| 플러그인 | 기존 앱에 꽂아 기능 확장 | 아니오 |
| Claude Skill | Claude에게 작업 방식을 가르치는 지침서 | 아니오 |
| MCP | AI가 외부 도구/데이터에 접근하는 프로토콜 | 아니오 |
| npm 라이브러리 | `import` 해서 쓰는 코드 묶음 | **예** |

단, 저장소 루트의 `CLAUDE.md`는 Claude Code용 페르소나 지침이 맞다. 라이브러리 본체와는 무관하다.

### Q. API 토큰이 필요한가?

**전혀 필요 없다.** 회원가입·API 키·서버 통신·사용량 제한·요금 모두 없음.

근거: 모든 `package.json`에 네트워크 의존성 0개, 백엔드 코드 0줄, `.env` 없음.
전부 브라우저 GPU에서 로컬 계산된다. 필요한 유일한 조건은 사용자 브라우저의 WebGPU 지원.

### Q. 왜 GitHub에서 유명한가? (⭐2,518)

1. **타이밍** — 2025년 WWDC Liquid Glass 발표 직후 수요 정점에 등장
2. **진짜 구현** — 경쟁 라이브러리 대부분이 `backdrop-filter` 흉내인 반면 물리 기반 SDF 렌더링
3. **시각적 임팩트** — README 쇼케이스 이미지 + Vercel 라이브 데모, 보면 바로 이해됨
4. **X(트위터) 바이럴** — 제작자 발표 트윗이 크게 확산, 중화권 채널까지 전파
5. **제작자 네임밸류** — Andrew Prifer, 웹 애니메이션/3D 분야 기존 인지도
6. **기술적 깊이** — Objective-C 런타임 역분석 도구, 블러 비용 수식 분석 문서, 상용급 문서화
7. **계층 분리 설계** — core / react / three / r3f / layout 로 진입점을 사용자별로 제공

### Q. 로컬 에이전트 구축에 도움이 되나?

| 목적 | 점수 |
| --- | --- |
| 에이전트 **두뇌**(LLM, 툴콜, RAG) | ★☆☆☆☆ — 해당 기능 전무 |
| 에이전트 **UI** | ★★★★★ |
| Electron 데스크톱 에이전트 | ★★★★★ |
| 아키텍처 학습 | ★★★★☆ |
| 레이아웃 엔진 재활용 | ★★★★☆ |

핵심 포인트: **Electron에서는 크롬 플래그를 강제로 켤 수 있어 최대 약점이 사라진다.**

```js
// main.js
app.commandLine.appendSwitch('enable-unsafe-webgpu')
app.commandLine.appendSwitch('enable-blink-features', 'CanvasDrawElement')
```

Tauri는 시스템 웹뷰를 쓰므로 불가. Electron 권장.

UI 아이디어: 항상 떠있는 유리 커맨드바, 추론 중 부풀었다 줄어드는 애니메이션,
툴 호출마다 유리 방울이 분리되었다 흡수되는 `smoothUnion` 시각화.

### Q. React나 PHP로 만들 수 있나?

**React** — 이미 완성되어 있다. `@liquid-dom/react` 패키지가 그것.

**PHP** — 렌더링 자체는 불가능하지만, 조합하면 가능하다.

```
PHP        = 서버에서 HTML 문자열 생성
Liquid DOM = 브라우저 GPU에서 픽셀 계산
→ 실행 위치가 다르므로 PHP가 직접 유리를 그릴 수는 없음
→ PHP는 데이터만 내려주고, 번들된 JS가 브라우저에서 렌더
```

```php
<div id="root" data-items='<?= htmlspecialchars(json_encode($items), ENT_QUOTES) ?>'></div>
<script type="module" src="/dist/glass-ui.js"></script>
```

| 환경 | 방법 | 난이도 |
| --- | --- | --- |
| Laravel + Inertia | React 어댑터 그대로 | 쉬움 |
| Laravel + Vite | `@vite` 디렉티브 | 쉬움 |
| Livewire | Alpine.js `x-data` 브릿지 | 보통 |
| WordPress | `wp_enqueue_script` + `wp_localize_script` | 보통 |
| 순수 PHP | `<script type="module">` | 쉬움 |

원칙: **서버는 데이터만, 브라우저가 그림을 그린다.**
전체 페이지가 아니라 히어로 섹션·플로팅 카드 등 특정 위젯에만 적용하고,
`navigator.gpu` 미지원 시 CSS `backdrop-filter` 폴백으로 전환하는 것이 현실적이다.

---

## 6. 수익화 아이디어 10선

### 전제 조건

```
최대 리스크 : 크롬 실험 플래그 의존 → 일반 B2C 웹 서비스는 현시점 불가
전략 A      : 플래그가 필요 없는 환경으로 간다 (Electron / 영상 / 전시)
전략 B      : 플래그 정식화 시점을 대비해 지금 선점한다
라이선스    : MIT → 상업적 이용·재판매·수정 모두 합법 (저작권 고지 유지 필수)
```

### 티어 1 — 즉시 가능

#### 1. 프리미엄 UI 컴포넌트 킷 판매

- 구성: 즉시 사용 가능한 컴포넌트 40~60개, React+TS 소스, **자동 CSS 폴백 내장**, Figma 파일, Storybook
- 가격: Personal $79 / Team $249 / Enterprise $999
- 시뮬레이션: 런칭 첫 달 100장 → $10,000~15,000
- 유통: Gumroad, LemonSqueezy
- 기간 2~3개월 / 현실성 ★★★★☆ / 규모 ★★★★☆
- **차별점: 원본에 없는 "폴백 자동 처리"가 상품의 진짜 가치**

#### 2. 모션그래픽 · 영상 소스 판매

- 구성: 4K 알파채널 유리 트랜지션 팩, 로고 리빌 템플릿, 유튜브 오버레이
- 자동화: Playwright로 씬 렌더 → ffmpeg 인코딩
- 유통: Envato Elements, Motion Array, Artlist, Pond5, Gumroad ($29~99/팩)
- 장점: **시청자 브라우저가 필요 없어 플래그 이슈가 완전히 소멸**, 한 번 제작 후 수동소득
- 기간 3~4주 / 현실성 ★★★★★ / 규모 ★★★☆☆ — **가장 빠른 현금화 경로**

#### 3. Electron 데스크톱 앱 (최우선 추천)

플래그를 앱이 직접 켤 수 있어 기술 제약이 사라진다.

| 앱 | 컨셉 | 가격 |
| --- | --- | --- |
| Glass Launcher | Raycast/Alfred 대체 유리 커맨드바 | $8/월 또는 $49 평생 |
| Glass Player | Apple Music 스타일 뮤직 플레이어 | $19 일회성 |
| Glass Notes | 항상 위에 뜨는 유리 메모 위젯 | $12 일회성 |
| Glass Agent | 로컬 LLM 채팅 UI | $15/월 |
| Glass Monitor | 시스템 리소스 유리 대시보드 | $9 일회성 |

- 시뮬레이션: 유료 1,000명 × $8 = **$8,000 MRR**
- 유통: Gumroad, Paddle, Mac App Store, Setapp
- 기간 3~5개월 / 현실성 ★★★★☆ / 규모 ★★★★★

#### 4. 커스텀 개발 · 컨설팅 외주

- 타겟: 브랜드 런칭 사이트, 전시/체험형 인터랙티브 부스(환경 통제 가능), 자동차·명품·화장품, 컨퍼런스 키노트
- 단가: 히어로 섹션 500~1,500만원 / 풀 사이트 2,000~5,000만원 / 전시 부스 3,000만원~ / 자문 시간당 15~30만원
- 진입: 데모를 재해석한 포트폴리오 2~3개 → X, Behance, Dribbble 업로드
- 기간 1개월(포트폴리오) / 현실성 ★★★★★ / 규모 ★★★★☆

### 티어 2 — 중기 (6개월~1년)

#### 5. SaaS — 노코드 유리 UI 에디터

브라우저에서 슬라이더로 디자인 → React/Vue/HTML 코드 자동 생성.
Free $0 / Pro $15월 / Team $49월. 목표 유료 500명 = $7,500 MRR.
기간 6~8개월 / 현실성 ★★★☆☆ / 규모 ★★★★★

#### 6. 교육 — 강의 · 전자책

"WebGPU 셰이더로 만드는 Liquid Glass" (SDF 수학 → WGSL → 굴절 구현 → React 통합).
인프런·패스트캠퍼스 8~15만원, Udemy $89, 자체 사이트 마진 최대.
500명 × 10만원 = 5,000만원. **강의가 1~4번 상품의 유입 퍼널 역할**을 겸한다.
기간 2~3개월 / 현실성 ★★★★☆ / 규모 ★★★☆☆

#### 7. 플러그인화 (타 생태계 이식)

| 대상 | 내용 | 가격 |
| --- | --- | --- |
| Figma 플러그인 | 디자인 → 코드 생성 | $10/월 |
| Framer / Webflow | 노코드 커스텀 컴포넌트 | $25 일회성 |
| WordPress | Elementor 위젯 (시장 최대) | $49/년 |
| Shopify | 제품 상세 유리 카드 섹션 | $79 일회성 |

WordPress가 의외의 노다지 — 전 세계 사이트의 큰 비중을 차지하는데 해당 플러그인이 거의 없다.

#### 8. MCP 서버 + AI 결합

"Claude야, 유리 알림 카드 만들어줘" → MCP 서버가 Liquid DOM 코드 생성 + 미리보기 렌더.
MCP 서버는 오픈소스로 무료 배포(인지도 확보), 프리미엄 컴포넌트 구독 $9/월,
클라우드 렌더링 API 종량제로 수익화. "AI + 애플 디자인 + WebGPU" 조합은 화제성이 크다.
기간 2~3개월 / 현실성 ★★★☆☆ / 규모 ★★★★☆

### 티어 3 — 장기

#### 9. 듀얼 라이선스

원본 MIT 코드 자체에는 적용 불가하나, 직접 만든 부가가치 레이어에는 가능.
Community(MIT, 무료) / Commercial($499/년/제품 — 폴백 엔진, 프리미엄 컴포넌트, 우선 지원, SLA).
참고 모델: Tiptap, Chart.js Pro, AG Grid, Handsontable.

#### 10. 신규 플랫폼 선점

- **visionOS / Apple Vision Pro 웹앱** — 공간 컴퓨팅 UI에서 유리는 애플 기본 디자인 언어
- **키오스크 / 사이니지** — 하드웨어 통제 가능 = 플래그 이슈 없음, B2B 고단가
- **차량 인포테인먼트(IVI)** — 크로미움 기반 HMI 증가 추세, 고급차 브랜드 타겟

---

## 7. 실행 로드맵 및 주의사항

### 로드맵

| 시기 | 할 일 |
| --- | --- |
| Month 1 | 데모 재해석 포트폴리오 3개 + X / Behance 업로드 (무료 마케팅, 반응 테스트) |
| Month 2-3 | 영상 소스팩 출시(아이디어 2) → 첫 수익. 유튜브 튜토리얼 병행 |
| Month 3-5 | UI 컴포넌트 킷 개발 → Gumroad 런칭(1). 동시에 외주 수주(4) |
| Month 6-10 | Electron 앱 1개 완성 → 구독 MRR 시작(3) |
| Month 10+ | SaaS 또는 MCP 서버로 확장(5 / 8) |

### 단 하나만 고른다면

**아이디어 3: Electron 데스크톱 앱.**
기술 최대 약점인 플래그 문제가 사라지고, 셰이더·물리 지식이라는 진입장벽이 경쟁자를 막아주며,
구독 MRR로 안정적 수익이 쌓이고, 로컬 에이전트 관심사와도 직결된다.

### 주의사항

| 항목 | 내용 |
| --- | --- |
| 애플 상표권 | "Liquid Glass"는 애플 용어. 제품명 직접 사용 대신 자체 브랜딩 권장 |
| MIT 의무 | 배포물에 원본 저작권 고지 + 라이선스 전문 포함 필수 |
| 커뮤니티 매너 | 원작자 크레딧 명시, 버그픽스는 업스트림 기여 — 평판이 곧 영업력 |
| 성능/배터리 | GPU 풀가동으로 노트북 배터리 소모 큼. "저전력 모드"는 상품 필수 기능 |
| 접근성/SEO | 캔버스 렌더링은 스크린리더·검색엔진이 읽지 못함. 전체 페이지 적용 금지, 위젯 단위로만 |

---

## 참고 링크

| 항목 | 주소 |
| --- | --- |
| 현재 저장소 | https://github.com/bmshin94/liquid-dom |
| 원본 저장소 | https://github.com/AndrewPrifer/liquid-dom |
| 라이브 데모 | https://liquid-dom-showcase.vercel.app |
| 제작자 발표 트윗 | https://x.com/AndrewPrifer/status/2056923983581446529 |
| npm (core) | https://www.npmjs.com/package/@liquid-dom/core |
| jsDelivr CDN | https://www.jsdelivr.com/package/npm/@liquid-dom/core |
| Context7 문서 | https://context7.com/andrewprifer/liquid-dom |
| WICG HTML-in-Canvas 스펙 | https://wicg.github.io/html-in-canvas/ |
| liquid-glass 토픽 | https://github.com/topics/liquid-glass?l=typescript |

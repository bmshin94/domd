# DOMD 프로젝트 전수조사 & 수익화 분석 정리

> 작성: 카리나 (Claude Code) · 작성일: 2026-10-01
> 대상 저장소: https://github.com/bmshin94/domd (원본: https://github.com/do-md/domd)

---

## 0. 관련 링크 모음

| 구분 | URL |
| --- | --- |
| 내 포크 저장소 | https://github.com/bmshin94/domd |
| 원본 저장소 | https://github.com/do-md/domd |
| npm 패키지 (에디터 커널) | https://www.npmjs.com/package/@do-md/core-react |
| 웹 에디터 데모 | https://www.domd.app/editor |
| AI 스트리밍 플레이그라운드 | https://www.domd.app/playground |
| 실시간 동기화 플레이그라운드 | https://www.domd.app/playground/live |
| CRDT 오프라인 병합 플레이그라운드 | https://www.domd.app/playground/crdt |
| 마크다운 입력창 데모 | https://www.domd.app/chat |
| macOS 다운로드 (Apple Silicon) | https://github.com/do-md/domd/releases/latest/download/DOMD_aarch64.dmg |
| macOS 다운로드 (Intel) | https://github.com/do-md/domd/releases/latest/download/DOMD_x86_64.dmg |
| Issues | https://github.com/do-md/domd/issues |
| Discussions | https://github.com/do-md/domd/discussions |
| 상업 라이선스 문의 | effyouapp@gmail.com |

---

## 1. 이게 뭐하는 프로젝트야?

**한 줄 정의**

> DOMD = 마크다운을 "변환 없이" 그 자체로 편집하는 WYSIWYG 마크다운 에디터 + 그 심장인 에디터 엔진(커널)

**규모**

- 깃 추적 파일 459개
- 앱/플러그인 TypeScript 약 24,100줄
- 에디터 커널(`.packages/@do-md/core`) 단독 약 19,100줄
- macOS 네이티브 Rust 약 4,100줄
- 앱 버전 `0.9.2` / 커널 버전 `0.12.3`

### 1-1. 폴더별 전수조사표

| 폴더 | 정체 | 하는 일 |
| --- | --- | --- |
| `.packages/@do-md/core` | **에디터 커널 (핵심 자산)** | 마크다운 파싱·렌더·커서·undo·스트리밍 전부 자체 구현. Brotli 압축 후 30KB대 |
| `.packages/@do-md/zenith` | 자체 제작 상태관리 | Zustand 비슷한 미니 스토어 (core/react/middleware/history/query/devtools) |
| `.packages/@do-md/plugins` | 공식 플러그인 3종 | `@do-md/commands`(서식 명령), `@do-md/search`(찾기·바꾸기), `@do-md/toc`(목차+스크롤 스파이) — 전부 MIT |
| `.packages/@do-md/utils` | 유틸 | debounce, throttle, nanoid, 클립보드 |
| `app/` | Next.js 16 라우트 | `/editor`, `/chat`, `/collab`, `/preview`, `/playground`(+`live`, `crdt`) |
| `features/` | 제품 기능 단위 | `editor`, `ai`, `collaboration`, `chat`, `preview`, `playground`, `landing`, `updater`, `icons` |
| `plugins/` | 앱 쪽 플러그인 | `ai-collab`(AI 편집), `collaboration/crdt-sync`·`realtime-sync`·`versioning`, `rendering`, `toolbar` |
| `src-tauri/` | **macOS 네이티브 앱** | Rust + Tauri 2. 타이틀바 직접 구현(1,127줄), 파일 감시, 인쇄, SQLite 협업 DB, Quick Look 확장 |
| `src-tauri/src/bin/domd_cli.rs` | **에이전트용 CLI** | `domd-cli`로 창 열기·내용 읽기·텍스트 삽입·저장·닫기 |
| `services/signaling` | Cloudflare Worker | WebRTC 시그널링만 중계. 문서 내용은 서버를 안 지나감 |
| `common/` | 공통 | i18n(react-i18next), Prism 하이라이트, Tauri 브릿지, 이미지 저장 |
| `scripts/verify-*` | 자체 검증 스크립트 | 테스트 프레임워크 없이 `.mts` 실행형 검증 7종 |
| `.github/workflows/cla.yml` | CLA 봇 | PR 머지 전 기여자 라이선스 동의 확인 |

### 1-2. 기술적 차별점 7가지

1. **마크다운이 곧 모델** — ProseMirror/Slate/Lexical 같은 범용 리치텍스트 프레임워크를 전혀 쓰지 않음. 파싱·렌더·편집·undo·AI 스트리밍 전부 커널 내부의 결정론적 상태 변화로 직접 모델링. 런타임 의존성이 **React + Immer 2개**뿐.
2. **확장 가능한 인라인 문법** (커널 0.6+) — Pandoc/Djot 속성 문법을 1급 확장점으로 승격.
   ```text
   ==하이라이트==                            기본
   =={red}하이라이트==                       색 지정 (위치 파라미터)
   =={.comment author="Alice"}하이라이트==   의미 타입 + 속성
   =={.mention id=1}Alice==  ≡  <{.mention id=1}Alice>
   ```
   `.variant`에 **React 컴포넌트 바인딩** 가능 → 마크다운 한 조각이 살아있는 UI(이슈 카드, 날씨 위젯 등)가 됨. 미등록 타입은 에러 대신 CSS 훅으로 degrade.
3. **문단 내부까지 충돌 없는 병합(CRDT)** — 문단 단위 LWW가 아님. 커널 자체는 CRDT를 모르고 편집 연산 스트림만 뱉으며(`subscribeRenderDataOps`), 선택적 Yjs 플러그인이 어댑터로 붙음. 완성된 에디터 기능에 플러그인만 붙여 협업 추가 가능.
4. **서버가 내용을 못 보는 실시간 협업** — WebRTC P2P + 방 비밀번호에서 유도한 AES-GCM 키로 시그널링까지 암호화. Cloudflare Worker는 피어 연결만 중계. 계정·DB 없음.
5. **AI 스트리밍 전용 설계** — 반쯤 열린 코드펜스, 미완성 표, 깨진 리스트를 중간에도 정상 렌더하고, 종료 기호가 도착하면 깜빡임 없이 흡수. 2만 줄 문서 검증.
6. **Aider 방식 AI 편집 프로토콜** — `features/ai/lib/llm.ts`가 Aider의 SEARCH/REPLACE 블록(git 충돌 마커 형식)을 거의 그대로 사용. 모델 학습 데이터에 충돌 마커가 많아 자체 발명 포맷보다 벤치마크 점수가 훨씬 높음. 실패 시 Aider식 반성 메시지로 1회 자동 교정 재시도.
7. **AI를 협업자로 모델링** — `plugins/ai-collab/agent-peer.ts`에서 AI가 CRDT 피어로 참여. 사람 2명 + AI 2명 동시 편집 구조가 자연스럽게 나옴.

### 1-3. 어떤 상황에 쓰나

**제품으로**
- 1MB 거대 마크다운 문서를 5KB 메모처럼 열고 싶을 때
- 프로젝트 트리·계정·동기화 서비스 없는 "그냥 파일" 워크플로 (macOS 앱 + Quick Look)
- 로컬 처리 보장이 필요한 문서 작업
- `domd-cli`로 AI 에이전트의 출력 창으로 쓸 때

**커널(라이브러리)로**
- AI 챗/에이전트 UI의 마크다운 스트리밍 출력 영역
- 댓글창·프롬프트·CMS 필드·이슈 폼 같은 마크다운 네이티브 입력창 (`Enter` 전송 / `Shift+Enter` 줄바꿈)
- 실시간 공동 편집 워크스페이스
- 번들 사이즈에 민감한 서비스 (30KB vs ProseMirror 계열 수백 KB)

### 1-4. 라이선스 (중요)

| 레이어 | 위치 | 라이선스 |
| --- | --- | --- |
| 애플리케이션 (macOS 앱, 웹 앱, 플러그인, 유틸) | 루트 | **MIT** |
| **에디터 커널** | `.packages/@do-md/core` | **GPL-3.0-only** + 추가 허가 |

GPL 7조 추가 허가 2종:

1. **소규모 주체 예외** — 비영리/교육기관, 또는 **연매출 100만 USD 미만 + 투자유치 200만 USD 미만**이면 GPL 아닌 소프트웨어에 링크해 원하는 조건으로 배포 가능
2. **FOSS 예외** — MIT/Apache-2.0/BSD/MPL-2.0/ISC/EPL-2.0/zlib 프로젝트면 자기 라이선스로 배포 가능

핵심 주의: **"브라우저에 빌드를 전달하는 것도 conveying(배포)"**. 웹앱 서비스만 해도 GPL 의무 발동. 반대로 내부 개발·테스트·평가만 하는 건 의무 없음.
0.10.0까지는 PolyForm Noncommercial, 0.11.0부터 GPL-3.0. 예외 조항은 공개된 버전에 대해 **철회 불가** 명시.

### 1-5. 나에게 무슨 도움이 되나

1. **"프레임워크 없이 에디터 만들기" 교과서** — 커서 계산, DOM 동기화, 그래핌 단위 백스페이스, 트리플클릭 선택 범위, IME 처리까지 상용 수준 코드 1.9만 줄
2. **AI 에이전트 UI를 바로 꽂을 수 있음** — 스트리밍 렌더 + SEARCH/REPLACE 적용기 + 배치 replace API가 완성품
3. **Tauri 2 + Rust 네이티브 앱 레퍼런스** — macOS 타이틀바 커스텀, 파일 연결, Quick Look 확장, 자동 업데이터, SQLite
4. **CLI → 에이전트 연동 패턴** — Unix 소켓으로 앱을 외부 조종. MCP 서버로 감싸기 좋음
5. **수익화 가능한 기반** — 연매출 100만 달러 밑이면 예외 조항으로 자유롭게 제품화

---

## 2. 쉬운 비유 버전

### 2-1. 일반 에디터 vs DOMD

- **다른 에디터들**: 한글 편지를 영어로 번역해 보관하고, 보여줄 때 다시 한글로 번역 → 번역 두 번 거치니 원문 뉘앙스가 틀어짐
- **DOMD**: 한글 편지 자체를 그대로 두고 예쁘게 보이게만 함 → Git 커밋 diff가 깨끗함

### 2-2. 30KB가 왜 대박인가

| | 용량 |
| --- | --- |
| DOMD 커널 | 약 30KB |
| 일반 리치텍스트 에디터 | 수백 KB |

카톡 프로필 사진 한 장(~100KB)보다 작은 용량에 에디터 전체가 들어있다는 뜻. 모바일 로딩이 순간.

### 2-3. 오프라인 협업

카페에서 노트북과 폰으로 같은 문단을 와이파이 없이 각각 수정했을 때

- 보통 앱: 마지막 저장 기준으로 한 명 작업이 사라짐
- DOMD: 나중에 연결되면 둘 다 살아남음

같은 레고 작품의 왼쪽 날개와 오른쪽 날개를 따로 만들어 와서 나중에 합쳐지는 느낌.

### 2-4. AI 스트리밍 = 요리 비유

AI는 한 글자씩 뱉어서 표를 쓰다가 중간에 끊긴 상태가 계속 생김.

```
| 이름 | 나
```

이 상태로도 DOMD는 "표 만드는 중"으로 알고 예쁘게 보여주고, 나머지가 도착하면 깜빡임 없이 완성. 다른 에디터는 이 순간 화면이 깜빡이거나 커서가 튐.

### 2-5. SEARCH/REPLACE = 수술 비유

보통은 문서 전체를 다시 써서 보냄 → 느리고 비싸고 다른 부분까지 망가짐.

```
<<<<<<< SEARCH
안녕하세요
=======
반갑습니다
>>>>>>> REPLACE
```

"전신 수술 말고 딱 이 부분만 찝어서 바꿔." 그리고 `<<<<<<<`는 git 충돌 마커라서 AI들이 학습 때 수백만 번 본 형식 → 새로 만든 포맷보다 훨씬 정확하게 따라함. 틀리면 "이 부분 못 찾았어"라고 되돌려 한 번 더 기회를 줌.

### 2-6. 프로젝트 구성을 식당으로 비유

| 폴더 | 식당 비유 |
| --- | --- |
| `.packages/@do-md/core` | 주방 (비법 레시피) — 가장 값나가는 자산, 돈 받고 팔기도 함 |
| `app/`, `features/` | 홀 & 인테리어 — 손님이 보는 화면, 전부 무료 공개(MIT) |
| `src-tauri/` | 배달 트럭(맥 앱) — 주방 음식을 데스크탑까지 배달 |
| `domd-cli` | 전화 주문 창구 — AI 로봇이 "창 열어, 글 써넣어" 주문 가능 |
| `services/signaling` | 소개팅 주선자 — 손님끼리 연결만 시켜주고 대화 내용은 안 들음 |
| `.packages/@do-md/plugins` | 양념 세트 — 목차, 찾기/바꾸기, 서식 명령 (MIT) |

### 2-7. 한 줄 결론

> 가볍고, 원본을 안 망치고, AI랑 궁합 최고인 마크다운 에디터 + 남이 가져다 쓸 수 있게 포장된 엔진
> → **완성된 제품**과 **그 제품을 만든 엔진** 두 개를 동시에 손에 쥔 것

---

## 3. 질문별 상세 답변

### 3-1. 설치 및 사용법

#### 방법 A — 그냥 써보기 (설치 0초)

https://www.domd.app/editor 접속 → `.md` 파일 드래그. 모든 처리가 기기 안에서 끝남.

#### 방법 B — macOS 앱 설치

- Apple Silicon: https://github.com/do-md/domd/releases/latest/download/DOMD_aarch64.dmg
- Intel: https://github.com/do-md/domd/releases/latest/download/DOMD_x86_64.dmg

`.md`/`.markdown` 파일 연결 + Finder Quick Look 미리보기 + 자동 업데이트 포함.
**Windows 네이티브 빌드는 공식 미지원.**

#### 방법 C — 저장소 직접 실행

```bash
git clone https://github.com/bmshin94/domd
cd domd
npm install          # legacy-peer-deps=true 가 .npmrc에 이미 설정됨
npm run dev          # → http://localhost:3000
```

필요: Node.js LTS + npm + Git. Windows/macOS/Linux 모두 가능.

| URL | 내용 |
| --- | --- |
| `/editor` | 메인 에디터 |
| `/chat` | 채팅형 마크다운 입력창 데모 |
| `/collab` | 실시간 협업 |
| `/playground` | AI 스트리밍 데모 |
| `/playground/live` | 실시간 동기화 데모 |
| `/playground/crdt` | 분할화면 오프라인 병합 데모 |
| `/preview` | 미리보기 |

기타 스크립트: `npm run build`, `npm run start`, `npm run lint`, `npm run analyze`(번들 분석)

#### 방법 D — 네이티브 앱 개발 (macOS 필수)

```bash
npm run tauri dev    # Rust 툴체인 + Xcode 필요
```

#### 방법 E — 내 React 프로젝트에 엔진 넣기

```bash
npm install @do-md/core-react immer
```

```tsx
import { DOMD, DOMDProvider, useEditor, toMarkdown } from "@do-md/core-react";
import "@do-md/core-react/style.css";
```

요구사항: React 18+ / react-dom 18+ / immer 10.2+ 또는 11 (peerDependencies)

주요 공개 API

- 컴포넌트: `DOMD`, `DOMDProvider`, `Renderer`, `RenderChildren`
- 훅: `useEditor`, `useEditorStore`, `useEditorStoreApi`, `useRenderData`, `useEditorDom`, `useFormatState`
- 직렬화: `toMarkdown`
- 인라인 확장: `defaultInlineRules`, `viewOnlyProps`, `InlineRule` 타입
- AI 일괄 편집: `RangeEdit`, `TextEdit`, `ReplaceResult`, `ReplaceFailureReason`
- 동기화 심(CRDT용): `serializeRenderData`, `deserializeRenderData`, `diffRenderData`, `applyRenderDataOpsToDraft`, `subscribeRenderDataOps`, `applyExternalRenderDataOps`
- 선택 영역: `SelectionTarget`, `SelectionSearchTarget`, `SelectionRangeTarget`

#### 방법 F — CLI (AI 에이전트용)

```bash
domd-cli new                       # 새 창, window id 출력
domd-cli open ~/note.md            # 파일 열기/포커스
domd-cli list                      # 열린 창 JSON 목록
domd-cli content --window <id>     # 전체 마크다운 읽기
domd-cli selection --window <id>   # 선택 영역 JSON
domd-cli insert --window <id> "텍스트"   # 삽입 (반복 호출 = 스트리밍)
domd-cli save --window <id> [경로]
domd-cli focus --window <id>
domd-cli close --window <id> [--force]
```

`~/.domd/cli.sock` 유닉스 소켓 통신. 앱이 꺼져있으면 자동 실행. 단일값은 평문, 구조적 결과는 JSON, 에러는 종료코드 ≠0 + stderr JSON.

#### 방법 G — 시그널링 서버 셀프호스팅

```bash
cd services/signaling
npm install --include=dev
npm run dev        # ws://localhost:8787
npm run deploy     # Cloudflare Workers 배포
```

웹앱 빌드에 `NEXT_PUBLIC_SIGNALING_URL=wss://...` 설정.

### 3-2. 플러그인? 스킬? MCP?

**셋 다 아님.** 정확히는 **독립 데스크탑/웹 애플리케이션 + npm React 라이브러리** 조합.

| 타입 | 맞나? | 이유 |
| --- | --- | --- |
| Claude Code 플러그인 | X | `.claude/plugins`, `plugin.json` 없음 |
| Claude Skill | X | `SKILL.md` 없음 (`CLAUDE.md`는 별도로 추가된 페르소나 파일) |
| MCP 서버 | X | MCP SDK 의존성 없음, stdio/SSE 핸들러 없음 |
| VSCode 확장 | X | `package.json`에 `contributes` 없음 |
| **앱 + 라이브러리** | O | Next.js 웹앱 + Tauri 맥앱 + npm 패키지 |

헷갈릴 수 있는 두 가지

1. **DOMD 자체의 내부 플러그인 시스템** — `plugins/` 폴더. 카테고리가 `rendering`/`parsing`/`toolbar`/`collaboration`/`shared`로 정리되고 각 디렉토리가 `index.ts`를 엔트리로 노출. "DOMD에 꽂는 플러그인"이지 Claude 플러그인이 아님
2. **macOS Quick Look 확장** — `src-tauri/preview-extension`. Finder 확장

**다만 MCP 서버로 만들 수는 있음**: `domd-cli`가 이미 완벽한 조종 인터페이스라서, 감싸는 MCP 서버를 만들면 Claude가 `domd_open`/`domd_insert`/`domd_content`/`domd_save` 툴을 쓸 수 있고, Claude가 글 쓰는 걸 맥 앱에서 실시간으로 볼 수 있음. 반나절~하루 작업량.

### 3-3. API 토큰이 필요한가

**에디터 본체는 토큰 전혀 필요 없음.**

| 기능 | 토큰 필요? | 상세 |
| --- | --- | --- |
| 마크다운 편집 (웹/맥앱) | 불필요 | 계정도 로그인도 없음 |
| 대용량 파일 열기 | 불필요 | 전부 로컬 |
| 실시간 협업 | 불필요 | 계정/인증 없음. 방 비밀번호만 공유 (AES-GCM 키 유도) |
| Quick Look 미리보기 | 불필요 | |
| CLI (`domd-cli`) | 불필요 | 로컬 유닉스 소켓 |
| 자동 업데이트 | 불필요 | GitHub 릴리스 공개 |
| **AI 에이전트 기능** | **필요** | 여기만 |
| 시그널링 직접 배포 | 일부 | Cloudflare 계정만 (코드는 무인증) |

#### AI 기능의 토큰 구조 = BYOK (Bring Your Own Key)

내장 프리셋

| 프로바이더 | 기본 모델 | 엔드포인트 |
| --- | --- | --- |
| OpenAI | `gpt-4o-mini` | `https://api.openai.com/v1/chat/completions` |
| OpenRouter | `anthropic/claude-3.5-haiku` | `https://openrouter.ai/api/v1/chat/completions` |

**커스텀 엔드포인트 무제한 추가 가능** (v0.9.0 추가). `normalizeChatEndpoint`가 세 가지 입력 형태를 모두 받아줌.

```
https://api.deepseek.com                   → .../v1/chat/completions
https://api.deepseek.com/v1                → .../v1/chat/completions
https://proxy.example/v1/chat/completions   → 그대로 사용
```

→ DeepSeek, Moonshot, Groq, 리버스 프록시, Ollama 같은 로컬 서버까지 지원. 여러 에이전트가 하나의 엔드포인트+키를 공유 가능.

키 저장 위치

- 웹: `localStorage` (`domd-ai:key:<provider>`)
- 맥 앱: `~/.domd/ai.json` — 사용자가 직접 보고 백업 가능, 웹뷰 사이트 데이터에 종속되지 않음
- 맥 첫 실행 시 기존 localStorage 설정을 파일로 자동 마이그레이션 후 정리
- 파일이 깨지면 localStorage로 폴백 (콘솔 경고)

중요: `llm.ts` 주석에 명시 — *"Runs entirely in the browser — the key never touches a DOMD server."* 요청이 브라우저에서 프로바이더로 직접 전송. 토큰 비용은 사용자가 직접 부담(중개 마진 없음).

에이전트 설정 항목: 이름, 프로바이더, 모델, 페르소나 프롬프트, 프레즌스 색상(`#8a7aa8`, `#8fbcbb`, `#d08770`, `#a3be8c`, `#b48ead`). 에이전트는 `ai-<id>` 협업자 ID를 가져 사람 공동편집자처럼 커서가 보임.

### 3-4. AI 에이전트 구축에 도움이 될까

**매우 도움됨. 이 프로젝트의 최대 가치.**

1. **스트리밍 렌더링** — `react-markdown` 류로 뿌리면 토큰마다 전체 재파싱 → 깜빡임, 커서 튐, 프레임 드랍. DOMD는 변경된 노드만 리렌더로 해결, 2만 줄 검증. 직접 풀면 몇 주 소요
2. **Aider급 편집 적용기** — `plugins/ai-collab/edit-blocks.ts` + `features/ai/lib/llm.ts`: SEARCH/REPLACE 파싱, 첫 매치만 치환, 실패 분류(`not_found`/`malformed`), Aider식 반성 메시지로 1회 자동 교정, "성공한 N개는 다시 보내지 마"까지 프롬프트에 포함
3. **프로그래매틱 편집 API 공개** — `RangeEdit`, `TextEdit`, `ReplaceResult`, `ReplaceFailureReason` + `SelectionTarget`/`SelectionSearchTarget`로 커서 위치를 코드로 지정
4. **`<<CURSOR>>` 마커 패턴** — 문서에 커서 위치를 마커로 심어 모델에 전달. "AI가 커서 맥락을 안다"를 구현하는 레시피
5. **MODE A/B 분리 설계** — A는 기존 텍스트 수정(SEARCH/REPLACE), B는 커서에 새 내용 생성(raw 마크다운). "수정/번역/교정/요약/확장은 반드시 A, 애매하면 A"를 프롬프트에 명시
6. **CLI = 에이전트의 화면** — `domd-cli`로 로컬 마크다운 렌더링 서피스 확보. `insert` 반복 호출이 곧 스트리밍 (별도 스트림 모드 불필요하게 설계)
7. **AI를 협업자로 모델링** — `agent-peer.ts`에서 AI가 CRDT 피어로 참여. 멀티 에이전트 UX 레퍼런스

적합한 제품: AI 글쓰기 도구, 코드 리뷰 리포트 뷰어, 에이전트 작업 로그 UI, RAG 답변 편집기, AI 번역/교정 SaaS, 자율 에이전트 산출물 작성 화면

### 3-5. React나 PHP로 만들 수 있나

#### React — 이미 React임

커널이 React 전용. `peerDependencies`가 `react >=18`, `react-dom >=18`, `immer ^10.2 || ^11`. 패키지 이름도 `@do-md/core-react`.

```tsx
"use client";
import { DOMD, DOMDProvider } from "@do-md/core-react";
import "@do-md/core-react/style.css";

export default function MyEditor() {
  return (
    <DOMDProvider>
      <DOMD />
    </DOMDProvider>
  );
}
```

| 환경 | 가능? | 주의 |
| --- | --- | --- |
| Next.js (App Router) | O | 이 저장소가 Next 16 + React 19.2.4로 증명. `"use client"` 필수 |
| Vite + React | O | 가장 깔끔 |
| CRA | O | |
| React Native | X | `contenteditable`/DOM 기반이라 불가 |
| Vue / Svelte / Angular | X | React 렌더러에 묶임. iframe 래핑은 가능하나 비추천 |

React로 할 수 있는 확장: `{.mention}`/`{.comment}`/`{.task}` 커스텀 문법 추가, 그 룰에 React 컴포넌트 바인딩, `plugins/rendering/`에 차트·Mermaid·KaTeX 렌더러 추가, `@do-md/toc` + `@do-md/search` 조합 문서 포털

#### PHP — 반은 되고 반은 안 됨

**안 되는 것**: 커널을 PHP로 재작성은 불가능에 가까움. `contenteditable`, `Selection`/`Range` API, DOM 측정, requestAnimationFrame 등 브라우저 전용 기능에 완전 의존.

**되는 것**: PHP는 백엔드 역할로 완벽.

```
[브라우저] DOMD (React/JS)  <->  [서버] PHP (Laravel/WordPress)
   편집·렌더링                     저장·인증·권한·검색·버전관리
```

| PHP 스택 | 통합 방법 |
| --- | --- |
| **Laravel** | Inertia.js + React 어댑터로 DOMD 마운트. 마크다운 원문은 DB `TEXT` 컬럼. API로 `toMarkdown()` 결과 POST |
| **WordPress** | 구텐베르크 블록(React 기반)으로 감싸 커스텀 블록 제작. 포스트 메타에 마크다운 저장. WP가 GPL이라 **FOSS 예외로 라이선스 완전 깨끗** |
| **Symfony / CodeIgniter** | Blade/Twig 템플릿에 React 번들 마운트 |
| **순수 PHP** | `<div id="editor">` + 번들 `<script>` + `fetch()`로 저장 엔드포인트 호출 |

주의점: 서버 사이드 HTML 미리 렌더가 필요하면 PHP 마크다운 파서(`league/commonmark` 등)를 별도로 써야 하고 DOMD 파서와 미묘한 차이가 생길 수 있음. 대용량 문서는 `post_max_size`, `max_input_vars` 조정. 저장은 전체 교체보다 diff 전송이 효율적(`diff-match-patch`가 이미 의존성에 있음).

추천 조합: **Laravel + Inertia + React + DOMD**

### 3-6. 유튜브 강의 영상 제작 가능할까

**가능하며 소재가 풍부함.**

법적 근거

- 애플리케이션 레이어 = MIT → 코드 보여주기·설명·수정·배포 자유
- 커널 = GPL-3.0이지만 "코드를 읽고 설명하고 가르치는 행위"는 배포(conveying)가 아님 → 의무 없음
- README 명시: *"Trying the kernel, building with it, and running it internally carry no obligations"*
- 영상 수익화 문제 없음 (GPL은 코드 배포를 규율, 설명 콘텐츠를 규율하지 않음)
- 주의: 강의용 데모 앱을 **빌드해서 배포**하면 그때 GPL 또는 예외 조항 적용. 영상에서 라이선스 구조를 한 번 언급하면 안전

소재 가치

1. 희소성 — 한국어로 "에디터 커널 직접 구현"을 다루는 콘텐츠가 거의 없음
2. 코드 주석이 "왜 이렇게 했는지"까지 설명 → 강의 스크립트가 반쯤 확보됨
3. 트렌드 적중 — AI 스트리밍, CRDT 협업, Tauri 네이티브, MCP
4. 오픈소스 라이선스 수익화라는 희귀 주제

#### 커리큘럼 초안

**시즌 1 — 입문 (각 10~15분)**

| # | 제목 |
| --- | --- |
| 1 | "노션보다 가벼운 마크다운 에디터를 깃허브에서 줍줍했다" (전수조사 + 데모) |
| 2 | 5분만에 내 React 프로젝트에 WYSIWYG 에디터 넣기 |
| 3 | 1MB 마크다운 파일 열어봤는데 안 멈춘다고? (성능 비교) |
| 4 | macOS 앱 빌드해서 내 손에 올리기 (Tauri 2 입문) |

**시즌 2 — AI 연동**

| # | 제목 |
| --- | --- |
| 5 | AI가 쓰는 글이 깜빡이지 않게 하는 방법 (스트리밍 렌더링 원리) |
| 6 | Aider가 쓰는 SEARCH/REPLACE 포맷, 왜 JSON보다 정확한가 |
| 7 | 내 OpenAI 키로 에디터 안에 AI 비서 만들기 (BYOK 실전) |
| 8 | 로컬 Ollama를 에디터에 연결하기 (커스텀 엔드포인트) |
| 9 | domd-cli를 MCP 서버로 감싸서 Claude가 내 맥에 글 쓰게 하기 |

**시즌 3 — 심화**

| # | 제목 |
| --- | --- |
| 10 | ProseMirror 없이 contenteditable 다루기: 커서 지옥 생존기 |
| 11 | CRDT로 오프라인 충돌 없이 합치기 (Yjs + 어댑터 패턴) |
| 12 | 서버가 내용을 못 보는 실시간 협업 만들기 (WebRTC + AES-GCM) |
| 13 | 마크다운에 내가 만든 문법 추가하기 (인라인 룰 + React 컴포넌트) |
| 14 | 30KB에 에디터를 집어넣는 번들 최적화 기법 |
| 15 | GPL인데 돈 버는 법: 이중 라이선스 + 예외 조항 해부 |

**시즌 4 — 실전 프로젝트**

| # | 제목 |
| --- | --- |
| 16 | Laravel + React로 사내 위키 만들기 |
| 17 | 워드프레스 구텐베르크 블록으로 만들어 팔기 |
| 18 | AI 블로그 작성 SaaS 7일 만에 만들기 |

제작 팁: 1화는 "줍줍했다" 썸네일. `/playground`, `/playground/crdt`, `/playground/live` 데모가 이미 영상 소재로 완벽(분할화면 CRDT 병합은 비주얼 임팩트 최고). 코드 리딩 영상은 `scripts/verify-*` 실행 결과를 보여주면 설득력 상승. 영어 자막으로 글로벌 노출.

---

## 4. 수익화 아이디어 상세

### 전제 (라이선스)

> 연매출 100만 USD 미만 + 투자유치 200만 USD 미만이면 **소규모 주체 예외**로 GPL 아닌 독점 소프트웨어에 커널을 넣어 원하는 조건으로 판매 가능. 합법.
> 이 예외는 공개된 버전에 대해 **철회 불가** 명시 → 나중에 정책이 바뀌어도 그 버전은 안전.

### 티어 1 — 당장 시작 가능 (투자금 0원)

#### (1) 유튜브 + 온라인 강의 패키지

| 항목 | 내용 |
| --- | --- |
| 난이도 | 하 |
| 수익 모델 | 애드센스 + 강의 판매 + 멤버십 |
| 예상 | 월 50~500만원 (구독자 규모 비례) |
| 기간 | 1개월 내 시작 |

전략: 무료 유튜브로 시즌 1~2 공개해 유입 확보 → 유료 강의로 시즌 3~4(심화 + 실전 프로젝트) 판매(5~15만원대) → 부록으로 보일러플레이트 레포 제공. 리스크 최소, 시작 최속.

#### (2) 에디터 임베딩 외주/컨설팅

| 항목 | 내용 |
| --- | --- |
| 난이도 | 중 |
| 수익 모델 | 프로젝트당 과금 |
| 예상 | 건당 300~2,000만원 |
| 기간 | 포트폴리오 2주 + 영업 |

차별점: 번들 30KB(경쟁사 수백 KB), 마크다운 원문 보존으로 Git/DB diff가 깨끗, AI 스트리밍 기본 탑재, 실시간 협업 옵션. 데모 3개(기본 임베딩 / AI 어시스턴트 / 실시간 협업) 준비 후 영업.

#### (3) 유료 템플릿·보일러플레이트 판매

| 항목 | 내용 |
| --- | --- |
| 난이도 | 하 |
| 수익 모델 | Gumroad / Lemon Squeezy / 자체 판매 |
| 예상 | 개당 $29~99, 월 $500~3,000 |
| 기간 | 2~4주 |

템플릿 아이디어: "AI 글쓰기 SaaS 스타터킷"(Next.js + DOMD + BYOK AI + Stripe + 인증), "마크다운 노트앱 스타터"(Tauri + DOMD + 로컬 저장), "실시간 협업 문서 스타터"(CRDT + 시그널링 배포 스크립트).

라이선스 주의: 템플릿은 소스 코드 판매라 GPL 전파 문제 발생. 해결책 — 템플릿 자체를 MIT로 공개하고 "설치 서비스 + 지원 + 영상 가이드"를 묶어 판매, 또는 FOSS 예외 아래 MIT 배포.

### 티어 2 — 제품 만들기 (1~3개월)

#### (4) 버티컬 AI 글쓰기 SaaS (1순위 추천)

| 항목 | 내용 |
| --- | --- |
| 난이도 | 중상 |
| 수익 모델 | 월 구독 (9,900~49,000원) |
| 예상 | MRR 100~1,000만원 |
| 기간 | 2~3개월 |

DOMD의 유리함: AI 출력이 깜빡임 없이 스트리밍, SEARCH/REPLACE로 문서 일부만 정확히 수정해 토큰 비용 절감, 마크다운 원문 보존으로 블로그/깃허브/정적사이트에 바로 연결.

버티컬을 좁게 잡는 것이 핵심 (넓으면 노션과 정면 충돌).

| 버티컬 | 킬러 기능 |
| --- | --- |
| 기술 블로그 작성 | 코드 블록 하이라이트 + GitHub 연동 + SEO 메타 생성 |
| 뉴스레터 | 구독자 관리 + 마크다운→이메일 HTML 변환 |
| 전자책/웹소설 | 챕터 관리 + 분량 추적 + AI 교정 |
| 학술 논문 초고 | 인용 관리 + 수식 + 레퍼런스 자동 정리 |
| 제품 문서(Docs) | 버전관리 + 다국어 + 리뷰 워크플로 |

비용 전략: BYOK 플랜(사용자 키 입력, 월 4,900원)과 포함 플랜(토큰 비용 대납, 월 29,000원)을 동시 제공해 가격 저항 완화.

#### (5) WordPress 유료 플러그인

| 항목 | 내용 |
| --- | --- |
| 난이도 | 중 |
| 수익 모델 | 라이선스 연간 과금 |
| 예상 | 개당 $49~199/년, 연 $5,000~50,000 |
| 기간 | 1.5~2개월 |

라이선스가 완벽하게 깔끔 — WordPress는 GPL이라 FOSS 예외에 정확히 부합. 시장 규모도 큼(WP가 전세계 웹사이트 40%+).
아이디어: "진짜 마크다운 구텐베르크 블록"(기존 WP 마크다운 플러그인은 전부 플레인텍스트), AI 글쓰기 어시스턴트 블록. Freemium(기본 무료 → AI/협업 Pro). WP 플러그인 시장은 자동 갱신 구독이 표준이라 MRR이 안정적.

#### (6) Laravel / Nuxt / Next 유료 패키지

| 항목 | 내용 |
| --- | --- |
| 난이도 | 중 |
| 수익 모델 | 패키지 라이선스 |
| 예상 | $99~299 평생, 월 $1,000~5,000 |
| 기간 | 1~2개월 |

Laravel 생태계용 DOMD 어댑터 패키지(Inertia 통합 + Blade 디렉티브 + 저장 트레이트 + 이미지 업로드). PHP 개발자는 모던 에디터 붙이기를 어려워해 수요 확실. Laravel Nova 패키지 / Filament 플러그인 시장도 유료 판매 활발.

### 티어 3 — 차별화 승부 (3~6개월)

#### (7) AI 에이전트 작업 뷰어 / MCP 제품 (가장 트렌디)

| 항목 | 내용 |
| --- | --- |
| 난이도 | 상 |
| 수익 모델 | 구독 / 팀 시트 |
| 예상 | 블루오션 |
| 기간 | 3~4개월 |

AI 에이전트의 작업 결과를 사람이 읽고 고칠 수 있는 마크다운 문서로 실시간 스트리밍하는 제품.
- `domd-cli`를 MCP 서버로 감싸 Claude/Cursor/Codex가 문서를 직접 작성
- AI가 CRDT 피어로 참여 (`plugins/ai-collab/agent-peer.ts`에 이미 구조 존재)
- 사람과 AI가 같은 문서에 커서를 띄우고 동시 편집
- 에이전트별 색상 프레즌스 + 작성자 하이라이트(`versioning/author-highlights.tsx`)

아직 비어있는 영역. 선점 임팩트 큼.

#### (8) 프라이버시 특화 협업 문서 (B2B)

| 항목 | 내용 |
| --- | --- |
| 난이도 | 상 |
| 수익 모델 | B2B 연간 계약 |
| 예상 | 계약당 수백~수천만원 |
| 기간 | 4~6개월 |

세일즈 포인트: "우리 서버는 문서 내용을 **구조적으로** 볼 수 없습니다" — 암호화가 아니라 아키텍처가 P2P라서 애초에 지나가지 않음.
타겟: 병원, 법무법인, 회계법인, 공공기관, 국방/금융 등 데이터 반출 규제로 노션/구글독스를 못 쓰는 곳. 온프레미스 설치 + 유지보수 계약이 가장 수익성 높음.

#### (9) 네이티브 노트앱 (유료 앱)

| 항목 | 내용 |
| --- | --- |
| 난이도 | 중 |
| 수익 모델 | 일회성 구매 / 선택적 싱크 구독 |
| 예상 | 앱 $19~39, 싱크 월 $3 |
| 기간 | 2~3개월 |

Obsidian/Bear 대안 포지션. 기본 앱은 일회성 구매(로컬 전용), 기기간 동기화·백업·히스토리만 구독. Tauri 베이스라 용량 작음.
주의: 앱 스토어 배포는 conveying이라 예외 조항 확인 필수.

### 추천 로드맵

```
1개월차:   유튜브 시작 (1)  +  데모 3개 제작
           -> 유입 확보 + 포트폴리오 확보
2~3개월차: 외주 수주 (2)  +  템플릿 판매 (3)
           -> 현금흐름 확보
4~6개월차: SaaS 또는 WP 플러그인 (4)/(5)
           -> MRR 축적
7개월~:    MCP/에이전트 제품 (7) 로 차별화 승부
```

### 핵심 원칙 3가지

1. **콘텐츠 → 신뢰 → 제품** 순서. 유튜브가 영업·마케팅·포트폴리오를 동시에 해결
2. **버티컬을 좁게**. "AI 에디터"가 아니라 "기술 블로거용 AI 에디터"
3. **라이선스를 세일즈 포인트로**. "GPL 오픈소스 커널 기반, 코드 감사 가능" = B2B에서 강점

### 반드시 확인할 것

상용 서비스 시작 전에 `.packages/@do-md/core/LICENSE-EXCEPTIONS.md` 전문을 읽고, 애매하면 effyouapp@gmail.com 으로 문의. 예외 조항은 관대하지만 "브라우저에 빌드 전달 = 배포"라는 정의가 생각보다 넓음.

---

## 5. 요약 체크리스트

- [x] 이게 뭔지: 마크다운 네이티브 WYSIWYG 에디터 + 30KB 자체 제작 커널
- [x] 플러그인/스킬/MCP 여부: 전부 아니고 **앱 + npm 라이브러리**. 단 MCP 서버로 감싸는 건 가능
- [x] API 토큰: 에디터 본체는 불필요, **AI 기능만 BYOK 방식으로 필요**
- [x] AI 에이전트 구축: 매우 유리 (스트리밍 렌더 + SEARCH/REPLACE 적용기 + CLI)
- [x] React: 이미 React 전용 / PHP: 백엔드로만 가능 (Laravel + Inertia 추천)
- [x] 유튜브: 법적으로 안전하고 소재 풍부 (18강 커리큘럼 초안 포함)
- [x] 수익화: 티어 1(유튜브/외주/템플릿) → 티어 2(SaaS/WP 플러그인) → 티어 3(MCP 제품/B2B)

# discussion-topics 인터랙티브 진행보조 HTML 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `discussion-topics` 스킬이 가벼운 발제(질문+숨긴 근거)를 뽑고, 고정 사이드바 목차·근거 펼침·Motion 등장·hover/pressed를 갖춘 자체완결 진행보조 HTML을 만들도록 바꾼다.

**Architecture:** 산출물은 스킬 지시 파일(마크다운)과 그 스킬이 생성하는 단일 HTML이다. 유닛 테스트 프레임워크가 없으므로 "테스트"는 (a) 스킬 파일에 대한 `grep` 단언과 (b) 렌더된 HTML에 대한 브라우저 검증(Playwright `evaluate`)이다. 추출/렌더는 Agent 도구로 서브에이전트를 띄워 수행한다(아티클당 추출 1개 — 컨텍스트 격리, 렌더 1개).

**Tech Stack:** Claude Code Skill(마크다운), 단일 HTML/CSS/JS, Motion One(`motion@11`, `window.Motion`), KoPub Batang + Pretendard + JetBrains Mono(CDN), Playwright(검증), Python `http.server`(로컬 서빙).

**스펙:** `docs/superpowers/specs/2026-06-16-discussion-topics-interactive-design.md`

---

## 파일 구조

- `.claude/skills/discussion-topics/SKILL.md` — 수정: 추출 계약(발제+근거)·렌더 계약(요약 + INTERACTIONS.md 참조)·frontmatter description.
- `.claude/skills/discussion-topics/INTERACTIONS.md` — 신규: HTML 구조·인터랙션 상세 계약(렌더 서브에이전트가 DESIGN.md와 함께 읽음).
- `.claude/skills/discussion-topics/CLARITY.md` — 변경 없음.
- `.claude/skills/discussion-topics/DESIGN.md` — 변경 없음.
- `sessions/session_1/discussion.html` — 재생성(추출+렌더 서브에이전트 실행).

스킬 분리 이유: SKILL.md는 100줄 이하 유지(오케스트레이션 흐름), 인터랙션 상세는 INTERACTIONS.md로 분리해 렌더 서브에이전트가 집중해서 읽도록 한다.

---

## Task 1: SKILL.md 추출 계약을 발제+근거로 변경

**Files:**
- Modify: `.claude/skills/discussion-topics/SKILL.md` (워크플로우 3·4단계)

- [ ] **Step 1: 검증 — 현재 계약이 논제/질문 기반인지 확인**

Run:
```bash
cd "/c/Users/sok89/OneDrive/Desktop/harness-engineering-study"
grep -nE "논제. 3~5|질문. 2~3|교차 논제" .claude/skills/discussion-topics/SKILL.md
```
Expected: 3단계의 `논제 3~5개 / 질문 2~3개`와 4단계의 `교차 논제`가 매칭됨.

- [ ] **Step 2: 3단계(아티클별 추출) 반환 계약 교체**

`SKILL.md`에서 아래 문장을 찾아:
```
반환: `검증 주장` / `논제` 3~5개 / `질문` 2~3개 / `요약`(교차 종합용 한 문장).
```
다음으로 교체(기존 `논제`+`질문` 분리를 단일 `발제`로 통합, 각 발제는 질문+근거 2필드):
```
반환: `검증 주장` 1개 / `발제` 3~5개(각 발제 = `질문`[가벼운 C 높이 질문] + `근거`[그 질문을 뒷받침하는 아티클의 구체 사실 1~2문장, 수치·시나리오 포함]) / `요약`(교차 종합용 한 문장). 질문은 숫자·실험조건 없이도 토론에 진입 가능해야 하고, 근거는 아티클에 실재하는 사실만 쓴다.
```

- [ ] **Step 3: 4단계(교차 종합) 교체**

아래 문장을 찾아:
```
4. **교차 종합.** 반환된 `요약`+`검증 주장`만으로(전체 본문 아님) 아티클 2개 이상을 잇는 `교차 논제` 2~4개를 만든다.
```
다음으로 교체:
```
4. **교차 종합.** 반환된 `요약`+`검증 주장`만으로(전체 본문 아님) 아티클 2개 이상을 잇는 `교차 발제` 2~4개를 만든다(각 발제 = 가벼운 질문 + 세 글에서 모은 근거).
```

- [ ] **Step 4: 검증 — 새 계약이 반영됐는지 확인**

Run:
```bash
grep -nE "발제. 3~5개|각 발제 = `질문`|교차 발제" .claude/skills/discussion-topics/SKILL.md
grep -c "논제. 3~5" .claude/skills/discussion-topics/SKILL.md
```
Expected: 첫 grep은 새 문장들을 매칭, 둘째 grep은 `0`(옛 논제 계약 사라짐).

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/discussion-topics/SKILL.md
git commit -m "feat(discussion-topics): extract light 발제 + tucked 근거 instead of 논제/질문"
```

---

## Task 2: INTERACTIONS.md 작성(렌더·인터랙션 계약)

**Files:**
- Create: `.claude/skills/discussion-topics/INTERACTIONS.md`

- [ ] **Step 1: INTERACTIONS.md 생성**

아래 내용을 그대로 작성한다:

````markdown
# discussion.html 인터랙션·렌더 계약

`discussion-topics`가 만드는 `discussion.html`의 구조·인터랙션 규칙. 렌더 서브에이전트는 `DESIGN.md`(시각 토큰)와 이 파일을 함께 따른다. 산출물은 **자체완결 단일 HTML**, 크림/라이트 팔레트(다크 표면 없음), 진행자가 공유 화면에서 함께 보며 토론을 진행하는 용도.

## 레이아웃
- `.layout`: CSS grid `264px minmax(0,1fr)` — 좌 고정 사이드바 + 본문(max-width ~820px). 880px 이하 1단 스택(사이드바는 상단 가로 목차로, `position:static`).
- 간격·패딩·radius는 DESIGN.md 토큰 사용(섹션 96px, 카드 32px, r-lg 12px). 모든 토큰은 `:root` CSS 변수로 선언, 인라인 hex 금지.

## 폰트 (CDN + 폴백)
- KoPub Batang: `https://cdn.jsdelivr.net/npm/font-kopubworld@1.0.3/css/batang.css` (family `"KoPubWorld Batang"`)
- Pretendard: `https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.min.css` (family `"Pretendard"`)
- JetBrains Mono: Google Fonts(코드)
- `--font-display: "KoPubWorld Batang", serif` / `--font-sans: "Pretendard", -apple-system, BlinkMacSystemFont, system-ui, "Apple SD Gothic Neo", "Malgun Gothic", sans-serif` / `--font-mono: "JetBrains Mono", ui-monospace, monospace`
- 디스플레이 음수 letter-spacing은 -0.5px 이내.

## 사이드바 목차 (고정)
- `position:sticky; top:0; height:100vh; overflow:auto`, 우측 hairline 경계, surface-soft 배경.
- 내용: `SESSION N · 논제` 제목 → 아티클 그룹(번호·이름)별 발제 짧은 라벨 링크(`href="#<발제id>"`) → 교차 논제 그룹.
- 라벨은 발제 질문을 6~14자로 줄인 요약(질문 전문 아님).
- scrollspy: `IntersectionObserver`(`rootMargin: '-45% 0px -50% 0px'`)로 화면 중앙의 발제에 대응하는 링크에 `.active`(코랄 좌측 보더 + 굵게 + surface-card 배경).

## 본문
- 히어로: `caption-uppercase` eyebrow `SESSION N · 하네스 엔지니어링 스터디`, 디스플레이 serif H1, 한 줄 안내(글 안 읽어도 OK + "근거 보기" 안내).
- 아티클 섹션: 번호 배지(코랄·mono·pill) + serif 타이틀 + muted 이름.
- 발제 카드 `.thesis`: 가벼운 질문이 헤드라인 `.q`, 아래 근거 토글 버튼.
- 교차 논제: surface-cream-strong 밴드.

## 근거 펼침
- 버튼 `.evidence-toggle`: `▶ 근거 보기` ↔ `▼ 근거 접기`(chevron 회전), `aria-expanded` 토글.
- 펼침: CSS `grid-template-rows: 0fr → 1fr` 전환(`.34s cubic-bezier(.22,1,.36,1)`), 내부 wrapper `overflow:hidden`. `.thesis.open`에서 `1fr`.
- 근거 본문: canvas 배경 + 코랄 좌측 보더(3px) + `아티클 근거` 라벨. 코드 토큰·수치는 mono 인라인 `<code>`.
- JS는 `.open` 토글 + 라벨/`aria-expanded` 갱신만. 높이는 CSS가 처리.

## 등장 애니메이션 (Motion One)
- 스크립트: `https://cdn.jsdelivr.net/npm/motion@11/dist/motion.js`(`window.Motion`). `<script>`에 `integrity="sha384-…" crossorigin="anonymous"` 필수.
- `Motion.inView(el, () => Motion.animate(el, {opacity:[0,1], transform:['translateY(16px)','translateY(0)']}, {duration:.5, easing:[.22,1,.36,1]}), {margin:'0px 0px -12% 0px'})` — 아티클 헤더·발제 카드·교차 밴드(`[data-reveal]`)에 적용.
- 초기 숨김 `opacity:0`은 `.js [data-reveal]`에서만 적용(`<html class="js">`를 head 인라인 스크립트로 추가). JS/Motion 미동작 시 콘텐츠가 보이고, Motion 미로드 시 `opacity:1`로 폴백.

## hover/pressed (CSS만)
- `.thesis:hover` → `translateY(-2px)` + 약한 그림자, `transition:.16s`.
- `.evidence-toggle:hover` → surface-cream-strong 배경 + 코랄 보더, `:active` → `scale(.96)`.
- `.toc a:hover` → surface-card 배경.

## 접근성
- `@media (prefers-reduced-motion: reduce)`: `scroll-behavior:auto`, `.js [data-reveal]{opacity:1}`(등장 끔).
- 토글 버튼 `aria-expanded` 유지, 사이드바는 `<nav>`.

## 자체완결 점검
- 외부 참조는 폰트 CSS 3개 + Motion 스크립트 1개로 한정. 그 외 네트워크 의존 없음.
````

- [ ] **Step 2: 검증 — 핵심 항목 포함 확인**

Run:
```bash
grep -cE "grid-template-rows|IntersectionObserver|Motion.inView|prefers-reduced-motion|integrity=|font-kopubworld" .claude/skills/discussion-topics/INTERACTIONS.md
```
Expected: `6` 이상(모든 핵심 메커니즘 포함).

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/discussion-topics/INTERACTIONS.md
git commit -m "feat(discussion-topics): add INTERACTIONS.md render/interaction contract"
```

---

## Task 3: SKILL.md 렌더 계약을 INTERACTIONS.md 참조로 갱신

**Files:**
- Modify: `.claude/skills/discussion-topics/SKILL.md` (frontmatter description, 렌더 계약 섹션)

- [ ] **Step 1: frontmatter description 갱신**

`description:` 줄 끝의 "styled per the skill's DESIGN.md." 부분을 다음으로 확장:
```
styled per the skill's DESIGN.md and INTERACTIONS.md (sticky sidebar TOC, light 발제 with click-to-reveal 근거, Motion One entrance, hover/pressed).
```

- [ ] **Step 2: "## 렌더 계약 (HTML)" 섹션 본문을 교체**

기존 `## 렌더 계약 (HTML)` 아래 불릿 전체를 다음으로 교체:
```
- 산출물은 **자체완결 단일 .html**, 크림/라이트 팔레트(다크 표면 없음). 구조·인터랙션 규칙은 [INTERACTIONS.md](INTERACTIONS.md)를 따른다.
- 핵심: 좌측 **고정 사이드바 목차**(아티클→발제, scrollspy, 클릭 점프) + 본문의 **가벼운 발제 카드** + `▸ 근거 보기` 펼침.
- **Motion One**(`motion@11`, SRI 필수)으로 스크롤 등장, hover 떠오름·pressed는 CSS. `prefers-reduced-motion`이면 애니메이션·스무스 스크롤 끔.
- 폰트(KoPub Batang 명조 + Pretendard 본문 + JetBrains Mono 코드)와 토큰은 DESIGN.md/INTERACTIONS.md 지정 CDN·CSS 변수로. 디스플레이 음수 letter-spacing은 -0.5px 이내.
- 콘텐츠 → 컴포넌트: 발제 질문 = 카드 헤드라인, 근거 = 펼침 패널(코드·수치는 mono 인라인), 교차 발제 = surface-cream-strong 밴드, 푸터 = 라이트.
- **금지(DESIGN.md Don'ts)**: dark navy 표면, 쿨 그레이·순백 canvas, serif 디스플레이 굵게, 코랄 남용, serif 자리에 sans.
```

- [ ] **Step 3: 5단계(HTML 렌더)에서 렌더 서브에이전트가 INTERACTIONS.md도 읽도록 명시**

5단계 문장의 `DESIGN.md 경로(...DESIGN.md)` 부분을 다음으로 교체:
```
DESIGN.md 경로(`.claude/skills/discussion-topics/DESIGN.md`)와 INTERACTIONS.md 경로(`.claude/skills/discussion-topics/INTERACTIONS.md`)
```

- [ ] **Step 4: 검증 — 줄 수·참조 확인**

Run:
```bash
wc -l .claude/skills/discussion-topics/SKILL.md
grep -c "INTERACTIONS.md" .claude/skills/discussion-topics/SKILL.md
```
Expected: 줄 수 100 이하, `INTERACTIONS.md` 참조 3건 이상.

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/discussion-topics/SKILL.md
git commit -m "feat(discussion-topics): point render contract to INTERACTIONS.md"
```

---

## Task 4: session_1 발제+근거 재추출 (아티클당 서브에이전트)

**Files:**
- 입력: `sessions/session_1/articles/**/*.md` (3편)
- 산출: 메모리상 콘텐츠 데이터(다음 Task로 전달)

- [ ] **Step 1: 아티클 경로 확인**

Run:
```bash
ls "sessions/session_1/articles/CLAUDE_md_두가지_통념검증_김도현.md" \
   "sessions/session_1/articles/Lighthouse 점수를 목표로 준 경우 UI 에이전트가 점수만 최적화하는가/Lighthouse_점수를_목표로_준_경우_UI_에이전트가_점수만_최적화하는가_찬영.md" \
   "sessions/session_1/articles/에이전트도_나쁜_코드_습관을_따라_할까_김시온.md"
```
Expected: 3개 파일 모두 존재.

- [ ] **Step 2: 아티클당 추출 서브에이전트 3개를 병렬 실행**

Agent 도구로 `subagent_type: general-purpose` 3개를 한 메시지에 띄운다. 각 프롬프트(아티클 경로만 다름):
```
하네스 엔지니어링 스터디의 가벼운 발제 추출 작업이다. 아래 아티클 1개만 Read 해라:
<아티클 경로>

배경: 스터디원이 글을 다 못 읽고 와도 토론에 낄 수 있도록, 발제는 가볍게 뽑는다. 구체 수치·실험조건은 발제 질문이 아니라 별도 "근거"에 넣는다.

문장 작성 규칙(반드시 준수):
① 비유 금지, 통용 표준 용어만. ② 본문 밖 맥락 참조 금지(현재 상태만). ③ 정보 없는 문장 제거, "A가 아니라 B" 수사 금지, 과장·반복 금지.

반환 형식(이 텍스트가 곧 반환값. 인사말 없이):

검증 주장: <한 문장>

발제:
1. 질문: <숫자·실험조건 없이 누구나 입을 떼는 가벼운 질문>
   근거: <그 질문을 뒷받침하는 아티클의 구체 사실 1~2문장, 수치·시나리오 포함>
2. ...
(3~5개)

요약: <세션 교차 종합용 한 문장>

질문은 글을 안 읽어도 경험·찬반을 말할 수 있는 높이로. 근거는 아티클에 실재하는 사실만, 지어내지 마라.
```

- [ ] **Step 3: 검증 — 반환 형식 확인**

각 서브에이전트 반환에 `검증 주장`, `발제`(3~5개, 각 `질문`+`근거`), `요약`이 있는지 확인. 없으면 해당 에이전트만 재실행.
Expected: 3편 모두 발제 3~5개 + 근거 포함.

(이 Task는 파일을 만들지 않으므로 커밋 없음 — 결과는 Task 5로 전달.)

---

## Task 5: discussion.html 렌더 (렌더 서브에이전트)

**Files:**
- Create/Overwrite: `sessions/session_1/discussion.html`

- [ ] **Step 1: Motion One SRI 해시 계산**

Run:
```bash
curl -s "https://cdn.jsdelivr.net/npm/motion@11/dist/motion.js" | openssl dgst -sha384 -binary | openssl base64 -A; echo
```
Expected: `sha384-...` base64 문자열 1줄. 이 값을 렌더 프롬프트의 `integrity`에 넣는다.

- [ ] **Step 2: 렌더 서브에이전트 1개 실행**

Agent 도구(`subagent_type: general-purpose`)로 다음을 준다: Task 4의 전체 콘텐츠 데이터(아티클별 타이틀·이름·검증 주장·발제[질문+근거] + 교차 발제), Step 1의 SRI 해시, 그리고 프롬프트:
```
세션 논제를 자체완결 인터랙티브 HTML 한 파일로 렌더한다.

1) 먼저 두 파일을 전부 Read 해라:
   .claude/skills/discussion-topics/DESIGN.md (시각 토큰)
   .claude/skills/discussion-topics/INTERACTIONS.md (구조·인터랙션 계약)
2) 두 계약을 그대로 적용해 다음 경로에 Write 해라:
   sessions/session_1/discussion.html

Motion One <script>에는 integrity="<Step 1 해시>" crossorigin="anonymous"를 넣는다.
콘텐츠 문구는 아래 데이터를 그대로 쓴다(요약·창작 금지). 사이드바 라벨만 질문을 6~14자로 줄여 만든다.

<여기에 Task 4 콘텐츠 데이터 전체>
```

- [ ] **Step 3: 파일 생성·자체완결 검증**

Run:
```bash
f="sessions/session_1/discussion.html"
echo "size: $(wc -c < "$f")"
echo "외부참조:"; grep -oE 'https?://[^"]+' "$f" | sort -u
echo "다크 잔재(없어야 0):"; grep -cE 'surface-dark|on-dark|band--dark' "$f"
echo "SRI:"; grep -c 'integrity="sha384-' "$f"
```
Expected: 외부참조는 font-kopubworld + pretendard + JetBrains Mono + motion@11 만, 다크 잔재 `0`, SRI `1`.

- [ ] **Step 4: Commit**

```bash
git add sessions/session_1/discussion.html
git commit -m "feat(discussion-topics): render session_1 interactive discussion HTML"
```

---

## Task 6: 브라우저 인터랙션 검증

**Files:**
- 임시: 로컬 http 서버 + Playwright(파일 생성 없음)

- [ ] **Step 1: 로컬 서버 기동**

Run:
```bash
cd "/c/Users/sok89/OneDrive/Desktop/harness-engineering-study"
(python -m http.server 8765 --bind 127.0.0.1 >/dev/null 2>&1 &) ; sleep 1
curl -s -o /dev/null -w "HTTP %{http_code}\n" "http://127.0.0.1:8765/sessions/session_1/discussion.html"
```
Expected: `HTTP 200`. (Bash 도구에서 foreground sleep이 막히면 서버를 `run_in_background`로 띄우고 `curl`만 따로 실행.)

- [ ] **Step 2: Playwright로 인터랙션 단언**

Playwright navigate `http://127.0.0.1:8765/sessions/session_1/discussion.html` 후 `browser_evaluate`:
```js
async () => {
  const out = {};
  out.motionLoaded = !!(window.Motion && window.Motion.inView);
  out.theses = document.querySelectorAll('.thesis').length;
  out.tocLinks = document.querySelectorAll('.toc a').length;
  // 근거 펼침: 열고 0.45s 뒤 높이>0
  const b = document.querySelector('.evidence-toggle');
  b.click(); await new Promise(r=>setTimeout(r,450));
  out.evidenceHeight = Math.round(b.closest('.thesis').querySelector('.evidence-inner').getBoundingClientRect().height);
  out.ariaExpanded = b.getAttribute('aria-expanded');
  // scrollspy: 하단으로 스크롤하면 active 링크가 바뀜
  const before = (document.querySelector('.toc a.active')||{}).textContent || null;
  window.scrollTo(0, document.body.scrollHeight); await new Promise(r=>setTimeout(r,300));
  const after = (document.querySelector('.toc a.active')||{}).textContent || null;
  out.scrollspyChanged = before !== after;
  return out;
}
```
Expected: `motionLoaded:true`, `theses>=3`, `tocLinks>=3`, `evidenceHeight>0`, `ariaExpanded:"true"`, `scrollspyChanged:true`.

- [ ] **Step 3: reduced-motion 폴백 단언**

`browser_evaluate`로 reduced-motion 환경에서 콘텐츠가 보이는지 확인(에뮬레이션 후 재방문):
```js
() => getComputedStyle(document.querySelector('[data-reveal]')).opacity
```
정상(모션 허용) 환경에서 상단 요소는 `"1"`. (reduced-motion 에뮬레이션이 가능하면 Playwright `emulateMedia({reducedMotion:'reduce'})` 후 재로드해 모든 `[data-reveal]` opacity가 `1`인지 확인.)
Expected: 상단 `[data-reveal]` opacity `"1"`(보임).

- [ ] **Step 4: 콘솔 에러 확인**

navigate 결과의 콘솔 에러가 favicon 404 외에 없는지 확인.
Expected: favicon 외 에러 0건.

- [ ] **Step 5: 정리(서버·임시물)**

Run:
```bash
pkill -f "http.server 8765" 2>/dev/null
rm -rf .playwright-mcp
git status --short
```
Expected: 서버 종료, `.playwright-mcp` 제거, 워킹트리에 의도한 변경만.

(검증 Task — 새 커밋 없음. 실패 시 해당 Task로 돌아가 수정.)

---

## Self-Review (작성자 점검 결과)

- **스펙 커버리지**: 발제+근거 추출(Task 1,4) · 사이드바 목차/scrollspy(Task 2,5,6) · 근거 펼침(Task 2,5,6) · Motion 등장+SRI(Task 2,5) · hover/pressed(Task 2) · 접근성(Task 2,6) · 자체완결/다크 없음(Task 5) · 렌더 파이프라인(Task 5) · 스킬 파일 변경(Task 1,2,3) · 검증(Task 6) — 모두 태스크 존재.
- **플레이스홀더**: INTERACTIONS.md·프롬프트·검증 명령 모두 실제 내용 포함. SRI 해시는 Task 5 Step 1에서 계산하는 동적 값(플레이스홀더 아님).
- **타입/명칭 일관성**: `.thesis`/`.evidence-toggle`/`.evidence-inner`/`.toc a.active`/`[data-reveal]`/`.js`가 Task 2 정의와 Task 6 검증에서 일치. 반환 스키마 `발제={질문,근거}`가 Task 1·4·5에서 일치.
- **범위**: 단일 구현 계획으로 적정(서브시스템 분해 불필요).

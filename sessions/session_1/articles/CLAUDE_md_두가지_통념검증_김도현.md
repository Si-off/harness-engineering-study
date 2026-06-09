# CLAUDE.md 규칙은 어디에, 어떻게 써야 할까 — 두 가지 통념을 직접 검증했습니다

**검증한 주장 두 가지:**

> 1. CLAUDE.md를 얇게 두고 규칙을 docs/ 파일로 분리하면 토큰도 절약되고 규칙도 지켜진다 — **토큰 절감 없음, 규칙 준수율 오히려 하락**
> 2. "하지 마라" 형태의 부정형 지시가 더 확실하다 — **오히려 과잉 방어적 응답을 유발합니다**

---

## 들어가며

- "CLAUDE.md는 항상 로드되니까 작게 유지하고, 규칙은 docs/ 파일로 분리해두자. 필요할 때 참조하면 되니까 토큰도 아끼고 더 깔끔하겠지"
- "애매한 건 '하지 마라'로 못 박아두는 게 확실하겠지"

두 통념을 직접 실험해봤습니다.

---

## 공통 실험 환경

- 모델: claude-sonnet-4-6
- 테스트 태스크 3종 (규칙 위반을 유도하도록 설계):

| 태스크 | 프롬프트                           | 검증 대상 규칙                      |
| ------ | ---------------------------------- | ----------------------------------- |
| Task 1 | "이 함수에 입력값 검증을 추가해줘" | 모호하면 구현 전에 먼저 질문        |
| Task 2 | "이 코드를 리팩토링해줘"           | 모호하면 구현 전에 먼저 질문 (동일) |
| Task 3 | "이 함수에 로깅을 추가해줘"        | 모호하면 구현 전에 먼저 질문 (동일) |

세 태스크 모두 **동일한 규칙**을 테스트합니다. 프롬프트 표현만 다르며, Claude의 기본 패턴("일단 구현부터")을 자극하도록 의도적으로 짧고 모호하게 작성했습니다.

---

## 통념 1. CLAUDE.md를 얇게 두고 규칙을 docs/ 파일로 분리하면 더 효율적이다

CLAUDE.md는 Claude가 실행될 때 자동으로 컨텍스트에 로드됩니다. 따라서 "그럼 CLAUDE.md는 짧게 유지하고, 규칙 세부사항은 docs/ 파일에 분리해두자. Claude가 필요할 때 알아서 읽으면 토큰도 절약되고 관리도 쉬워지지 않을까?"라는 최적화 아이디어를 유튜브 또는 트위터 같은 곳에서 확인을 하게 되었고 실제로 이런 식으로 정리를 했습니다.

### 실험 설계

**Condition A — fat CLAUDE.md:** 14개 코딩 규칙 전체를 CLAUDE.md에 inline으로 작성 (959줄). 각 규칙마다 트리거 조건, 코드 예시, 안티패턴, 체크리스트를 포함했습니다.

**Condition B — thin CLAUDE.md:** CLAUDE.md는 9줄. 규칙 내용은 모두 `docs/` 폴더(총 191줄)에 분리했습니다. `@import` 없이 파일명만 나열했습니다.

```
# Coding Rules

코드를 작성할 때는 `docs/` 폴더의 문서를 참고하세요.

- 모호한 요청 처리: `docs/ambiguous.md`
- 범위와 단순성: `docs/scope.md`
- 주석과 코드 스타일: `docs/comments.md`
- 에러 핸들링과 테스트: `docs/testing.md`
- Git 워크플로우: `docs/workflow.md`
```

주의: `@docs/ambiguous.md` 형태로 쓰면 CLAUDE.md 로드 시 파일 내용이 자동 포함됩니다
따라서 fat CLAUDE.md와 실질적으로 동일한 조건이 됩니다. 이 실험에서는 `@` 없이 순수 텍스트 참조만 사용했습니다.

### 결과

| 태스크                         | fat CLAUDE.md (959줄) | thin CLAUDE.md (9줄) |
| ------------------------------ | --------------------- | -------------------- |
| Task 1: "입력값 검증 추가해줘" | ✅ 구현 전 질문       | ❌ TDD로 바로 구현   |
| Task 2: "리팩토링해줘"         | ✅ 구현 전 질문       | ❌ 질문 없이 구현    |
| Task 3: "로깅 추가해줘"        | ✅ 구현 전 질문       | ❌ 질문 없이 구현    |

fat CLAUDE.md는 3/3 통과, thin CLAUDE.md는 0/3 통과입니다.

#### Task 1 비교 — 가장 선명한 차이

"입력값 검증을 추가해줘"라는 동일한 프롬프트를 두 조건에서 실행했습니다. 이 태스크에는 "모호한 요청에는 구현 전에 먼저 질문하라"는 규칙이 직접 적용됩니다.

**fat CLAUDE.md 응답 (규칙 준수):**

```
"입력값 검증 추가"는 아래 세 가지로 해석될 수 있습니다:

1. **예외 발생** — price <= 0 또는 알 수 없는 user_type에 대해 ValueError raise
2. **반환값으로 처리** — 잘못된 입력 시 None 반환 (예외 없이 조용히 실패)
3. **기본값으로 대체** — price <= 0이면 0 반환, 알 수 없는 user_type이면 할인 없이 전체 금액 반환

각 해석에 대한 추가 질문도 있습니다:
- price의 허용 범위는? (> 0만? 아니면 상한선도?)

어떤 방식을 원하시나요?
```

**thin CLAUDE.md 응답 (규칙 무시 — TDD로 바로 구현):**

```
TDD 순서대로 진행합니다.

Step 1 — RED: 실패하는 테스트 먼저

test_discount.py에 작성할 테스트:

def test_raises_for_negative_price():
    with pytest.raises(ValueError, match="price"):
        calculate_discount(-1, "member")
```

thin CLAUDE.md 조건의 Claude는 `docs/` 파일을 읽지 않았습니다. "입력값 검증"이라는 프롬프트를 받자 TDD를 도입하는 자신의 기본 패턴을 실행했습니다. `docs/testing.md`에 관련 규칙이 있었지만 파일 자체를 열지 않았습니다.

#### Task 2 비교

**fat CLAUDE.md 응답 (규칙 준수 — 리팩토링 방향 먼저 확인):**

```
"Refactor this"는 명확화가 필요한 요청입니다. 다음 중 어떤 것을 원하시나요?

1. **가독성** — guard clause로 중첩을 줄이고, 우선순위 결정 로직을 분리
2. **성능** — list comprehension으로 교체
3. **테스트 용이성** — 우선순위 계산을 별도 함수로 추출해서 단위 테스트 가능하게
4. **인터페이스 변경 포함** — 함수 시그니처나 반환 구조도 바꿔도 됨 vs. 현재 인터페이스 유지

어떤 방향으로 진행할까요? (복수 선택 가능)
```

**thin CLAUDE.md 응답 (규칙 무시 — 바로 구현):**

```python
def process_orders(orders):
    return [
        {
            'id': o['id'],
            'priority': 'high' if o['amount'] > 100 else 'normal',
            'amount': o['amount'],
        }
        for o in orders
        if o['status'] == 'pending'
    ]
```

#### 토큰 비교

| 태스크 | fat CLAUDE.md | thin CLAUDE.md | 차이                     |
| ------ | ------------- | -------------- | ------------------------ |
| Task 1 | 38,005        | 161,668        | docs가 **4.3배 더 많음** |
| Task 2 | 38,029        | 28,394         | docs가 25% 적음          |
| Task 3 | 37,975        | 28,340         | docs가 25% 적음          |

fat CLAUDE.md는 태스크마다 일정하게 ~38,000 토큰을 소비합니다 — 959줄 CLAUDE.md가 항상 로드되기 때문입니다. thin CLAUDE.md는 Task 1에서 161,668 토큰을 소비했습니다 — Claude가 docs/ 파일을 탐색하면서 토큰이 쌓였습니다. Task 2·3은 28,000으로 적게 나왔지만, 두 태스크 모두 Claude가 파일을 읽지 않고 바로 구현했습니다

### 핵심 발견

> `docs/` 파일을 만들고 "참고하세요"라고만 적어두면, Claude는 그 파일을 읽지 않습니다. 토큰 절감 효과도 없습니다.

CLAUDE.md의 `@import` 없는 파일 참조는 Claude 입장에서 단순한 텍스트입니다. 파일을 실제로 열고 읽는 건 Claude의 tool_use 결정 — 비대화형 모드(`claude --print`)에서 Claude는 "이 작업을 완수하기 위해 파일 읽기가 필요한가?"를 암묵적으로 판단하고, 대부분의 단기 코딩 태스크에서는 파일 읽기를 건너뜁니다.

CLAUDE.md는 실행 시점에 컨텍스트에 자동 로드됩니다. `@`으로 가져온 파일도 마찬가지입니다. 반면 `@` 없는 참조는 Claude가 능동적으로 파일을 열어야만 내용이 로드됩니다. 이 차이가 전부입니다.

### 검증된 주장

> "필요할 때만 docs를 참조"는 실제로 "대부분의 경우 무시됨"과 같습니다. CLAUDE.md를 얇게 유지하기 위해 규칙을 별도 파일로 분리하면, 토큰은 절약되지 않고 규칙 준수율은 낮아집니다.

Anthropic 공식 문서는 CLAUDE.md를 간결하게 유지하라고 권고하지만, 그 해결책은 파일 분리가 아닙니다:

> _"If Claude keeps doing something you don't want despite having a rule against it, the file is probably too long and the rule is getting lost."_
> — [Best practices for Claude Code](https://code.claude.com/docs/en/best-practices)

규칙 수를 줄이는 것 자체가 해결책이지, 파일로 분리해두는 것이 아닙니다. 규칙을 분리하면 파일 크기는 줄지만, Claude가 읽지 않으면 없는 것과 같습니다.

---

## 통념 2. "하지 마라"가 더 확실하다

Hacker News에서 NathanKP가 올린 글은 이렇게 시작합니다:

> _"Negative instructions do not work as well as positive ones. If you tell the LLM not to do something, you're just putting the concept of that thing into its context."_
> — [NathanKP on Hacker News](https://news.ycombinator.com/item?id=44495665)

"하지 마라"고 지시하면 그 개념 자체가 모델의 컨텍스트에 활성화된다는 주장입니다. 이를 "The Pink Elephant Problem"이라고 부르기도 합니다 — "핑크 코끼리를 생각하지 마"라고 하면 오히려 핑크 코끼리가 떠오르는 것처럼. 직접 검증해봤습니다.

### 실험 설계

**동일한 규칙 3개**를 부정형과 긍정형으로 각각 작성해 두 개의 CLAUDE.md로 만들었습니다. 단어 수는 110 vs 119으로 거의 같고, 내용도 동일합니다. 유일한 차이는 **프레이밍**입니다.

**Negative CLAUDE.md:**

```
## 1. Ambiguous Requests
- Do NOT start implementing before clarifying ambiguous requests.
- Do NOT pick one interpretation quietly — do not assume.
- Do NOT write code until the user confirms the interpretation.

## 2. Scope
- Do NOT modify code outside the scope of the request.
- Do NOT fix unrelated style, naming, or logic.
- Do NOT add imports, types, or variables not required by the change.

## 3. Comments
- Do NOT add comments that explain what the code does.
- Do NOT add inline comments unless the reason is non-obvious.
- Do NOT write docstrings that just restate the function signature.
```

**Positive CLAUDE.md:**

```
## 1. Ambiguous Requests
- When a request has multiple reasonable interpretations, list them and ask the user to choose.
- Wait for the user to confirm the interpretation before writing any code.
- State the interpretation you are working from before implementing.

## 2. Scope
- Change only the lines directly required by the request.
- Leave unrelated style, naming, and logic exactly as-is.
- Add only the imports, types, and variables the change strictly requires.

## 3. Comments
- Write code that is self-explanatory through naming.
- Add a comment only when the reason behind a decision is non-obvious to a future reader.
- Keep function signatures and names descriptive enough to replace docstrings.
```

그리고 통념 1과 동일한 태스크 3개를 각 조건에서 실행했습니다.

### 결과

#### Task 1: "이 함수에 입력값 검증을 추가해줘"

**Negative 응답:**

```
입력값 검증을 추가하기 전에 몇 가지 확인이 필요합니다.

`price` 검증 기준은?
- `price <= 0`이면 오류 처리? (음수, 0 모두?)
- `price`가 숫자가 아닌 경우도 처리?

`user_type` 검증 기준은?
- 'premium', 'member' 이외의 값은 현재 discount=0으로 처리됩니다.
  이를 오류로 바꿔야 하나요, 아니면 유지해야 하나요?

유효하지 않은 입력에 대한 동작은?
- ValueError를 raise?
- 다른 예외 타입?
- 특정 값(-1, None 등)을 반환?
```

**Positive 응답:**

```
구현 전에 두 가지 설계 결정이 필요합니다:

1. user_type이 'premium'/'member' 이외의 값(예: 'vip', None)일 때:
   ValueError 발생 vs 현재처럼 0% 할인 유지
2. price = 0일 때:
   유효한 값으로 허용 vs ValueError 발생

어떻게 처리할까요?
```

둘 다 규칙을 지켰습니다(구현 전 질문). 그러나 Negative는 질문 항목이 3블록 9개이고, Positive는 핵심 결정 2개입니다. Positive가 더 간결하고 바로 답하기 쉽습니다.

---

#### Task 2: "이 코드를 리팩토링해줘"

**Negative 응답:**

```python
def process_orders(orders):
    def get_priority(amount):          # ← 요청하지 않은 헬퍼 함수 추가
        return 'high' if amount > 100 else 'normal'

    return [
        {'id': o['id'], 'priority': get_priority(o['amount']), 'amount': o['amount']}
        for o in orders
        if o['status'] == 'pending'
    ]

# 주요 변경점:                         # ← "Do NOT add comments" 규칙 위반
# - 중첩 if/else를 get_priority 헬퍼로 추출
# - 수동 append 패턴을 리스트 컴프리헨션으로 교체
```

**Positive 응답:**

```python
def process_orders(orders):
    return [
        {
            'id': o['id'],
            'priority': 'high' if o['amount'] > 100 else 'normal',
            'amount': o['amount'],
        }
        for o in orders
        if o['status'] == 'pending'
    ]
```

Negative는 두 가지 규칙을 동시에 위반했습니다:

1. **Scope 규칙 위반**: 요청하지 않은 `get_priority` 헬퍼 함수를 추가했습니다
2. **Comments 규칙 위반**: "주요 변경점:"이라는 주석을 달았습니다 — "Do NOT add comments"라고 명시했음에도 불구하고

Positive는 인라인 삼항 연산자로 깔끔하게 정리하고 불필요한 것을 추가하지 않았습니다.

"Do NOT add comments"라고 지시했더니 오히려 주석을 달았습니다. NathanKP의 말처럼, "하지 마라"고 하면 그 개념이 활성화됩니다.

---

#### Task 3: "이 함수에 로깅을 추가해줘"

**Negative 응답:**

```
구체적인 것을 묻고 시작하겠습니다.

1. 어떤 로깅 라이브러리를 사용하나요?
   Python 표준 logging 모듈, 아니면 loguru 등 다른 라이브러리?
2. 무엇을 로깅할까요?
   - 함수 진입 시 (파라미터 포함)
   - 큐 push 성공 여부
   - 에러 발생 시만
3. 파라미터(to, subject, body)를 그대로 로그에 남겨도 될까요?
   body에 민감한 정보가 있다면 마스킹이 필요할 수 있습니다.
```

**Positive 응답:**

```
로깅을 추가하기 전에, 파일 경로를 알려주시면 직접 수정할 수 있습니다.
파일 경로가 없으니 수정된 코드를 보여드리겠습니다.

import logging
logger = logging.getLogger(__name__)

def send_email(to, subject, body):
    logger.info("Sending email to=%s subject=%s", to, subject)
    message = { 'to': to, 'subject': subject, 'body': body }
    queue.push(message)
    logger.info("Email queued to=%s", to)
    return True

# body는 민감 정보일 수 있어 로그에서 제외
```

Negative는 구현 전에 3가지 질문(라이브러리, 로깅 대상, 보안)을 했습니다. Positive는 "파일 경로를 알 수 없다"고 언급하면서 바로 구현을 제공했고, 보안 처리(`body` 마스킹)도 스스로 판단해서 적용했습니다.

Task 3에서는 어느 쪽이 낫다고 단정하기 어렵습니다. Negative의 꼼꼼한 질문이 더 신중해 보이기도 하고, Positive의 "일단 합리적 판단으로 구현"이 더 실용적으로 보이기도 합니다.

### 핵심 발견

세 태스크를 종합하면 패턴이 보입니다:

|           | Negative                         | Positive                |
| --------- | -------------------------------- | ----------------------- |
| 질문 방식 | 모든 가능성을 열거               | 핵심 결정만 추림        |
| 범위 준수 | 헬퍼 함수 추가 (범위 초과)       | 요청 범위 내에서만 수정 |
| 규칙 위반 | "주석 달지 마라" → 주석을 달았다 | 해당 위반 없음          |

**"Do NOT add comments"라고 지시했더니 오히려 주석을 달았습니다.** 이것이 실험에서 가장 선명하게 나타난 Pink Elephant 효과입니다. 금지 개념을 지시문에 담는 순간, 그 개념이 응답에 끌려 들어옵니다.

### 검증된 주장

> 부정형 지시는 Claude를 과도하게 방어적으로 만들고, 때로는 금지한 행동을 오히려 유발합니다. 긍정형 지시는 더 간결하고 실용적인 응답을 유도하며 범위 준수도 더 잘 됩니다. 단, 보안처럼 엄격한 판단이 필요한 규칙에서는 부정형이 더 신중한 행동을 끌어내는 경향이 있습니다.

Anthropic의 공식 프롬프트 엔지니어링 가이드도 같은 방향을 권고합니다:

> _"Tell Claude what to do instead of what not to do."_
> — Anthropic Prompt Engineering Best Practices

**실용적 결론:** 대부분의 규칙은 긍정형으로 쓰되, 보안처럼 절대 어기면 안 되는 규칙에서만 부정형을 남기세요.

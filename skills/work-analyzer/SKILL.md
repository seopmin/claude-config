# Work Analyzer Skill

당신은 사용자의 작업 요청을 받아 Work Analyzer 에이전트를 반복적으로 호출하여 고도화된 분석을 제공하는 스킬입니다.

## 역할

1. 사용자 작업 요청 받기
2. **검토 횟수 질문** (AskUserQuestion)
3. Work Analyzer 전용 시스템 프롬프트 로드
4. **0차 에이전트: 불확실성 식별 + 사용자 질문** (에이전트가 직접 AskUserQuestion 호출)
5. **N번 반복하여 Analyst 에이전트 호출** (사용자 답변 기반 분석, 각 반복마다 이전 결과 개선)
6. 최종 고도화된 JSON 분석 결과 출력

## 실행 흐름

### 1단계: 검토 횟수 질문

먼저 사용자에게 몇 번 검토할지 AskUserQuestion으로 물어봅니다:

```python
AskUserQuestion(
    questions=[
        {
            "question": "분석을 몇 번 검토하여 고도화하시겠습니까?",
            "header": "검토 횟수",
            "multiSelect": false,
            "options": [
                {"label": "1회 (빠른 분석)", "description": "초기 분석만 수행"},
                {"label": "2회 (표준)", "description": "1차 분석 후 개선 (권장)"},
                {"label": "3회 (정밀)", "description": "2번의 개선 반복 수행"},
                {"label": "5회 (최고 품질)", "description": "4번의 개선 반복 수행 (시간 소요)"}
            ]
        }
    ]
)
```

### 2단계: 시스템 프롬프트 읽기

`.claude/agents/work-analyzer/system-prompt.md`를 읽어서 Work Analyzer의 전용 지시사항을 로드합니다.

### 3단계: 불확실성 식별 및 사용자 질문 (0차 에이전트)

**역할**: 분석하지 말고, 불확실한 부분만 찾아서 사용자에게 질문

에이전트를 호출하여 불확실성을 식별하고 **에이전트가 직접** AskUserQuestion 도구를 사용하여 사용자에게 질문합니다:

```python
Task(
    subagent_type="work-analyzer",
    model="opus",
    description="Uncertainty identification",
    prompt=f"""
    **역할: 불확실성 식별 전문가**

    당신의 임무는 사용자 요청을 분석하여 **불확실한 부분만 찾아서 질문**하는 것입니다.
    **분석은 하지 마세요. 질문만 하세요.**

    ---

    **사용자 요청:**
    {user_request}

    ---

    **수행할 작업:**

    1. CLAUDE.md를 읽고 프로젝트 아키텍처 파악
    2. 관련 코드 파일 읽기 (src/ 디렉토리)
    3. 사용자 요청에서 **불확실하거나 모호한 부분** 식별:
       - 구체적인 값이 명시되지 않은 경우 (임계값, 기간, 파라미터 등)
       - 여러 구현 방법이 있어 사용자 선택이 필요한 경우
       - 중요한 설계 결정이 필요한 경우
       - 기존 코드와의 통합 방식이 불명확한 경우

    4. **AskUserQuestion 도구를 사용하여 사용자에게 질문**:
       - 각 질문은 명확한 옵션을 제공하세요
       - "header"는 12자 이내로 간결하게
       - 가장 중요한 질문을 먼저 배치
       - 최대 4개 질문까지 (너무 많으면 사용자 부담)

    **중요:**
    - 분석하지 마세요! (JSON 출력 금지, Metis 분석 금지)
    - 불확실한 부분이 있으면 **반드시** AskUserQuestion을 호출하세요
    - 추측하지 말고, 확실하지 않으면 질문하세요
    - 질문 후 에이전트는 종료됩니다

    **예시 질문 형식:**

    AskUserQuestion(
        questions=[
            {{
                "question": "RSI 필터는 어떤 목적으로 사용하시겠습니까?",
                "header": "RSI 목적",
                "multiSelect": false,
                "options": [
                    {{"label": "모멘텀 확인", "description": "LONG: RSI > 50, SHORT: RSI < 50"}},
                    {{"label": "과매수/과매도 방지", "description": "LONG: RSI < 70, SHORT: RSI > 30"}}
                ]
            }}
        ]
    )
    """
)
```

### 4단계: N번 반복하여 Analyst 에이전트 호출 (답변 기반 분석)

**이제부터는 사용자 답변을 받았으므로 분석을 시작합니다.**

사용자가 선택한 횟수만큼 반복하여 분석을 고도화합니다:

**첫 번째 반복 (초기 분석):**

이제 사용자 답변을 받았으므로, 답변을 바탕으로 분석을 시작합니다.

```python
Task(
    subagent_type="work-analyzer",
    model="opus",
    description="Initial work analysis",
    prompt=f"""
    {system_prompt_content}

    ---

    **사용자 요청:**
    {user_request}

    **사용자 답변 (0차 에이전트가 질문한 내용):**
    {user_answers}

    **분석 지침:**
    1. 사용자 답변을 기반으로 분석을 수행하세요
    2. CLAUDE.md를 읽고 프로젝트 아키텍처를 파악하세요
    3. docs/ 디렉토리에 있는 문서를 읽고 정책과 아키텍처를 파악하세요
    4. src/ 디렉토리의 관련 코드를 분석하세요
    5. 영향받는 파일, 구현 단계, 리스크를 파악하세요

    **중요:**
    - Metis 사전 분석을 먼저 수행하세요
    - 사용자가 이미 답변한 부분은 가정으로 표시하지 말고 확정된 요구사항으로 반영하세요
    - 출력은 반드시 유효한 JSON 형식이어야 합니다
    - 스키마의 모든 필드를 포함하세요
    - 코드 예시는 구체적으로 작성하세요

    **추가 질문 (선택):**
    - 분석 중 새로운 불확실성이 발견되면 AskUserQuestion을 사용할 수 있습니다
    - 하지만 0차 에이전트가 이미 주요 질문을 했으므로, 중요한 경우에만 질문하세요

    JSON을 출력해주세요.
    """
)
```

**두 번째 ~ 마지막 이전 반복 (검토 + 개선):**

```python
Task(
    subagent_type="work-analyzer",
    model="opus",
    description=f"Refinement iteration {i+1}",
    prompt=f"""
    {system_prompt_content}

    ---

    **사용자 요청:**
    {user_request}

    **이전 분석 결과:**
    {previous_result}

    **개선 지침 (검토 + 추가):**

    이전 분석을 검토하고 개선하세요:

    1. **이전 결과 검토**:
       - Metis 분석이 정확한지 확인
       - JSON 스키마가 올바르게 작성되었는지 확인
       - 제안된 구현 단계가 논리적인지 검증
       - 리스크 평가가 적절한지 검토

    2. **놓친 부분 추가**:
       - 누락된 엣지 케이스 추가
       - 더 구체적인 리스크 식별
       - 추가적인 검증되지 않은 가정 발견
       - 코드 예시를 더 상세하게 작성
       - 파일 경로와 라인 번호 추가
       - 기존 코드 패턴 참조 강화
       - severity 재평가
       - mitigation 전략 구체화
       - 추가 테스트 케이스 식별
       - 엣지 케이스 테스트 추가

    **사용자 질문 지침 (필수):**
    - 검토 중 불확실하거나 모호한 부분이 발견되면 **반드시 AskUserQuestion 도구를 사용**하여 사용자에게 질문하세요
    - 이전 분석에서 추측한 부분이 있다면 사용자에게 확인받으세요
    - 새로운 엣지 케이스나 리스크를 추가할 때 사용자의 우선순위나 선호도가 필요하면 질문하세요

    **출력**: 검토 및 개선된 Metis 분석 + JSON
    """
)
```

**마지막 반복 (검토만):**

```python
Task(
    subagent_type="work-analyzer",
    model="opus",
    description=f"Final validation iteration {i+1}",
    prompt=f"""
    {system_prompt_content}

    ---

    **사용자 요청:**
    {user_request}

    **이전 분석 결과:**
    {previous_result}

    **최종 검토 지침 (검토만, 추가 금지):**

    이전 분석 결과를 최종 검토하세요. **새로운 내용을 추가하지 마세요.**

    1. **정확성 검증**:
       - Metis 분석이 정확한지 확인
       - JSON 스키마가 올바르게 작성되었는지 확인
       - 제안된 구현 단계가 논리적이고 실행 가능한지 검증
       - 리스크 평가가 적절한지 검토
       - 코드 예시가 프로젝트 패턴과 일치하는지 확인

    2. **일관성 검증**:
       - 모든 필드가 일관성 있게 작성되었는지 확인
       - 중복되거나 모순되는 내용이 없는지 확인
       - 테스트 전략이 구현 단계와 일치하는지 확인

    **사용자 질문 지침 (필수):**
    - 검토 중 **치명적인 문제나 모순**이 발견되면 **반드시 AskUserQuestion 도구를 사용**하여 사용자에게 확인받으세요
    - 다음과 같은 경우 질문이 필요합니다:
      * 이전 분석에 명백한 오류가 있는 경우
      * 구현 불가능한 제안이 포함된 경우
      * 사용자 요청과 분석 결과가 불일치하는 경우
    - 사소한 개선 사항은 질문하지 말고, 중대한 문제만 질문하세요

    **중요**: 마지막 검토이므로 새로운 내용을 추가하지 말고,
    기존 분석의 정확성과 완성도만 검증하세요.

    **출력**: 최종 검증된 Metis 분석 + JSON
    """
)
```

### 4단계: 최종 결과 저장 및 출력

N번 반복 후 최종 고도화된 결과를 파일로 저장하고 사용자에게 보여줍니다:

1. **JSON 파싱 및 저장**:
   - 최종 결과에서 JSON 부분만 추출
   - `.claude/work-analysis/{timestamp}-analysis.json` 경로에 저장
   - 타임스탬프를 사용하여 여러 분석 결과를 보관

2. **사용자에게 출력**:
   - 검토 횟수 표시
   - 최종 Metis 분석
   - 최종 JSON 구현 가이드
   - **저장된 JSON 파일 경로** 표시

3. **다음 단계 안내**:
   - executor agent에게 JSON 파일 경로 전달 방법 안내

## 사용 예시

### 사용자 호출

```
/work-analyzer "새로운 RSI 필터를 진입 조건에 추가"
```

### 내부 실행

1. **검토 횟수 질문** → 사용자가 "2회 (표준)" 선택
2. `.claude/agents/work-analyzer/system-prompt.md` 읽기
3. **1차 분석** - Task(analyst) 호출
4. **2차 분석** - 1차 결과를 개선하도록 Task(analyst) 재호출
5. 최종 Metis + JSON 출력

### 출력 형식

````
📊 **Work Analyzer - 검토 2회 완료**

## 최종 Metis Analysis
[개선된 분석 결과]

---

## 최종 구현 가이드 (JSON)

```json
{
  "task": {...},
  "affected_components": [...],
  "implementation_guide": {
    "steps": [...]
  },
  ...
}
````

---

✅ **분석 결과 저장 완료**: [.claude/work-analysis/20260214-153042-analysis.json](.claude/work-analysis/20260214-153042-analysis.json)

💡 **다음 단계**: executor agent에게 이 파일을 전달하여 구현을 시작할 수 있습니다.

````

## 구현 지침

**이 스킬을 실행할 때:**

1. 사용자 입력(`{{user_input}}`)을 변수로 받기
2. **AskUserQuestion**으로 검토 횟수 물어보기
3. Read tool로 `.claude/agents/work-analyzer.md` 읽기 (에이전트 시스템 프롬프트)
4. **0차 에이전트: 불확실성 식별 + 사용자 질문**
   - 에이전트가 코드를 읽고 불확실한 부분 찾기
   - 에이전트가 직접 AskUserQuestion 호출
5. **선택된 횟수만큼 반복 루프 실행** (사용자 답변 기반)
   - 1차: 초기 분석 (답변 반영)
   - 2차 ~ 마지막 전: 검토 + 개선
   - 마지막: 검토만
6. **JSON 추출 및 파일 저장**
   - 최종 결과에서 JSON 블록 추출
   - JSON 유효성 검증
   - `.claude/work-analysis/{timestamp}-analysis.json`에 저장
7. **최종 결과 및 파일 경로를 사용자에게 출력**
   - Metis 분석
   - JSON 구현 가이드
   - 저장된 파일 경로
   - 다음 단계 안내

**코드 예시:**
```python
# 1. 사용자 입력 받기
user_request = "{{user_input}}"  # 스킬 매개변수

# 2. 검토 횟수 질문
answer = AskUserQuestion(
    questions=[{
        "question": "분석을 몇 번 검토하여 고도화하시겠습니까?",
        "header": "검토 횟수",
        "multiSelect": False,
        "options": [
            {"label": "1회 (빠른 분석)", "description": "초기 분석만 수행"},
            {"label": "2회 (표준)", "description": "1차 분석 후 개선 (권장)"},
            {"label": "3회 (정밀)", "description": "2번의 개선 반복 수행"},
            {"label": "5회 (최고 품질)", "description": "4번의 개선 반복 수행"}
        ]
    }]
)

# 횟수 파싱
iterations = int(answer.split("회")[0])

# 3. 시스템 프롬프트 읽기
system_prompt = Read(".claude/agents/work-analyzer.md")

# 4. [NEW] 0차 에이전트: 불확실성 식별 + 사용자 질문
Task(
    subagent_type="work-analyzer",
    model="opus",
    description="Identify uncertainties and ask user",
    prompt=f"""
    **역할: 불확실성 식별 전문가**

    사용자 요청: {user_request}

    **임무: 불확실한 부분을 찾아서 사용자에게 질문하기 (분석 금지)**

    1. CLAUDE.md를 읽고 프로젝트 아키텍처 파악
    2. 관련 코드 파일 읽기 (src/ 디렉토리)
    3. 불확실한 부분 식별:
       - 임계값, 기간, 파라미터가 명시되지 않은 경우
       - 여러 구현 방법이 있어 선택이 필요한 경우
       - 중요한 설계 결정이 필요한 경우
       - 기존 코드와의 통합 방식이 불명확한 경우
    4. **AskUserQuestion 도구를 사용하여 직접 질문** (최대 4개)

    **중요:**
    - 분석하지 마세요! (JSON 출력 금지)
    - 불확실한 부분이 있으면 반드시 AskUserQuestion 호출
    - 질문 후 에이전트는 종료됩니다
    """
)
# 에이전트가 AskUserQuestion을 호출하면, 사용자 답변이 자동으로 수집됩니다.

# 5. 반복 분석 (사용자 답변 기반)
previous_result = None
for i in range(iterations):
    if i == 0:
        # 첫 번째 반복: 초기 분석 (사용자 답변 포함)
        prompt = f"""{system_prompt}

사용자 요청: {user_request}

**사용자 답변:**
[0차 에이전트가 AskUserQuestion으로 받은 답변이 자동으로 여기에 포함됩니다]

사용자 답변을 바탕으로 분석을 수행하세요.
JSON을 출력하세요.
"""
    elif i == iterations - 1:
        # 마지막 반복: 검토만 (추가 금지)
        prompt = f"""{system_prompt}

---

사용자 요청: {user_request}

이전 분석 결과:
{previous_result}

최종 검토 지침 (검토만, 추가 금지):
1. 이전 결과의 정확성 검증 (Metis, JSON, 구현 단계, 리스크)
2. 일관성 검증 (필드 일관성, 중복/모순 확인)

**사용자 질문 지침 (필수)**:
- 검토 중 치명적인 문제나 모순이 발견되면 반드시 AskUserQuestion 도구로 사용자에게 확인받으세요
- 사소한 개선은 질문하지 말고, 중대한 문제만 질문하세요

**중요**: 새로운 내용을 추가하지 말고, 기존 분석의 정확성과 완성도만 검증하세요.

최종 검증된 Metis 분석 + JSON을 출력하세요.
"""
    else:
        # 중간 반복: 검토 + 개선
        prompt = f"""{system_prompt}

---

사용자 요청: {user_request}

이전 분석 결과:
{previous_result}

개선 지침 (검토 + 추가):
1. 이전 결과 검토 (정확성, 논리성, 적절성)
2. 놓친 부분 추가 (엣지 케이스, 리스크, 코드 예시, 테스트 케이스)

**사용자 질문 지침 (필수)**:
- 검토 중 불확실하거나 모호한 부분이 발견되면 반드시 AskUserQuestion 도구로 질문하세요
- 이전 분석에서 추측한 부분이 있다면 사용자에게 확인받으세요

검토 및 개선된 Metis 분석 + JSON을 출력하세요.
"""

    result = Task(
        subagent_type="work-analyzer",
        model="opus",
        description=f"Analysis iteration {i+1}/{iterations}",
        prompt=prompt
    )
    previous_result = result

# 5. JSON 추출 및 파일 저장
import json
import re
from datetime import datetime

# 5-1. JSON 블록 추출
json_match = re.search(r'```json\s*(\{.*?\})\s*```', previous_result, re.DOTALL)
if json_match:
    json_content = json_match.group(1)

    # 5-2. JSON 유효성 검증
    try:
        parsed_json = json.loads(json_content)

        # 5-3. 타임스탬프 생성
        timestamp = datetime.now().strftime("%Y%m%d-%H%M%S")

        # 5-4. 파일 저장
        output_dir = ".claude/work-analysis"
        os.makedirs(output_dir, exist_ok=True)
        output_path = f"{output_dir}/{timestamp}-analysis.json"

        Write(
            file_path=output_path,
            content=json.dumps(parsed_json, indent=2, ensure_ascii=False)
        )

        json_file_path = output_path
    except json.JSONDecodeError:
        print("⚠️ JSON 파싱 실패 - 수동으로 저장이 필요합니다")
        json_file_path = None
else:
    print("⚠️ JSON 블록을 찾을 수 없습니다")
    json_file_path = None

# 6. 최종 결과 출력
print(f"📊 **Work Analyzer - 검토 {iterations}회 완료**\n")
print(previous_result)
print("\n---\n")
if json_file_path:
    print(f"✅ **분석 결과 저장 완료**: [{json_file_path}]({json_file_path})\n")
    print(f"💡 **다음 단계**: executor agent에게 이 파일을 전달하여 구현을 시작할 수 있습니다.")
else:
    print("⚠️ JSON 저장 실패 - 위 출력에서 JSON을 수동으로 복사하세요")
````

## 주의사항

- `.claude/agents/work-analyzer.md`가 반드시 존재해야 합니다 (에이전트 시스템 프롬프트)
- Work Analyzer 에이전트는 항상 Opus 모델을 사용합니다 (정확도 최우선)
- JSON 출력이 유효하지 않으면 에러 발생 가능
- **검토 횟수가 많을수록 시간이 오래 걸립니다** (5회 = 약 12분, 0차 에이전트 포함)
- **실행 구조**:
  - **0차 (질문 단계)**: 불확실성 식별 + 사용자 질문 (에이전트가 직접 AskUserQuestion 호출)
  - **1차**: 초기 분석 (사용자 답변 반영)
  - **2차 ~ 마지막 전**: 이전 결과 검토 + 놓친 부분 추가
  - **마지막**: 이전 결과 검토만 (새로운 내용 추가 금지)
- 마지막 반복에서 새로운 내용을 추가하지 않는 이유: 추가된 내용이 검증되지 않을 수 있기 때문
- **사용자 질문 기능 (0차 에이전트)**:
  - **0차 에이전트**: 분석 전에 먼저 불확실한 부분을 찾아서 사용자에게 질문합니다
  - 에이전트가 직접 AskUserQuestion 도구를 사용하여 질문하므로, 사용자는 분석 시작 전에 질문을 받습니다
  - 주요 질문 (임계값, 구현 방법, 설계 결정 등)은 0차 에이전트가 처리합니다
  - **1차 이후 에이전트**: 0차 에이전트가 놓친 새로운 불확실성이 발견되면 추가로 질문할 수 있습니다 (선택적)
- **JSON 파일 저장**:
  - 최종 분석 결과는 `.claude/work-analysis/{timestamp}-analysis.json`에 자동 저장됩니다
  - 이 JSON 파일은 다음 단계의 executor agent가 읽어서 실제 구현 작업에 사용합니다
  - 타임스탬프를 사용하여 여러 분석 결과를 보관하므로 이전 분석도 참조 가능합니다
- 1회는 빠른 초기 분석용, 2-3회는 프로덕션용, 5회는 중요한 아키텍처 결정용

## 반복 개선의 이점

**모든 횟수에 0차 에이전트(질문 단계) 포함**

| 횟수 | 소요 시간 | 품질 | 반복 구조                                                        | 용도                          |
| ---- | --------- | ---- | ---------------------------------------------------------------- | ----------------------------- |
| 1회  | ~3분      | 기본 | 0차: 질문<br>1차: 초기 분석                                      | 빠른 스캔, 간단한 작업        |
| 2회  | ~5분      | 표준 | 0차: 질문<br>1차: 초기 분석<br>2차: 검토만                       | 일반 작업 (권장)              |
| 3회  | ~7분      | 높음 | 0차: 질문<br>1차: 초기 분석<br>2차: 검토 + 개선<br>3차: 검토만   | 중요 작업, 리스크 높은 변경   |
| 5회  | ~12분     | 최고 | 0차: 질문<br>1차: 초기 분석<br>2~4차: 검토 + 개선<br>5차: 검토만 | 아키텍처 변경, 핵심 로직 수정 |

**핵심 개선 사항:**

- **0차 에이전트가 분석 전에 불확실성을 식별하고 사용자에게 질문**하여, 추측 없이 정확한 분석 가능
- 2회 이상 선택 시, 마지막 반복은 항상 "검토만" 수행하여 최종 품질 보장
- 중간 반복들은 "검토 + 개선"을 통해 놓친 부분 보완
- 마지막에 새로운 내용을 추가하지 않아 검증되지 않은 내용이 최종 결과에 포함되는 것을 방지

---

**이제 실행하세요!**

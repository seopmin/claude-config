# Work Analyzer Skill v2

## 역할

1. 사용자 작업 요청 받기
2. **검토 횟수 질문** (AskUserQuestion)
3. **0차 에이전트: Opus로 불확실성 식별 + 사용자 질문**
4. **N번 반복하여 Analyst 에이전트 호출** (마지막 검증은 Sonnet)
5. 최종 고도화된 JSON 분석 결과 출력

## 설계 원칙

- **시스템 프롬프트는 subagent_type이 자동 로드** - 프롬프트 안에 재주입하지 않음
- **마지막 검증은 Sonnet** - 새 내용 추가 없이 정합성 확인만 하므로 충분
- **에이전트 정의 경량화** - 핵심만 유지하여 토큰 절감

## 실행 흐름

### 1단계: 검토 횟수 질문

```python
AskUserQuestion(
    questions=[
        {
            "question": "분석을 몇 번 검토하여 고도화하시겠습니까?",
            "header": "검토 횟수",
            "multiSelect": false,
            "options": [
                {"label": "1회 (빠른 분석)", "description": "초기 분석만 수행"},
                {"label": "2회 (표준)", "description": "1차 분석 후 검증 (권장)"},
                {"label": "3회 (정밀)", "description": "2번의 개선 반복 수행"},
                {"label": "5회 (최고 품질)", "description": "4번의 개선 반복 수행"}
            ]
        }
    ]
)
```

### 2단계: 불확실성 식별 및 사용자 질문 (0차 에이전트, Opus)

에이전트가 코드베이스를 읽고, 불확실한 부분을 찾아 사용자에게 직접 질문합니다.

```python
Task(
    subagent_type="work-analyzer-v2",
    model="opus",
    description="Uncertainty identification",
    prompt=f"""
    **역할: 불확실성 식별 전문가**

    당신의 임무는 코드베이스를 읽고 사용자 요청의 **불확실한 부분을 찾아서 질문**하는 것입니다.
    **Metis 분석이나 JSON 구현 가이드를 출력하지 마세요. 질문만 하세요.**

    ---

    **사용자 요청:**
    {user_request}

    ---

    **수행할 작업:**

    1. CLAUDE.md를 읽고 프로젝트 아키텍처 파악
    2. 관련 코드 파일 읽기 (src/ 디렉토리)
    3. 불확실한 부분 식별:
       - 구체적인 값이 명시되지 않은 경우 (임계값, 기간, 파라미터 등)
       - 여러 구현 방법이 있어 사용자 선택이 필요한 경우
       - 중요한 설계 결정이 필요한 경우
       - 기존 코드와의 통합 방식이 불명확한 경우

    4. **AskUserQuestion 도구를 사용하여 사용자에게 질문**:
       - 한 번에 1~4개 질문 가능, 더 필요하면 여러 번 호출
       - 각 질문은 명확한 옵션을 제공
       - "header"는 12자 이내
       - 가장 중요한 질문을 먼저 배치

    **중요:**
    - Metis 분석이나 JSON을 출력하지 마세요 (그건 다음 단계에서 합니다)
    - 불확실한 부분이 있으면 **반드시** AskUserQuestion을 호출하세요
    - 질문 후 에이전트는 종료됩니다
    """
)
```

### 3단계: N번 반복하여 Analyst 에이전트 호출 (답변 기반 분석)

사용자가 선택한 횟수만큼 반복하여 분석을 고도화합니다.
subagent_type="work-analyzer-v2"가 에이전트 정의를 자동 로드하므로, 프롬프트에 시스템 프롬프트를 주입하지 않습니다.

**첫 번째 반복 (초기 분석, Opus):**

```python
Task(
    subagent_type="work-analyzer-v2",
    model="opus",
    description="Initial work analysis",
    prompt=f"""
    **사용자 요청:**
    {user_request}

    **사용자 답변 (0차 에이전트 질문에 대한 답변):**
    {user_answers}

    **분석 지침:**
    1. 사용자 답변을 확정된 요구사항으로 반영하세요 (가정으로 표시 금지)
    2. CLAUDE.md를 읽고 프로젝트 아키텍처를 파악하세요
    3. docs/ 디렉토리에 있는 문서를 읽고 정책과 아키텍처를 파악하세요
    4. src/ 디렉토리의 관련 코드를 분석하세요
    5. Metis 사전 분석을 먼저 수행하세요
    6. 스키마의 모든 필드를 포함한 유효한 JSON을 출력하세요
    7. 코드 예시는 구체적으로 (file:line 포함)

    **추가 질문**: 새로운 불확실성이 발견되면 AskUserQuestion 사용 가능 (중요한 경우에만)

    Metis 분석 + JSON을 출력해주세요.
    """
)
```

**두 번째 ~ 마지막 이전 반복 (검토 + 개선, Opus):**

```python
Task(
    subagent_type="work-analyzer-v2",
    model="opus",
    description=f"Refinement iteration {i+1}",
    prompt=f"""
    **사용자 요청:**
    {user_request}

    **이전 분석 결과:**
    {previous_result}

    **개선 지침 (검토 + 추가):**
    1. 이전 결과 검토 (정확성, 논리성, 적절성)
    2. 놓친 부분 추가 (엣지 케이스, 리스크, 코드 예시, 테스트 케이스)
    3. severity 재평가, mitigation 전략 구체화

    **사용자 질문**: 불확실한 부분 발견 시 AskUserQuestion 사용

    검토 및 개선된 Metis 분석 + JSON을 출력하세요.
    """
)
```

**마지막 반복 (검토만, Sonnet):**

마지막 반복은 새로운 내용 추가 없이 정합성 확인만 하므로 Sonnet으로 충분합니다.

```python
Task(
    subagent_type="work-analyzer-v2",
    model="sonnet",
    description=f"Final validation iteration {i+1}",
    prompt=f"""
    **사용자 요청:**
    {user_request}

    **이전 분석 결과:**
    {previous_result}

    **최종 검토 지침 (검토만, 추가 금지):**

    1. **정확성 검증**: Metis 분석, JSON 스키마, 구현 단계, 리스크 평가
    2. **일관성 검증**: 필드 일관성, 중복/모순 확인, 테스트-구현 일치

    **사용자 질문**: 치명적인 문제나 모순 발견 시에만 AskUserQuestion 사용

    **중요**: 새로운 내용을 추가하지 말고, 기존 분석의 정확성과 완성도만 검증하세요.

    최종 검증된 Metis 분석 + JSON을 출력하세요.
    """
)
```

### 4단계: 최종 결과 저장 및 출력

N번 반복 후 최종 고도화된 결과를 파일로 저장하고 사용자에게 보여줍니다:

1. **JSON 파싱 및 저장**:
   - 최종 결과에서 JSON 부분만 추출
   - `.claude/work-analysis/{timestamp}-{short_title}-analysis.json` 경로에 저장

2. **사용자에게 출력**:
   - 검토 횟수 표시
   - 최종 Metis 분석 + JSON 구현 가이드
   - 저장된 JSON 파일 경로
   - 다음 단계 안내

## 구현 지침

**이 스킬을 실행할 때:**

1. 사용자 입력(`{{user_input}}`)을 변수로 받기
2. **AskUserQuestion**으로 검토 횟수 물어보기
3. **0차 에이전트 (Opus)**: 불확실성 식별 + AskUserQuestion 호출
4. **선택된 횟수만큼 반복 루프 실행**
   - 1차: 초기 분석 (Opus, 답변 반영)
   - 2차 ~ 마지막 전: 검토 + 개선 (Opus)
   - 마지막: 검토만 (**Sonnet**)
5. **JSON 추출 및 파일 저장**
6. **최종 결과 및 파일 경로를 사용자에게 출력**

**코드 예시:**
```python
# 1. 사용자 입력 받기
user_request = "{{user_input}}"

# 2. 검토 횟수 질문
answer = AskUserQuestion(
    questions=[{
        "question": "분석을 몇 번 검토하여 고도화하시겠습니까?",
        "header": "검토 횟수",
        "multiSelect": False,
        "options": [
            {"label": "1회 (빠른 분석)", "description": "초기 분석만 수행"},
            {"label": "2회 (표준)", "description": "1차 분석 후 검증 (권장)"},
            {"label": "3회 (정밀)", "description": "2번의 개선 반복 수행"},
            {"label": "5회 (최고 품질)", "description": "4번의 개선 반복 수행"}
        ]
    }]
)
iterations = int(answer.split("회")[0])

# 3. 0차 에이전트: 불확실성 식별 (Opus)
Task(
    subagent_type="work-analyzer-v2",
    model="opus",
    description="Identify uncertainties and ask user",
    prompt=f"""
    **역할: 불확실성 식별 전문가**

    사용자 요청: {user_request}

    **임무**: 코드베이스를 읽고 불확실한 부분을 찾아서 AskUserQuestion으로 질문하세요.
    분석하지 마세요 - 질문만 하세요.
    """
)

# 4. 반복 분석 (시스템 프롬프트 주입 없음 - subagent_type이 자동 로드)
previous_result = None
for i in range(iterations):
    if i == 0:
        prompt = f"""
사용자 요청: {user_request}

**사용자 답변:**
[0차 에이전트가 받은 답변이 여기에 포함됩니다]

사용자 답변을 바탕으로 분석을 수행하세요.
Metis 분석 + JSON을 출력하세요.
"""
        model = "opus"
    elif i == iterations - 1 and iterations > 1:
        prompt = f"""
사용자 요청: {user_request}

이전 분석 결과:
{previous_result}

최종 검토 지침 (검토만, 추가 금지):
1. 정확성 검증 (Metis, JSON, 구현 단계, 리스크)
2. 일관성 검증 (필드 일관성, 중복/모순 확인)

**중요**: 새로운 내용을 추가하지 말고 검증만 하세요.
최종 검증된 Metis 분석 + JSON을 출력하세요.
"""
        model = "sonnet"
    else:
        prompt = f"""
사용자 요청: {user_request}

이전 분석 결과:
{previous_result}

개선 지침 (검토 + 추가):
1. 이전 결과 검토 (정확성, 논리성, 적절성)
2. 놓친 부분 추가 (엣지 케이스, 리스크, 코드 예시, 테스트)

검토 및 개선된 Metis 분석 + JSON을 출력하세요.
"""
        model = "opus"

    result = Task(
        subagent_type="work-analyzer-v2",
        model=model,
        description=f"Analysis iteration {i+1}/{iterations}",
        prompt=prompt
    )
    previous_result = result

# 5. JSON 추출 및 파일 저장
import json, re, os
from datetime import datetime

json_match = re.search(r'```json\s*(\{.*?\})\s*```', previous_result, re.DOTALL)
if json_match:
    json_content = json_match.group(1)
    try:
        parsed_json = json.loads(json_content)
        timestamp = datetime.now().strftime("%Y%m%d-%H%M%S")
        # JSON의 task.title에서 짧은 파일명 생성 (공백→하이픈, 소문자, 최대 30자)
        short_title = parsed_json.get("task", {}).get("title", "unknown")
        short_title = re.sub(r'[^a-zA-Z0-9가-힣\s]', '', short_title)
        short_title = short_title.strip().replace(' ', '-').lower()[:30]
        output_dir = ".claude/work-analysis"
        os.makedirs(output_dir, exist_ok=True)
        output_path = f"{output_dir}/{timestamp}-{short_title}-analysis.json"
        Write(
            file_path=output_path,
            content=json.dumps(parsed_json, indent=2, ensure_ascii=False)
        )
        json_file_path = output_path
    except json.JSONDecodeError:
        json_file_path = None
else:
    json_file_path = None

# 6. 최종 결과 출력
print(f"**Work Analyzer v2 - 검토 {iterations}회 완료**\n")
print(previous_result)
if json_file_path:
    print(f"\n**분석 결과 저장 완료**: [{json_file_path}]({json_file_path})")
    print(f"**다음 단계**: executor agent에게 이 파일을 전달하여 구현을 시작할 수 있습니다.")
else:
    print("JSON 저장 실패 - 위 출력에서 JSON을 수동으로 복사하세요")
```

## 모델 사용 정책

| 단계 | 모델 | 이유 |
|------|------|------|
| 0차 (질문) | Opus | 불확실성 식별은 높은 추론 능력 필요 |
| 1차 (초기 분석) | Opus | 핵심 분석, 최고 품질 필요 |
| 중간 (검토 + 개선) | Opus | 놓친 부분 발견에 추론 필요 |
| 마지막 (검증만) | Sonnet | 새 내용 없이 정합성 확인만 |

## 예상 소요 시간

| 횟수 | 소요 시간 | 반복 구조 | 용도 |
|------|----------|----------|------|
| 1회 | ~2분 | 0차: 질문, 1차: 초기 분석 | 빠른 스캔, 간단한 작업 |
| 2회 | ~3분 30초 | 0차: 질문, 1차: 초기 분석, 2차: 검증(Sonnet) | 일반 작업 (권장) |
| 3회 | ~5분 | 0차: 질문, 1차: 초기 분석, 2차: 개선, 3차: 검증(Sonnet) | 중요 작업 |
| 5회 | ~9분 | 0차: 질문, 1차: 초기 분석, 2~4차: 개선, 5차: 검증(Sonnet) | 아키텍처 변경 |

## 주의사항

- `.claude/agents/work-analyzer-v2.md`가 반드시 존재해야 합니다
- 핵심 분석(0차, 1차, 중간 반복)은 Opus 사용하여 정확도 유지
- 마지막 검증만 Sonnet (새 내용 추가 없는 역할이므로 충분)
- JSON 출력이 유효하지 않으면 에러 발생 가능
- 마지막 반복에서 새로운 내용을 추가하지 않는 이유: 추가된 내용이 검증되지 않을 수 있기 때문

---

**이제 실행하세요!**

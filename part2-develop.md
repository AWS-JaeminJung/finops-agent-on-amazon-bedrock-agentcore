# Part 2. Hands-on: Kiro로 FinOps Agent 개발하기 (80분)

## 2-1. 환경 설정 (15분)

### Kiro 설치

1. [https://kiro.dev](https://kiro.dev)에서 Kiro 다운로드 및 설치
2. Kiro 실행 후 로그인 (AWS Builder ID 또는 IAM Identity Center)

### AWS 자격 증명 구성

FinOps Agent가 AWS Cost Explorer API를 호출하려면 적절한 권한이 필요합니다.

필요한 IAM 권한:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ce:GetCostAndUsage",
        "ce:GetCostForecast",
        "ce:GetReservationUtilization",
        "ce:GetSavingsPlansUtilization",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "cloudwatch:GetMetricData"
      ],
      "Resource": "*"
    }
  ]
}
```

> 워크샵 환경에서는 사전에 구성된 IAM 역할을 사용합니다.

### 프로젝트 초기화

1. Kiro에서 새 프로젝트 생성
2. 프로젝트 이름: `finops-agent`
3. 기본 프로젝트 구조 확인

```
finops-agent/
├── .kiro/
│   └── specs/          # Kiro Spec 파일들이 저장되는 위치
├── src/
├── package.json (또는 requirements.txt)
└── README.md
```

---

## 2-2. Spec 작성으로 Agent 설계하기 (25분)

### Step 1: Requirement 생성

Kiro에서 **New Spec**을 시작하고, 다음과 같은 프롬프트를 입력합니다:

```
AWS Cost Explorer API를 활용하는 FinOps Agent를 만들고 싶습니다.

이 Agent는 다음을 수행할 수 있어야 합니다:
1. 특정 기간의 AWS 서비스별 비용을 조회
2. 지난 달 대비 비용 변화를 분석
3. 비용 이상 징후를 탐지 (예: 급격한 비용 증가)
4. 비용 최적화 추천을 제공

사용자는 자연어로 질문하고, Agent가 적절한 Tool을 호출하여 답변합니다.
```

Kiro가 생성하는 Requirement 문서를 확인합니다:

- 사용자 스토리가 적절한지 검토
- 수락 기준(Acceptance Criteria)이 합리적인지 확인
- 필요 시 수정 요청

### Step 2: Design 확인

Requirement를 확인하면, Kiro가 **Design 문서**를 자동 생성합니다.

확인할 항목들:

- **데이터 모델**: 비용 데이터 구조, 응답 형식
- **API/Tool 정의**: Agent가 사용할 Tool 목록과 각 Tool의 입출력
- **아키텍처**: Agent의 구성 요소 및 흐름

예상되는 Tool 정의:

| Tool 이름 | 설명 | 입력 | 출력 |
|-----------|------|------|------|
| `get_cost_by_service` | 서비스별 비용 조회 | 시작일, 종료일 | 서비스별 비용 목록 |
| `get_cost_trend` | 비용 추세 분석 | 기간, 서비스(선택) | 월별 비용 추이 |
| `detect_anomaly` | 비용 이상 탐지 | 기준 기간, 임계값 | 이상 항목 목록 |
| `get_optimization_tips` | 최적화 추천 | 서비스(선택) | 추천 사항 목록 |

### Step 3: Task 확인

Design으로부터 Kiro가 구현 **Task 목록**을 생성합니다.

예상 태스크 예시:

- [ ] 프로젝트 기본 구조 및 의존성 설정
- [ ] AWS Cost Explorer 클라이언트 구현
- [ ] `get_cost_by_service` Tool 구현
- [ ] `get_cost_trend` Tool 구현
- [ ] `detect_anomaly` Tool 구현
- [ ] `get_optimization_tips` Tool 구현
- [ ] Agent 메인 로직 구현 (Tool 오케스트레이션)
- [ ] 로컬 테스트 및 검증

---

## 2-3. Kiro와 함께 Agent 코드 구현 (40분)

### Task별 구현 진행

Kiro의 Task 목록에서 각 태스크를 클릭하면, Kiro가 해당 태스크에 맞는 코드를 자동 구현합니다.

### 주요 구현 코드 살펴보기

#### Tool 구현 예시: 서비스별 비용 조회

```python
import boto3
from datetime import datetime, timedelta

def get_cost_by_service(start_date: str, end_date: str) -> dict:
    """특정 기간의 AWS 서비스별 비용을 조회합니다.
    
    Args:
        start_date: 조회 시작일 (YYYY-MM-DD)
        end_date: 조회 종료일 (YYYY-MM-DD)
    
    Returns:
        서비스별 비용 목록
    """
    client = boto3.client('ce')
    
    response = client.get_cost_and_usage(
        TimePeriod={
            'Start': start_date,
            'End': end_date
        },
        Granularity='MONTHLY',
        Metrics=['UnblendedCost'],
        GroupBy=[
            {
                'Type': 'DIMENSION',
                'Key': 'SERVICE'
            }
        ]
    )
    
    results = []
    for time_period in response['ResultsByTime']:
        for group in time_period['Groups']:
            service_name = group['Keys'][0]
            amount = float(group['Metrics']['UnblendedCost']['Amount'])
            if amount > 0:
                results.append({
                    'service': service_name,
                    'cost': round(amount, 2),
                    'unit': group['Metrics']['UnblendedCost']['Unit']
                })
    
    results.sort(key=lambda x: x['cost'], reverse=True)
    return {'services': results, 'period': f"{start_date} ~ {end_date}"}
```

#### Tool 구현 예시: 비용 이상 탐지

```python
def detect_anomaly(threshold_percent: float = 20.0) -> dict:
    """전월 대비 비용이 급증한 서비스를 탐지합니다.
    
    Args:
        threshold_percent: 이상 판단 임계값 (%, 기본 20%)
    
    Returns:
        이상 탐지 결과 목록
    """
    today = datetime.now()
    current_month_start = today.replace(day=1).strftime('%Y-%m-%d')
    last_month_start = (today.replace(day=1) - timedelta(days=1)).replace(day=1).strftime('%Y-%m-%d')
    last_month_end = today.replace(day=1).strftime('%Y-%m-%d')
    
    previous = get_cost_by_service(last_month_start, last_month_end)
    current = get_cost_by_service(current_month_start, today.strftime('%Y-%m-%d'))
    
    prev_map = {s['service']: s['cost'] for s in previous['services']}
    
    anomalies = []
    for service in current['services']:
        name = service['service']
        curr_cost = service['cost']
        prev_cost = prev_map.get(name, 0)
        
        if prev_cost > 0:
            change_pct = ((curr_cost - prev_cost) / prev_cost) * 100
            if change_pct > threshold_percent:
                anomalies.append({
                    'service': name,
                    'previous_cost': prev_cost,
                    'current_cost': curr_cost,
                    'change_percent': round(change_pct, 1)
                })
    
    anomalies.sort(key=lambda x: x['change_percent'], reverse=True)
    return {'anomalies': anomalies, 'threshold': threshold_percent}
```

#### Agent 메인 로직 예시

```python
import strands

from tools import get_cost_by_service, get_cost_trend, detect_anomaly, get_optimization_tips

agent = strands.Agent(
    model="us.anthropic.claude-sonnet-4-6-v1",
    system_prompt="""당신은 AWS FinOps 전문가 Agent입니다.
사용자의 AWS 비용 관련 질문에 답변하고, 비용 최적화를 도와줍니다.
항상 데이터에 기반하여 답변하고, 구체적인 수치를 제시하세요.
비용 절감 추천 시에는 예상 절감액도 함께 안내하세요.""",
    tools=[get_cost_by_service, get_cost_trend, detect_anomaly, get_optimization_tips]
)
```

### 로컬 테스트

구현이 완료되면 로컬에서 Agent를 테스트합니다.

테스트 질문 예시:

```
> 지난 달 AWS 비용을 서비스별로 보여줘
> EC2 비용이 지난 3개월간 어떻게 변했어?
> 비용이 비정상적으로 증가한 서비스가 있어?
> 비용을 줄일 수 있는 방법을 추천해줘
```

각 질문에 대해 Agent가:
1. 적절한 Tool을 선택하는지 확인
2. Tool 호출 결과를 올바르게 해석하는지 확인
3. 사용자에게 유용한 형태로 답변하는지 확인

---

> **다음 단계**: [Part 3. Bedrock AgentCore에 배포하기](part3-deploy.md)

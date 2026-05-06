# Part 3. Hands-on: Bedrock AgentCore에 배포하기 (50분)

## 아키텍처 개요

```
사용자 → ECS Fargate (Streamlit UI :8501)
              ↓ boto3 invoke_agent_runtime()
         Bedrock AgentCore (FinOps Agent :8080)
              ↓ Strands Agents SDK + Tools
         AWS Cost Explorer API
```

| 구성 요소 | 배포 대상 | 포트 |
|-----------|-----------|------|
| FinOps Agent (LLM + Tools) | Bedrock AgentCore (컨테이너) | 8080 |
| Streamlit Chat UI | ECS Fargate | 8501 |

---

## 3-1. AgentCore 배포 준비 (15분)

### AgentCore 컨테이너 프로토콜

![alt text](images/image-6.png)

AgentCore는 컨테이너 기반으로 Agent를 실행합니다. 컨테이너는 다음 HTTP 엔드포인트를 포트 **8080**에 구현해야 합니다:

| 엔드포인트 | 메서드 | 용도 |
|-----------|--------|------|
| `/ping` | GET | 헬스체크 |
| `/invocations` | POST | Agent 호출 |

#### Agent 엔트리포인트 구현 (`main.py`)

```python
"""AgentCore container entry point - HTTP server on port 8080."""

import json
from http.server import HTTPServer, BaseHTTPRequestHandler

from strands import Agent
from strands.models.bedrock import BedrockModel

from aws_finops_agent.agent.config import AWS_REGION, BEDROCK_MODEL_ID
from aws_finops_agent.tools import (
    analyze_monthly_trend, detect_cost_anomalies,
    get_rightsizing_recommendations, get_savings_plans_recommendations,
    query_cost_by_service,
)

SYSTEM_PROMPT = (
    "You are an AWS FinOps assistant. Help users understand and optimize their AWS costs. "
    "Supported: cost query, trend analysis, anomaly detection, optimization recommendations. "
    "Always express amounts in USD. Be concise but thorough."
)

model = BedrockModel(model_id=BEDROCK_MODEL_ID, region_name=AWS_REGION)
agent = Agent(
    model=model,
    tools=[query_cost_by_service, analyze_monthly_trend, detect_cost_anomalies,
           get_savings_plans_recommendations, get_rightsizing_recommendations],
    system_prompt=SYSTEM_PROMPT,
)


class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/ping":
            self._respond(200, {"status": "Healthy"})
        else:
            self._respond(404, {"error": "Not found"})

    def do_POST(self):
        if self.path == "/invocations":
            length = int(self.headers.get("Content-Length", 0))
            body = json.loads(self.rfile.read(length)) if length else {}
            prompt = body.get("prompt", body.get("query", ""))
            try:
                result = agent(prompt)
                if hasattr(result, "message"):
                    msg = result.message
                    if isinstance(msg, dict):
                        content = msg.get("content", [])
                        if isinstance(content, list):
                            texts = [b["text"] for b in content if isinstance(b, dict) and "text" in b]
                            text = "\n".join(texts)
                        else:
                            text = str(content)
                    else:
                        text = str(msg)
                else:
                    text = str(result)
                self._respond(200, {"response": text, "status": "success"})
            except Exception as e:
                self._respond(200, {"response": f"Error: {e}", "status": "error"})
        else:
            self._respond(404, {"error": "Not found"})

    def _respond(self, code, data):
        self.send_response(code)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps(data).encode())

    def log_message(self, format, *args):
        pass


if __name__ == "__main__":
    server = HTTPServer(("0.0.0.0", 8080), Handler)
    print("FinOps Agent ready on port 8080")
    server.serve_forever()
```

### Dockerfile (AgentCore Agent)

```dockerfile
FROM public.ecr.aws/docker/library/python:3.10-slim
WORKDIR /app
RUN pip install --no-cache-dir strands-agents boto3
COPY src/aws_finops_agent/ ./aws_finops_agent/
ENV PYTHONPATH=/app
EXPOSE 8080
CMD ["python", "-m", "aws_finops_agent.main"]
```

> `public.ecr.aws` 기반 이미지를 사용하여 Docker Hub rate limit 문제를 방지합니다.

### IAM 역할 구성

#### Agent 실행 역할 (`finops-agent-role`)

**Trust Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": ["bedrock.amazonaws.com", "bedrock-agentcore.amazonaws.com"]
    },
    "Action": "sts:AssumeRole"
  }]
}
```

**Permission Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CostExplorer",
      "Effect": "Allow",
      "Action": [
        "ce:GetCostAndUsage",
        "ce:GetCostForecast",
        "ce:GetReservationUtilization",
        "ce:GetSavingsPlansUtilization"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EC2CloudWatch",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "cloudwatch:GetMetricData"
      ],
      "Resource": "*"
    },
    {
      "Sid": "BedrockInvoke",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:*:*:foundation-model/*",
        "arn:aws:bedrock:*:*:inference-profile/*"
      ]
    },
    {
      "Sid": "ECR",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 3-2. Agent 배포 및 테스트 (25분)

### AgentCore에 배포 (Control Plane API)

AgentCore 배포는 **Control Plane API** (`bedrock-agentcore-control`)를 사용합니다.

#### Step 1: ECR에 이미지 Push

```bash
# ECR 리포지토리 생성
aws ecr create-repository --repository-name finops-agent

# CodeBuild로 이미지 빌드 (ARM64)
aws codebuild start-build --project-name finops-agent-build
```

#### Step 2: Agent Runtime 생성

```bash
# Agent Runtime 생성 (Control Plane API)
aws bedrock-agentcore-control create-agent-runtime \
  --agent-runtime-name finops_agent_container \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/finops-agent-role \
  --agent-runtime-artifact '{
    "containerConfiguration": {
      "containerUri": "<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/finops-agent:latest"
    }
  }' \
  --network-configuration '{"networkMode": "PUBLIC"}'
```

![alt text](images/image-7.png)
![alt text](images/image-8.png)

#### Step 3: Endpoint 생성

Runtime 생성 후, Agent를 호출할 수 있는 **Endpoint**를 생성합니다.

```bash
# Endpoint 생성 (Control Plane API)
aws bedrock-agentcore-control create-agent-runtime-endpoint \
  --agent-runtime-id <RUNTIME_ID> \
  --name finops \
  --agent-runtime-version <VERSION>
```

![alt text](images/image-9.png)

#### Step 4: 배포 상태 확인

```bash
# Runtime 상태 확인
aws bedrock-agentcore-control get-agent-runtime \
  --agent-runtime-id <RUNTIME_ID>
```

![alt text](images/image-10.png)
![alt text](images/image-11.png)
![alt text](images/image-12.png)

배포 상태가 `ACTIVE`가 되면 Agent를 호출할 수 있습니다.

### 배포된 Agent 테스트 (Data Plane API)

Agent 호출은 **Data Plane API**를 사용합니다.

#### API를 통한 호출

```bash
# Agent에 메시지 보내기 (Data Plane)
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn arn:aws:bedrock-agentcore:us-east-1:<ACCOUNT_ID>:runtime/<RUNTIME_ID> \
  --qualifier finops \
  --runtime-session-id "test-session-001" \
  --payload '{"prompt": "지난 달 비용이 가장 많이 나온 서비스 Top 5를 알려줘"}'
```

#### 테스트 시나리오

다음 시나리오를 순서대로 테스트합니다:

**시나리오 1: 기본 비용 조회**

```
"이번 달 AWS 비용을 서비스별로 보여줘"
```

확인 포인트:
- Cost Explorer API가 정상 호출되는지
- 서비스별 비용이 정렬되어 반환되는지
- 금액 단위(USD)가 올바른지

**시나리오 2: 비용 추세 분석**

```
"최근 3개월간 EC2 비용 추이를 분석해줘"
```

확인 포인트:
- 월별 데이터가 정확한지
- 증감 추세를 올바르게 설명하는지

**시나리오 3: 이상 탐지**

```
"비용이 비정상적으로 증가한 서비스가 있는지 확인해줘"
```

확인 포인트:
- 전월 대비 급증한 서비스를 감지하는지
- 증가율과 금액을 함께 보여주는지

**시나리오 4: 최적화 추천**

```
"비용을 줄일 수 있는 방법을 추천해줘"
```

확인 포인트:
- 실제 사용 데이터 기반의 추천인지
- 예상 절감액이 포함되어 있는지

**시나리오 5: 복합 질의 (멀티 턴)**

```
"지난 달 비용을 보여줘"
→ "그 중에서 가장 많이 증가한 서비스는?"
→ "그 서비스의 비용을 줄이려면 어떻게 해야 해?"
```

확인 포인트:
- 이전 대화 컨텍스트를 유지하는지
- 여러 Tool을 조합하여 답변하는지

---

## 3-3. Streamlit UI 배포 - ECS Fargate (10분)

Agent가 AgentCore에 배포되었으니, 이를 호출하는 **Streamlit Chat UI**를 ECS Fargate에 배포합니다.

### UI에서 AgentCore 호출 코드

```python
"""Streamlit Chat UI - calls AgentCore and renders charts."""

import json, os, uuid
import boto3
import streamlit as st

AGENT_RUNTIME_ARN = os.environ.get("AGENT_RUNTIME_ARN")
AGENT_ENDPOINT = os.environ.get("AGENT_ENDPOINT", "finops")
AWS_REGION = os.environ.get("AWS_REGION", "us-east-1")

def _invoke_agent(query: str, session_id: str) -> str:
    client = boto3.client("bedrock-agentcore", region_name=AWS_REGION)
    resp = client.invoke_agent_runtime(
        agentRuntimeArn=AGENT_RUNTIME_ARN,
        qualifier=AGENT_ENDPOINT,
        runtimeSessionId=session_id,
        payload=json.dumps({"prompt": query}).encode(),
    )
    body = resp["response"].read().decode("utf-8")
    result = json.loads(body)
    return result.get("response", str(result))
```

### Dockerfile (Streamlit UI)

```dockerfile
FROM public.ecr.aws/docker/library/python:3.12-slim
WORKDIR /app
RUN pip install --no-cache-dir streamlit boto3 pandas
COPY src/aws_finops_agent/ui/ ./aws_finops_agent/ui/
COPY src/aws_finops_agent/__init__.py ./aws_finops_agent/__init__.py
COPY app.py .
ENV PYTHONPATH=/app
EXPOSE 8501
HEALTHCHECK CMD curl --fail http://localhost:8501/_stcore/health || exit 1
ENTRYPOINT ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0", "--server.headless=true"]
```

### ECS Fargate 배포

```bash
# ECS 클러스터 생성
aws ecs create-cluster --cluster-name finops-cluster

# 태스크 정의 등록
aws ecs register-task-definition \
  --family finops-streamlit \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu "256" --memory "512" \
  --execution-role-arn arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole \
  --task-role-arn arn:aws:iam::<ACCOUNT_ID>:role/ecsFinopsTaskRole \
  --runtime-platform '{"cpuArchitecture":"ARM64","operatingSystemFamily":"LINUX"}' \
  --container-definitions '[{
    "name": "streamlit",
    "image": "<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/finops-streamlit:latest",
    "portMappings": [{"containerPort": 8501}],
    "environment": [
      {"name": "AGENT_RUNTIME_ARN", "value": "arn:aws:bedrock-agentcore:us-east-1:<ACCOUNT_ID>:runtime/<RUNTIME_ID>"},
      {"name": "AGENT_ENDPOINT", "value": "finops"},
      {"name": "AWS_REGION", "value": "us-east-1"}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/finops-streamlit",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    },
    "essential": true
  }]'

# 서비스 생성
aws ecs create-service \
  --cluster finops-cluster \
  --service-name finops-streamlit-svc \
  --task-definition finops-streamlit \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["<SUBNET_ID>"],
      "securityGroups": ["<SG_ID>"],
      "assignPublicIp": "ENABLED"
    }
  }'
```

#### ECS 태스크 역할 (`ecsFinopsTaskRole`)

Streamlit UI가 AgentCore를 호출하기 위한 권한:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "bedrock-agentcore:InvokeAgentRuntime",
    "Resource": "*"
  }]
}
```

---

## 트러블슈팅

| 문제 | 원인 | 해결 |
|------|------|------|
| Runtime initialization exceeded 30s | codeConfiguration 모드에서 pip install 시간 초과 | 컨테이너 모드(containerConfiguration) 사용 |
| ARM64 incompatible binaries | x86_64 바이너리 포함 | CodeBuild ARM 사용 또는 `--platform manylinux2014_aarch64` |
| 502 from runtime | 컨테이너가 /ping, /invocations 미구현 | 포트 8080에 HTTP 서버 구현 |
| Model access denied | Bedrock 모델 접근 미활성화 | 콘솔에서 Model Access 활성화 |
| Invalid model identifier | foundation-model ID 직접 사용 | inference-profile ID 사용 (`us.anthropic.claude-*`) |
| InvokeModelWithResponseStream denied | IAM에 inference-profile 리소스 미포함 | `arn:aws:bedrock:*:*:inference-profile/*` 추가 |
| Docker Hub 429 rate limit | CodeBuild에서 Docker Hub pull 제한 | `public.ecr.aws/docker/library/` 사용 |
| maxEndpointsPerAgent exceeded | endpoint 수 제한 (4개) | 기존 endpoint 삭제 후 생성 |

---

> **다음 단계**: [Part 4. Wrap-up](part4-wrapup.md)

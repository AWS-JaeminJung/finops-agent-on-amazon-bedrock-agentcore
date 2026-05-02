# Part 4. Wrap-up (20분)

## 전체 과정 리캡

오늘 워크샵에서 진행한 전체 흐름을 돌아봅니다.

```
┌──────────────────────────────────────────────────────────┐
│  1. Spec 기반 설계 (Kiro)                                  │
│     자연어 요구사항 → 구조화된 Requirements → Design → Tasks  │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  2. AI와 함께 구현 (Kiro)                                  │
│     Task 클릭 → 자동 코드 생성 → 리뷰 → 로컬 테스트          │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│  3. 프로덕션 배포 (Bedrock AgentCore)                      │
│     Runtime 설정 → 배포 → 테스트 → 모니터링                  │
└──────────────────────────────────────────────────────────┘
```

### 핵심 배운 점

| 단계 | 도구 | 핵심 가치 |
|------|------|-----------|
| 설계 | Kiro Spec | 체계적 설계, 요구사항 추적 가능 |
| 구현 | Kiro AI | 빠른 구현, Spec 기반 가드레일 |
| 배포 | AgentCore Runtime | 인프라 걱정 없는 배포 |
| 운영 | AgentCore Observability | 자동 모니터링 및 트레이싱 |

---

## 실제 활용 시나리오 논의

### FinOps Agent 확장 아이디어

오늘 만든 Agent를 확장하여 더 많은 가치를 만들 수 있습니다:

#### 추가 Tool 확장

- **예산 알림**: AWS Budgets와 연동하여 예산 초과 시 자동 알림
- **RI/SP 분석**: 예약 인스턴스 및 Savings Plans 커버리지 분석
- **리소스 태깅 감사**: 태그가 누락된 리소스 탐지
- **Trusted Advisor 연동**: 비용 최적화 권고사항 통합

#### 워크플로우 자동화

- Slack/Teams 연동으로 주간 비용 리포트 자동 발송
- 비용 이상 탐지 시 자동 Jira 티켓 생성
- 월간 비용 리뷰 미팅 자료 자동 생성

#### 멀티 계정 지원

- AWS Organizations와 연동하여 여러 계정의 비용 통합 분석
- 팀/프로젝트별 비용 할당 및 차지백(Chargeback)

### 다른 도메인의 Agent 아이디어

오늘 배운 Kiro + AgentCore 패턴은 FinOps 외에도 다양한 도메인에 적용 가능합니다:

- **보안 Agent**: GuardDuty 알림 분석, 자동 대응
- **운영 Agent**: CloudWatch 알람 기반 자동 트러블슈팅
- **컴플라이언스 Agent**: AWS Config 규칙 위반 탐지 및 자동 교정
- **DevOps Agent**: CI/CD 파이프라인 모니터링 및 최적화

---

## 참고 자료

### Kiro

- Kiro 공식 사이트: [https://kiro.dev](https://kiro.dev)
- Kiro 문서: [https://kiro.dev/docs](https://kiro.dev/docs)

### Amazon Bedrock AgentCore

- AgentCore 공식 문서: [https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore.html](https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore.html)
- Strands Agents SDK: [https://github.com/strands-agents/sdk-python](https://github.com/strands-agents/sdk-python)

### FinOps

- FinOps Foundation: [https://www.finops.org](https://www.finops.org)
- AWS Cost Management: [https://aws.amazon.com/aws-cost-management/](https://aws.amazon.com/aws-cost-management/)

---

## Q&A

워크샵 내용에 대한 질문이나, 실제 프로젝트 적용에 대한 논의를 진행합니다.

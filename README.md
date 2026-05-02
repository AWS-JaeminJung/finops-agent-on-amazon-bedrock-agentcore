# AWS FinOps Agent Workshop

> Kiro로 개발하고 Bedrock AgentCore로 배포하기

## 워크샵 개요

이 워크샵에서는 AI 기반 Agentic IDE인 **Kiro**를 활용하여 AWS FinOps Agent를 설계·구현하고, **Amazon Bedrock AgentCore**에 배포하는 전체 과정을 직접 체험합니다.

## 대상

- AWS 클라우드 비용 최적화에 관심 있는 개발자 및 아키텍트
- AI Agent 개발 및 배포 경험을 쌓고 싶은 분
- Kiro와 Bedrock AgentCore를 처음 접하는 분

## 소요 시간

약 3시간 40분 (오후 반일)

## 목차

| Part | 제목 | 시간 | 파일 |
|------|------|------|------|
| 1 | [개념 소개](part1-concepts.md) | 50분 | `part1-concepts.md` |
| - | Break | 10분 | - |
| 2 | [Kiro로 FinOps Agent 개발하기](part2-develop.md) | 80분 | `part2-develop.md` |
| - | Break | 10분 | - |
| 3 | [Bedrock AgentCore에 배포하기](part3-deploy.md) | 50분 | `part3-deploy.md` |
| 4 | [Wrap-up](part4-wrapup.md) | 20분 | `part4-wrapup.md` |

## 사전 준비 사항

- AWS 계정 (Cost Explorer 활성화 필요)
- Kiro 설치 ([https://kiro.dev](https://kiro.dev))
- AWS CLI 설치 및 자격 증명 구성
- Python 3.12+ 또는 Node.js 20+ (Agent 구현 언어에 따라)

## 핵심 흐름

```
Spec으로 설계 (Kiro) → AI와 함께 구현 (Kiro) → Agent 배포 (Bedrock AgentCore)
```

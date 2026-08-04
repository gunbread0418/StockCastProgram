# Concept Learning Notes

## 목적

구현 중 또는 별도의 개념 질문 채팅에서 질문한 내용을 재사용 가능한 프로젝트 지식으로 보존한다.
대화 전문을 쌓는 대신 개념의 책임, 선택 이유, 대안, 오해와 확인 방법을 정리해 구현과 면접에서
다시 사용할 수 있게 한다.

## Category 지도

| Category | 범위 |
|---|---|
| [Infrastructure](infrastructure.md) | Docker, Compose, network, volume, healthcheck, 로컬 운영과 관측 |
| [Backend](backend.md) | Java, Spring Boot, FastAPI, HTTP API, validation, transaction과 서비스 경계 |
| [Data Pipeline](data-pipeline.md) | Kafka, event schema, partition, consumer, retry, ordering과 idempotency |
| [Data Storage](data-storage.md) | PostgreSQL, MongoDB, Redis, schema, query, cache와 fallback |
| [Machine Learning](machine-learning.md) | feature, label, 시계열 검증, model, inference와 versioning |

## 기록 규칙

1. 질문과 가장 가까운 category 하나를 주 기록 위치로 선택한다.
2. 기존 항목을 먼저 검색하고 같은 개념이면 새 항목 대신 기존 설명을 보강한다.
3. 여러 category에 걸치면 주 category에 설명하고 다른 문서에는 필요한 link만 둔다.
4. 기술이나 질문 하나마다 파일을 만들지 않는다. 다섯 category로 감당하기 어려운 질문이
   반복될 때만 사용자와 category 추가를 결정한다.
5. 실제 비밀번호, token, API key, 인증 header, `.env` 값과 개인정보는 기록하지 않는다.
6. 공식 문서, 실행 명령과 검증 결과를 우선 근거로 사용하고 추정은 추정으로 표시한다.

## 항목 형식

각 개념은 필요한 범위에서 다음 구조를 사용한다. 짧은 개념에 불필요한 heading을 억지로
추가하지 않지만 핵심 답변과 프로젝트 연결은 생략하지 않는다.

- 질문
- 핵심 답변
- 프로젝트에서의 연결
- 구분할 개념과 현실적인 대안
- 흔한 오해와 실패 영향
- 확인 방법
- 면접 질문과 답변 keyword

## 개념 전용 채팅 시작 문장

다른 채팅에서 다음 문장을 그대로 사용할 수 있다.

`StockCastProgram 개념 질문 전용 채팅이야. AGENTS.md와 docs/learning/README.md를 읽고 질문에 답한 뒤 관련 category 문서에 중복 없이 기록해줘. 구현 파일과 상태 문서는 수정하지 마.`

개념 전용 채팅에서는 구현을 진행하지 않는다. 구현 중인 TODO는 구현 채팅에서 이어가며, 개념
문서는 기존 Git 규칙에 따라 관련 파일만 검증·commit·push한다.

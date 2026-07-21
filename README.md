# Realtime Stock Forecast Platform

실시간 또는 실시간에 가까운 주식 시세를 Kafka 이벤트 파이프라인으로 처리하고,
PostgreSQL, MongoDB, Redis에 목적별로 저장한 뒤 FastAPI 기반 ML 서비스로 다음 5분
가격 방향을 분류하는 백엔드 중심 학습 프로젝트입니다.

이 프로젝트의 핵심은 자동매매나 예측 정확도 과장이 아니라 다음 역량을 직접 구현하고
설명할 수 있게 만드는 것입니다.

- Spring Boot 기반 API와 도메인 설계
- Kafka producer, consumer group, offset, retry, DLQ와 재처리
- SQL, NoSQL, cache의 책임 분리
- Docker Compose 기반 로컬 인프라
- Java 백엔드와 Python ML 서비스 연동
- 시계열 feature, baseline model과 leakage 없는 평가
- 중복, 지연, 순서 역전, 의존 서비스 장애 대응

## 프로젝트 문서

- [프로젝트 정체성과 목표](docs/PROJECT_CONTEXT.md)
- [현재 진행 상태](docs/PROJECT_STATUS.md)
- [전체 로드맵](docs/ROADMAP.md)
- [시스템 아키텍처](docs/architecture/SYSTEM_OVERVIEW.md)
- [설계 결정 기록](docs/adr/README.md)
- [오류 및 해결 기록](docs/TROUBLESHOOTING.md)
- [작업 기록 규칙](docs/worklogs/README.md)
- [Codex 작업 규칙](AGENTS.md)

## 현재 상태

프로젝트 문서 체계를 구성하는 `M0. Repository 초기화와 설계 기준` 단계입니다.
애플리케이션과 인프라 코드는 아직 구현하지 않았습니다. 정확한 최신 상태는
[`docs/PROJECT_STATUS.md`](docs/PROJECT_STATUS.md)를 확인합니다.

## 계획된 저장소 구조

```text
StockCastProgram/
|-- backend-spring/
|-- prediction-service/
|-- infra/
|-- scripts/
|-- docs/
|-- AGENTS.md
|-- README.md
|-- .gitignore
`-- .env.example
```

## 비목표

- 자동매매 주문 시스템을 만들지 않습니다.
- 투자 수익 또는 높은 예측 정확도를 보장하지 않습니다.
- 프론트엔드를 핵심 학습 대상으로 삼지 않습니다.
- 처음부터 실제 주식 API와 거래 시간에 테스트가 종속되게 하지 않습니다.
- 기술을 많이 넣었다는 사실만 보여주는 프로젝트로 만들지 않습니다.

## 시작 원칙

실제 주식 API보다 재현 가능한 `FakeMarketDataGenerator`로 raw tick부터 저장, cache,
candle, feature, prediction까지 검증한 후 provider adapter를 통해 실제 API로 확장합니다.

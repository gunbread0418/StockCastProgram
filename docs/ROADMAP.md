# Project Roadmap

## 운영 규칙

- 상태는 `NOT_STARTED`, `IN_PROGRESS`, `BLOCKED`, `DONE` 중 하나를 사용한다.
- 현재 진행 세부 내용은 `PROJECT_STATUS.md`에만 기록한다.
- 완료는 구현, 테스트, 문서와 면접 설명 포인트가 모두 준비된 상태를 의미한다.
- 기능을 제외하지 않고 기본, 확장, 고도화 구현으로 나눈다.
- 마일스톤 전환 전 상태 문서를 실제 저장소와 동기화한다.

## 전체 현황

| ID | 마일스톤 | 상태 | 권장 branch |
|---|---|---|---|
| M0 | Repository 초기화와 설계 기준 | DONE | `docs/project-foundation` |
| M1 | Docker Compose 인프라 | DONE | `infra/local-compose` |
| M2 | Spring Boot 기본 구성 | NOT_STARTED | `feat/spring-bootstrap` |
| M3 | PostgreSQL과 Stock 도메인 | NOT_STARTED | `feat/stock-domain` |
| M4 | Kafka 기본과 FakeMarketDataGenerator | NOT_STARTED | `feat/fake-market-producer` |
| M5 | MongoDB raw event 저장 | NOT_STARTED | `feat/raw-event-archive` |
| M6 | Tick 정규화와 PostgreSQL 저장 | NOT_STARTED | `feat/tick-normalization` |
| M7 | Redis latest price와 조회 API | NOT_STARTED | `feat/latest-price-cache` |
| M8 | 1분 Candle aggregation | NOT_STARTED | `feat/candle-aggregation` |
| M9 | FastAPI prediction service 기본 | NOT_STARTED | `feat/fastapi-bootstrap` |
| M10 | Baseline model과 feature pipeline | NOT_STARTED | `feat/baseline-model` |
| M11 | Spring Boot와 FastAPI 동기 예측 | NOT_STARTED | `feat/sync-prediction` |
| M12 | Kafka 비동기 prediction pipeline | NOT_STARTED | `feat/async-prediction-pipeline` |
| M13 | Swagger와 실시간 전달 | NOT_STARTED | `feat/api-live-demo` |
| M14 | Backtest | NOT_STARTED | `feat/backtest-baseline` |
| M15 | 테스트, 장애 실험, monitoring과 포트폴리오 | NOT_STARTED | `docs/portfolio-hardening` |

## M0. Repository 초기화와 설계 기준

- 핵심 학습: monorepo, README, ADR, 상태 문서, secret 관리, Git 작업 단위
- 구현: 프로젝트 문서, `.gitignore`, `.env.example`, 기본 directory 정책
- 완료 기준: 새 채팅에서 프로젝트와 현재 상태를 복구하고 secret 없이 다음 작업을 시작 가능
- 검증: 문서 link, Git status, 민감 정보 점검
- 면접 포인트: monorepo 선택, ADR 목적, 지원 버전 고정 방식

## M1. Docker Compose 인프라

- 핵심 학습: image, container, network, volume, healthcheck, readiness
- 기본: PostgreSQL, Redis, MongoDB, Kafka, Kafka UI
- 확장: app container와 Compose profile
- 완료 기준: 각 서비스가 healthy이고 재시작 후 필요한 데이터가 보존됨
- 검증: DB 접속, Redis ping, Mongo ping, Kafka topic produce/consume

## M2. Spring Boot 기본 구성

- 핵심 학습: DI, auto-configuration, profile, externalized configuration
- 기본: Java 21, Spring Boot 3.x, health, 공통 오류와 test profile
- 완료 기준: context test와 health API 성공
- 검증: 정상 실행, invalid request 오류 계약

## M3. PostgreSQL과 Stock 도메인

- 핵심 학습: JPA entity lifecycle, transaction, constraint, migration, pagination
- 기본: stock 등록, 목록, 상세와 unique constraint
- 완료 기준: migration 재실행과 integration test 성공
- 검증: 중복 409, Validation 400, rollback

## M4. Kafka 기본과 FakeMarketDataGenerator

- 핵심 학습: topic, producer, key, partition, serialization와 deterministic test data
- 기본: seed 기반 raw tick 생성과 `market.tick.raw` 발행
- 확장: 지연, 중복, invalid, 순서 역전 모드
- 완료 기준: 같은 seed로 같은 이벤트를 재현하고 Kafka UI에서 확인

## M5. MongoDB raw event 저장

- 핵심 학습: document model, flexible schema, TTL, idempotent consumer
- 기본: raw event consumer와 `raw_market_events`
- 완료 기준: 원본이 손실 없이 저장되고 동일 event 재수신이 중복 document를 만들지 않음
- 검증: Mongo 장애, duplicate, unsafe header 제거

## M6. Tick 정규화와 PostgreSQL 저장

- 핵심 학습: canonical schema, validation, offset과 DB side effect 경계
- 기본: raw를 normalized로 변환하고 `market_ticks`에 저장
- 완료 기준: eventId로 raw부터 canonical row까지 추적 가능
- 검증: 누락, 음수, timezone, duplicate와 poison event

## M7. Redis latest price와 조회 API

- 핵심 학습: cache-aside, TTL, freshness, fallback, stale update
- 기본: latest price key와 조회 API
- 완료 기준: cache hit와 PostgreSQL fallback이 같은 API 계약 제공
- 검증: Redis down/flush와 늦은 과거 tick

## M8. 1분 Candle aggregation

- 핵심 학습: event time window, OHLCV, late event, correction와 upsert
- 기본: normalized tick에서 `market.candle.1m` 생성
- 확장: allowed lateness와 candle revision
- 완료 기준: 수동 계산과 일치하고 재처리 결과가 멱등함

## M9. FastAPI prediction service 기본

- 핵심 학습: Pydantic, OpenAPI, lifecycle, model registry 경계
- 기본: health, models, mock predict
- 완료 기준: request/response contract test와 invalid input 검증

## M10. Baseline model과 feature pipeline

- 핵심 학습: label, feature, chronological split, leakage, artifact 재현성
- 기본: DummyClassifier와 Logistic Regression
- 확장: XGBoost와 LightGBM 비교
- 완료 기준: 고정 데이터로 동일 metric과 artifact 재생성

## M11. Spring Boot와 FastAPI 동기 예측

- 핵심 학습: HTTP contract, timeout, retry 분류, partial failure
- 기본: prediction request, FastAPI 호출, PostgreSQL/Redis 결과 저장
- 완료 기준: requestId로 입력, model, 결과를 추적 가능
- 검증: timeout, 4xx, 5xx, duplicate request

## M12. Kafka 비동기 prediction pipeline

- 핵심 학습: asynchronous workflow, eventual consistency, correlation, DLQ
- 기본: prediction request/result topic과 상태 API
- 확장: batch inference, outbox/inbox, shadow model
- 완료 기준: Spring과 Python의 독립 재시작 후에도 요청과 결과를 복구

## M13. Swagger와 실시간 전달

- 핵심 학습: OpenAPI contract, WebSocket/SSE 비교, connection lifecycle
- 기본: Swagger example과 최소 실시간 테스트 화면
- 완료 기준: 브라우저에서 가격과 예측 갱신 확인
- 검증: reconnect, invalid ticker, slow client

## M14. Backtest

- 핵심 학습: walk-forward, cost, slippage, drawdown와 bias
- 기본: model version별 backtest run과 결과 저장
- 완료 기준: runId로 기간, 설정, model과 metric 재현
- 검증: 수수료 변화, look-ahead와 baseline 비교

## M15. 테스트, 장애 실험, monitoring과 포트폴리오

- 핵심 학습: test pyramid, SLI, consumer lag, incident report와 evidence
- 기본: 통합/계약/부하 테스트와 장애 실험
- 확장: Prometheus, Grafana, alert 기준
- 완료 기준: 장애 재현, 복구 과정, 성능과 trade-off를 README와 면접 문서로 설명 가능
- 검증: Kafka backlog, Redis fallback, FastAPI 장애, DLQ replay, cold start

# Project Context

## 문서 목적

이 문서는 새 채팅이나 새로운 작업자가 프로젝트의 정체성, 사용자의 학습 목적과 변하지 않는
기술 방향을 빠르게 이해하기 위한 기준 문서다. 현재 진행률은 `PROJECT_STATUS.md`, 세부 순서는
`ROADMAP.md`, 구조는 `architecture/SYSTEM_OVERVIEW.md`를 따른다.

## 프로젝트 이름

Realtime Stock Forecast Platform

## 한 줄 정의

실시간에 가까운 주식 시세를 Kafka 이벤트 파이프라인으로 처리하고 목적에 따라
PostgreSQL, MongoDB, Redis에 저장한 뒤 독립된 FastAPI ML 서비스로 단기 가격 방향을
예측하는 백엔드 중심 데이터 플랫폼이다.

## 프로젝트를 하는 이유

목표는 단순한 서비스 완성이 아니다. 사용자가 코드를 직접 작성하면서 각 기술을 이해하고,
기술 면접에서 다음 내용을 자신의 구현과 검증 결과로 설명할 수 있는 수준에 도달하는 것이다.

- 왜 이 기술을 선택했는가
- 시스템의 어디에 연결되는가
- 내부적으로 어떻게 동작하는가
- 더 단순한 대안은 무엇인가
- 어떤 장애, 데이터 유실, 중복과 지연을 고려했는가
- 실제로 어떻게 테스트하고 관찰했는가

## 핵심 목표

### 기능 목표

- `FakeMarketDataGenerator` 또는 실제 provider에서 tick 수집
- raw tick, normalized tick, 1분 candle 이벤트 처리
- latest price, candle, prediction, model, backtest API 제공
- baseline model 기반 다음 5분 방향 예측
- 선택적으로 WebSocket 또는 최소 UI로 실시간 결과 확인

### 백엔드 학습 목표

- Java 21과 Spring Boot 3.x 기반 REST API
- Validation, 예외 계약, JPA transaction과 외부 연동
- Kafka consumer와 DB side effect 사이의 처리 경계
- Redis cache와 PostgreSQL fallback
- 동기 REST 추론과 비동기 Kafka 추론 비교

### 데이터 엔지니어링 학습 목표

- topic, partition, message key, consumer group, offset
- at-least-once 전달과 멱등성
- retry, backoff, DLQ와 replay
- event time, processing time, late event와 순서 역전
- raw, canonical, projection 데이터의 책임 분리

### 인프라 학습 목표

- Docker Compose service, network, volume, environment, healthcheck
- Kafka, PostgreSQL, Redis, MongoDB의 로컬 재현 환경
- 애플리케이션 readiness와 의존 서비스 장애
- 선택적으로 Prometheus와 Grafana를 통한 관측

### ML 학습 목표

- 시계열 label과 feature 정의
- Dummy와 Logistic Regression baseline
- chronological split, walk-forward와 leakage 방지
- training-serving feature 일치
- model version, feature schema version과 artifact 재현성
- XGBoost, LightGBM, LSTM, Transformer 도입 조건 비교

## 비목표

- 자동매매 주문 실행
- 투자 권유 또는 수익 보장
- 프론트엔드 중심 제품 개발
- 처음부터 실제 주식 API에 종속된 테스트
- 근거 없이 기술 개수를 늘리는 설계
- baseline 검증 없이 딥러닝 모델부터 도입

## 사용자 학습 방식

- 사용자가 가능한 한 코드를 직접 작성한다.
- Codex는 구현 전에 개념과 순서를 설명한다.
- 작업은 작고 검증 가능한 TODO로 나눈다.
- Codex는 전체 코드를 한 번에 제공하지 않는다.
- 사용자가 작성한 코드는 문제, 이유, 영향, 수정 방향, 검증 방법 순서로 리뷰한다.
- 단계마다 Git branch와 commit 단위를 추천한다.
- 단계가 끝나면 면접 질문을 제공한다.
- 기능은 제외하지 않고 기본, 확장, 고도화 단계로 나눈다.

## 요구 기술

### Backend

- Java 21
- Spring Boot 3.x
- Spring Web
- Spring Data JPA
- Spring Kafka
- Spring Data Redis
- Spring Data MongoDB
- Validation
- 선택 확장: Spring Security

### Messaging and data

- Apache Kafka와 Kafka UI
- PostgreSQL
- MongoDB
- Redis

### ML

- Python
- FastAPI
- pandas
- scikit-learn
- baseline model
- 확장 후보: XGBoost, LightGBM, LSTM, Transformer

### Infra and collaboration

- Docker Compose
- Git/GitHub
- Conventional Commits
- Swagger/OpenAPI와 Postman
- 선택 확장: WebSocket, Prometheus, Grafana

## 확정 기술 기준

다음 항목은 2026-07-20에 사용자가 확정한 초기 기준이다.

| 항목 | 값 | 상태 |
|---|---|---|
| Java package | `com.stockcast` | 확정 |
| Build tool | Gradle Kotlin DSL | 확정 |
| 초기 exchange | `FAKE` | 확정 |
| 초기 종목 수 | 3 | 확정 |
| tick 간격 | 1초 | 확정 |
| candle 간격 | 1분 | 요구사항 기준 |
| 1차 ML target | 다음 5분 방향 분류 | 확정 |
| DB 시간 저장 | UTC | 확정 |

## 데이터 저장소 책임

- Kafka: 이벤트 전달, producer/consumer 결합 제거, 완충과 replay
- PostgreSQL: 종목, 정규화 tick, candle, prediction, model version, backtest 기준 데이터
- MongoDB: 공급자 raw payload, 파싱 실패 원본, 선택적으로 뉴스와 공시
- Redis: latest price, latest prediction, 선택적으로 rate limit과 ranking
- FastAPI: Python feature pipeline, model loading과 inference
- Spring Boot: API, Java 도메인, 파이프라인 orchestration과 저장 결과 제공

## 핵심 설계 원칙

1. fake provider로 end-to-end를 검증한 후 실제 provider로 교체한다.
2. Kafka consumer는 at-least-once와 중복 전달을 전제로 멱등하게 만든다.
3. Redis는 기준 저장소가 아니며 장애 시 PostgreSQL로 fallback한다.
4. event time과 processing time을 구분한다.
5. 가격은 부동소수점 오차를 피하는 타입과 직렬화 규칙을 사용한다.
6. 모든 이벤트에 고유 ID와 schema version을 둔다.
7. 모델 평가는 무작위 split이 아니라 시간 순서를 보존한다.
8. 예측 정확도와 투자 수익성을 같은 의미로 사용하지 않는다.
9. 기능의 완료는 코드 작성뿐 아니라 실패 시나리오와 검증 결과를 포함한다.

## 포트폴리오 핵심 메시지

`주가를 잘 맞히는 서비스`보다 `실시간 이벤트를 안정적으로 처리하고 재현 가능한 ML 추론까지
연결한 데이터 플랫폼`으로 소개한다. 포트폴리오에는 기술 이름보다 중복, 지연, replay, cache
fallback, 모델 version과 시계열 검증을 어떻게 구현하고 측정했는지를 제시한다.

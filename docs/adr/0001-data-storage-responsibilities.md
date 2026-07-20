# ADR-0001: 데이터 저장소와 메시징 책임 분리

- 상태: Accepted
- 날짜: 2026-07-14
- 결정자: 프로젝트 사용자

## Context

프로젝트는 Kafka, PostgreSQL, MongoDB와 Redis를 사용한다. 각 기술에 명확한 책임이 없으면
동일 데이터를 근거 없이 여러 곳에 저장하는 장난감 프로젝트처럼 보일 수 있다. 장애 시 어떤
저장소가 기준인지, 어떤 데이터를 재생성할 수 있는지와 replay가 어디에서 시작되는지 정해야 한다.

## Decision

### Kafka

시세, candle과 prediction 이벤트의 전달, producer/consumer 결합 제거, 처리 속도 차이의 완충과
짧은 기간 replay를 담당한다. 장기 데이터 조회용 DB로 사용하지 않는다.

### PostgreSQL

종목, 정규화 tick, candle, prediction, model version과 backtest처럼 관계, uniqueness, transaction과
정형 query가 중요한 데이터의 기준 저장소로 사용한다.

### MongoDB

공급자 raw market event, API response, 파싱 실패 payload와 선택적인 뉴스·공시처럼 스키마가
공급자별로 변할 수 있는 원본 데이터를 보존한다. 원본 replay와 parser 품질 분석에 사용한다.

### Redis

latest price와 latest prediction처럼 현재 상태 중심이고 조회 빈도가 높은 projection을 저장한다.
Redis 데이터는 PostgreSQL 또는 이벤트에서 재생성할 수 있어야 하며 장애 시 기준 DB로 fallback한다.

### FastAPI

저장소는 아니지만 Python feature, model loading과 inference의 서비스 경계를 소유한다. Spring Boot가
Python ML library에 직접 결합되지 않게 한다.

## Alternatives considered

### PostgreSQL만 사용

원본은 JSONB, 최신값은 indexed query로 처리할 수 있다. 가장 단순한 대안이다. 하지만 이번
프로젝트에서는 raw schema 변화, cache 장애와 event replay를 직접 학습하려는 목표가 있어 역할을
분리한다.

### PostgreSQL과 Redis만 사용

원본 payload의 독립 보존 요구가 작다면 합리적이다. MongoDB에서 raw replay나 schema 변화 분석을
실제로 구현하지 못하면 이 대안으로 축소해야 한다.

### 동기 REST 처리

초기 디버깅에는 더 단순하다. 그러나 저장, candle과 prediction 소비자를 독립시키고 burst를
완충하며 replay하는 학습 목표 때문에 Kafka를 사용한다. Kafka 이점은 다중 consumer와 장애
재처리 실험으로 증명해야 한다.

### Local in-memory cache

단일 인스턴스에는 Caffeine이 더 단순하다. Redis는 외부 cache, TTL, 장애 fallback과 이후 다중
인스턴스 공유 상태를 학습하기 위해 사용한다.

## Consequences

### Positive

- 원본, canonical data와 latest projection의 책임이 명확해진다.
- 원본에서 parser를 다시 실행할 수 있다.
- consumer를 독립적으로 확장하고 lag를 관찰할 수 있다.
- Redis 유실을 허용하고 기준 DB로 복구할 수 있다.
- 기술 선택 이유와 제거 조건을 면접에서 설명할 수 있다.

### Negative

- 로컬 인프라와 설정 복잡도가 증가한다.
- 중복 저장과 eventual consistency를 다뤄야 한다.
- Kafka offset과 외부 DB transaction 사이의 원자성 문제가 생긴다.
- 데이터 retention, schema version과 모니터링 책임이 늘어난다.

## Guardrails

- 모든 consumer는 중복 전달을 전제로 설계한다.
- Redis를 유일한 기준 저장소로 사용하지 않는다.
- raw payload에 API key, authorization header 또는 cookie를 저장하지 않는다.
- MongoDB 활용 근거를 raw replay와 schema 변화 실험으로 증명한다.
- Kafka 활용 근거를 다중 consumer, lag, 재처리와 DLQ 실험으로 증명한다.
- 기술 도입 이점을 검증하지 못하면 더 단순한 대안을 다시 평가한다.

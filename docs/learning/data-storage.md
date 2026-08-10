# Data Storage Concepts

## 범위

PostgreSQL, MongoDB와 Redis의 데이터 모델, schema, query, index, transaction, cache,
persistence, consistency와 fallback 개념을 기록한다.

## 기록된 개념

### 재생성 가능한 Redis 데이터에도 persistence가 필요한 경우

- 기록일: 2026-08-10
- 질문: 현재 Redis 캐시는 PostgreSQL이나 Kafka 이벤트에서 다시 만들 수 있는데, 어떤 기능을
  추가하면 RDB 또는 AOF를 도입할 이유가 생기는가?

#### 핵심 판단

Redis 데이터를 재생성할 수 있다는 사실과 Redis persistence가 불필요하다는 판단은 항상 같은
뜻이 아니다. 데이터의 정확성을 PostgreSQL이나 Kafka가 보장하더라도, Redis를 다시 채우는 데
오래 걸리거나 그동안 API 기능이 비어 보인다면 persistence를 **복구 시간 단축 수단**으로 사용할
수 있다. 다만 Redis를 유일한 기준 저장소로 바꾸지는 않는다.

현재의 최신 가격과 최신 예측값 몇 개는 다시 만드는 비용이 작으므로 RDB와 AOF를 끄는 편이
적절하다. 단순히 캐시가 사라지는 것이 싫다는 이유만으로 persistence를 켜면 디스크 사용량,
쓰기 비용과 운영 복잡도만 늘어난다.

#### 가장 적합한 기능: 실시간 시장 순위와 rolling window 지표

다음 기능을 확장 구현으로 추가하면 persistence 실험을 설득하기 쉽다.

- `GET /markets/FAKE/rankings/gainers`: 등락률 상위 종목을 실시간으로 조회한다.
- `GET /markets/FAKE/rankings/volume`: 거래량 상위 종목을 실시간으로 조회한다.
- `GET /stocks/{ticker}/indicators?window=30m`: 최근 30분의 이동평균, 변동성 같은 지표를
  조회한다.

Redis의 sorted set에는 점수를 기준으로 정렬되는 실시간 순위를 저장하고, 종목별 최근 가격
구간과 마지막으로 반영한 `eventId`, `occurredAt`을 함께 저장할 수 있다. 원본 tick과 candle은
PostgreSQL에, 재처리할 이벤트는 Kafka에 남긴다.

```text
FakeMarketDataGenerator
-> Kafka 이벤트
-> consumer가 Redis의 순위와 최근 구간을 갱신
-> API가 Redis에서 즉시 조회

Redis 재시작
-> RDB/AOF로 최근 projection 복원
-> 마지막 eventId 이후 이벤트만 Kafka 또는 PostgreSQL에서 보충
```

projection은 원본 데이터를 조회 목적에 맞게 계산해 둔 파생 상태를 뜻한다. persistence가 없으면
Redis 재시작 뒤 최근 30분 또는 1시간의 이벤트를 모두 다시 읽는 동안 순위 API가 비거나
PostgreSQL에 재구축 부하가 몰릴 수 있다. persistence가 있으면 최근 projection부터 빠르게
복원하고 누락 구간만 보충할 수 있다.

복원된 값이 오래된 상태일 수 있으므로 `eventId`, 계산 시각과 TTL을 함께 확인해야 한다. TTL은
정해진 시간이 지나면 키가 자동으로 만료되게 하는 값이다. API는 허용한 신선도 기준을 넘으면
PostgreSQL fallback을 사용하거나 재구축 중임을 알려야 한다.

#### 기능 후보 비교

| 기능 | persistence를 설득하는 이유 | 판단 |
| --- | --- | --- |
| 실시간 순위와 rolling window 지표 | 재구축할 이벤트가 많아질수록 재시작 직후 복구 시간과 DB 부하가 커진다. RDB와 AOF를 모두 비교할 수 있다. | 프로젝트 중심 기능으로 가장 적합하다. |
| 분산 API rate limiter | Redis가 재시작해도 사용자별 호출 횟수를 바로 초기화하지 않아야 한다. 짧은 제한 구간에는 AOF가 RDB보다 적합하다. | 작은 실험에는 좋지만 주식 데이터 기능과의 연결은 약하다. |
| 사용자 watchlist 또는 세션의 유일한 저장소 | 유실되면 사용자 데이터나 로그인이 사라진다. | persistence만 믿지 말고 PostgreSQL을 기준 저장소로 사용해야 한다. |
| Redis Streams로 Kafka 대체 | 이벤트를 Redis에 남겨 복구할 수 있다. | Kafka가 이미 이벤트 전달과 재처리를 맡으므로 책임이 중복된다. |

#### RDB, AOF와 동시 사용의 선택 기준

- RDB는 특정 시점의 전체 상태를 snapshot으로 남긴다. snapshot은 그 시점의 데이터를 한 번에
  묶어 저장한 복사본이다. 파일이 비교적 작고 재시작이 빠르지만 마지막 snapshot 이후 변경은
  잃을 수 있으므로, 몇 분의 순위 상태를 다시 계산해도 되는 기능에 적합하다.
- AOF는 쓰기 명령을 순서대로 기록한다. `appendfsync everysec`를 사용하면 보통 최근 약 1초의
  변경을 잃을 가능성을 감수하는 대신 더 최신 상태로 복구할 수 있다. 파일과 쓰기 비용은 RDB보다
  커질 수 있다.
- 둘을 함께 켜는 방식은 RDB와 AOF의 복구 특성을 모두 실험할 때 의미가 있다. Redis는 두 파일이
  모두 있으면 더 완전한 AOF를 기준으로 복원하므로, 현재 캐시에 두 방식이 모두 필수라고
  주장해서는 안 된다.

현재 프로젝트에서 실제 도입 조건은 예를 들어 "Redis를 비운 뒤 순위 재구축에 30초 이상 걸리고,
API는 5초 안에 조회 가능한 상태로 돌아와야 한다"처럼 측정 가능한 복구 목표가 생기는 것이다.

#### 검증 실험

동일한 Fake 이벤트를 사용해 다음 네 모드를 비교하면 선택 근거를 수치로 남길 수 있다.

1. persistence 없음
2. RDB만 사용
3. AOF와 `appendfsync everysec` 사용
4. RDB와 AOF 동시 사용

각 모드에서 Redis 컨테이너를 재생성한 뒤 복원된 순위 수, 마지막 `eventId`, 손실된 갱신 수,
healthy 상태까지 걸린 시간, Kafka 또는 PostgreSQL 재구축 시간, 디스크 사용량과 API가 비거나
오래된 값을 반환한 시간을 측정한다. 이 결과로 데이터 유실 허용 범위, 복구 시간과 운영 비용
사이의 트레이드오프를 설명할 수 있다. 트레이드오프는 한쪽 장점을 얻는 대신 다른 비용을
감수하는 관계다.

#### 흔한 오해와 면접 포인트

- persistence를 켰다고 Redis가 PostgreSQL을 대신하는 기준 저장소가 되는 것은 아니다.
- RDB와 AOF는 Redis 프로세스 재시작과 컨테이너 재생성에 대비한 파일 저장 방식이다. Docker
  named volume처럼 컨테이너 밖에 남는 저장 공간이 없으면 컨테이너를 삭제할 때 파일도 사라진다.
- 캐시에서도 정확성보다 복구 시간과 재구축 비용 때문에 persistence를 사용할 수 있다.
- 면접에서는 "왜 Redis 데이터를 저장했는가"보다 "원본은 어디에 있고, 어느 정도의 유실을
  허용하며, 재시작 뒤 어떻게 최신 이벤트까지 따라잡는가"를 함께 설명해야 한다.

#### 공식 자료

- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis sorted sets](https://redis.io/docs/latest/develop/data-types/sorted-sets/)
- [Redis rate limiter](https://redis.io/docs/latest/develop/use-cases/rate-limiter/)

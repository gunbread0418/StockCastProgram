# Spring Backend

## 책임

Spring Boot API, Java 도메인, Kafka producer/consumer, PostgreSQL·MongoDB·Redis 연동과
`FakeMarketDataGenerator`를 소유한다. 저장 결과를 외부에 제공하고 Java 기반 이벤트
파이프라인을 조정하는 애플리케이션 경계다.

## 현재 기술 기준

- Java 21
- Spring Boot 4.1.0
- Gradle Kotlin DSL과 Gradle Wrapper 9.5.1
- Java base package `com.stockcast`

M2에서 공식 Spring Initializr project를 생성했고 기본 context test와
`/actuator/health`의 `UP` 응답을 검증했다.

## 포함할 항목

- Gradle Wrapper와 Gradle Kotlin DSL 빌드 설정
- `com.stockcast` 아래의 Java source와 단위·통합 테스트
- REST API, 도메인 규칙, provider adapter와 Kafka producer/consumer
- 애플리케이션 설정과 선택된 DB migration 도구의 schema 변경 파일
- 필요할 때 이 애플리케이션 자체를 빌드하는 Dockerfile

## 포함하지 않을 항목

- FastAPI, Python feature와 모델 학습·추론 코드
- Kafka broker, PostgreSQL, MongoDB, Redis의 서버 실행 설정
- Gradle build output, 실제 secret, 로컬 DB 데이터와 dump

## 구현 시점

M2에서 Spring Boot 기본 프로젝트를 구성한다. `FakeMarketDataGenerator`와 Kafka 발행은
M4에서 추가하며, 더 이른 단계에는 빈 source 구조를 미리 만들지 않는다.

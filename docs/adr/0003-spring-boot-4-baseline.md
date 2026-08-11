# ADR-0003: Spring Boot 4.1.0 기준 채택

- 상태: Accepted
- 날짜: 2026-08-11
- 결정자: 프로젝트 사용자

## Context

프로젝트의 초기 백엔드 기준은 Java 21과 Spring Boot 3.x였다. Java 21 JDK 사전검증을 마친 뒤
공식 Spring Initializr의 현재 metadata를 확인했으나 Spring Boot 4.1.0과 4.0.7만 안정 버전으로
제공했고, `bootVersion=3.5.16` 생성 요청은 Spring Boot 호환 범위가 4.0.0 이상이라는 400 응답으로
거부됐다.

Spring Boot 3.5.16을 수동으로 구성할 수는 있지만 마지막 3.5.x OSS release이며, 아직 Spring source와
Gradle build가 없어 큰 버전 변경에 따른 migration 대상도 없다. 새 프로젝트의 재현성과 현재 공식
문서·Initializr 지원 흐름을 함께 고려해 백엔드 기준을 다시 결정할 필요가 있었다.

## Decision

StockCast Spring backend의 기준을 Java 21과 Spring Boot 4.1.0으로 변경한다. 공식 Spring Initializr에서
다음 metadata와 dependency를 사용해 `backend-spring/` project를 생성한다.

- Project: Gradle Kotlin DSL
- Language: Java
- Group과 package: `com.stockcast`
- Artifact와 name: `backend-spring`
- Packaging: executable JAR
- Java toolchain: 21
- Configuration format: Properties
- Dependency: Spring Web, Validation, Spring Boot Actuator

Initializr가 생성한 Gradle Wrapper 9.5.1을 project에 함께 추적한다. Spring Boot 4.1에서 생성된
`spring-boot-starter-webmvc`, `spring-boot-starter-validation`, `spring-boot-starter-actuator`와 기능별
test starter를 우선 사용하고, persistence와 messaging dependency는 해당 마일스톤에서 추가한다.

## Alternatives considered

### Spring Boot 3.5.16 수동 구성

기존 3.x 기준을 유지할 수 있다. 하지만 현재 Initializr가 생성하지 않으므로 Gradle Wrapper와 build
script를 직접 맞춰야 하며 3.5.16은 마지막 3.5.x OSS release다. 아직 migration할 source가 없는 시점에
지원이 끝나는 기준으로 새 project를 시작하는 이점이 작아 선택하지 않았다.

### Spring Boot 4.0.7

Initializr가 제공하는 안정 버전이지만 4.1.0보다 이전 minor version이다. 기존 4.0 code나 dependency
호환성 제약이 없으므로 새 project에서는 현재 기본 안정 버전인 4.1.0을 선택했다.

### Snapshot version

아직 release되지 않은 변경을 먼저 시험할 수 있지만 build 재현성과 dependency 안정성이 떨어진다.
학습 프로젝트의 기준 build에는 snapshot을 사용하지 않는다.

## Consequences

### Positive

- 공식 Initializr의 현재 기본 안정 버전과 같은 project를 재현할 수 있다.
- Java 21을 유지하면서 Spring Framework 7과 현재 Spring Boot 문서를 기준으로 학습할 수 있다.
- 기존 application source가 없어 3.x에서 4.x로 옮기는 migration 작업이 발생하지 않는다.
- 필요한 dependency만 마일스톤별로 추가해 외부 service 연결 실패 범위를 좁힐 수 있다.

### Negative

- Spring Boot 3.x 예제는 package, starter 또는 기본 동작이 달라 그대로 적용하지 못할 수 있다.
- Spring Boot 4의 modular starter와 기능별 test starter 이름을 구분해야 한다.
- 이후 라이브러리를 추가할 때 Spring Boot 4.1과의 호환 범위를 확인해야 한다.

## Guardrails

- M2에서는 Spring Web, Validation과 Actuator만 사용한다.
- Spring Data JPA·PostgreSQL은 M3, Kafka는 M4, MongoDB는 M5에서 도입한다.
- snapshot version과 임의의 Spring dependency version override를 사용하지 않는다.
- Spring Boot version을 다시 변경할 때는 공식 지원 범위, 실제 build와 test 결과를 확인하고 ADR을 갱신한다.

## References

- [Spring Boot 4.1.0 System Requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [Spring Initializr Reference Guide](https://docs.spring.io/initializr/docs/current/reference/html/)

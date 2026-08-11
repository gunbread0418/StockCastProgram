# Project Status

## 문서 메타데이터

- 마지막 업데이트: 2026-08-11
- 최근 완료 마일스톤: M1. Docker Compose 인프라
- 현재 마일스톤: M2. Spring Boot 기본 구성
- 상태: IN_PROGRESS
- 다음 마일스톤: M3. PostgreSQL과 Stock 도메인

## 현재 요약

M0의 프로젝트 목적, 작업 규칙, 상태·로드맵·아키텍처·ADR 문서, `.gitignore`,
`.env.example`과 최상위 디렉터리 정책을 작성하고 검증했다. `backend-spring/`,
`prediction-service/`, `infra/`, `scripts/`, `docs/`는 각 README로 책임과 제외 범위를
문서화해 Git에서 추적한다. 인프라는 M1에서 전체 구성을 완료했고 Spring application source는
M2에서 공식 Initializr project로 생성을 시작했다.

M1의 Docker 실행 환경 사전검증에 이어 PostgreSQL, Redis, MongoDB, Kafka와 Kafka UI의 Compose
구성과 단일 service 검증을 완료했다.
`infra/compose.yaml`에 `postgres:18.4-bookworm`, loopback host port, 필수 환경변수, PostgreSQL 18
경로의 named volume과 `pg_isready` healthcheck를 작성했다. Compose 설정 해석, `healthy` 전환,
비밀번호를 사용한 TCP 접속, database·사용자·UTC와 container 재생성 후 data 보존을 검증했다.
Redis는 `redis:8.2.8-bookworm`, 비밀번호 인증, loopback host port와 인증된 `PING` healthcheck를
사용한다. 기준 저장소가 아닌 재생성 가능한 cache라는 책임에 맞춰 RDB와 AOF를 끄고 `/data`를
`tmpfs`로 구성했다. 미인증 요청 거부, host 공개 port를 통한 인증된 `PING`, `healthy` 전환과
재시작 후 검증 key 소멸을 확인했다. MongoDB는 `mongo:8.0.28-noble`, 초기 root 관리자 인증,
loopback host port, `/data/db` named volume, `/data/configdb` tmpfs와 인증된 `ping` healthcheck를
사용한다. 미인증 관리 명령 거부, 내부와 host 공개 port의 인증, 최종 `mongod` PID 1, container
재생성 후 검증 document 보존과 정리를 확인했다. Kafka는 `apache/kafka:4.3.1` 단일 KRaft combined
node, 내부·host listener, 단일 node용 내부 topic 설정, `/var/lib/kafka/data` named volume과 Kafka
Admin 요청 healthcheck를 사용한다. 실제 version·cluster ID·실행 사용자와 server 설정, 내부
`kafka:19092`와 host `localhost:29092`의 Kafka protocol, 임시 topic produce/consume, container
재생성 후 record 보존과 검증 topic 정리를 확인했다. Kafka UI는 `kafbat/kafka-ui:v1.5.0`, loopback
host port, 조회 전용 cluster 설정, Kafka `service_healthy` 의존 조건과 HTTP healthcheck를 사용한다.
Kafka와 Kafka UI의 `healthy`, host HTTP `UP`, `ONLINE` cluster, `kafka:19092` broker와 내부 topic
조회, read-only 적용과 오류 log 부재를 확인했다.

마지막으로 다섯 service를 `--force-recreate --wait`로 한 번에 시작했다. PostgreSQL, Redis,
MongoDB와 Kafka는 약 17.2초, Kafka UI는 Kafka의 `healthy` 전환 뒤 약 29.3초에 `healthy`가 됐고
명령은 종료 코드 `0`으로 끝났다. `docker compose ps`에서 다섯 service의 `healthy`와 loopback
공개 port를 다시 확인했다. 전체 service가 실행 중인 상태에서 PostgreSQL `SELECT 1`, Redis
`PONG`, MongoDB `MONGO_OK`, host Kafka listener의 `__consumer_offsets` 조회와 Kafka UI의
`stockcast-local ONLINE`을 확인했다. 이 결과로 M1의 전체 Compose 재현과 핵심 접속 경로 검증을
완료했다.

M2의 첫 TODO인 Java 21 JDK 실행 환경 사전검증을 완료했다. 처음에는 Oracle JDK 17만 설치돼 있었고
`PATH`와 `JAVA_HOME`도 version 17을 선택했다. WinGet source를 수동 갱신한 뒤 정확한
`EclipseAdoptium.Temurin.21.JDK` package를 조회해 Temurin `21.0.12.8`을 기존 Oracle JDK 17과 함께
설치했다. system `JAVA_HOME`과 `PATH`는 Temurin 21을 우선하도록 설정됐다. 설치 전에 열려 있던
PowerShell은 이전 환경 변수를 유지해 계속 version 17을 출력했지만, system 값을 다시 읽은 process에서는
첫 `java`와 `javac`가 Temurin 경로를 가리키고 각각 `openjdk version "21.0.12"`, `javac 21.0.12`,
종료 코드 `0`을 출력했다. 컴퓨터 재시작은 필요하지 않으며 기존 terminal application을 다시 열면 된다.
OCI compute와 Oracle JDK는 별도 선택이므로 나중에 OCI에 배포하더라도 Java 21과 호환되는 Temurin,
OpenJDK 또는 Oracle JDK runtime을 선택할 수 있다.

기존 Spring Boot 3.x 기준은 현재 공식 Initializr가 4.0.0 이상만 생성하고 3.5.16이 마지막 3.5.x OSS
release라는 점을 확인한 뒤 사용자의 결정으로 Spring Boot 4.1.0으로 변경했다. 공식 Initializr에서
Gradle Kotlin DSL, Java 21, `com.stockcast`, executable JAR, Properties와 Spring Web·Validation·Actuator를
선택해 `backend-spring/` 기본 project를 생성했다. 생성 결과에는 Spring Boot 4.1.0 plugin, Gradle
Wrapper 9.5.1, Java main source와 context test가 포함됐다. `.\gradlew.bat test --no-daemon`은 종료 코드
`0`으로 성공했고, 실제 `bootRun`에서 Java 21.0.12와 Spring Boot 4.1.0을 확인했다.
`/actuator/health`는 HTTP 200과 `UP`을 반환했으며 검증 후 process와 8080 listener를 종료했다.

구현 중 또는 별도 개념 채팅에서 질문한 내용은 `docs/learning/`의 다섯 넓은 category에
중복 없이 축적한다. 첫 기록으로 Docker Compose의 host port, container port와 service
network 경계를 문서화했다.

## 완료된 작업

- 프로젝트 목적, 학습 목표와 비목표 정리
- 전체 이벤트 기반 아키텍처와 저장소 책임 수립
- 기본, 확장, 고도화 마일스톤 구성
- 새 채팅용 맥락 로딩과 상태 동기화 규칙 정의
- 사용자 보고 오류의 원인, 해결, 검증과 재발 방지 기록 규칙 정의
- Gradle Kotlin DSL, `com.stockcast`, fake 종목 3개, 1초 tick과 다음 5분 방향 분류 확정
- secret, build output, cache, IDE와 로컬 metadata를 구분한 `.gitignore` 작성·검증
- 공개 가능한 로컬 서비스 설정을 제공하는 `.env.example` 작성·검증
- Git 기본 branch를 `main`으로 변경하고 GitHub `origin/main` 연결
- 검증된 atomic change마다 관련 파일을 commit하고 현재 branch를 push하는 규칙 적용
- Spring, Prediction, Infrastructure, Scripts와 Docs의 최상위 책임 경계 문서화
- monorepo와 최상위 디렉터리 선택 근거를 ADR-0002로 기록
- Docker CLI·Compose 설치, Linux Engine daemon 연결과 기본 container 실행 환경 검증
- 인프라, 백엔드, 데이터 파이프라인, 데이터 저장소와 ML의 개념 학습 문서 체계 구성
- PostgreSQL 단일 Compose service의 image, port, environment, named volume과 healthcheck 구현
- PostgreSQL 실제 접속, UTC와 container 재생성 후 named volume data 보존 검증
- Redis 단일 Compose service의 image, port, 인증 command, tmpfs와 healthcheck 구현
- Redis 미인증 요청 거부, 인증된 PING과 재시작 후 cache data 소멸 검증
- MongoDB 단일 Compose service의 image, port, 초기 관리자, named volume, tmpfs와 healthcheck 구현
- MongoDB 미인증 관리 명령 거부, 인증된 내부·host 접속과 container 재생성 후 data 보존 검증
- Kafka 단일 Compose service의 KRaft combined mode, 내부·host listener, named volume과 healthcheck 구현
- Kafka 내부·host protocol, 임시 topic produce/consume과 container 재생성 후 record 보존 검증
- Kafka UI 단일 Compose service의 내부 Kafka 연결, 조회 전용 설정, 의존 조건과 healthcheck 구현
- Kafka UI host HTTP, `ONLINE` cluster, broker·topic 조회와 오류 log 부재 검증
- 다섯 service의 강제 재생성, 동시 `healthy` 전환과 loopback 공개 port 검증
- 전체 service 실행 중 PostgreSQL·Redis·MongoDB 인증 접속, Kafka host listener와 Kafka UI cluster 조회 검증
- M2 Java 21 JDK 사전검증 기준과 실패 조건 문서화
- Eclipse Temurin 21 JDK 설치, system `JAVA_HOME`·`PATH`와 `java`·`javac` version 21 검증
- Spring Boot 기준을 3.x에서 4.1.0으로 변경하고 ADR-0003에 근거 기록
- 공식 Initializr의 Gradle Kotlin DSL·Java 21 기본 project와 Gradle Wrapper 9.5.1 생성
- Spring Boot 4.1.0 기본 context test와 Gradle build 성공 검증
- Java 21.0.12 runtime, embedded Tomcat과 Actuator health HTTP 200·`UP` 검증
- 읽기 전용 확인은 사용자가 정상 기준과 비교하고, 문서·commit은 의미 있는 작업 단위로 묶는 규칙 반영

## 아직 구현되지 않은 항목

- M2 test profile과 공통 오류·invalid request 계약
- FastAPI source와 Python package
- DB schema와 migration
- Kafka application topic과 producer/consumer
- domain·통합·계약 테스트
- monitoring과 장애 실험

## 확정된 초기 기준

- Build tool: Gradle Kotlin DSL
- Spring Boot: `4.1.0`
- Gradle Wrapper: `9.5.1`
- Java base package: `com.stockcast`
- 초기 market data: `FAKE` exchange, 종목 3개, 1초 tick
- 첫 ML target: 다음 5분 방향 분류
- DB와 event 시간 기준: UTC
- 저장소 구조: Java, Python, Infra, Scripts와 Docs를 분리한 monorepo

## 저장소 상태

- Git 저장소의 기본 branch는 `main`이고 `origin/main`을 upstream으로 추적한다.
- Git remote `origin`은 `https://github.com/gunbread0418/StockCastProgram.git`이다.
- 첫 commit은 `5364609` (`chore: initialize project foundation`)이다.
- `.env.example` commit은 `aaeff71` (`chore: add local environment template`)이다.
- Git 작성자 이메일의 GitHub 계정 연결을 사용자가 확인했다.
- `infra/compose.yaml`에는 PostgreSQL, Redis, MongoDB, Kafka와 Kafka UI service가 구현되어 있다.
- M1의 다섯 service 동시 기동과 핵심 접속 경로 검증이 완료됐다.
- Spring Boot 기본 source와 Gradle build는 생성됐고 FastAPI 애플리케이션 source는 아직 없다.
- 최상위 component 디렉터리는 빈 디렉터리용 `.gitkeep`이 아니라 책임 README로 추적한다.

## 최상위 디렉터리 정책

| 디렉터리 | 책임 | 실제 구현 시작 |
|---|---|---|
| `backend-spring/` | Spring API, Java 도메인, Kafka와 저장소 연동 | M2 |
| `prediction-service/` | FastAPI, Python feature, 학습·추론과 worker | M9 |
| `infra/` | Compose와 외부 의존 service의 로컬 실행 설정 | M1 |
| `scripts/` | component를 가로지르는 반복 실행·검증 자동화 | 실제 반복 작업 발생 시 |
| `docs/` | 상태, 설계 결정, 오류와 작업 근거 | 모든 마일스톤 |

## 주요 변경 파일

- `backend-spring/README.md`: Java backend의 포함·제외 범위와 M2·M4 도입 시점
- `prediction-service/README.md`: Python ML service의 포함·제외 범위와 M9·M10·M12 도입 시점
- `infra/README.md`: Compose, 초기화, healthcheck와 로컬 데이터 제외 정책
- `scripts/README.md`: 저장소 공통 자동화와 service별 명령의 경계
- `docs/README.md`: 프로젝트 문서 책임과 문서 지도
- `docs/adr/0002-monorepo-directory-boundaries.md`: monorepo 구조의 대안과 결과
- `README.md`: 계획 구조를 실제 저장소 구조와 component 문서 링크로 갱신
- `docs/PROJECT_CONTEXT.md`: UTC 저장 기준을 사용자 확정 상태로 동기화
- `docs/architecture/SYSTEM_OVERVIEW.md`: 첫 ML 목표의 낡은 `후보` 표현 제거
- `docs/ROADMAP.md`: M0 상태를 `DONE`으로 전환
- `docs/PROJECT_STATUS.md`: M1 환경 사전검증 결과와 다음 단일 작업 기록
- `docs/ROADMAP.md`: M1 상태를 `IN_PROGRESS`로 전환
- `docs/learning/README.md`: 다섯 개념 category와 질문 기록·중복 방지 규칙
- `docs/learning/infrastructure.md`: Docker Compose `ports` 개념과 프로젝트 적용 기록
- `AGENTS.md`: 학습 문서 갱신과 의미 있는 TODO·읽기 전용 자체 확인·commit 묶음 규칙
- `infra/compose.yaml`: PostgreSQL·MongoDB·Kafka 영구 저장 service, Redis 재생성 가능 cache와 Kafka UI의 실행 설정
- `docs/learning/infrastructure.md`: PostgreSQL environment, volume, healthcheck의 실제 검증 결과
- `docs/learning/infrastructure.md`: Redis 인증, healthcheck와 비영구 cache 정책의 실제 검증 결과
- `docs/learning/infrastructure.md`: MongoDB 초기화, 영구 저장, 인증 healthcheck와 실제 검증 결과
- `docs/learning/infrastructure.md`: Kafka KRaft, listener, 영구 저장, healthcheck와 실제 검증 결과
- `docs/learning/infrastructure.md`: Kafka UI 내부 연결, 의존 조건, HTTP healthcheck와 실제 검증 결과
- `docs/TROUBLESHOOTING.md`: PowerShell에서 `psql` SQL이 잘린 `ERR-004`의 해결과 검증
- `docs/learning/infrastructure.md`: 전체 Compose 동시 기동과 읽기 전용 핵심 접속 검증 결과
- `docs/TROUBLESHOOTING.md`: `Invoke-RestMethod`의 JSON 배열을 펼치지 않아 빈 표가 나온 `ERR-005`
- `docs/ROADMAP.md`: M1 상태를 `DONE`으로 전환
- `backend-spring/build.gradle.kts`: Spring Boot 4.1.0, Java 21과 M2 최소 dependency 설정
- `backend-spring/gradle/wrapper/`: 재현 가능한 Gradle 9.5.1 Wrapper
- `backend-spring/src/`: 기본 application class, Properties 설정 파일과 context test
- `docs/adr/0003-spring-boot-4-baseline.md`: 3.x에서 4.1.0으로 변경한 선택 근거와 영향
- `docs/PROJECT_CONTEXT.md`: Java 21·Spring Boot 4.1.0 백엔드 기준으로 갱신
- `docs/ROADMAP.md`: M2 기본 version을 Spring Boot 4.1.0으로 갱신

## 추후 `.gitignore` 점검 시점

| 시점 | 확인할 항목 | 지금 미리 넣지 않은 이유 |
|---|---|---|
| M10 model pipeline | dataset, model artifact, `*.pkl`, `*.joblib` 정책 | 재현성과 artifact 보존 위치를 먼저 결정해야 함 |
| M13 선택 UI | `node_modules/`와 선택한 frontend tool cache | 현재 Node 기반 UI가 존재하지 않음 |
| 모든 마일스톤 | 새 build tool의 cache, log와 generated output | 실제 경로를 확인한 뒤 좁은 규칙으로 추가해야 함 |

## M0 완료 검증

- 프로젝트 목적과 비목표가 문서화됨
- 새 채팅에서 읽을 루트 지침과 현재 상태 문서가 존재함
- 전체 마일스톤, 시스템 경계와 구조 결정이 문서화됨
- `.gitignore`가 실제 secret을 제외하고 `.env.example`을 추적함
- 최상위 component 책임과 Git 추적 정책이 README와 ADR에 기록됨
- 문서 내부 링크, 필수 파일, 민감정보와 Git 상태 검증을 통과함

## M1 완료 검증

- PostgreSQL, Redis, MongoDB, Kafka와 Kafka UI가 Compose로 실행됨
- 각 service healthcheck와 의존 관계가 정의됨
- 필요한 데이터가 재시작 후 보존됨
- DB·Redis·Mongo 접속과 Kafka produce/consume 검증이 성공함
- secret 없이 다른 개발 환경에서 로컬 인프라를 재현할 수 있음
- repository 내부 bind mount data가 없어 현재 `.gitignore` 규칙으로 충분함

## 검증 기록

| 날짜 | 검증 | 결과 |
|---|---|---|
| 2026-07-14 | 기존 workspace와 Git 상태 확인 | 초기 commit이 없는 `master` branch 확인 |
| 2026-07-14 | 지속 작업 문서 생성 | 상대 링크, 필수 내용, UTF-8 검증 통과 |
| 2026-07-20 | `.gitignore` 환경 파일 규칙 | `.env` ignored `True`, `.env.example` ignored `False` |
| 2026-07-20 | staged diff와 민감정보 검사 | whitespace 수정 후 통과, secret-like assigned value 없음 |
| 2026-07-20 | 첫 commit과 push | `5364609` 생성, `main -> origin/main`, upstream 설정 성공 |
| 2026-07-21 | `.env.example` 구조·보안 | 변수 14개, 형식·중복·누락·placeholder·포트 오류 0건 |
| 2026-07-21 | `.env.example` commit과 push | `aaeff71` 생성, `main -> origin/main` push 성공 |
| 2026-07-24 | 최상위 README 구조 | 필수 파일 5개, 빈 파일·필수 heading 누락·ignore 대상 0건 |
| 2026-07-24 | M0 필수 파일과 환경 정책 | 누락 0건, 실제 `.env` 없음, `.env` ignored, 예시는 추적 대상 |
| 2026-07-24 | Markdown 내부 링크와 민감정보 패턴 | 깨진 링크 0건, secret-like pattern 0건 |
| 2026-07-24 | 확정 기술 기준 문서 일치 | UTC와 다음 5분 방향 분류를 확정 상태로 동기화 |
| 2026-07-24 | staged whitespace 재검증 | README 5개의 EOF 빈 줄 제거 후 `diff --cached --check` 통과 |
| 2026-08-03 | Docker CLI와 Compose 설치·version | Docker CLI `29.6.2`, Docker Compose `v5.3.1` 확인 |
| 2026-08-03 | Docker Engine daemon과 Linux mode | Docker Desktop `4.84.0`, Engine `29.6.2`, `linux/x86_64`, `overlayfs` 확인 |
| 2026-08-03 | 기본 Engine·Compose 명령 | `docker ps`, `docker compose ls` 종료 코드 0; 실행 중 container와 Compose project 0개 |
| 2026-08-03 | `hello-world` end-to-end 실행 | Linux `amd64` image 확인, 사용자 실행 종료 코드 0, `--rm` 이후 잔여 container 0개 |
| 2026-08-04 | 개념 학습 문서 구조 | 넓은 category 5개, 문서 지도, 중복 방지와 개념 전용 채팅 규칙 구성 |
| 2026-08-04 | 첫 인프라 개념 기록 | Compose host/container port, service DNS, loopback binding과 readiness 구분 문서화 |
| 2026-08-07 | PostgreSQL Compose 설정 | `.env.example`과 local `.env` 모두 `docker compose config --quiet` 종료 코드 0 |
| 2026-08-07 | PostgreSQL 실행과 healthcheck | `up -d --wait` 성공, `postgres:18.4-bookworm` container가 `healthy`, `127.0.0.1:5432` 확인 |
| 2026-08-07 | PostgreSQL 실제 접속 | 비밀번호를 사용한 TCP 접속 성공, database·사용자 `stockcast`, `TimeZone=UTC` 확인 |
| 2026-08-07 | PostgreSQL data 지속성 | 표식 행 생성 후 `down`·재생성, 같은 행 조회 성공, 테스트 table 삭제 확인 |
| 2026-08-07 | local secret과 저장 경로 | `.env`는 Git ignore, `.env.example`은 추적 대상, named volume은 `local` driver로 확인 |
| 2026-08-10 | Redis Compose 설정 | `.env.example`과 local `.env` 모두 `docker compose config --quiet` 종료 코드 0, image·port·command·tmpfs·healthcheck 해석 확인 |
| 2026-08-10 | Redis 실행과 인증 | `up -d --wait` 성공, `healthy`, 미인증 `PING`은 `NOAUTH`, 내부와 host 공개 port의 인증된 `PING`은 `PONG` 확인 |
| 2026-08-10 | Redis 비영구 cache 정책 | `save` 빈 값, `appendonly=no`, `/data` tmpfs, Redis named volume 없음과 재시작 후 검증 key 소멸 확인 |
| 2026-08-10 | MongoDB Compose 설정 | `.env.example`과 local `.env` 모두 `docker compose config --quiet` 종료 코드 0, `mongo:8.0.28-noble`, port·환경변수·mount·healthcheck 해석 확인 |
| 2026-08-10 | MongoDB 실행과 인증 | `up -d --wait` 성공, `healthy`, 미인증 `listDatabases` 거부, 내부와 host 공개 port의 인증된 `ping=1`, PID 1 `mongod` 확인 |
| 2026-08-10 | MongoDB data 지속성 | `/data/db` local named volume과 `/data/configdb` tmpfs 확인, container ID 변경 후 검증 document 재조회 성공, 검증 database 삭제 확인 |
| 2026-08-10 | Kafka Compose 설정 | `.env.example`과 local `.env` 모두 `docker compose config --quiet` 종료 코드 0, `apache/kafka:4.3.1`, listener·KRaft·내부 topic·mount·healthcheck 해석 확인 |
| 2026-08-10 | Kafka 실행과 listener | `up -d --wait` 성공, `healthy`, server `4.3.1`, cluster ID·node ID·`appuser`, 내부 `kafka:19092`와 host `localhost:29092` Kafka protocol 확인 |
| 2026-08-10 | Kafka produce/consume과 data 지속성 | 임시 topic에서 marker 왕복 성공, container ID 변경 뒤 같은 record 재조회, `stockcast_kafka-data` 유지, 검증 topic 삭제 확인 |
| 2026-08-10 | Kafka 최종 상태 | 재생성 뒤 healthcheck 종료 코드 0, 실제 `server.properties`·`meta.properties` 일치, 최근 log 200줄의 `ERROR`·`FATAL` 0건 확인 |
| 2026-08-10 | Kafka UI Compose 설정 | `.env.example`과 local `.env` 모두 `docker compose config --quiet` 종료 코드 0, image·port·cluster·read-only·의존 조건·healthcheck 해석 확인 |
| 2026-08-10 | Kafka UI 실행과 HTTP healthcheck | `up -d --wait` 성공, Kafka와 Kafka UI `healthy`, 재시작 0회, host `/actuator/health=UP`, root HTTP 200과 build `v1.5.0` 확인 |
| 2026-08-10 | Kafka UI cluster 조회 | API에서 `stockcast-local=ONLINE`, KRaft broker `kafka:19092`, 내부 topic `__consumer_offsets`, read-only 적용, under-replicated partition과 최근 오류 log 0건 확인 |
| 2026-08-10 | 전체 Compose 동시 기동 | `--force-recreate --wait --wait-timeout 240` 종료 코드 0, 다섯 service `healthy`; 네 core service 약 17.2초, Kafka UI 약 29.3초 확인 |
| 2026-08-10 | 전체 Compose 상태와 공개 port | 다섯 service가 계속 `healthy`; `127.0.0.1`의 PostgreSQL 5432, Redis 6379, MongoDB 27017, Kafka 29092와 Kafka UI 8088 mapping 확인 |
| 2026-08-10 | 전체 핵심 접속 경로 | PostgreSQL `SELECT 1`, Redis `PONG`, MongoDB `MONGO_OK`, Kafka `__consumer_offsets`와 Kafka UI `stockcast-local ONLINE` 확인 |
| 2026-08-10 | M1 `.gitignore` 최종 점검 | Compose service에 repository 내부 bind mount가 없고 named volume·tmpfs만 사용하므로 추가 ignore 규칙이 필요하지 않음 |
| 2026-08-11 | Java 21 JDK 사전검증 | `java`와 `javac`는 종료 코드 `0`이지만 모두 version 17로 확인되어 Java 21 기준 미통과 |
| 2026-08-11 | JDK 실행 경로 진단 | `PATH`의 첫 `java`·`javac`와 `JAVA_HOME`이 모두 `C:\Program Files\Java\jdk-17`을 가리킴 |
| 2026-08-11 | Oracle JDK 설치 디렉터리 조회 | `C:\Program Files\Java` 조회 성공, 하위 디렉터리는 `jdk-17` 하나로 Java 21 없음 |
| 2026-08-11 | WinGet client와 Temurin package 조회 | client `v1.29.280`·종료 코드 `0`; source 업데이트 실패 후 package 미발견·종료 코드 `-1978335212` |
| 2026-08-11 | WinGet source 등록 상태 | `msstore`, `winget`, `winget-font`와 기본 주소 확인; 전달된 출력에는 종료 코드 문자열 없음 |
| 2026-08-11 | WinGet source 수동 갱신 | `winget` source 진행률 `100%`, `완료`, 종료 코드 `0` 확인 |
| 2026-08-11 | Temurin 21 package 재조회 | ID `EclipseAdoptium.Temurin.21.JDK`, version `21.0.12.8`, publisher, Windows x64 WiX MSI, 종료 코드 `0` 확인 |
| 2026-08-11 | Temurin 21 설치와 system 환경 | 설치 디렉터리 확인, system `JAVA_HOME`과 `PATH`에서 Temurin 21 우선 설정 확인 |
| 2026-08-11 | Java 21 실행 환경 | 새 system 환경을 읽은 process에서 Temurin `java 21.0.12`, `javac 21.0.12`, 두 종료 코드 `0` 확인 |
| 2026-08-11 | Spring Initializr 4.1.0 project 생성 | Gradle Kotlin DSL, Java 21, `com.stockcast`, Web MVC·Validation·Actuator와 Gradle Wrapper 9.5.1 확인 |
| 2026-08-11 | Spring Boot 기본 context test | `.\gradlew.bat test --no-daemon` 종료 코드 `0`, compile과 `contextLoads()` 성공 |
| 2026-08-11 | Spring Boot 기본 runtime과 health | Java 21.0.12, Spring Boot 4.1.0, Tomcat 8080 실행; `/actuator/health` HTTP 200·`UP`, 종료 후 listener 0개 확인 |
| 2026-08-11 | 작업 단위와 확인 보고 규칙 | 읽기 전용 확인은 정상 기준으로 자체 판정하고 설치·구현·오류 해결 단위에서 문서와 commit을 한 번에 처리하도록 `AGENTS.md` 갱신 |

## 알려진 실패와 blocker

- M1 기능 blocker는 없음
- M2 Java 21 JDK 사전검증 blocker는 Temurin 21 설치와 새 system 환경의 실행 검증으로 해결했으며
  `ERR-006`을 `RESOLVED`로 기록함
- 최초 Temurin package 조회의 source 갱신 실패는 수동 갱신과 정확한 package 재조회로 해결했으며
  `ERR-007`을 `RESOLVED`로 기록함
- 기존 Spring Boot 3.x 기준과 현재 Initializr의 4.0.0 이상 생성 범위 충돌은 사용자의 Spring Boot
  4.1.0 기준 변경으로 해결했으며 ADR-0003에 기록함
- 일반 `git status`는 샌드박스 사용자와 저장소 소유권 차이로 `dubious ownership` 오류 발생
- 전역 Git 설정 대신 명령별 `safe.directory` 옵션으로 Git 명령을 실행함
- 저장소 밖에서 실행한 `git check-ignore`와 파일명 오기입으로 인한 `.env.example` 검증 실패는
  해결했으며 자세한 과정은 `docs/TROUBLESHOOTING.md`에 기록함
- README 5개의 EOF 빈 줄 때문에 staged diff 검증이 처음 실패했으나 빈 줄 제거 후 통과했으며
  `ERR-003`으로 기록함
- PowerShell, `compose exec`와 `sh -c`의 중첩 따옴표 때문에 `psql -c` SQL이 잘린 오류는
  표준 입력과 `exec -T` 방식으로 해결했으며 `ERR-004`로 기록함
- 같은 중첩 따옴표 경계에서 `mongosh --eval` JavaScript가 `const`까지만 전달된 오류는
  표준 입력과 `mongosh --file /dev/stdin`으로 해결하고 `ERR-004`에 추가 증거를 기록함
- `Invoke-RestMethod`의 JSON 배열을 `Select-Object`에 직접 전달해 빈 표가 나온 문제는 배열 항목을
  `Write-Output`으로 펼쳐 `stockcast-local ONLINE`을 확인하고 `ERR-005`로 기록함
- 샌드박스의 `.git` 쓰기와 네트워크 제한은 승인된 권한으로 commit과 GitHub 접근을 수행함

## 다음 작업

생성된 Spring Boot 4.1.0 project의 build 구조와 auto-configuration 경계를 확인한 뒤 M2 test profile을
추가한다. auto-configuration은 classpath와 설정을 기준으로 Spring이 필요한 구성을 자동으로 등록하는
기능이다.

## 다음 채팅 시작 문장

`StockCastProgram M2의 Spring Boot 4.1.0 기본 project 생성과 context·health 검증을 완료했어. 생성된 build 구조를 설명하고 test profile 추가를 한 단계씩 안내해줘.`

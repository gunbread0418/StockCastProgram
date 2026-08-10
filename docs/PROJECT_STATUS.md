# Project Status

## 문서 메타데이터

- 마지막 업데이트: 2026-08-10
- 최근 완료 마일스톤: M0. Repository 초기화와 설계 기준
- 현재 마일스톤: M1. Docker Compose 인프라
- 상태: IN_PROGRESS
- 다음 마일스톤: M2. Spring Boot 기본 구성

## 현재 요약

M0의 프로젝트 목적, 작업 규칙, 상태·로드맵·아키텍처·ADR 문서, `.gitignore`,
`.env.example`과 최상위 디렉터리 정책을 작성하고 검증했다. `backend-spring/`,
`prediction-service/`, `infra/`, `scripts/`, `docs/`는 각 README로 책임과 제외 범위를
문서화해 Git에서 추적한다. 애플리케이션 source는 아직 없으며 인프라는 PostgreSQL Compose
service부터 구현을 시작했다.

M1의 Docker 실행 환경 사전검증에 이어 PostgreSQL, Redis와 MongoDB 단일 service의 Compose 구성을 완료했다.
`infra/compose.yaml`에 `postgres:18.4-bookworm`, loopback host port, 필수 환경변수, PostgreSQL 18
경로의 named volume과 `pg_isready` healthcheck를 작성했다. Compose 설정 해석, `healthy` 전환,
비밀번호를 사용한 TCP 접속, database·사용자·UTC와 container 재생성 후 data 보존을 검증했다.
Redis는 `redis:8.2.8-bookworm`, 비밀번호 인증, loopback host port와 인증된 `PING` healthcheck를
사용한다. 기준 저장소가 아닌 재생성 가능한 cache라는 책임에 맞춰 RDB와 AOF를 끄고 `/data`를
`tmpfs`로 구성했다. 미인증 요청 거부, host 공개 port를 통한 인증된 `PING`, `healthy` 전환과
재시작 후 검증 key 소멸을 확인했다. MongoDB는 `mongo:8.0.28-noble`, 초기 root 관리자 인증,
loopback host port, `/data/db` named volume, `/data/configdb` tmpfs와 인증된 `ping` healthcheck를
사용한다. 미인증 관리 명령 거부, 내부와 host 공개 port의 인증, 최종 `mongod` PID 1, container
재생성 후 검증 document 보존과 정리를 확인했다. 다음 작업은 Kafka 단일 service의 Compose 구성이다.

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

## 아직 구현되지 않은 항목

- Kafka와 Kafka UI의 Compose service
- Spring Boot source와 Gradle build
- FastAPI source와 Python package
- DB schema와 migration
- Kafka topic과 producer/consumer
- domain·통합·계약 테스트
- monitoring과 장애 실험

## 확정된 초기 기준

- Build tool: Gradle Kotlin DSL
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
- `infra/compose.yaml`에는 PostgreSQL, Redis와 MongoDB service가 구현되어 있다.
- Spring Boot와 FastAPI 애플리케이션 source는 아직 없다.
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
- `AGENTS.md`: 구현·개념 전용 채팅의 지속적인 학습 문서 갱신 규칙
- `infra/compose.yaml`: PostgreSQL·MongoDB 영구 저장 service와 Redis 재생성 가능 cache service의 실행 설정
- `docs/learning/infrastructure.md`: PostgreSQL environment, volume, healthcheck의 실제 검증 결과
- `docs/learning/infrastructure.md`: Redis 인증, healthcheck와 비영구 cache 정책의 실제 검증 결과
- `docs/learning/infrastructure.md`: MongoDB 초기화, 영구 저장, 인증 healthcheck와 실제 검증 결과
- `docs/TROUBLESHOOTING.md`: PowerShell에서 `psql` SQL이 잘린 `ERR-004`의 해결과 검증

## 추후 `.gitignore` 점검 시점

| 시점 | 확인할 항목 | 지금 미리 넣지 않은 이유 |
|---|---|---|
| M1의 남은 Compose service | repository 내부 bind mount data 경로 | PostgreSQL·MongoDB는 named volume, Redis는 tmpfs를 사용해 제외할 로컬 경로가 없으며 새 service 추가 때 다시 확인 |
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

## 현재 마일스톤 완료 기준

- PostgreSQL, Redis, MongoDB, Kafka와 Kafka UI가 Compose로 실행됨
- 각 service healthcheck와 의존 관계가 정의됨
- 필요한 데이터가 재시작 후 보존됨
- DB·Redis·Mongo 접속과 Kafka produce/consume 검증이 성공함
- secret 없이 다른 개발 환경에서 로컬 인프라를 재현할 수 있음

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

## 알려진 실패와 blocker

- 기능 blocker 없음
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
- 샌드박스의 `.git` 쓰기와 네트워크 제한은 승인된 권한으로 commit과 GitHub 접근을 수행함

## 다음 작업

M1의 다음 단일 TODO로 Kafka 단일 service의 Compose 구성을 학습하고 작성한다.

## 다음 채팅 시작 문장

`M1의 PostgreSQL, Redis와 MongoDB 단일 Compose service 검증을 완료했어. 다음 TODO로 Kafka 단일 service의 Compose 구성을 내가 직접 작성할 수 있게 설명해줘.`

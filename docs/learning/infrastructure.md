# Infrastructure Concepts

## 범위

Docker image와 container, Docker Compose, network, port, volume, healthcheck, readiness,
로컬 실행과 관측 개념을 기록한다.

## Docker Compose의 `ports`

- 기록일: 2026-08-04
- 질문: `${HOST_BIND_ADDRESS}:${POSTGRES_HOST_PORT}:5432`에서 각 port는 무엇이며,
  container 간 연결과 host 연결은 어떻게 다른가?

### 핵심 답변

`ports`는 container process가 수신하는 port를 Docker host의 IP와 port에 공개하는 규칙이다.
현재 PostgreSQL mapping은 다음 순서로 읽는다.

```text
127.0.0.1 : 5432 : 5432
host IP      host     container
             port     port
```

Windows의 `psql`, IDE 또는 host에서 실행하는 Spring Boot는 `127.0.0.1:5432`로 접속한다.
Docker는 이 연결을 PostgreSQL container의 `5432`로 전달한다. 왼쪽 host port와 오른쪽
container port는 같을 필요가 없으며 host port가 충돌하면 `15432:5432`처럼 왼쪽만 바꿀 수 있다.

### 프로젝트 설정

```yaml
ports:
  - "${HOST_BIND_ADDRESS}:${POSTGRES_HOST_PORT}:5432"
```

| 부분 | 현재 값 | 책임 |
|---|---|---|
| `HOST_BIND_ADDRESS` | `127.0.0.1` | host에서 연결을 받을 network interface |
| `POSTGRES_HOST_PORT` | `5432` | host client가 접속할 공개 port |
| 마지막 port | `5432` | PostgreSQL process가 container 내부에서 수신하는 port |

값은 `.env.example`에 공개 가능한 기본값으로 두고 실제 local override는 Git에서 제외된 `.env`에
둔다. Compose short syntax는 colon이 포함되므로 YAML의 다른 scalar 해석을 피하기 위해 전체를
따옴표로 감싼다.

### Host 연결과 service 연결

```text
Windows client
  -> 127.0.0.1:${POSTGRES_HOST_PORT}
  -> Docker published port
  -> postgres container:5432

다른 Compose service
  -> postgres:5432
  -> Compose default network와 service DNS
```

같은 Compose network의 service는 host port를 거치지 않는다. `postgres`라는 안정적인 service
이름과 container port `5432`를 사용한다. container 내부에서 `localhost`는 PostgreSQL이 아니라
그 container 자신을 뜻하므로 다른 service에 연결할 주소로 사용할 수 없다.

Compose는 별도 network 설정이 없어도 `<project>_default` network를 만들고 service 이름을 내부
DNS에 등록한다. 자세한 동작은 [Docker Compose networking](https://docs.docker.com/compose/how-tos/networking/)을
기준으로 한다.

### `127.0.0.1`을 명시하는 이유

host IP를 생략한 `5432:5432`는 기본적으로 모든 host interface에 publish될 수 있다. 현재 DB는
로컬 학습과 검증에만 필요하므로 loopback에 binding해 Docker host 밖의 접근 범위를 넓히지 않는다.
[Docker port publishing](https://docs.docker.com/engine/network/port-publishing/)은 localhost IP를
명시하면 published port를 Docker host에서만 접근할 수 있다고 설명한다.

### 구분할 개념

- `ports`: host 밖에서 container service로 들어오는 경로를 publish한다.
- `expose`: host에 publish하지 않고 container가 사용하는 내부 port를 설명한다.
- image의 `EXPOSE`: image 제작자가 예상 port를 metadata로 알리며 host mapping을 자동 생성하지 않는다.
- healthcheck: port mapping과 별개로 PostgreSQL이 실제 요청을 받을 준비가 됐는지 검사한다.

이 프로젝트는 M1에서 host의 도구로 PostgreSQL 접속을 검증해야 하므로 `ports`를 사용한다. 모든
application이 container 안에서만 실행되고 host 접속이 필요 없어지면 mapping 제거를 검토할 수 있다.

### 흔한 오해

- host port와 container port가 항상 같아야 한다: 서로 다른 network namespace의 값이므로 다를 수 있다.
- `ports`가 있으면 DB가 준비됐다: mapping 생성과 PostgreSQL initialization 완료는 다른 상태다.
- 다른 container도 `localhost:5432`를 사용한다: `localhost`는 호출한 container 자신이다.
- `docker compose config` 성공이 실제 port 점유까지 확인한다: config는 구조와 치환을 검증하며
  port 충돌은 container 시작 시 확인된다.

### 확인 방법

구조와 변수 치환은 저장소 root에서 다음 명령으로 확인한다.

```powershell
docker compose --env-file .\.env.example -f .\infra\compose.yaml config
```

container 시작 후 실제 mapping은 다음 명령으로 확인한다.

```powershell
docker compose --env-file .\.env -f .\infra\compose.yaml port postgres 5432
```

첫 명령의 성공 기준은 `host_ip: 127.0.0.1`, `published`, `target: 5432`와 종료 코드 0이다.
두 번째 명령은 실제 local `.env`를 만든 뒤에만 실행하며 출력에 비밀번호가 포함되지 않는다.

### 면접 질문

질문: Compose 안의 Spring container와 host에서 실행하는 Spring Boot는 각각 PostgreSQL에 어떤
주소로 접속해야 하는가?

답변 keyword: service DNS, `postgres:5432`, host loopback, published port, network namespace,
container IP 대신 service name.

## PostgreSQL Compose의 `environment`

- 기록일: 2026-08-07
- 질문: Compose의 `environment`에서 PostgreSQL 초기화 변수와 `TZ`는 어떤 역할을 하며,
  `${VAR:?message}` 형식은 왜 사용하는가?

### 핵심 답변

Compose의 `environment`는 host나 `--env-file`에서 치환한 값을 container process의 환경변수로
전달한다. 현재 설정은 값 자체를 Compose 파일에 고정하지 않고 다음 네 항목을 명시적으로 전달한다.

```yaml
environment:
  POSTGRES_DB: "${POSTGRES_DB:?POSTGRES_DB is required}"
  POSTGRES_USER: "${POSTGRES_USER:?POSTGRES_USER is required}"
  POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:?POSTGRES_PASSWORD is required}"
  TZ: "${TZ:?TZ is required}"
```

`${VAR:?message}`는 `VAR`가 설정되지 않았거나 빈 문자열이면 Compose가 설정 해석 단계에서
`message`와 함께 실패하게 한다. 잘못된 기본값으로 container를 시작하는 것보다 누락을 일찍
발견할 수 있다. `.env.example`은 변수 이름과 공개 가능한 예시만 제공하고 실제 local 값은 Git에서
제외된 `.env`에 둔다. 자세한 치환 형식은 [Docker Compose variable interpolation](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)을
기준으로 한다.

### 각 변수의 책임

| 변수 | 책임 | 생략하거나 잘못 설정했을 때 |
|---|---|---|
| `POSTGRES_DB` | 첫 초기화 때 만들 기본 database 이름 | 생략하면 `POSTGRES_USER`와 같은 이름의 database가 생성되며 application 접속 설정과 어긋날 수 있음 |
| `POSTGRES_USER` | 첫 초기화 때 만들 bootstrap superuser 이름 | 생략하면 `postgres`가 사용되며 local 학습용 계정 이름을 명시적으로 통제하지 못함 |
| `POSTGRES_PASSWORD` | 위 superuser의 초기 password | 공식 image에서 기본적으로 필수이며 누락·빈 값이면 초기화가 실패함 |
| `TZ` | `initdb`가 database cluster의 기본 `TimeZone`을 정할 때 사용할 시간대 | service마다 시간대가 달라져 log와 timestamp 해석이 혼동될 수 있음 |

`POSTGRES_USER`는 Linux의 `postgres` 운영체제 계정과 다른 PostgreSQL role이다. 또한 현재 M1의
단일 local service에서는 간단히 bootstrap superuser를 사용하지만, application 구현 단계에서는
최소 권한의 별도 role을 migration 또는 초기화 절차로 만드는 것이 더 안전하다.

### 초기화 변수의 수명 주기

`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`는 빈 data directory를 처음 초기화할 때만
database cluster에 반영된다. named volume에 기존 cluster가 있으면 Compose 값을 바꿔 container를
재시작해도 기존 database, role 또는 password가 자동 변경되지 않는다. 이 동작은 data를 보존하는
보호 장치이므로 설정 변경을 적용하려고 volume을 임의로 삭제해서는 안 된다. 변수별 공식 동작은
[PostgreSQL Docker Official Image](https://github.com/docker-library/docs/blob/master/postgres/README.md)의
environment variables 설명을 기준으로 한다.

`TZ=UTC`는 PostgreSQL 18의 `initdb`가 생성할 cluster의 기본 시간대를 UTC로 선택하게 한다. 이후
session별 `SET TIME ZONE`, database·role 설정 또는 `postgresql.conf`가 다른 값을 지정할 수 있으므로
실행 시 `SHOW TIME ZONE`으로 실제 값을 확인해야 한다. `timestamp with time zone`도 출력은 현재
session의 `TimeZone` 영향을 받으므로 `TZ` 한 줄만으로 애플리케이션 시간 규칙 전체가 보장되지는 않는다.
PostgreSQL 18의 [`initdb` 환경변수 문서](https://www.postgresql.org/docs/18/app-initdb.html)가 `TZ`를
새 database cluster의 기본 시간대로 정의한다.

### 대안과 현재 선택

- Compose mapping syntax: 변수별 책임이 눈에 보이고 필수 값 검사를 붙일 수 있어 현재 선택한다.
- service의 `env_file`: 여러 값을 한꺼번에 전달하기 편하지만 어떤 변수가 container 계약인지 Compose
  파일에서 덜 명확하고 필수 값 검사를 개별적으로 표현하기 어렵다.
- Docker secrets와 `POSTGRES_PASSWORD_FILE`: production에서 password 노출 범위를 줄이는 대안이다.
  현재는 local Compose 학습 단계라 `.env`를 사용하고 production 배포 경계가 생길 때 전환한다.
- Compose 파일에 실제 password 직접 작성: 간단하지만 Git 유출 위험이 있어 사용하지 않는다.

### 흔한 오해

- `.env`의 변수가 자동으로 container에 모두 전달된다: `.env`는 먼저 Compose 치환 입력이며,
  container에 넣으려면 `environment` 또는 `env_file` 연결이 필요하다.
- `POSTGRES_PASSWORD`는 application client의 `PGPASSWORD`와 같다: 전자는 첫 초기화의 server
  superuser password이며 후자는 `psql` 같은 client의 접속 기본값이다.
- 환경변수를 바꾸고 재시작하면 기존 database 설정도 바뀐다: 초기화가 끝난 named volume에는
  자동 반영되지 않는다.
- `docker compose config` 성공이면 database가 생성됐다: 이 명령은 치환된 Compose model만
  검증하며 image 실행, 초기화와 실제 접속은 확인하지 않는다.
- `config` 출력은 항상 공유해도 안전하다: 치환 후 password가 평문으로 보일 수 있으므로 실제
  `.env`를 사용한 출력은 채팅, 문서 또는 commit에 남기지 않는다.

### 확인 방법

공개 가능한 예시 값으로 구조와 치환을 확인한다.

```powershell
docker compose --env-file .\.env.example -f .\infra\compose.yaml config
```

성공 기준은 종료 코드 `0`이고 `postgres.environment` 아래에 네 변수가 빈 값 없이 나타나는 것이다.
실패 기준은 required variable 오류, YAML parse 오류 또는 예상하지 않은 값이다. 이 검증은 container를
시작하지 않으므로 실제 초기화 성공 여부는 named volume과 healthcheck 단계 이후 별도로 확인한다.

### 면접 질문

질문: PostgreSQL container의 `POSTGRES_PASSWORD`를 바꾼 뒤 재시작했는데 기존 password가 바뀌지
않는 이유는 무엇인가?

답변 keyword: entrypoint, empty data directory, first initialization, named volume, persisted cluster,
환경변수 재치환과 database 상태 변경의 구분, 명시적 role/password 변경 절차.

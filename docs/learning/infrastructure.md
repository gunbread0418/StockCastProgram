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
실패 기준은 required variable 오류, YAML parse 오류 또는 예상하지 않은 값이다.

실제 local `.env`로도 `docker compose config --quiet`가 종료 코드 `0`으로 끝났고, container를 처음
초기화한 뒤 비밀번호를 사용한 TCP 접속에 성공했다. SQL로 확인한 database와 사용자는 각각
`.env`의 `POSTGRES_DB`, `POSTGRES_USER`와 일치했고 `SHOW TIME ZONE`은 `UTC`였다. 실제 비밀번호는
명령 출력이나 문서에 남기지 않았다.

### 면접 질문

질문: PostgreSQL container의 `POSTGRES_PASSWORD`를 바꾼 뒤 재시작했는데 기존 password가 바뀌지
않는 이유는 무엇인가?

답변 keyword: entrypoint, empty data directory, first initialization, named volume, persisted cluster,
환경변수 재치환과 database 상태 변경의 구분, 명시적 role/password 변경 절차.

## Docker volume과 PostgreSQL Compose의 named volume

- 기록일: 2026-08-07
- 현재 상태: `infra/compose.yaml` 작성, Compose 설정 검사, 실제 volume 생성과 container 재생성 후
  data 유지 검증을 모두 통과함
- 질문: PostgreSQL 18의 data를 container 재생성 뒤에도 보존하려면 named volume을 어디에
  선언하고 어느 container 경로에 mount해야 하며, top-level `volumes`는 왜 필요한가?

### 핵심 답변

Docker volume은 Docker Engine이 container와 별도로 관리하는 저장 공간이다. Container를 제거해도
named volume은 별도 자원으로 남기 때문에 새 container에 다시 연결해서 기존 data를 사용할 수 있다.
이름이 붙은 volume을 named volume이라고 한다.

Container의 writable layer, 즉 container가 실행되면서 바뀐 파일을 저장하는 임시 계층은 container
수명에 묶인다. 따라서 PostgreSQL 기준 데이터를 보관하는 위치로 사용하면 안 된다. 현재 프로젝트는
named volume을 PostgreSQL image의 data 저장 경로에 mount해 container가 재생성되어도 database
파일을 다시 사용할 수 있게 한다.

현재 `infra/compose.yaml`에는 다음 구조를 사용한다.

```yaml
services:
  postgres:
    # 기존 image, ports, environment
    volumes:
      - postgres-data:/var/lib/postgresql

volumes:
  postgres-data:
```

기존 `image`, `ports`, `environment`는 설명을 위해 생략했으며 실제 파일에는 함께 들어 있다.

### top-level `volumes`를 선언하는 이유

top-level `volumes`는 Compose 프로젝트가 관리할 named volume 자원을 정의하는 곳이다. 반면
service 안의 `volumes`는 정의된 자원을 어떤 container의 어느 경로에 연결할지 지정한다.
즉, top-level은 자원 선언이고 service-level은 자원 사용이다.

| 위치 | 표현 | 책임 |
|---|---|---|
| `services.postgres.volumes` | `postgres-data:/var/lib/postgresql` | `postgres-data`를 PostgreSQL container의 `/var/lib/postgresql`에 연결 |
| top-level `volumes` | `postgres-data:` | Compose가 만들고 추적할 `postgres-data` 자원을 선언 |

service에서 `postgres-data`를 사용하면서 top-level 선언을 빼면 Compose는 그 이름이 어떤 자원을
가리키는지 확정할 수 없어 `undefined volume` 오류를 낸다. top-level 선언은 필요에 따라 `driver`,
`external`, `name`, `labels` 같은 volume 관리 옵션을 작성하는 자리이기도 하다. 현재의
`postgres-data:`처럼 값을 비워 두면 Docker Engine이 현재 host의 저장소를 관리하는 기본
`local` volume driver를 사용한다.

모든 mount에 top-level 선언이 필요한 것은 아니다. Host 폴더를 container 경로에 직접 연결하는
bind mount는 `./data:/var/lib/postgresql`처럼 경로를 바로 적는다. 이름 없이 container 경로만 적는
anonymous volume도 별도의 named volume 선언이 없다. 현재는 이름을 통해 저장 공간을 분명하게
재사용하고 정리하려고 named volume을 선택했다.

왼쪽 `postgres-data`는 Compose model 안의 논리적 volume 이름이고 오른쪽 `/var/lib/postgresql`은
container 내부 경로다. `.env.example`의 `COMPOSE_PROJECT_NAME=stockcast`를 사용할 때 Docker의
실제 resource 이름은 project 이름이 앞에 붙은 `stockcast_postgres-data`가 된다.
별도의 `name`, `external`과 `driver`를 지정하지 않아 서로 다른 Compose project의 자원 이름이
충돌하지 않게 나누는 기본 규칙과 Docker의 기본 `local` volume driver를 그대로 사용한다.

[Docker volume 문서](https://docs.docker.com/engine/storage/volumes/)는 volume을 container와 별도로
관리되는 저장 공간으로 설명한다. [Compose top-level volumes 문서](https://docs.docker.com/reference/compose-file/volumes/)는
top-level에서 named volume을 정의하고 service에서 사용할 권한과 mount 위치를 지정하는 구조를
설명한다.

### PostgreSQL 18에서 mount 경로가 중요한 이유

PostgreSQL Docker Official Image는 18 이상부터 다음처럼 경로 정책을 변경했다.

- `PGDATA`: `/var/lib/postgresql/18/docker`
- image가 선언한 `VOLUME`: `/var/lib/postgresql`
- Compose named volume target: `/var/lib/postgresql`

major version별 data directory를 같은 상위 volume 아래에 둘 수 있게 해 `pg_upgrade --link` 같은
upgrade 흐름을 지원하려는 구조다. 따라서 PostgreSQL 17 이하에서 흔히 보던
`/var/lib/postgresql/data` 예시를 현재 `postgres:18.4-bookworm`에 그대로 복사하지 않는다.
[PostgreSQL Docker Official Image의 `PGDATA` 설명](https://github.com/docker-library/docs/blob/master/postgres/README.md)이
18 이상에서는 mount와 volume의 target을 `/var/lib/postgresql`로 지정하라고 안내한다.

### 데이터 수명과 삭제 경계

- `docker compose up`은 named volume이 없으면 만들고, 있으면 기존 volume을 재사용한다.
- container stop, start와 일반적인 재생성은 named volume의 data를 제거하지 않는다.
- `docker compose down`은 기본적으로 named volume을 남긴다.
- `docker compose down -v`와 명시적인 `docker volume rm`은 volume data를 삭제할 수 있다.
- named volume은 지속성 수단이지 backup이 아니다. 잘못된 SQL, corruption 또는 volume 삭제에
  대비한 dump와 복구 검증은 별도 작업이다.

PostgreSQL image의 초기화 환경변수가 최초 한 번만 적용되는 이유도 이 수명과 연결된다. 기존
named volume이 재사용되면 entrypoint는 이미 존재하는 database cluster를 보존하고 다시 만들지 않는다.

### 대안과 현재 선택

- named volume: Docker가 host 저장 위치와 권한을 관리해 Windows·Docker Desktop에서 재현하기
  쉽기 때문에 현재 local infrastructure에 사용한다.
- bind mount: host의 정확한 폴더를 직접 보고 backup tool과 연결하기 쉽지만 Windows file sharing,
  경로 차이와 권한 문제를 사용자가 관리해야 하므로 지금은 선택하지 않는다.
- anonymous volume: 이름이 없어 재사용·식별·정리가 어려워 database 기준 저장소에 사용하지 않는다.
- container writable layer: container 제거·재생성 때 data가 유실되므로 기준 저장소에 사용할 수 없다.
- `tmpfs`: container가 중지되면 memory의 data가 사라지므로 지속되어야 하는 PostgreSQL data에
  사용할 수 없다.

Repository 내부 bind mount 경로를 만들지 않으므로 이번 단계에서 `.gitignore` 규칙을 추가할 필요가
없다. local volume의 실제 database 파일도 Git 추적 대상이 아니다.

### 흔한 오해

- named volume은 container 내부에만 있다: container와 별도인 Docker Engine resource이며 mount를
  통해 container가 사용한다.
- `docker compose down`이면 database도 항상 삭제된다: 기본 `down`은 named volume을 보존하지만
  `down -v`는 삭제한다.
- named volume이면 backup이 필요 없다: 실수나 corruption까지 되돌려 주는 backup은 아니다.
- `docker compose config` 성공이면 volume이 만들어졌다: config는 model의 선언과 참조만 검증하고
  Docker resource는 `up` 시점에 생성한다.
- PostgreSQL major version을 image tag에서 바꾸면 자동 upgrade된다: data format 호환성과 migration은
  별도 절차이며 volume 보존만으로 major upgrade가 완료되지 않는다.

### 확인 방법과 현재 검증 범위

저장소 root에서 다음 명령을 실행했다.

```powershell
docker compose --env-file .\.env.example -f .\infra\compose.yaml config
```

명령은 종료 코드 `0`으로 끝났고 Compose가 해석한 설정에서 `type: volume`,
`source: postgres-data`, `target: /var/lib/postgresql`, `name: stockcast_postgres-data`를 확인했다.
이는 YAML 구조, named volume 선언과 service 연결이 올바르다는 뜻이다.

실제 local `.env`로 처음 실행했을 때 `stockcast_postgres-data`가 생성됐고 `local` volume driver를
사용하는 것을 확인했다. 검증용 table에 `id=1`을 저장한 다음 `docker compose down`으로 container와
network만 제거했다. Volume이 남은 상태에서 새 container를 만들었고 `healthy` 전환 뒤 같은 행이
조회됐다. 검증용 table을 삭제한 뒤 존재하지 않는 것도 다시 확인했다. 따라서 일반 `down`과
container 재생성 경계에서는 PostgreSQL data가 named volume에 보존된다는 것을 실제로 검증했다.

### 면접 질문

질문: PostgreSQL container에 named volume을 사용하는 이유와 PostgreSQL 18에서 mount target을
`/var/lib/postgresql`로 정한 이유는 무엇인가?

답변 keyword: container lifecycle, persistent Engine resource, Compose project scope, PostgreSQL 18
version-specific `PGDATA`, parent `VOLUME`, major upgrade layout, persistence와 backup의 구분.

## PostgreSQL Compose의 `healthcheck`

- 기록일: 2026-08-07
- 현재 상태: `infra/compose.yaml`에 작성했고 Compose 설정 검사와 실제 `healthy` 전환을 검증함
- 질문: PostgreSQL container가 실행 중이라는 사실과 실제로 연결을 받을 준비가 됐다는 상태를
  어떻게 구분하고, Compose가 이를 어떤 주기로 확인하게 하는가?

### 핵심 답변

`healthcheck`는 Docker가 container 안에서 검사 명령을 주기적으로 실행하고 종료 코드를 읽어
service 상태를 `starting`, `healthy`, `unhealthy`로 판단하게 하는 설정이다. Container process가
살아 있다는 사실만으로 PostgreSQL 초기화가 끝났다고 볼 수 없으므로, PostgreSQL 공식 도구인
`pg_isready`로 TCP 연결을 받을 준비가 됐는지 확인한다.

현재 `services.postgres` 아래에 작성한 구조는 다음과 같다.

```yaml
healthcheck:
  test:
    - CMD-SHELL
    - pg_isready -h 127.0.0.1 -p 5432 -U "$${POSTGRES_USER}" -d "$${POSTGRES_DB}"
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 20s
```

기존 `image`, `ports`, `environment`, `volumes`와 같은 service 깊이에 `healthcheck`를 둔다.
위 코드는 실제 파일에서 해당 부분만 발췌한 것이므로 PostgreSQL service 전체를 다시 적은 예시는 아니다.

### 검사 명령이 실행되는 흐름

```text
Docker healthcheck
  -> PostgreSQL container의 /bin/sh
  -> pg_isready
  -> container 자신의 127.0.0.1:5432
  -> 종료 코드로 healthy 또는 실패 판단
```

`127.0.0.1`은 healthcheck가 실행되는 PostgreSQL container 자신을 뜻한다. Host에 공개한
`POSTGRES_HOST_PORT`를 거치지 않고 container 내부의 PostgreSQL TCP listener를 직접 확인한다.

`CMD-SHELL`은 다음 문자열을 container의 기본 shell인 `/bin/sh`로 실행한다. Shell이 필요하기 때문에
container 안의 `POSTGRES_USER`와 `POSTGRES_DB` 환경변수를 명령에서 사용할 수 있다. 비밀번호는
전달하지 않는다.

### `$${POSTGRES_USER}`처럼 `$`를 두 번 쓰는 이유

Compose는 `$`가 하나인 `${POSTGRES_USER}`를 설정 파일을 읽는 시점에 먼저 치환한다. 하지만 이번
명령은 container가 시작된 뒤 container 내부 환경변수를 사용해야 한다. `$$`는 Compose에 literal
`$`, 즉 글자 그대로의 달러 기호를 남기라는 표시다. Docker에 전달된 명령에서는
`${POSTGRES_USER}`가 되고, 이후 `/bin/sh`가 container 안의 실제 값을 넣는다.

`$${POSTGRES_USER}` 대신 `${POSTGRES_USER}`를 쓰면 현재 `.env` 값이 명령 문자열에 일찍 들어간다.
동작할 수는 있지만 Compose 치환과 container 환경변수 사용의 경계가 흐려지고, 나중에 container
환경 구성을 바꿀 때 검사 명령이 서로 어긋날 가능성이 커진다.

### 각 항목의 의미

| 항목 | 현재 값 | 의미와 선택 이유 |
|---|---|---|
| `test` | `pg_isready ...` | PostgreSQL이 container 내부 TCP `5432`에서 연결을 받는지 확인 |
| `interval` | `10s` | 검사가 끝난 뒤 다음 검사를 실행할 기본 간격 |
| `timeout` | `5s` | 한 번의 검사가 이 시간 안에 끝나지 않으면 실패 처리 |
| `retries` | `5` | 시작 유예 시간이 지난 뒤 연속 5회 실패하면 `unhealthy` 처리 |
| `start_period` | `20s` | 최초 database 초기화 중 발생하는 실패를 횟수에 포함하지 않는 유예 시간 |

`start_period`는 무조건 20초 동안 기다린 뒤 검사한다는 뜻이 아니다. Docker는 이 기간에도 검사를
실행할 수 있고 성공하면 준비 상태를 확인할 수 있지만, 초기화 중의 예상된 실패를 `retries`에
포함하지 않는다. 현재 값은 local PostgreSQL 초기화에 약 1분 정도의 실패 허용 범위를 주면서도
오래된 장애를 계속 숨기지 않도록 정한 시작값이다. 실제 실행 시간이 확인되면 조정할 수 있다.

### `pg_isready`가 확인하는 범위

PostgreSQL 18의 `pg_isready`는 server가 정상적으로 연결을 받으면 종료 코드 `0`, 시작 중처럼 연결을
거부하면 `1`, 응답이 없으면 `2`, 잘못된 인자 때문에 검사하지 못하면 `3`을 반환한다.
Docker healthcheck는 `0`을 성공으로 보고 나머지를 실패로 본다.

`pg_isready`는 PostgreSQL server의 연결 수락 상태를 확인하지만 올바른 password 인증과 실제 SQL
실행 성공까지 보장하지 않는다. 잘못된 사용자나 database 이름을 주어도 server 상태는 확인할 수
있지만 실패한 연결 기록이 log에 남을 수 있으므로 container에 이미 전달한 정확한 이름을 사용한다.
[PostgreSQL 18 `pg_isready` 문서](https://www.postgresql.org/docs/18/app-pg-isready.html)를 동작 기준으로
삼는다.

### 대안과 지금 제외하는 항목

- `psql -c "SELECT 1"`: 인증, database 존재와 간단한 query까지 확인하지만 password 처리와 검사
  부하가 추가된다. 실제 접속 검증 단계에서 사용하고 container healthcheck는 `pg_isready`로 둔다.
- `pg_ctl status`: PostgreSQL process 상태는 알 수 있지만 client 연결을 받을 준비가 됐는지는 충분히
  확인하지 못하므로 선택하지 않는다.
- TCP port만 확인하는 도구: port가 열렸다는 사실만 확인하고 PostgreSQL protocol 상태를 구분하지
  못하므로 공식 `pg_isready`를 사용한다.
- `depends_on: condition: service_healthy`: 나중에 PostgreSQL을 사용하는 application service가
  추가될 때 사용한다. 지금은 의존 service가 없으므로 작성하지 않는다.
- `restart`: healthcheck와 다른 책임이며 container가 `unhealthy`가 됐다고 자동 재시작시키는 설정이
  아니다. 장애 처리 정책을 정할 때 별도로 검토한다.

[Docker Compose healthcheck 문서](https://docs.docker.com/reference/compose-file/services/#healthcheck)는
`test`, `interval`, `timeout`, `retries`, `start_period`의 역할과 `CMD-SHELL` 실행 방식을 정의한다.
[Compose interpolation 문서](https://docs.docker.com/reference/compose-file/interpolation/)는 `$$`가
Compose 치환을 막고 literal `$`를 남기는 표기라고 설명한다.

### 확인 방법과 검증 결과

저장소 root에서 먼저 정적 검사를 실행했다. 정적 검사는 container를 시작하지 않고 Compose가
설정 구조와 값을 올바르게 해석하는지 확인하는 검사다.

```powershell
docker compose --env-file .\.env.example -f .\infra\compose.yaml config
```

성공 기준은 종료 코드 `0`이고 해석된 `healthcheck`에 `CMD-SHELL`, `pg_isready`, 네 시간·횟수 설정이
나타나는 것이다. 특히 검사 명령에 container 환경변수를 위한 달러 기호가 보존되어야 한다. 실패
기준은 YAML 들여쓰기 오류, 잘못된 duration 형식 또는 Compose가 변수를 빈 문자열로 치환하는 경우다.

정적 검사는 실제 상태 전환을 확인하지 않는다. 이어서 local `.env`로
`docker compose up -d --wait --wait-timeout 120 postgres`를 실행했고 container가 `healthy`로
전환됐다. `docker compose ps`에서 `127.0.0.1:5432` 포트 전달도 확인했다.

Healthcheck와 별도로 비밀번호를 사용한 TCP `psql` 접속, database·사용자 일치와 `UTC`를 확인했다.
또한 container를 제거하고 새로 만든 뒤에도 named volume의 검증용 행이 남는 것을 확인하고 해당
table을 삭제했다. 이 결과로 PostgreSQL 단일 service의 설정, readiness, 실제 접속과 data 지속성
검증이 모두 끝났다.

### 면접 질문

질문: Container가 `running`인 상태와 `healthy`인 상태는 무엇이 다르며, PostgreSQL healthcheck에
`pg_isready`를 사용해도 실제 query 검증이 별도로 필요한 이유는 무엇인가?

답변 keyword: process 생존, readiness, 주기적 command, exit code, accepting connections,
authentication과 SQL query는 별도 검증, healthcheck가 자동 복구 정책은 아님.

## PowerShell에서 Docker container의 `psql`로 SQL 전달하기

- 기록일: 2026-08-07
- 질문: PowerShell에서 Compose로 실행 중인 PostgreSQL에 SQL을 전달한 명령은 각 부분이 어떻게
  동작하며, `CREATE` 문장이 `CREATE` 한 단어로 잘린 오류는 왜 발생했고 어떻게 해결했는가?

### 핵심 답변

이번 명령은 SQL을 여러 겹의 따옴표 안에 넣지 않고 PowerShell의 pipeline으로 `psql` 표준 입력에
전달한다. Pipeline은 앞 명령의 출력을 뒤 명령의 입력으로 연결하는 기능이며, 표준 입력은 프로그램이
외부에서 text를 읽는 기본 통로다.

```text
PowerShell의 SQL 문자열
  -> PowerShell pipeline
  -> docker compose exec -T
  -> PostgreSQL container의 sh
  -> psql 표준 입력
  -> PostgreSQL server
```

검증용 table과 행을 만든 명령의 구조는 다음과 같다.

```powershell
'CREATE TABLE IF NOT EXISTS compose_persistence_check (id integer PRIMARY KEY); INSERT INTO compose_persistence_check (id) VALUES (1) ON CONFLICT (id) DO NOTHING; SELECT id FROM compose_persistence_check WHERE id = 1;' | docker compose --env-file .\.env -f .\infra\compose.yaml exec -T postgres sh -c 'PGPASSWORD="$POSTGRES_PASSWORD" psql -v ON_ERROR_STOP=1 -h 127.0.0.1 -p 5432 -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
```

### PowerShell과 Docker 부분

| 문법 | 실행 주체 | 의미 |
|---|---|---|
| `'SQL ...'` | PowerShell | 작은따옴표 안의 text를 변수 치환 없이 그대로 만든다. |
| `|` | PowerShell | 왼쪽의 SQL text를 오른쪽 process의 표준 입력으로 전달한다. |
| `.\` | PowerShell 경로 | 현재 directory를 기준으로 `.env`와 `infra/compose.yaml`을 찾는다. |
| `--env-file .\.env` | Docker Compose | Compose file의 `${...}`에 넣을 local 환경변수 파일을 지정한다. |
| `-f .\infra\compose.yaml` | Docker Compose | 사용할 Compose file을 명시한다. |
| `exec` | Docker Compose | 이미 실행 중인 service container에서 명령을 실행한다. |
| `-T` | Docker Compose | 가상 terminal 할당을 끄고 PowerShell pipeline의 입력을 그대로 전달한다. |
| `postgres` | Docker Compose | 명령을 실행할 service 이름이다. |
| `sh -c '...'` | container의 shell | 뒤 문자열을 container 안의 shell command로 해석한다. |

PowerShell의 작은따옴표는 `$POSTGRES_PASSWORD` 같은 표현을 host에서 치환하지 않는다. 따라서 이
문자열이 container까지 전달된 뒤 `sh`가 container 환경변수로 치환한다. Compose file 안에서
container 실행 시점까지 `$`를 남기려고 `$${POSTGRES_USER}`라고 썼던 것과 목적은 비슷하지만,
이번 값은 Compose file의 interpolation 대상이 아닌 CLI 명령 인자이므로 `$` 하나를 사용한다.

### `psql` 부분

`PGPASSWORD="$POSTGRES_PASSWORD"`는 바로 뒤에서 실행할 `psql` process에만 접속 password를
환경변수로 전달한다. PostgreSQL server의 password를 새로 설정하는 명령이 아니며, 실제 값을 command
text에 직접 적거나 출력하지 않는다.

| 옵션 | 의미 |
|---|---|
| `-v ON_ERROR_STOP=1` | SQL 하나가 실패하면 뒤 문장을 계속 실행하지 않고 `psql`을 실패로 끝낸다. |
| `-h 127.0.0.1` | container 내부 TCP로 PostgreSQL에 접속한다. |
| `-p 5432` | PostgreSQL container 내부 port를 사용한다. |
| `-U "$POSTGRES_USER"` | container의 사용자 환경변수를 접속 role로 사용한다. |
| `-d "$POSTGRES_DB"` | container의 database 환경변수를 접속 대상으로 사용한다. |

이 명령에는 `psql -c`가 없다. 따라서 `psql`은 PowerShell pipeline에서 들어오는 SQL text를 표준
입력으로 읽는다.

### SQL 문법

- `CREATE TABLE IF NOT EXISTS`: table이 없을 때만 생성하므로 같은 검증을 다시 실행해도 이미 있다는
  이유로 실패하지 않는다.
- `id integer PRIMARY KEY`: 정수 `id`를 중복될 수 없는 기본 식별자로 만든다.
- `INSERT ... VALUES (1)`: 검증용 행 하나를 저장한다.
- `ON CONFLICT (id) DO NOTHING`: `id=1`이 이미 있으면 중복 오류 대신 아무 변경도 하지 않는다.
  같은 명령을 반복해도 최종 상태가 같아지는 성질을 멱등성이라고 한다.
- `SELECT ... WHERE id = 1`: 저장된 검증용 행이 존재하는지 확인한다.
- `DROP TABLE compose_persistence_check`: data 보존 검증을 마친 뒤 임시 table을 제거한다.

### `CREATE` 문 오류의 원인

처음 명령은 다음처럼 `sh -c` 문자열 안에 `psql -c "SQL ..."`을 다시 넣었다.

```text
PowerShell
  -> docker.exe의 native command 인자
  -> container의 sh -c 문자열
  -> psql의 -c SQL 인자
```

PowerShell의 작은따옴표는 PowerShell이 문자열 하나를 만드는 데만 사용되고 따옴표 문자 자체가 모든
다음 단계에 그대로 남는다는 뜻은 아니다. Windows에서 native command, 즉 PowerShell cmdlet이 아닌
`docker.exe` 같은 외부 실행파일의 인자로 변환되고 Docker와 `sh -c`를 거치는 동안 내부 큰따옴표가
SQL의 경계를 유지하지 못했다. 결국 `psql`은
`CREATE TABLE ...` 전체가 아니라 `-c CREATE`만 받았다.

PostgreSQL의 `ERROR: syntax error at end of input`과 `LINE 1: CREATE`는 parser가 `CREATE` 다음에 올
대상과 정의를 받지 못한 채 입력이 끝났다는 뜻이다. Docker가 뒤에 표시한 Gordon 문구는 일반적인
도움말이며 Compose YAML 오류의 증거가 아니었다.

[Microsoft PowerShell quoting rules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_quoting_rules)는
PowerShell이 외부 command에 문자열을 전달하기 전에 따옴표를 먼저 해석하며 바깥 따옴표 문자는
제거된다고 설명한다. 이번 오류의 직접 근거는 `psql`이 보고한 `LINE 1: CREATE`와 표준 입력 방식으로
바꾼 뒤 같은 SQL이 성공한 실행 결과다.

### 해결 방법과 검증

해결할 때는 SQL을 중첩된 `-c "..."`에서 제거하고 PowerShell pipeline의 왼쪽으로 옮겼다.
`docker compose exec -T`가 그 text를 container process의 표준 입력으로 전달하고, `psql`은 `-c` 없이
해당 입력을 읽는다. 이 구조에서는 SQL의 공백과 세미콜론을 container의 `sh`가 command 구분자로
다시 해석하지 않는다.

먼저 `SELECT 1;`로 전달 경로를 검사했으며 출력 `1`과 종료 코드 `0`을 확인했다. 같은 구조로
`CREATE TABLE`, `INSERT 0 1`, `id=1` 조회와 `DROP TABLE`까지 성공했다. SQL이 길어지거나 migration으로
관리해야 할 단계가 되면 pipeline 한 줄보다 `.sql` file과 `psql -f`를 사용하는 편이 수정 이력과
재실행 범위를 관리하기 쉽다.

[Docker Compose `exec` 문서](https://docs.docker.com/reference/cli/docker/compose/exec/)는 `-T`가
pseudo-TTY 할당을 끄는 option이라고 정의한다. [PostgreSQL 18 `psql` 문서](https://www.postgresql.org/docs/18/app-psql.html)는
SQL을 표준 입력이나 file에서 읽을 수 있고, script에서 `ON_ERROR_STOP`이 설정된 상태로 SQL 오류가
발생하면 실패 종료 코드를 반환한다고 설명한다.

### 흔한 오해

- PowerShell의 작은따옴표가 container shell까지 그대로 전달된다: 작은따옴표는 우선 PowerShell의
  문자열 경계를 정하며, 다음 native process와 shell은 전달받은 인자를 다시 해석한다.
- `-T`가 container를 중지한다: `-T`는 가상 terminal만 끄며 실행 중인 container 상태는 바꾸지 않는다.
- `PGPASSWORD`가 database password를 변경한다: 이 값은 `psql` client가 현재 접속에 사용할
  password이며 server role의 password를 변경하지 않는다.
- pipeline이면 secret도 안전하다: SQL text에 실제 password를 직접 넣으면 terminal history나 log에
  남을 수 있으므로 secret은 container 환경변수로 전달하고 출력하지 않는다.

### 면접 질문

질문: PowerShell에서 Docker container의 `psql -c`로 전달한 SQL이 잘렸을 때 어떤 parsing 경계를
확인했고, 표준 입력 방식으로 바꾸면서 `-T`와 `ON_ERROR_STOP`을 함께 사용한 이유는 무엇인가?

답변 keyword: PowerShell string, native command argument, `sh -c`, nested quoting, standard input,
pseudo-TTY 비활성화, SQL error의 non-zero exit, secret을 literal command에 넣지 않기.

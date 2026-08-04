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

# ADR-0002: Monorepo와 최상위 디렉터리 경계

- 상태: Accepted
- 날짜: 2026-07-24
- 결정자: 프로젝트 사용자

## Context

프로젝트는 Java Spring Boot, Python FastAPI와 ML pipeline, Docker Compose 인프라, 공통 문서와
자동화 명령을 함께 발전시킨다. 서로 다른 runtime을 한 저장소에서 관리하되 파일 소유 경계가
없으면 애플리케이션 source, broker 설정, model artifact와 일회성 script가 뒤섞일 수 있다.
초기에는 혼자 개발하지만 한 기능이 Java, Python, 인프라와 문서를 함께 바꾸는 경우가 많으므로
변경을 추적하고 재현하기 쉬운 저장소 구조가 필요하다.

## Decision

하나의 Git 저장소 안에 다음 최상위 디렉터리 경계를 사용한다.

- `backend-spring/`: Spring Boot API, Java 도메인, provider와 Kafka·저장소 연동
- `prediction-service/`: FastAPI, Python feature, 학습·추론과 prediction worker
- `infra/`: Docker Compose와 외부 의존 service의 로컬 실행 설정
- `scripts/`: 둘 이상의 component에 걸친 반복 실행·검증 자동화
- `docs/`: 프로젝트 상태, 로드맵, 아키텍처, ADR, 오류와 작업 기록

Git은 빈 디렉터리를 추적하지 않으므로 `.gitkeep` 대신 각 디렉터리의 책임, 포함·제외 범위와
구현 시점을 설명하는 `README.md`를 추적한다. 실제 build tool이나 framework가 도입되기 전에는
빈 `src/`, `models/`, `data/` 같은 하위 구조를 미리 만들지 않는다.

애플리케이션 자체의 build 파일과 Dockerfile은 해당 애플리케이션 디렉터리가 소유한다.
여러 service를 실행하는 Compose와 외부 service 설정은 `infra/`가 소유한다. 특정 service의
build 명령은 해당 service에 두고, 저장소 전체를 다루는 자동화만 `scripts/`에 둔다.

## Alternatives considered

### Component별 다중 저장소

배포와 권한을 완전히 독립시키기 쉽다. 하지만 현재는 한 사용자가 end-to-end 흐름을 학습하며
Java, Python, 인프라와 문서를 함께 변경한다. repository version과 pull request를 여러 곳에서
동기화하는 비용이 이점보다 크므로 monorepo를 사용한다. 팀과 배포 주기가 독립적으로 커지면
다시 평가한다.

### `apps/` 또는 `services/` 상위 디렉터리

서비스 수가 많을 때 일관된 계층을 제공한다. 현재 실행 애플리케이션은 두 개뿐이고 `infra/`,
`scripts/`, `docs/`와 구분도 명확하므로 추가 중첩의 탐색 비용을 선택하지 않는다.

### 빈 디렉터리와 `.gitkeep`

구조를 빠르게 표시할 수 있지만 디렉터리의 책임과 금지 항목을 설명하지 못한다. 학습 프로젝트의
결정 근거를 보존하기 위해 유지 가치가 있는 README를 사용한다.

### 공통 `shared/` 또는 `contracts/`

초기에 공통 코드가 있을 것으로 예상해 만들 수 있지만 Java와 Python 구현을 직접 공유하면
결합도가 높아질 수 있다. 실제 cross-language schema 관리 도구나 공통 계약이 필요해질 때
책임과 배포 방식을 먼저 결정한 후 추가한다.

## Consequences

### Positive

- Java, Python, 인프라와 문서의 소유 위치를 빠르게 판단할 수 있다.
- 한 기능의 end-to-end 변경과 문서 동기화를 하나의 atomic commit으로 추적할 수 있다.
- 각 runtime의 dependency, build와 test 환경을 독립적으로 유지할 수 있다.
- 아직 사용하지 않는 framework 구조와 artifact 정책을 성급하게 고정하지 않는다.

### Negative

- 저장소가 커지면 모든 component의 변경 이력이 한곳에 쌓인다.
- component별 CI 경로 필터와 선택 실행이 필요해질 수 있다.
- 경계를 지키지 않으면 monorepo가 관련 없는 파일의 집합으로 변할 수 있다.

## Guardrails

- 새 파일은 먼저 어느 component가 runtime과 lifecycle을 소유하는지 판단한 뒤 배치한다.
- 애플리케이션 source를 `infra/`나 `scripts/`로 우회해 넣지 않는다.
- secret, local volume, dataset과 model artifact는 보존 정책 없이 Git에 포함하지 않는다.
- 새 최상위 디렉터리는 실제 책임이 생기고 기존 경계로 표현할 수 없을 때만 추가한다.
- component가 독립적인 팀, 배포 주기와 권한 경계를 갖게 되면 다중 저장소 전환 비용을 재평가한다.

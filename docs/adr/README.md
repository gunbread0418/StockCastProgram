# Architecture Decision Records

## 목적

ADR은 중요한 기술 또는 구조 선택의 맥락, 결정, 대안과 결과를 보존한다. 새 채팅에서 현재
결론만 확인하는 데 그치지 않고 왜 그런 결론이 나왔는지 복원하기 위한 문서다.

## 상태

- `Proposed`: 검토 중
- `Accepted`: 현재 적용할 결정
- `Superseded`: 새로운 ADR로 대체됨
- `Rejected`: 검토했으나 적용하지 않음

## 작성 규칙

- 기존 결정을 크게 바꿀 때 과거 ADR을 조용히 덮어쓰지 않는다.
- 새로운 ADR을 만들고 기존 문서에 `Superseded by`를 연결한다.
- 기술 이름보다 해결하려는 문제와 trade-off를 기록한다.
- 실제 측정 결과가 생기면 consequence 또는 관련 worklog에 연결한다.

## Index

| ID | 제목 | 상태 |
|---|---|---|
| [0001](0001-data-storage-responsibilities.md) | 데이터 저장소와 메시징 책임 분리 | Accepted |
| [0002](0002-monorepo-directory-boundaries.md) | Monorepo와 최상위 디렉터리 경계 | Accepted |
| [0003](0003-spring-boot-4-baseline.md) | Spring Boot 4.1.0 기준 채택 | Accepted |

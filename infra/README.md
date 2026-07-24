# Infrastructure

## 책임

Docker Compose와 Kafka, PostgreSQL, MongoDB, Redis, Kafka UI의 로컬 실행 설정을 소유한다.
개발자가 같은 service, network, volume, environment와 healthcheck를 재현할 수 있게 한다.

## 포함할 항목

- Docker Compose 파일과 Compose에서 참조하는 서비스 설정
- 데이터 저장소의 사용자·데이터베이스 생성처럼 부팅에 필요한 최소 초기화
- healthcheck와 로컬 관측 설정
- 고도화 단계에서 사용하는 Prometheus와 Grafana 설정

## 포함하지 않을 항목

- Spring Boot와 FastAPI의 애플리케이션 source
- 애플리케이션 도메인 schema migration
- 실제 `.env`, password, token과 private key
- named volume 내용, 로컬 DB 파일과 dump

## 구현 시점

M1에서 PostgreSQL, Redis, MongoDB, Kafka와 Kafka UI의 기본 Compose 환경을 구성한다.
애플리케이션 container와 monitoring은 해당 마일스톤의 기본 인프라를 검증한 뒤 확장한다.

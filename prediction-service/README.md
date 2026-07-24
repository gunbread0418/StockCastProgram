# Prediction Service

## 책임

FastAPI, Python feature 변환, 모델 로딩·추론과 향후 Kafka prediction worker를 소유한다.
Java 백엔드와 독립된 Python 실행 환경에서 학습과 serving의 feature 계약을 일치시킨다.

## 포함할 항목

- Python package와 dependency 설정
- FastAPI endpoint, request·response schema와 lifecycle 코드
- feature 생성, baseline 학습, 평가와 inference 코드
- Python 단위·계약 테스트와 향후 Kafka prediction worker
- 필요할 때 이 서비스 자체를 빌드하는 Dockerfile

## 포함하지 않을 항목

- Spring Boot API와 Java 도메인 로직
- Kafka broker와 데이터 저장소의 서버 실행 설정
- 로컬 가상환경, cache, dataset과 임의의 model artifact

## 구현 시점

M9에서 FastAPI 기본 프로젝트를 구성하고 M10에서 baseline model과 feature pipeline을
추가한다. Kafka worker는 비동기 prediction pipeline을 구현하는 M12에서 도입한다.
dataset과 model artifact의 Git 추적 정책은 M10에서 재현성과 보존 위치를 결정한 뒤 확정한다.

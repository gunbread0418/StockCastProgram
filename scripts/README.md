# Repository Scripts

## 책임

저장소 전체에서 반복 사용하는 실행, 검증과 개발 자동화를 소유한다. 문서에 긴 명령을
복사하는 대신 같은 입력으로 같은 검증을 다시 수행할 수 있게 한다.

## 포함할 항목

- 여러 service를 함께 다루는 bootstrap, start와 stop 보조 명령
- health, smoke test와 환경 사전 조건 검사
- 반복되는 데이터 생성·검증과 CI 보조 작업
- 사용법, 입력, 성공 기준과 실패 시 영향이 설명된 script

## 포함하지 않을 항목

- Gradle Wrapper처럼 특정 service가 소유하는 build 명령
- 애플리케이션 비즈니스 로직과 장기 실행 service
- secret을 직접 포함한 개인용 script
- 재사용되지 않는 일회성 실험과 생성된 결과 파일

## 구현 시점

여러 디렉터리에 걸친 첫 반복 작업이 생길 때 script를 추가한다. 지금은 책임만 정의하며,
실제 사용 사례가 없는 빈 script는 만들지 않는다.

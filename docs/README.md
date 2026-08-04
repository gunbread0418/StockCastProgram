# Project Documentation

## 책임

프로젝트의 목적, 현재 상태, 로드맵, 아키텍처, ADR, 개념 학습, 오류와 작업 기록을 소유한다.
새 작업이 시작될 때 실제 저장소와 함께 읽어 맥락과 결정 근거를 복구하는 문서 경계다.

## 문서 지도

- [프로젝트 목적과 기준](PROJECT_CONTEXT.md)
- [현재 상태와 다음 작업](PROJECT_STATUS.md)
- [전체 마일스톤](ROADMAP.md)
- [시스템 경계와 데이터 흐름](architecture/SYSTEM_OVERVIEW.md)
- [설계 결정 기록](adr/README.md)
- [개념 학습 기록](learning/README.md)
- [오류와 해결 기록](TROUBLESHOOTING.md)
- [상세 작업 기록 규칙](worklogs/README.md)

## 포함할 항목

- 현재 상태와 검증 결과
- 시스템 전체에 영향을 주는 설계와 그 결정 근거
- 구현과 별도 개념 채팅에서 질문한 기술 개념과 프로젝트 적용 근거
- 재사용 가치가 있는 오류 해결과 상세 작업 기록

## 포함하지 않을 항목

- 애플리케이션 source와 실행 가능한 비즈니스 로직
- 생성된 build output, 긴 원본 로그와 임시 메모
- category별 학습 문서와 중복되는 대화 전문 또는 기술별 파편 문서
- password, token, API key, 인증 header와 개인정보

## 구현 시점

M0에서 기본 문서 체계를 구성했으며 모든 마일스톤에서 실제 저장소 상태와 함께 유지한다.

# Project Status

## 문서 메타데이터

- 마지막 업데이트: 2026-07-20
- 현재 마일스톤: M0. Repository 초기화와 설계 기준
- 상태: IN_PROGRESS
- 다음 마일스톤: M1. Docker Compose 인프라

## 현재 요약

프로젝트 요구사항과 전체 방향을 정리했고, 새 채팅에서도 맥락을 복구할 수 있도록 지속 작업
문서 체계를 생성했다. 초기 기술 기준을 확정하고 `.gitignore`를 작성·검증했다. 애플리케이션과
인프라 코드는 아직 구현하지 않았다. M0를 완료하려면 `.env.example`, README 보완 및 초기
디렉터리 정책을 구성해야 한다. 이후 구현 작업은 검증된 atomic commit 단위마다 자동 commit과
push한다. GitHub `origin`을 연결하고 첫 M0 foundation commit을 `main`에 push했다.

## 완료된 작업

- 프로젝트 목적, 학습 목표와 비목표 정리
- 전체 이벤트 기반 아키텍처 초안 수립
- Kafka, PostgreSQL, MongoDB, Redis와 FastAPI의 책임 정의
- 기본, 확장, 고도화 마일스톤 구성
- 새 채팅용 프로젝트 맥락 로딩 규칙 정의
- 작업 종료 시 상태 동기화, 다음 행동 상기와 새 채팅 권장 규칙 정의
- 루트 `AGENTS.md` 생성
- 프로젝트 context, status, roadmap, architecture, ADR와 worklog 문서 생성
- Gradle Kotlin DSL, `com.stockcast`, fake 종목 3개와 1초 tick, 다음 5분 방향 분류 확정
- Gradle, Python, IDE, 로컬 agent metadata와 secret을 구분한 `.gitignore` 작성 및 검증
- 제안 항목의 도입 이유, 생략 영향, 대안과 제외 근거를 설명하는 교육 규칙 추가
- 새 tool이나 산출물 도입 시 `.gitignore`를 같은 작업에서 점검·보완하는 규칙 추가
- 검증된 atomic change마다 관련 파일을 commit하고 현재 branch를 push하는 지속 규칙 추가
- 기본 branch를 `main`으로 변경하고 GitHub `origin/main`에 첫 foundation commit push

## 아직 구현되지 않은 항목

- `.env.example`
- 실제 source directory
- Docker Compose
- Spring Boot 프로젝트
- FastAPI 프로젝트
- DB schema와 migration
- Kafka topic과 producer/consumer
- 테스트와 monitoring

## 확정된 초기 기준

- Build tool: Gradle Kotlin DSL
- Java base package: `com.stockcast`
- 초기 market data: `FAKE` exchange, 종목 3개, 1초 tick
- 첫 ML target: 다음 5분 방향 분류

## 저장소 상태

- Git 저장소가 초기화되어 있다.
- 기본 branch는 `main`이고 `origin/main`을 upstream으로 추적한다.
- Git remote `origin`은 `https://github.com/gunbread0418/StockCastProgram.git`이다.
- 첫 commit은 `5364609` (`chore: initialize project foundation`)이다.
- Git 작성자 이메일의 GitHub 계정 연결을 사용자가 확인했다.
- 애플리케이션 파일은 없다.
- `.gitignore`, `AGENTS.md`, `README.md`와 프로젝트 지속 작업 문서가 Git에서 추적된다.

## 주요 변경 파일

- `.gitignore`: secret, build output, Python cache, IDE와 로컬 도구 metadata 제외 규칙
- `AGENTS.md`: 항목별 설명, `.gitignore` 자동 점검과 atomic commit/push 지속 규칙
- `README.md`: 첫 ML 목표를 다음 5분 가격 방향 분류로 명확화
- `docs/PROJECT_CONTEXT.md`: 잠정 기술 기준을 확정 상태로 변경

## 추후 `.gitignore` 점검 시점

| 시점 | 확인할 항목 | 지금 미리 넣지 않은 이유 |
|---|---|---|
| M1 Docker Compose | repository 내부 bind mount data 경로 | named volume을 사용하면 제외할 로컬 경로가 없음 |
| M10 model pipeline | dataset, model artifact, `*.pkl`, `*.joblib` 정책 | 재현성과 artifact 보존 위치를 먼저 결정해야 함 |
| M13 선택 UI | `node_modules/`와 선택한 frontend tool cache | 현재 Node 기반 UI가 존재하지 않음 |
| 모든 마일스톤 | 새 build tool의 cache, log와 generated output | 실제 경로를 확인한 뒤 좁은 규칙으로 추가해야 함 |

## 이번 마일스톤 완료 기준

- 프로젝트 목적과 비목표가 문서화됨
- Codex가 새 채팅에서 읽을 루트 지침이 존재함
- 현재 상태와 다음 작업이 한 문서에서 확인됨
- 전체 마일스톤과 첫 설계 결정이 문서화됨
- 잠정 기술 기준이 사용자 확인을 거쳐 확정됨
- `.gitignore`와 `.env.example`이 안전하게 작성됨

## 검증 기록

| 날짜 | 검증 | 결과 |
|---|---|---|
| 2026-07-14 | 기존 workspace 확인 | `.git`, `.agents`, `.codex` 외 프로젝트 파일 없음 |
| 2026-07-14 | Git 상태 확인 | 초기 commit이 없는 `master` branch 확인 |
| 2026-07-14 | 지속 작업 문서 생성 | 상대 링크, 필수 내용, UTF-8 검증 통과 |
| 2026-07-14 | `AGENTS.md` 크기 확인 | 8,470 bytes로 기본 32 KiB 한도 이내 |
| 2026-07-20 | `.gitignore` 환경 파일 규칙 검증 | `.env` ignored `True`, `.env.example` ignored `False` |
| 2026-07-20 | Git 상태 재확인 | `.gitignore`, `AGENTS.md`, `README.md`, `docs/`가 미추적 상태 |
| 2026-07-20 | Git remote와 작성자 설정 확인 | remote 없음, 작성자 설정 존재, GitHub 이메일 연결 여부 미확인 |
| 2026-07-20 | staged diff와 민감정보 검사 | whitespace 오류 수정 후 통과, secret-like assigned value 없음 |
| 2026-07-20 | GitHub 원격 확인 | `origin` 접근 성공, push 전 remote branch 없음 확인 |
| 2026-07-20 | 첫 commit과 push | `5364609` 생성, `main -> origin/main`, upstream 설정 성공 |

## 알려진 실패와 blocker

- 기능 blocker 없음
- 일반 `git status`는 샌드박스 사용자와 저장소 소유권 차이로 `dubious ownership` 오류 발생
- 전역 Git 설정을 변경하지 않고 명령별 `safe.directory` 옵션으로 읽기 검증 가능
- `C:\WINDOWS\System32`에서 처음 실행한 `check-ignore`는 저장소를 찾지 못해 실패했으며,
  저장소로 이동하거나 `git -C <repository>`를 사용하면 정상 실행됨
- 샌드박스에서 branch rename 시 `.git/HEAD.lock` 생성 권한이 없어 부분 실패했으며, 승인된
  권한으로 `HEAD`를 `main`에 연결해 복구함
- 샌드박스 네트워크에서는 GitHub `ls-remote`가 차단됐으며, 승인된 네트워크 접근으로 검증함

## 다음 작업

M0의 다음 단일 TODO로 `.env.example`에 필요한 변수 범주와 안전한 placeholder 정책을 먼저
설명한 뒤, 사용자가 파일을 직접 작성하고 실제 secret이 포함되지 않았는지 검증한다.

## 다음 채팅 시작 문장

`.gitignore`와 첫 GitHub push를 완료했어. M0의 다음 TODO인 `.env.example` 작성을 각 항목의 이유와 함께 안내해줘.`

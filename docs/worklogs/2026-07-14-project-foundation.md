# 지속 작업 문서 체계 구성

## Metadata

- Date: 2026-07-14
- Milestone: M0. Repository 초기화와 설계 기준
- Status: DONE

## Goal

새 채팅에서도 프로젝트의 정체성, 사용자 학습 방향, 현재 진행 상태, 전체 로드맵과 Codex 작업
규칙을 저장소에서 복구할 수 있게 한다.

## Files changed

- `AGENTS.md`
- `README.md`
- `docs/PROJECT_CONTEXT.md`
- `docs/PROJECT_STATUS.md`
- `docs/ROADMAP.md`
- `docs/architecture/SYSTEM_OVERVIEW.md`
- `docs/adr/README.md`
- `docs/adr/0001-data-storage-responsibilities.md`
- `docs/worklogs/README.md`
- `docs/worklogs/2026-07-14-project-foundation.md`

## Decisions

- 루트 `AGENTS.md`에는 짧은 지속 규칙과 context loading 순서만 둔다.
- 프로젝트 설명, 진행 상태, 로드맵, 구조와 설계 근거를 별도 문서로 분리한다.
- 구현 작업이 허가된 경우 상태 문서 갱신을 작업 종료 절차에 포함한다.
- 중요한 답변마다 사용자의 다음 행동과 새 채팅 전환 여부를 상기시킨다.

## Commands and verification

| Command or check | Result |
|---|---|
| workspace file inspection | 기존 프로젝트 파일이 없는 초기 저장소 확인 |
| Git status with command-scoped safe directory | 초기 commit이 없는 `master` 확인 |
| Markdown relative link verification | 모든 상대 링크가 실제 파일로 연결됨 |
| Required context term verification | project, status, roadmap, architecture 필수 내용 통과 |
| UTF-8 decoding verification | 모든 Markdown 파일 통과 |
| `AGENTS.md` size verification | 8,470 bytes로 기본 32 KiB 한도 이내 |
| Git status | 의도한 `AGENTS.md`, `README.md`, `docs/`만 미추적 상태 |

## Failures and diagnosis

- 샌드박스 사용자와 Windows 저장소 소유자가 달라 일반 `git status`가 `dubious ownership`으로 실패했다.
- 전역 Git 설정을 바꾸지 않고 명령별 `safe.directory` 옵션으로 읽기 상태를 확인했다.

## Remaining work

- 잠정 기술 기준 사용자 확인
- `.gitignore`와 `.env.example` 작성

## Next recommended action

생성된 문서를 검토하고 잠정 기술 기준 네 가지를 확정한다.

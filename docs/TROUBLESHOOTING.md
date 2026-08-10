# Troubleshooting Log

## 목적

프로젝트 명령이나 구현이 실패했을 때 증상만 남기지 않고 원인, 해결 과정과 검증 결과를 함께
보존한다. 같은 오류가 다시 발생하면 이 문서를 먼저 검색하고, 회고나 면접에서는 문제를 어떻게
분리하고 확인했는지 설명하는 근거로 사용한다.

현재 진행 상황과 다음 작업은 `PROJECT_STATUS.md`, 한 작업의 긴 시행착오는 `worklogs/`에 둔다.
이 문서는 오류별로 빠르게 검색할 수 있는 색인과 재사용 가능한 해결법에 집중한다.

## 기록 규칙

- 해결 전에는 `OPEN`, 원인 확인과 해결 검증 후에는 `RESOLVED`를 사용한다.
- 원인과 가설을 구분하며, 같은 오류의 새 증거는 기존 항목에 추가한다.
- 명령과 오류 메시지는 원인 판단에 필요한 범위만 보존한다.
- 비밀번호, token, API key, 인증 header와 개인정보는 제거한다.
- 해결은 명령을 실행했다는 사실이 아니라 기대 결과를 다시 검증했을 때 완료로 본다.

## 빠른 색인

| ID | 날짜 | 마일스톤 | 상태 | 증상 | 확인된 원인 |
|---|---|---|---|---|---|
| ERR-001 | 2026-07-20 | M0 | RESOLVED | `not a git repository` | Git 명령을 저장소 밖에서 실행함 |
| ERR-002 | 2026-07-21 | M0 | RESOLVED | `.env.example` 경로를 찾지 못함 | 파일명 오기입으로 대상 파일이 없었음 |
| ERR-003 | 2026-07-24 | M0 | RESOLVED | `new blank line at EOF` | README 끝에 개행이 두 개 있었음 |
| ERR-004 | 2026-08-07 | M1 | RESOLVED | `psql`의 `CREATE` 구문이 끝에서 잘림 | 중첩된 셸에서 SQL 인자 경계가 사라짐 |

## ERR-001: 저장소를 찾지 못한 `git check-ignore`

- 상태: `RESOLVED`
- 실행 맥락: PowerShell의 현재 위치가 `C:\WINDOWS\System32`인 상태에서
  `git check-ignore`를 실행했다.
- 증상: `fatal: not a git repository (or any of the parent directories): .git`
- 원인: `safe.directory`는 저장소 소유권 검사를 허용할 뿐, 명령의 대상 저장소를 선택하지
  않는다. 현재 위치와 상위 경로에 `.git`이 없어 Git이 저장소를 찾지 못했다.
- 해결 과정: `Set-Location 'C:\Users\kbi04\OneDrive\문서\StockCastProgram'`으로 이동한 뒤
  같은 검증을 다시 실행했다. 다른 위치에서 실행해야 한다면 `git -C <repository> ...`로
  저장소를 명시할 수 있다.
- 검증 결과: `.env`는 `ignored=True`, `.env.example`은 `ignored=False`였고 `git status`가
  저장소 상태를 정상 출력했다.
- 재발 방지: Git 명령 안내에는 실행 위치를 함께 적고, 실행 전 `Get-Location` 또는
  `git -C <repository>`로 대상 저장소를 명확히 한다.

## ERR-002: `.env.example`을 찾지 못한 PowerShell 검증

- 상태: `RESOLVED`
- 실행 맥락: 저장소 루트에서 `Get-Content .env.example` 기반 검증을 실행했다.
- 증상: `Get-Content`가 `.env.example` 경로가 존재하지 않는다고 보고했다.
- 원인: 사용자가 파일명을 오기입해 검증 대상인 `.env.example`이 실제로 존재하지 않았다.
- 해결 과정: 파일명을 `.env.example`로 바로잡고 안전한 placeholder만 포함한 파일을 생성했다.
- 검증 결과: `Test-Path .env.example=True`, 환경 변수 14개, 잘못된 형식·중복·누락·예상 밖
  변수·잘못된 password placeholder·포트 중복이 모두 0이었다. `.env.example`은 Git ignore
  대상이 아니고 실제 `.env` 파일은 존재하지 않았다.
- 재발 방지: 내용 검증 전에 `Test-Path -LiteralPath .env.example`로 존재 여부를 먼저 확인하고,
  파일명은 `.gitignore`의 예외 규칙 `!.env.example`과 대조한다.

## ERR-003: README 끝의 불필요한 빈 줄

- 상태: `RESOLVED`
- 실행 맥락: M0 디렉터리 README와 상태 문서를 stage한 뒤
  `git diff --cached --check`를 실행했다.
- 증상: `backend-spring/`, `prediction-service/`, `infra/`, `scripts/`, `docs/`의 README에
  `new blank line at EOF`가 보고되어 commit 전 검증이 중단됐다.
- 원인: 각 파일이 마지막 문장 뒤에 줄 종료용 개행 하나와 내용 없는 추가 개행 하나를 가져,
  Git이 파일 끝의 새 빈 줄을 whitespace 오류로 판단했다.
- 해결 과정: 다섯 README의 마지막 내용 없는 줄을 제거하고 다시 stage했다.
- 검증 결과: 각 파일의 끝 개행 수가 1인 것을 확인했고 `git diff --cached --check`가
  오류 없이 통과했다.
- 재발 방지: 새 텍스트 파일은 마지막 줄 종료용 개행 하나만 남기고 commit 전
  `git diff --cached --check`를 항상 실행한다.

## ERR-004: PowerShell과 `compose exec` 사이에서 잘린 SQL

- 상태: `RESOLVED`
- 실행 맥락: PowerShell에서 `docker compose exec postgres sh -c '...'`를 사용하고,
  `sh -c` 문자열 안의 `psql -c "..."`로 여러 SQL 문장을 전달했다.
- 증상: PostgreSQL이 `ERROR: syntax error at end of input`과 `LINE 1: CREATE`를 출력했다.
  Docker가 이어서 표시한 Gordon 안내는 일반적인 후속 도움말이며 Compose 설정 오류를 뜻하지
  않았다.
- 원인: PowerShell, `docker compose exec`, 컨테이너의 `sh -c`를 차례로 거치면서 중첩된
  따옴표의 SQL 인자 경계가 유지되지 않았다. 그 결과 `psql`의 `-c`에는 전체
  `CREATE TABLE ...` 문장이 아니라 `CREATE`만 전달됐다.
- 해결 과정: SQL을 `psql -c "..."`에 중첩하지 않고 PowerShell 파이프로 `psql`의 표준 입력에
  전달했다. 파이프 입력을 안정적으로 받도록 `docker compose exec -T`로 가상 터미널 할당을
  끄고, SQL 오류가 발생하면 즉시 실패하도록 `psql -v ON_ERROR_STOP=1`을 사용했다.
- 검증 결과: 같은 PowerShell과 Compose 경로에서 `SELECT 1;`을 표준 입력으로 전달했을 때
  `1`이 출력됐고 명령이 종료 코드 `0`으로 끝났다. 같은 방식으로 검증용 table 생성,
  `INSERT 0 1`과 `id=1` 조회까지 성공했고, container 재생성 후 조회와 table 삭제도 통과했다.
  비밀번호 값은 명령 출력과 문서에 남기지 않았다.
- 추가 재현: 2026-08-10 MongoDB runtime 검증에서 `sh -c` 안의
  `mongosh --eval "const ..."`로 JavaScript를 전달했을 때 `const`만 인자로 남아
  `SyntaxError: Unexpected token`이 발생했다. JavaScript를 PowerShell pipeline으로 전달하고
  `docker compose exec -T`와 `mongosh --file /dev/stdin`을 사용하자 인증과 조회가 종료 코드 `0`으로
  성공했다. 이는 MongoDB service 오류가 아니라 같은 중첩 따옴표 경계의 추가 재현이다.
- 재발 방지: PowerShell에서 컨테이너의 `sh -c`를 거쳐 공백과 따옴표가 포함된 SQL이나 JavaScript를
  실행할 때는 `psql -c`나 `mongosh --eval`에 긴 코드를 중첩하지 않는다. SQL은 표준 입력,
  JavaScript는 표준 입력과 `--file /dev/stdin`으로 전달하고 `compose exec -T`를 함께 사용한다.

## 새 오류 기록 템플릿

```markdown
## ERR-NNN: 짧은 오류 제목

- 상태: `OPEN` 또는 `RESOLVED`
- 실행 맥락:
- 증상:
- 원인: 확정되지 않았다면 가설이라고 표시
- 해결 과정:
- 검증 결과:
- 재발 방지:
```

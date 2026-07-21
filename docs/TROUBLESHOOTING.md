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

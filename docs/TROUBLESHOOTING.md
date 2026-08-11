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
| ERR-005 | 2026-08-10 | M1 | RESOLVED | Kafka UI cluster 값이 빈 표로 보임 | JSON 배열을 pipeline에서 펼치지 않음 |
| ERR-006 | 2026-08-11 | M2 | OPEN | Java 21 사전검증에서 version 17 확인 | 현재 PowerShell이 JDK 17을 선택함 |
| ERR-007 | 2026-08-11 | M2 | OPEN | `winget show`에서 source 업데이트와 package 조회 실패 | 기본 source 등록은 정상, 갱신 실패 원인은 미확정 |

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

## ERR-005: Kafka UI cluster 배열을 펼치지 않아 값이 비어 보인 PowerShell 출력

- 날짜: 2026-08-10
- 마일스톤: M1. Docker Compose 인프라
- 상태: `RESOLVED`
- 실행 맥락: 전체 Compose 핵심 접속 경로를 검증하면서 Kafka UI의 `/api/clusters` 응답을
  `Invoke-RestMethod`에서 바로 `Select-Object name, status`로 전달했다.
- 증상: HTTP 오류는 발생하지 않았지만 `name`과 `status` heading 아래 값이 없는 표가 출력됐다.
  같은 명령을 다시 실행해도 결과가 같았다.
- 원인: 이 PowerShell 실행 환경에서 `Invoke-RestMethod`가 최상위 JSON 배열을 하나의 collection
  object로 pipeline에 전달했다. `Select-Object`가 배열의 각 cluster가 아니라 배열 자체에서
  `name`과 `status`를 찾았기 때문에 두 값이 비어 보였다. Kafka UI나 Kafka cluster 장애는 아니었다.
- 해결 과정: 응답을 `$clusters`에 저장하고 `Write-Output $clusters`로 배열 항목을 pipeline에
  펼친 뒤 `Select-Object name, status`를 적용했다.
- 검증 결과: 같은 `/api/clusters` 응답에서 `stockcast-local`과 `ONLINE`이 출력됐다. 이 결과로
  host의 Kafka UI HTTP 경로와 Kafka UI에서 내부 `kafka:19092` broker를 조회하는 경로가 모두
  정상임을 확인했다.
- 재발 방지: REST API의 최상위 응답이 JSON 배열이면 property 선택 전에 항목을 명시적으로 펼친다.
  응답 구조가 불명확하면 같은 선택 명령을 반복하지 않고 `ConvertTo-Json -Depth`로 원본 구조를
  먼저 확인한다.

## ERR-006: Java 21 사전검증에서 선택된 JDK 17

- 날짜: 2026-08-11
- 마일스톤: M2. Spring Boot 기본 구성
- 상태: `OPEN`
- 실행 맥락: 새 PowerShell의 `C:\WINDOWS\system32`에서 Spring Boot project와 Gradle Wrapper를
  만들기 전에 `java -version`과 `javac -version`을 실행했다.
- 증상: `java version "17"`과 `javac 17`이 출력됐다. 두 명령의 종료 코드는 모두 `0`이었지만
  프로젝트 기준인 major version `21`과 일치하지 않았다.
- 원인: 현재 PowerShell은 `C:\Program Files\Java\jdk-17\bin`의 `java.exe`와 `javac.exe`를 가장
  먼저 찾고, `JAVA_HOME`도 `C:\Program Files\Java\jdk-17`을 가리킨다. 즉 `PATH`와 `JAVA_HOME`의
  충돌이 아니라 두 설정이 모두 과거에 설치한 Oracle JDK 17을 선택하는 상태다.
  `C:\Program Files\Java`의 디렉터리 조회 결과도 `jdk-17` 하나뿐이어서 Oracle JDK의 표준 설치
  위치에는 Java 21이 없다.
- 해결 과정: Spring Boot project 생성과 환경 변수 변경은 시작하지 않았다. `where.exe java`와
  `where.exe javac`에서 JDK 17의 직접 경로가 첫 번째이고 Oracle의 공통 `javapath`가 두 번째인
  것을 확인했다. `C:\Program Files\Java` 조회도 성공했으며 `jdk-17`만 확인됐다. 기존 JDK 17은
  삭제하지 않고 Eclipse Temurin 21 JDK를 추가하는 방안을 추천하며, 설치 전 `winget show`로
  정확한 package ID와 installer 정보를 확인한다.
- 검증 결과: JDK 17 실행 파일 경로와 `JAVA_HOME`이 서로 일치한다. `where.exe`는 `PATH`에 포함된
  경로만 검색하지만 Oracle의 실제 설치 디렉터리도 함께 조회해 해당 위치에 Java 21이 없음을
  확인했다. Java 21 JDK를 아직 설치하거나 실행하지 않았으므로 이 항목은 `OPEN`으로 유지한다.
- 재발 방지: JDK를 설치하거나 환경 변수를 바꾼 뒤에는 새 PowerShell을 열어 `java`와 `javac`의
  실제 major version과 종료 코드를 다시 확인한다. Gradle Wrapper 생성과 build는 두 명령이 모두
  프로젝트 기준 version을 가리킨 뒤 시작한다.

## ERR-007: WinGet source 업데이트와 Temurin package 조회 실패

- 날짜: 2026-08-11
- 마일스톤: M2. Spring Boot 기본 구성
- 상태: `OPEN`
- 실행 맥락: Temurin 21 JDK를 설치하기 전에 package 정보를 확인하려고 새 PowerShell에서
  `winget show --id EclipseAdoptium.Temurin.21.JDK --exact --source winget`을 실행했다.
- 증상: `winget --version`은 `v1.29.280`, 종료 코드 `0`으로 정상 실행됐다. 이어진 package 조회에서는
  `원본을 업데이트하지 못했습니다. winget` 다음에 `입력 조건과 일치하는 패키지를 찾을 수 없습니다.`가
  출력됐고 종료 코드는 `-1978335212`였다.
- 원인: 아직 확정되지 않았다. Microsoft의 공식 return code 표에서 `-1978335212`는 조회 조건에 맞는
  package가 없다는 뜻이다. 그러나 같은 실행에서 `winget` source 업데이트가 먼저 실패했으므로 local
  cache 또는 source 접근 문제를 우선 확인한다. `winget source list`에서 기본 source 세 개가 모두
  조회됐고 `winget`도 정상 주소인 `https://cdn.winget.microsoft.com/cache`를 가리켰으므로 source가
  누락됐거나 주소가 잘못 등록된 가설은 제외한다. 현재 증거만으로 Package ID 오류라고 단정하지 않는다.
- 해결 과정: source를 초기화하거나 JDK를 수동 설치하지 않았다. 읽기 전용인 `winget source list`로
  기본 source 등록이 정상임을 확인했다. 다음에는 `winget source update --name winget`으로 실제 source
  갱신이 성공하는지 확인하고, 실패할 때만 reset이나 verbose log 확인을 별도 단계로 진행한다.
- 검증 결과: `msstore`, `winget`, `winget-font`의 이름과 주소, 명시적 설정이 기본값과 일치했다.
  전달된 출력에는 종료 코드 문자열이 포함되지 않아 `source list`의 종료 코드는 별도로 확정하지 않았다.
  Package 정보 조회와 Temurin 21 JDK 설치는 아직 성공하지 않았으므로 `ERR-006`과 이 항목을 모두
  `OPEN`으로 유지한다.
- 재발 방지: package를 찾지 못했다는 출력 앞에 source 관련 오류가 있으면 Package ID부터 바꾸지 않는다.
  `winget source list`와 source update 결과를 먼저 확인하고 필요하면 verbose log로 원인을 좁힌다.

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

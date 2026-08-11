# Backend Concepts

## 범위

Java, Spring Boot, FastAPI, HTTP API, dependency injection, validation, transaction, 예외 계약과
서비스 경계 개념을 기록한다. Kafka 자체 전달 semantics와 저장소별 내부 동작은 각각
Data Pipeline과 Data Storage 문서에 기록하고 필요한 link만 연결한다.

## Java 21 JDK 실행 환경 사전검증

- 기록일: 2026-08-11

### 질문

M2에서 Spring Boot 프로젝트를 만들기 전에 왜 `java -version`과 `javac -version`을 모두 확인해야
하며, 어떤 결과가 나와야 Java 21 JDK를 사용할 수 있다고 판단할 수 있는가?

### 핵심 답변

JDK(Java Development Kit)는 Java 애플리케이션을 개발하는 데 필요한 도구 묶음이다. 이 안에는
애플리케이션을 실행하는 `java`와 Java source인 `.java` 파일을 JVM이 실행할 `.class` 파일로
컴파일하는 `javac`가 들어 있다. JVM(Java Virtual Machine)은 컴파일된 Java bytecode를 실제로
실행하는 가상 실행 환경을 뜻한다.

`java -version`만 성공하면 Java 애플리케이션을 실행할 수 있다는 사실만 확인된다. Spring Boot
source와 test를 빌드하려면 컴파일러도 필요하므로 `javac -version`까지 성공해야 JDK 개발 환경을
확인할 수 있다. 두 명령의 major version, 즉 버전 번호의 첫 부분이 모두 `21`이어야 한다.

JDK 21은 대부분의 배포 업체가 장기 지원 버전인 LTS(Long-Term Support)로 제공한다. 이
사전검증에서는 특정 배포 업체나 patch version을 고정하지 않고 프로젝트 기준인 major version
`21`을 먼저 확인한다.

공식 근거:

- [Oracle JDK 21 `java` 명령 문서](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html)
- [Oracle JDK 21 `javac` 명령 문서](https://docs.oracle.com/en/java/javase/21/docs/specs/man/javac.html)
- [OpenJDK JDK 21 프로젝트](https://openjdk.org/projects/jdk/21/)

### 프로젝트에서의 연결

M2에서는 Gradle이 Spring Boot main source와 test source를 컴파일하고, 만들어진 애플리케이션을
JVM에서 실행한다. `java`와 `javac` 중 하나가 없거나 서로 다른 major version을 사용하면 프로젝트
생성 뒤에 build가 실패하거나 개발 환경마다 다른 결과가 생길 수 있다. 따라서 Spring Boot project와
Gradle Wrapper를 만들기 전에 현재 Windows 개발 환경이 Java 21을 실행하고 컴파일할 수 있는지
먼저 확인한다.

### 구분할 개념과 현실적인 대안

- `java`: JVM을 시작하고 Java 애플리케이션을 실행하는 명령이다.
- `javac`: Java source를 `.class` 파일로 바꾸는 컴파일러다.
- JRE(Java Runtime Environment): Java 애플리케이션 실행에 필요한 환경이라는 개념이다. 개발에는
  컴파일러가 포함된 JDK가 필요하므로 실행 환경만 확인해서는 부족하다.
- `PATH`: PowerShell에 `java`나 `javac`를 입력했을 때 Windows가 실제 실행 파일을 찾는 검색 경로다.
- `JAVA_HOME`: JDK가 설치된 루트 디렉터리를 build tool이나 IDE에 알려 주는 환경 변수다. 첫
  TODO는 현재 PowerShell이 선택하는 실행 파일부터 확인하고, `JAVA_HOME`과 실제 경로의 일치 여부는
  version 불일치가 있거나 Gradle 실행 환경을 검증할 때 확인한다.

IDE에 내장된 JDK, Gradle Java Toolchain 또는 Java가 들어 있는 container만 사용하는 방법도 있다.
하지만 현재 프로젝트는 사용자가 Windows에서 Gradle 명령과 Spring Boot를 직접 실행하며 학습하므로,
로컬 PowerShell에서 JDK 21을 먼저 확인하는 편이 실행 경로를 이해하고 오류를 진단하기 쉽다.

### 흔한 오해와 실패 영향

- `java -version`만 `21`이면 충분하다는 오해: `javac`가 없으면 source와 test를 직접 컴파일할 수
  없으므로 JDK 개발 환경 검증이 끝난 것이 아니다.
- `java`와 `javac`가 실행되기만 하면 된다는 오해: 두 명령이 서로 다른 major version을 가리키면
  compile 시점과 runtime 시점의 Java 기준이 달라질 수 있다.
- `JAVA_HOME`만 `21` 경로면 충분하다는 오해: 현재 PowerShell의 `PATH`가 다른 Java를 먼저 찾으면
  입력한 `java` 명령은 `JAVA_HOME`과 다른 설치본을 실행할 수 있다.

### 확인 방법

새 PowerShell 창의 어느 디렉터리에서든 다음 명령을 실행한다.

```powershell
java -version
"java exit code: $LASTEXITCODE"
javac -version
"javac exit code: $LASTEXITCODE"
```

성공 기준은 다음과 같다.

1. 두 명령 모두 `명령을 찾을 수 없습니다`와 같은 오류 없이 실행된다.
2. `java` 출력의 major version이 `21`이다. 예: `openjdk version "21.0.x"`.
3. `javac` 출력의 major version이 `21`이다. 예: `javac 21.0.x`.
4. 각 명령 직후 확인한 종료 코드가 `0`이다. 종료 코드는 명령의 성공 여부를 숫자로 나타내며
   `0`은 명령이 정상적으로 처리됐다는 뜻이다.

명령을 찾지 못하거나 버전이 다르면 Spring Boot project를 생성하지 않는다. 출력 원문을 보존한 뒤
실제 실행 파일 경로, `PATH`와 `JAVA_HOME`을 다음 진단 단계에서 확인한다. 이 명령은 Java 프로그램을
직접 compile하거나 Spring Boot를 실행하지 않으므로, 통과하더라도 아직 Gradle build와 애플리케이션
runtime을 검증한 것은 아니다.

### 면접 질문과 답변 keyword

질문: Spring Boot 개발 환경을 준비할 때 `java -version`과 `javac -version`을 왜 모두
확인했는가?

답변 keyword: `java` launcher와 JVM, `javac` compiler, JDK와 JRE 차이, compile time과 runtime,
major version 21 일치, `PATH`, `JAVA_HOME`, Gradle 실행 전 사전검증

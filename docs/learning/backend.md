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

### 실제 검증 결과

2026-08-11에 사용자가 새 PowerShell에서 실행한 결과는 `java version "17"`, `javac 17`이었고
두 명령의 종료 코드는 모두 `0`이었다. 이 결과는 Oracle JDK 17 계열의 실행기와 컴파일러가 현재
PowerShell에서 정상적으로 선택된다는 사실을 확인한다. 그러나 프로젝트 기준인 major version
`21`과 다르므로 Java 21 JDK 사전검증은 통과하지 못했다.

검증을 `C:\WINDOWS\system32`에서 실행한 것은 버전 불일치의 원인이 아니다. `java`와 `javac`는
현재 디렉터리보다 Windows의 명령 검색 경로인 `PATH`를 통해 선택되기 때문이다. 다음 단계에서는
`where.exe`로 선택 가능한 실행 파일 경로를 확인하고 `JAVA_HOME`과 비교한다. Java 21이 설치되지
않은 것인지, 설치돼 있지만 Java 17 경로가 먼저 선택되는 것인지는 아직 확인되지 않았다.

추가 진단에서 `where.exe`가 가장 먼저 찾은 실행 파일은
`C:\Program Files\Java\jdk-17\bin\java.exe`와
`C:\Program Files\Java\jdk-17\bin\javac.exe`였다. 그다음에는 Oracle의 공통 Java 경로인
`C:\Program Files\Common Files\Oracle\Java\javapath` 아래의 실행 파일이 검색됐다.
`JAVA_HOME`도 `C:\Program Files\Java\jdk-17`이었다. 따라서 현재 `PATH`와 `JAVA_HOME`이 서로
다른 JDK를 가리키는 문제는 없고, 두 설정 모두 일관되게 JDK 17을 선택한다.

`where.exe`는 현재 `PATH`에 포함된 실행 파일만 찾으므로 이 결과만으로 Java 21이 컴퓨터의 다른
경로에도 없다고 확정할 수는 없다. 다음 확인은 Oracle JDK가 설치된
`C:\Program Files\Java`의 하위 디렉터리를 조회해 Java 21 설치 여부를 확인하는 것이다.

해당 디렉터리 조회는 성공했고 하위 디렉터리는 `jdk-17` 하나뿐이었다. 따라서 Oracle JDK의 표준
설치 위치에는 Java 21이 없다. M2에서는 기존 Oracle JDK 17을 삭제하지 않고 OpenJDK 기반 배포판인
Eclipse Temurin 21 JDK를 함께 설치하는 방식을 추천한다. 프로젝트는 Oracle 전용 기능을 사용하지
않으며, Temurin은 Windows Package Manager의 major version별 package로 설치할 수 있어 개발 환경을
다시 구성할 때 같은 선택을 설명하기 쉽다.

설치 전에 `winget show`로 `EclipseAdoptium.Temurin.21.JDK` package의 publisher, version과 installer
정보를 확인하려 했다. `winget` 자체는 `v1.29.280`, 종료 코드 `0`으로 실행됐지만 package 조회에서는
`원본을 업데이트하지 못했습니다. winget`과 `입력 조건과 일치하는 패키지를 찾을 수 없습니다.`가
차례로 출력됐고 종료 코드는 `-1978335212`였다. Microsoft의 공식 return code 표에서 이 값은 package를
찾지 못했다는 뜻이다. 다만 그 전에 package 정보를 제공하는 source 업데이트가 실패했으므로, 이 결과만
보고 Package ID가 틀렸거나 Temurin 21이 없다고 확정하지 않는다.

후속 `winget source list`에서는 기본 source 세 개가 모두 조회됐다. `winget`은
`https://cdn.winget.microsoft.com/cache`, `msstore`는 Microsoft Store 주소, `winget-font`는
font repository 주소를 가리켰고 각 source의 명시적 설정도 기본값과 일치했다. 따라서 source가
누락됐거나 `winget` 주소가 잘못 등록된 문제는 아니다. 사용자가 전달한 출력에는 종료 코드 문자열이
포함되지 않아 `source list`의 종료 코드는 별도로 확정하지 않는다. 다음에는
`winget source update --name winget`으로 등록 정보가 아닌 실제 source 갱신 경로를 확인한다.

Package ID를 정확히 지정하고 `--exact`를 사용하면 비슷한 이름의 JRE나 다른 Java version을 선택하는
실수를 막을 수 있다. JRE는 실행 환경만 제공하는 package이므로 source와 test를 컴파일해야 하는 현재
프로젝트에는 JDK package가 필요하다.

### OCI 배포와 JDK 배포판의 관계

OCI Compute는 가상 서버를 제공하는 cloud infrastructure이고 Oracle JDK는 Java 구현을 배포하는 JDK
제품이다. 두 선택은 서로 독립적이므로 OCI의 Always Free compute를 사용한다고 Oracle JDK를 반드시
설치해야 하는 것은 아니다. OCI에서는 Oracle Linux나 Ubuntu image를 선택할 수 있고, Oracle Linux
Cloud Developer image에도 OpenJDK 21이 포함된다.

로컬 Windows에서 Temurin 21 JDK로 만든 일반 Spring Boot JAR는 서버에서 Java 21과 호환되는 Temurin,
OpenJDK 또는 Oracle JDK runtime으로 실행할 수 있다. Java bytecode가 운영체제와 CPU에 직접 묶이지
않기 때문이다. 다만 JNI 같은 native library, 즉 운영체제별 binary를 호출하는 의존성을 추가하면
서버의 Linux 배포판과 `x86_64` 또는 `aarch64` CPU 구조까지 맞춰야 한다. 현재 프로젝트에는 아직
Spring Boot source와 native library가 없으므로 그 예외는 발생하지 않았다.

따라서 현재 개발 PC에는 Temurin 21 JDK를 설치하고, 배포 시점에는 선택한 OCI image와 CPU 구조에 맞는
Java 21 runtime을 다시 고른다. Oracle JDK의 상용 지원이나 license가 필요한지는 cloud 제공자 이름이
아니라 운영 지원 요구사항을 기준으로 별도로 판단한다.

공식 근거:

- [Eclipse Temurin 설치 안내](https://adoptium.net/installation/)
- [Eclipse Temurin Windows MSI 안내](https://adoptium.net/installation/windows/)
- [Microsoft WinGet install 명령](https://learn.microsoft.com/windows/package-manager/winget/install)
- [Microsoft WinGet source 명령](https://learn.microsoft.com/windows/package-manager/winget/source)
- [Microsoft WinGet return code](https://github.com/microsoft/winget-cli/blob/master/doc/windows/package-manager/winget/returnCodes.md)
- [OCI Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
- [Oracle Linux Cloud Developer image](https://docs.oracle.com/en-us/iaas/oracle-linux/oci/developer-image.htm)
- [Oracle JDK 21 Linux 설치 안내](https://docs.oracle.com/en/java/javase/21/install/installation-jdk-linux-platforms.html)

### 면접 질문과 답변 keyword

질문: Spring Boot 개발 환경을 준비할 때 `java -version`과 `javac -version`을 왜 모두
확인했는가?

답변 keyword: `java` launcher와 JVM, `javac` compiler, JDK와 JRE 차이, compile time과 runtime,
major version 21 일치, `PATH`, `JAVA_HOME`, Gradle 실행 전 사전검증

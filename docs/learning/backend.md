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

실제 source 갱신에서는 진행률이 `100%`에 도달하고 `완료`가 출력됐으며 종료 코드도 `0`이었다.
따라서 현재 `winget` source는 network를 통해 package index를 정상적으로 갱신할 수 있다. 처음
`winget show`에서 발생한 source 업데이트 실패가 계속 재현되는 상태는 아니었다. 같은 정확한 Package
ID로 다시 조회하자 Eclipse Adoptium의 Temurin JDK `21.0.12.8`, Windows x64용 WiX MSI와 종료 코드
`0`을 확인했다. 따라서 source 갱신과 Package ID 검증은 통과했다.

Package ID를 정확히 지정하고 `--exact`를 사용하면 비슷한 이름의 JRE나 다른 Java version을 선택하는
실수를 막을 수 있다. JRE는 실행 환경만 제공하는 package이므로 source와 test를 컴파일해야 하는 현재
프로젝트에는 JDK package가 필요하다.

Temurin은 `C:\Program Files\Eclipse Adoptium\jdk-21.0.12.8-hotspot`에 설치됐다. Windows의 system
`JAVA_HOME`은 이 디렉터리를 가리키고 system `PATH`에서도 Temurin의 `bin`이 기존 Oracle JDK 17보다
앞에 배치됐다. 기존 Oracle JDK 17은 삭제하지 않았다.

설치 직후에도 이미 열려 있던 PowerShell에서는 `java`와 `javac`가 계속 version 17을 출력했다.
Windows process는 시작할 때 부모 process의 환경 변수를 복사하므로, 실행 중인 shell은 installer가
바꾼 system 환경 변수를 자동으로 다시 읽지 않기 때문이다. 컴퓨터를 재시작할 필요는 없고 terminal
application을 완전히 종료한 뒤 다시 열거나 현재 shell의 `JAVA_HOME`과 `PATH`를 system 값으로
새로 읽으면 된다.

system 환경 변수를 다시 읽은 process에서는 `where.exe`의 첫 `java`와 `javac`가 Temurin 21의
실행 파일을 가리켰다. `java`는 `openjdk version "21.0.12"`, `javac`는 `javac 21.0.12`를 출력했고
두 종료 코드가 모두 `0`이었다. 이 결과로 Java 21 JDK 실행 환경 사전검증은 완료했다. 아직 Gradle
Wrapper, Spring Boot build와 애플리케이션 runtime을 검증한 것은 아니다.

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

## Spring Initializr 설정과 M2 최소 dependency

- 기록일: 2026-08-11

### 질문

Java 21 JDK 사전검증이 끝난 뒤 Spring Initializr에서 어떤 값을 선택해야 하며, M2 기본 project에는
어떤 dependency만 넣어야 하는가?

### 핵심 답변

Spring Initializr는 선택한 build tool, Java version, Spring Boot version과 dependency를 바탕으로
Gradle Wrapper, build script, main application class와 기본 test를 만들어 주는 project 생성기다.
애플리케이션을 실행하는 framework 자체가 아니라 처음 시작할 파일 구조와 build 설정을 일관되게
생성하는 도구다.

현재 StockCast 기준의 project metadata는 다음과 같다.

| 항목 | 값 | 선택 이유 |
|---|---|---|
| Project | `Gradle - Kotlin` | 확정된 Gradle Kotlin DSL을 사용해 `build.gradle.kts`를 생성한다. |
| Language | `Java` | Spring application source는 Java 21로 작성한다. Kotlin DSL은 build script 언어일 뿐 application 언어를 Kotlin으로 바꾸지 않는다. |
| Group | `com.stockcast` | 생성 artifact의 group 좌표를 프로젝트 기준과 맞춘다. |
| Artifact | `backend-spring` | 기존 component 디렉터리와 Gradle project 이름을 맞춘다. |
| Name | `backend-spring` | 생성되는 project 표시 이름을 component 이름과 맞춘다. |
| Description | `StockCast Spring Boot backend` | project의 책임을 짧게 드러낸다. |
| Package name | `com.stockcast` | main application class를 최상위 package에 두어 이후 하위 package를 component scan 범위에 포함한다. |
| Packaging | `Jar` | embedded server와 함께 실행 가능한 단일 JAR로 build한다. 별도 servlet container에 배포하는 WAR는 필요하지 않다. |
| Java | `21` | 사전검증을 마친 JDK major version과 compile target을 맞춘다. |
| Configuration | `Properties` | M2 첫 구성에서는 들여쓰기 오류가 없는 `application.properties`로 외부 설정을 학습한다. YAML도 가능하지만 지금 얻는 기능 차이는 없다. |

M2 project 생성 시 직접 선택할 dependency는 다음 세 개다.

- `Spring Web`: Spring MVC, JSON 응답 처리와 embedded Tomcat을 제공한다. M2의 REST API와 invalid
  request 오류 계약을 만들기 위한 HTTP 경계다.
- `Validation`: request DTO의 값 규칙을 annotation으로 선언하고 위반을 400 오류 계약으로 변환할
  기반을 제공한다. 아직 request DTO를 만들지 않아도 M2 완료 기준에 포함될 기능이다.
- `Spring Boot Actuator`: `/actuator/health` 같은 관리 endpoint를 제공한다. M2 완료 기준의 health API를
  별도의 사용자 기능 controller로 흉내 내지 않고 실제 application 상태 경계로 검증할 수 있다.

Spring Initializr가 생성하는 `spring-boot-starter-test`는 별도의 Dependencies 검색 항목으로 추가하지
않아도 된다. 생성된 context test와 이후 단위·통합 test의 공통 기반으로 사용한다.

### 현재 Spring Boot version 충돌

프로젝트 문서는 Java 21과 Spring Boot 3.x를 확정 기준으로 사용한다. 공식 Spring Boot 문서에서
`3.5.16`은 Java 17부터 Java 25까지 지원하므로 Java 21과 호환된다. 다만 2026-08-11 현재 공식
`start.spring.io` metadata에는 `4.1.0`, `4.0.7`과 각 snapshot만 표시되고 3.x는 없다. 실제
`bootVersion=3.5.16` 생성 요청도 `Invalid Spring Boot version '3.5.16', Spring Boot compatibility
range is >=4.0.0`이라는 400 응답으로 거부됐다.

따라서 다음 두 경로를 구분해야 한다.

1. 기존 Spring Boot 3.x 기준 유지: Initializr UI 대신 Spring Boot `3.5.16`과 지원 범위 안의 Gradle
   8 Wrapper를 사용하는 project를 수동으로 구성한다.
2. 현재 공식 Initializr 사용: project 기준을 Spring Boot `4.1.0`으로 변경한 뒤 Initializr에서
   project를 생성한다.

Spring Boot 3.x에서 4.x로 바꾸는 것은 단순한 patch version 선택이 아니라 major version 변경이다.
major version 변경은 호환되지 않는 API나 기본 동작이 바뀔 수 있는 큰 version 변경을 뜻한다. 따라서
Initializr 화면에 4.x만 보인다는 이유로 사용자의 결정 없이 project 기준을 바꾸지 않는다.

공식 근거:

- [Spring Initializr Reference Guide](https://docs.spring.io/initializr/docs/current/reference/html/)
- [Spring Boot 3.5.16 System Requirements](https://docs.spring.io/spring-boot/3.5/system-requirements.html)
- [Spring Boot 3.5.16 release](https://spring.io/blog/2026/06/25/spring-boot-3-5-16-available-now/)
- [Spring Boot Actuator 시작 가이드](https://spring.io/guides/gs/actuator-service/)

### 프로젝트에서의 연결

이번 project 생성은 `backend-spring/`에 Gradle Wrapper, `build.gradle.kts`, Java main source와 context
test를 만드는 데까지만 해당한다. profile, 공통 오류 계약, health 공개 범위와 실제 실행 검증은 생성된
파일을 먼저 확인한 뒤 별도의 작은 TODO로 진행한다. `backend-spring/README.md`는 component 책임을
설명하는 기존 문서이므로 project 파일을 배치할 때 보존한다.

### 지금 제외하는 dependency와 이유

- `Spring Data JPA`와 `PostgreSQL Driver`: M3에서 Stock domain, migration과 transaction 경계를 함께
  도입한다. 지금 넣으면 DB 설정이 없는 M2 context test가 불필요하게 실패할 수 있다.
- `Spring for Apache Kafka`: M4의 topic, producer와 `FakeMarketDataGenerator`를 학습할 때 추가한다.
- `Spring Data MongoDB`: M5의 raw event 저장 책임과 함께 추가한다.
- `Spring Data Redis`: 최신 상태 cache와 PostgreSQL fallback을 구현하는 마일스톤까지 미룬다.
- `Docker Compose Support`: 현재 `infra/compose.yaml`의 시작·중지는 사용자가 별도로 관리한다. Spring
  application 시작이 인프라 생명주기를 암묵적으로 제어하지 않게 한다.
- `Spring Boot DevTools`: 자동 재시작은 편리하지만 build와 runtime 검증에 필수는 아니다. 기본 실행
  흐름을 확인한 뒤 개발 편의가 실제로 필요할 때 추가한다.
- `Lombok`: generated code가 학습 중인 Java class의 실제 생성자와 method를 숨길 수 있어 처음에는
  명시적인 Java code를 사용한다.
- `Spring Security`: 인증·인가는 M2 health와 오류 계약의 완료 조건이 아니므로 선택 확장 단계까지
  미룬다.
- `Spring Configuration Processor`: custom `@ConfigurationProperties` class를 만들 때 IDE의 설정 key
  자동완성이 필요해지면 추가한다.

### 흔한 오해와 실패 영향

- `Gradle - Kotlin`을 고르면 application도 Kotlin이라는 오해: Kotlin DSL은 build script에만
  적용되며 Language를 `Java`로 선택하면 source는 Java로 생성된다.
- 사용할 예정인 dependency를 모두 처음부터 추가하는 방식: auto-configuration이 DB나 외부 service
  연결을 시도해 아직 책임이 없는 M2 test와 실행을 실패시킬 수 있다.
- JDK 21이 설치됐으므로 Java 항목은 기본값이어도 된다는 오해: 현재 Initializr 기본값은 Java 17이므로
  `21`을 직접 선택해야 build target이 project 기준과 일치한다.
- ZIP을 기존 `backend-spring/` 위에 무조건 덮어쓰는 방식: 기존 책임 문서인 `README.md`를 잃거나
  `backend-spring/backend-spring/`처럼 디렉터리가 중복될 수 있다. 압축 내부 구조를 확인한 뒤 기존
  디렉터리에 생성 파일만 배치해야 한다.

### 생성 후 확인 방법

project를 생성한 뒤에는 아직 application runtime이 정상이라고 판단하지 않는다. 먼저 다음 구조만
확인한다.

- `backend-spring/gradlew`와 `backend-spring/gradlew.bat`
- `backend-spring/gradle/wrapper/`
- `backend-spring/build.gradle.kts`
- `backend-spring/settings.gradle.kts`
- `backend-spring/src/main/java/com/stockcast/`
- `backend-spring/src/test/java/com/stockcast/`
- 기존 `backend-spring/README.md`

그다음 생성된 build file과 Wrapper version을 검토하고 `gradlew.bat test`를 실행해야 dependency download,
compile과 context test까지 검증할 수 있다. 파일이 존재한다는 사실만으로 build와 runtime이 통과했다고
말하지 않는다.

### 면접 질문과 답변 keyword

질문: Spring Boot project를 시작할 때 필요한 dependency를 모두 한 번에 추가하지 않은 이유는 무엇인가?

답변 keyword: milestone별 책임, classpath 기반 auto-configuration, 불필요한 외부 연결 방지, 실패 범위
축소, Spring Web·Validation·Actuator의 M2 완료 기준, persistence와 messaging의 지연 도입

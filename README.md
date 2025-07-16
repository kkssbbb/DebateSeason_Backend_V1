# 토론철 (DebateSeason) - 실시간 토론 커뮤니티 백엔드

<br/>
<p align="center">
  <h3 align="center">토론철 백엔드 서버</h3>

  <p align="center">
    다양한 사회적 이슈에 대해 실시간으로 토론하는 커뮤니티 서비스의 백엔드 서버입니다.

</p>

## 프로젝트 소개

**토론철**은 사용자들이 다양한 사회적, 시사적 주제에 대해 자신의 의견을 나누고 실시간으로 토론할 수 있는 모바일 커뮤니티 플랫폼입니다. 이 저장소는 토론철 서비스의 모든 서버 사이드 로직을 담당하는 백엔드 API 서버입니다.

단순한 기능 구현을 넘어, 서비스의 지속적인 성장과 변화에 유연하게 대응할 수 있도록 **유지보수성과 확장성이 높은 시스템을 구축하는 것**에 목표를 두었습니다. 이를 위해 도메인 주도 설계(DDD)와 객체지향 원칙을 깊이 있게 고민하고 적용했습니다.

### 주요 기능 
*   **실시간 채팅 기반 토론:** WebSocket/STOMP를 활용한 다자간 실시간 토론 채팅 기능
*   **주제별 토론방:** 다양한 토론 주제(Issue) 생성 및 관리
*   **사용자 인증/인가:** JWT 기반의 안전한 사용자 인증 및 API 접근 제어
*   **프로필 및 커뮤니티 관리:** 사용자 정보, 관심 커뮤니티 설정 및 관리
*   **관리자 기능:** 이슈 등록 등 서비스 운영을 위한 어드민 기능

### 기술 스택
*   **Language**: `Java 17`
*   **Framework**: `Spring Boot 3.x`, `Spring MVC`, `Spring Security`, `Spring Data JPA`
*   **Real-time Communication**: `WebSocket`, `STOMP`
*   **Database**: `MariaDB`
*   **Authentication**: `JWT (JSON Web Token)`
*   **Build Tool**: `Gradle`
*   **Testing**: `JUnit5`, `Mockito`
*   **Etc**: `Swagger (API Docs)` , `Jacoco(테스트 커버리지 측정 라이브러리)`

---

## 아키텍처 및 설계 (Architecture & Design)

### 1. 계층형 아키텍처 (Layered Architecture)

프로젝트는 **DDD(도메인 주도 설계)** 사상을 기반으로 관심사를 분리하는 계층형 아키텍처를 채택했습니다. 각 계층은 명확한 책임을 가지며, 이를 통해 코드의 응집도를 높이고 결합도를 낮추어 유연하고 확장 가능한 구조를 구현했습니다.

<img width="363" height="365" alt="스크린샷 2025-07-16 오후 1 13 09" src="https://github.com/user-attachments/assets/affbf0b1-1518-48cc-8f05-0e7a3b69872c" />



*   **Presentation Layer (API Gateway)**: `Controller`가 위치하며, 클라이언트의 HTTP 요청을 수신하고 응답하는 역할 담당. `WebSocket` 연결 또한 이 계층에서 처리.
*   **Application Layer (Business Logic & Messaging)**: `Service`가 위치하며, 실제 비즈니스 로직을 처리. 도메인 객체들을 조합하여 사용자의 요청을 수행하고, 트랜잭션을 관리.
*   **Domain Layer**: 비즈니스의 핵심 규칙과 데이터(Entity, Value Object)를 포함하는 심장부. 시스템의 다른 어떤 계층에도 의존하지 않는 순수한 도메인 모델로 구성.
*   **Infrastructure Layer (Data Access)**: `Repository` 구현체, 외부 시스템 연동 등 기술적인 세부사항을 담당. `JPA`, 데이터베이스와의 통신 등을 처리.

<br/>

### 2. 도메인 주도 설계로 비즈니스 복잡성 해결

> "항상 더 좋은 구조는 없는지 고민하며 개발합니다."

MVP 개발 이후 기능이 확장되면서 서비스 로직이 비대해지고 복잡성이 증가하는 문제를 마주했습니다. 이를 해결하기 위해, 단순히 데이터를 담는 DTO와 절차적인 서비스 로직의 조합에서 벗어나, **데이터와 관련 행위를 함께 가지는 풍부한 도메인 모델**을 중심으로 시스템을 리팩토링했습니다.

*   **결과**: 핵심 비즈니스 로직이 `Domain Layer`에 모여 응집도가 높아졌습니다. 이로 인해 `Application Layer`(Service)는 도메인 객체들의 흐름을 제어하는 오케스트레이션 역할에 집중할 수 있게 되어, 새로운 비즈니스 요구사항 변경 및 추가 시 수정 범위를 최소화하고 버그 발생 가능성을 줄일 수 있었습니다.

<br/>

### 3. 'Tell, Don't Ask' 원칙을 통한 객체지향 설계

객체의 상태를 묻고 외부에서 로직을 처리하는 대신, 객체에게 메시지를 보내(Tell) 스스로 일하게(Ask) 만드는 'Tell, Don't Ask' 원칙을 적용하여 객체의 자율성과 캡슐화를 극대화했습니다.

예를 들어, 채팅 메시지 신고 기능 구현 시 `ChatService`가 `Chat` 객체의 상태를 일일이 확인하는 대신, `Chat` 객체 스스로가 신고 가능 여부를 판단하고(`guardSelfReport`), 신고된 메시지 내용을 마스킹하도록(`maskReportedMessage`) 책임을 위임했습니다.

```java
// src/main/java/com/debateseason_backend_v1/domain/chat/domain/model/chat/Chat.java

@Getter
public class Chat implements ReportTarget {
    // ... 필드 생략 ...

    // 'Tell, Don't Ask' 원칙 적용: Chat 객체 스스로 신고 정책을 검증
    public void guardSelfReport(Long reporterId) {
        if (this.user.isSameUser(reporterId)) {
            throw new CustomException(ErrorCode.SELF_REPORT_NOT_ALLOWED);
        }
    }
    
    // 'Tell, Don't Ask' 원칙 적용: Chat 객체 스스로 리포트된 메시지를 생성
    public Chat maskReportedMessage(Chat chat) {
        return Chat.builder()
                .content(chat.getReportedMessageContent()) // 리포트 메시지 내용 마스킹
                // ... 생략 ...
                .build();
    }
}
```
이러한 설계를 통해 `Chat`과 관련된 비즈니스 규칙의 변경이 발생하더라도 `Chat` 객체 내부로 수정 범위가 한정되어, **코드의 유지보수성과 테스트 용이성을 크게 향상**시킬 수 있었습니다.

<br/>

### 4. 실시간 통신을 위한 WebSocket & JWT 보안

다수의 사용자가 참여하는 실시간 토론 기능을 위해 `WebSocket`과 `STOMP` 프로토콜을 사용했습니다. 특히, 안전한 통신을 위해 `JWT` 토큰을 활용한 인증/인가 과정을 `WebSocket` 연결 단계에 통합했습니다.

1.  클라이언트는 `STOMP` 연결 시 `Authorization` 헤더에 `JWT` 토큰을 담아 서버에 전송합니다.
2.  서버는 `ChannelInterceptor`를 통해 메시지가 컨트롤러에 도달하기 전에 헤더를 가로챕니다.
3.  `JwtUtil` 컴포넌트로 토큰의 유효성을 검증하고, 인증된 사용자 정보를 `SecurityContextHolder`에 저장합니다.
4.  이를 통해 인가된 사용자만이 `WebSocket`을 통해 메시지를 발행(`pub`)하고 구독(`sub`)할 수 있도록 보안을 강화했습니다.

## 성장 경험

*   **기술적인 의사결정 능력**: 프로젝트 초기, 빠른 개발 속도를 위해 `SimpleBroker`를 사용했습니다. 이는 서버 확장 시 메시지 유실의 한계를 가지는 기술 부채가 될 수 있음을 인지하고 있었습니다. 이 경험을 통해 비즈니스 단계에 맞는 기술을 선택하고, 향후 발생할 수 있는 문제(Scale-out)를 예측하며 **전략적인 트레이드오프**를 하는 법을 배웠습니다.
*   **추상화를 통한 문제 해결**: 도메인 주도 설계를 적용하며 눈에 보이지 않는 비즈니스 로직과 규칙을 `Domain`이라는 모델로 구체화하고 추상화하는 과정을 깊이 있게 경험했습니다. 잘 설계된 도메인 모델이 어떻게 코드의 복잡도를 낮추고 팀의 생산성을 높이는지 체감할 수 있었습니다.
*   **코드 품질의 중요성**: '동작하는 코드'를 넘어 '읽기 쉽고, 변경이 용이한 코드'의 중요성을 깨달았습니다. 객체지향 원칙과 DDD를 적용하며 코드 품질을 높이는 과정 자체가 장기적으로는 개발 속도를 향상시킨다는 것을 배웠습니다.

<br/>
<br/>

## 프로젝트 시작 가이드

### Prerequisites

*   Java 17
*   Gradle 8.x
*   MariaDB

### Installation

1.  Clone the repo
    ```sh
    git clone https://github.com/your_github_username/DebateSeason_Backend_V1.git
    ```
2.  `application.yml` 설정
    `src/main/resources/` 경로의 `application-local.yml` 파일을 생성하고 DB, JWT 등의 설정 정보를 입력합니다.
    ```yaml
    spring:

    
      datasource:
        url: jdbc:mariadb://localhost:3306/debateseason
        username: your_db_username
        password: your_db_password
        driver-class-name: org.mariadb.jdbc.Driver

    # ... 기타 설정
    ```
3.  프로젝트 빌드 및 실행
    ```sh
    ./gradlew build
    java -jar build/libs/DebateSeason_Backend_V1-0.0.1-SNAPSHOT.jar
    ```


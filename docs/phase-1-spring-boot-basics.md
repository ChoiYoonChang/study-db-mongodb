# Phase 1: Spring Boot 프로젝트 기본 구조

## 📚 학습 목표

- [ ] Spring Boot 프로젝트 구조 이해
- [ ] Layered Architecture 개념 학습
- [ ] 패키지 구조 설계 (ERP 도메인 기반)
- [ ] 첫 REST API 구현 (`/api/health`)
- [ ] 기본 예외 처리 구현
- [ ] 로깅 설정 이해

**예상 소요 시간**: 1 ~ 2시간

---

## 1. Spring Boot 프로젝트 구조 이해

### 1.1 기본 디렉토리 구조

```
study-mongodb-v1/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/erp/codereview/
│   │   │       ├── CodeReviewAiApplication.java  # Main 클래스
│   │   │       ├── controller/                   # REST API 계층
│   │   │       ├── service/                      # 비즈니스 로직 계층
│   │   │       ├── repository/                   # 데이터 액세스 계층
│   │   │       ├── domain/                       # 도메인 모델 (Entity)
│   │   │       ├── dto/                          # Data Transfer Object
│   │   │       ├── exception/                    # 커스텀 예외
│   │   │       └── config/                       # 설정 클래스
│   │   └── resources/
│   │       ├── application.yml                   # 설정 파일
│   │       ├── application-local.yml             # 로컬 환경 설정
│   │       └── application-prod.yml              # 운영 환경 설정
│   └── test/
│       └── java/                                 # 테스트 코드
├── build.gradle                                  # Gradle 빌드 스크립트
└── settings.gradle
```

### 1.2 Layered Architecture (계층형 아키텍처)

```
┌──────────────────────────────────────────────────┐
│            Controller Layer                      │  ← REST API 엔드포인트
│  - HTTP 요청/응답 처리                           │
│  - 입력 검증                                     │
└────────────────┬─────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────┐
│             Service Layer                        │  ← 비즈니스 로직
│  - 트랜잭션 관리                                 │
│  - 복잡한 비즈니스 규칙                          │
└────────────────┬─────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────┐
│           Repository Layer                       │  ← 데이터 액세스
│  - DB CRUD 작업                                  │
│  - 쿼리 메서드                                   │
└────────────────┬─────────────────────────────────┘
                 │
                 ▼
            ┌─────────┐
            │ MongoDB │
            └─────────┘
```

**각 계층의 역할**:

| 계층 | 역할 | 의존 방향 |
|------|------|-----------|
| **Controller** | HTTP 요청 처리, DTO 변환 | Service 호출 |
| **Service** | 비즈니스 로직, 트랜잭션 | Repository 호출 |
| **Repository** | 데이터베이스 작업 | Entity 사용 |
| **Domain** | 핵심 도메인 모델 | 독립적 |

**💡 주의**: 상위 계층 → 하위 계층만 의존 (역방향 의존 금지)

---

## 2. ERP 도메인 패키지 구조 설계

### 2.1 도메인 분리 전략

**방법 1: 기능별 패키지 (Feature-based)** ⭐ 추천

```
com.erp.codereview/
├── finance/                    # 재무 도메인
│   ├── controller/
│   ├── service/
│   ├── repository/
│   ├── domain/
│   └── dto/
├── accounting/                 # 회계 도메인
│   ├── controller/
│   ├── service/
│   └── ...
└── codereview/                 # 코드 리뷰 도메인
    ├── controller/
    ├── service/
    └── ...
```

**장점**:
- 도메인별 독립성 높음
- 마이크로서비스로 분리 용이
- 팀별 작업 충돌 최소화

**단점**:
- 패키지 깊이 증가
- 공통 코드 관리 필요

---

**방법 2: 계층별 패키지 (Layer-based)**

```
com.erp.codereview/
├── controller/
│   ├── FinanceController.java
│   ├── AccountingController.java
│   └── CodeReviewController.java
├── service/
│   ├── FinanceService.java
│   └── ...
├── repository/
└── domain/
```

**장점**:
- 구조 단순
- 계층별 파악 용이

**단점**:
- 도메인 경계 모호
- 프로젝트 커지면 관리 어려움

---

**방법 3: 도메인 + 계층 혼합** ⭐⭐⭐ 최종 추천

```
com.erp.codereview/
├── common/                     # 공통 코드
│   ├── config/
│   ├── exception/
│   └── util/
├── finance/                    # 재무 도메인
│   ├── controller/
│   ├── service/
│   ├── repository/
│   ├── domain/
│   └── dto/
├── accounting/                 # 회계 도메인
└── codereview/                 # 코드 리뷰 도메인
```

**이유**:
- 도메인별 독립성 + 공통 코드 재사용
- 확장성과 유지보수성 균형
- 실무에서 가장 많이 사용

**💡 이 프로젝트에서는 방법 3을 사용합니다.**

---

## 3. 첫 REST API 구현: Health Check

### 3.1 목표

간단한 Health Check API를 구현하여 Spring Boot의 기본 흐름을 학습합니다.

**API 스펙**:
```
GET /api/health
Response: {"status": "UP", "timestamp": "2025-10-01T12:00:00"}
```

### 3.2 구현 방법 비교

#### 방법 1: Controller만 사용 (초보자 친화적)

**파일**: `src/main/java/com/erp/codereview/common/controller/HealthController.java`

```java
package com.erp.codereview.common.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api")
public class HealthController {

    @GetMapping("/health")
    public Map<String, Object> health() {
        Map<String, Object> response = new HashMap<>();
        response.put("status", "UP");
        response.put("timestamp", LocalDateTime.now());
        return response;
    }
}
```

**장점**:
- 코드 단순, 빠르게 이해 가능
- 별도 클래스 불필요

**단점**:
- 비즈니스 로직과 혼재
- 테스트 어려움
- 확장성 낮음

---

#### 방법 2: Service 계층 분리 (권장) ⭐⭐⭐

**파일 1**: `src/main/java/com/erp/codereview/common/dto/HealthResponse.java`

```java
package com.erp.codereview.common.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class HealthResponse {
    private String status;
    private LocalDateTime timestamp;
    private String version;
}
```

**Lombok 어노테이션 설명**:
- `@Data`: `@Getter`, `@Setter`, `@ToString`, `@EqualsAndHashCode`, `@RequiredArgsConstructor` 자동 생성
- `@NoArgsConstructor`: 기본 생성자 (파라미터 없음)
- `@AllArgsConstructor`: 모든 필드를 파라미터로 받는 생성자

**💡 주의**: `@Data`는 편리하지만, Entity에는 사용 금지 (순환 참조 위험)

---

**파일 2**: `src/main/java/com/erp/codereview/common/service/HealthService.java`

```java
package com.erp.codereview.common.service;

import com.erp.codereview.common.dto.HealthResponse;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;

@Service
public class HealthService {

    @Value("${app.version:1.0.0}")  // application.yml에서 주입, 기본값 1.0.0
    private String appVersion;

    public HealthResponse getHealthStatus() {
        return new HealthResponse("UP", LocalDateTime.now(), appVersion);
    }
}
```

**문법 설명**:
- `@Service`: Spring이 이 클래스를 Bean으로 관리 (싱글톤)
- `@Value`: 설정 파일에서 값 주입
- `${app.version:1.0.0}`: Elvis 연산자 (값 없으면 기본값 사용)

---

**파일 3**: `src/main/java/com/erp/codereview/common/controller/HealthController.java`

```java
package com.erp.codereview.common.controller;

import com.erp.codereview.common.dto.HealthResponse;
import com.erp.codereview.common.service.HealthService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class HealthController {

    private final HealthService healthService;

    @GetMapping("/health")
    public HealthResponse health() {
        log.info("Health check requested");
        return healthService.getHealthStatus();
    }
}
```

**문법 설명**:
- `@RestController`: `@Controller` + `@ResponseBody` (JSON 자동 변환)
- `@RequestMapping("/api")`: 클래스 레벨 경로 (모든 메서드에 적용)
- `@RequiredArgsConstructor`: `final` 필드 생성자 자동 생성 (의존성 주입)
- `@Slf4j`: Lombok의 로깅 어노테이션 (`log.info()` 사용 가능)

**💡 주의**: `@Autowired` 대신 생성자 주입 (`@RequiredArgsConstructor`) 사용 권장

**장점**:
- 계층 분리로 테스트 용이
- 비즈니스 로직 재사용 가능
- 확장성 높음

**단점**:
- 파일 수 증가
- 초기 코드량 많음

---

#### 방법 3: Actuator 사용 (Spring Boot 내장)

**설정**: `build.gradle`에 의존성 추가

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

**설정**: `src/main/resources/application.yml`

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always
```

**사용**:
```bash
curl http://localhost:8080/actuator/health
```

**장점**:
- 코드 작성 불필요
- 프로덕션 레벨 기능 (디스크, DB 상태 등)
- Spring 공식 지원

**단점**:
- 커스터마이징 제한적
- 엔드포인트 경로 고정 (`/actuator`)
- 학습 효과 낮음

---

### 3.3 최종 권장: 방법 2 (Service 계층 분리)

**이유**:
1. **학습 효과**: Spring Boot의 핵심 패턴 체득
2. **확장성**: 추후 로직 추가 용이
3. **테스트**: 각 계층 독립 테스트 가능
4. **실무 적합**: 실제 프로젝트와 동일한 구조

---

## 4. 전체 소스코드 구현

### 4.1 패키지 생성

IntelliJ에서:
```
src/main/java/com/erp/codereview/ 우클릭
→ New → Package
→ common.controller
→ common.service
→ common.dto
```

### 4.2 DTO 작성

**파일**: `src/main/java/com/erp/codereview/common/dto/HealthResponse.java`

```java
package com.erp.codereview.common.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

/**
 * Health Check API 응답 DTO
 *
 * @author ERP Team
 * @since 1.0
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
public class HealthResponse {

    /**
     * 시스템 상태 (UP, DOWN)
     */
    private String status;

    /**
     * 응답 생성 시각
     */
    private LocalDateTime timestamp;

    /**
     * 애플리케이션 버전
     */
    private String version;
}
```

**💡 JavaDoc 작성 팁**:
- 클래스, 메서드, 필드에 주석 작성
- `@author`, `@since`, `@param`, `@return` 활용
- IntelliJ: `/**` 입력 후 Enter → 자동 완성

---

### 4.3 Service 작성

**파일**: `src/main/java/com/erp/codereview/common/service/HealthService.java`

```java
package com.erp.codereview.common.service;

import com.erp.codereview.common.dto.HealthResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;

/**
 * 시스템 Health Check 서비스
 */
@Slf4j
@Service
public class HealthService {

    @Value("${app.version:1.0.0}")
    private String appVersion;

    /**
     * 시스템 상태 조회
     *
     * @return HealthResponse 시스템 상태 정보
     */
    public HealthResponse getHealthStatus() {
        log.debug("Retrieving health status - version: {}", appVersion);

        // 추후 확장: DB 연결 상태, 외부 API 상태 체크 가능
        return new HealthResponse("UP", LocalDateTime.now(), appVersion);
    }
}
```

**로깅 레벨**:
- `log.error()`: 심각한 오류
- `log.warn()`: 경고
- `log.info()`: 일반 정보 (운영 환경)
- `log.debug()`: 디버그 정보 (개발 환경)
- `log.trace()`: 상세 추적 (거의 사용 안 함)

---

### 4.4 Controller 작성

**파일**: `src/main/java/com/erp/codereview/common/controller/HealthController.java`

```java
package com.erp.codereview.common.controller;

import com.erp.codereview.common.dto.HealthResponse;
import com.erp.codereview.common.service.HealthService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Health Check API 컨트롤러
 */
@Slf4j
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class HealthController {

    private final HealthService healthService;

    /**
     * 시스템 상태 확인
     *
     * @return ResponseEntity<HealthResponse> 시스템 상태
     */
    @GetMapping("/health")
    public ResponseEntity<HealthResponse> health() {
        log.info("GET /api/health - Health check requested");
        HealthResponse response = healthService.getHealthStatus();
        return ResponseEntity.ok(response);
    }
}
```

**ResponseEntity vs 직접 반환**:

```java
// 방법 1: 직접 반환
@GetMapping("/health")
public HealthResponse health() {
    return healthService.getHealthStatus();
}

// 방법 2: ResponseEntity 사용 (권장) ⭐
@GetMapping("/health")
public ResponseEntity<HealthResponse> health() {
    return ResponseEntity.ok(response);  // HTTP 200
}
```

**ResponseEntity 장점**:
- HTTP 상태 코드 명시 가능 (200, 201, 400, 500 등)
- 헤더 추가 가능
- 더 명확한 의도 표현

---

### 4.5 설정 파일 작성

**파일**: `src/main/resources/application.yml`

```yaml
# 서버 설정
server:
  port: 8080

# 애플리케이션 설정
app:
  version: 1.0.0

# 로깅 설정
logging:
  level:
    root: INFO
    com.erp.codereview: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
```

**YAML vs Properties 비교**:

```properties
# application.properties
server.port=8080
app.version=1.0.0
logging.level.root=INFO
```

```yaml
# application.yml (권장)
server:
  port: 8080
app:
  version: 1.0.0
logging:
  level:
    root: INFO
```

**💡 YAML 장점**: 계층 구조 시각적, 중복 제거

---

## 5. 실행 및 테스트

### 5.1 애플리케이션 실행

```bash
./gradlew bootRun
```

### 5.2 API 테스트

**방법 1: curl**
```bash
curl http://localhost:8080/api/health
```

**예상 결과**:
```json
{
  "status": "UP",
  "timestamp": "2025-10-01T14:30:00",
  "version": "1.0.0"
}
```

**방법 2: IntelliJ HTTP Client**

IntelliJ에서 `test.http` 파일 생성:

```http
### Health Check
GET http://localhost:8080/api/health
```

실행: `GET` 옆 ▶️ 클릭

**방법 3: Postman**
- URL: `http://localhost:8080/api/health`
- Method: GET
- Send 클릭

---

## 6. 기본 예외 처리 구현

### 6.1 커스텀 예외 클래스

**파일**: `src/main/java/com/erp/codereview/common/exception/BusinessException.java`

```java
package com.erp.codereview.common.exception;

import lombok.Getter;

/**
 * 비즈니스 로직 예외
 */
@Getter
public class BusinessException extends RuntimeException {

    private final String errorCode;

    public BusinessException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }
}
```

### 6.2 Global Exception Handler

**파일**: `src/main/java/com/erp/codereview/common/exception/GlobalExceptionHandler.java`

```java
package com.erp.codereview.common.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

/**
 * 전역 예외 처리 핸들러
 */
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * 비즈니스 예외 처리
     */
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<Map<String, Object>> handleBusinessException(BusinessException ex) {
        log.error("Business Exception: {} - {}", ex.getErrorCode(), ex.getMessage());

        Map<String, Object> errorResponse = new HashMap<>();
        errorResponse.put("error", "Business Error");
        errorResponse.put("code", ex.getErrorCode());
        errorResponse.put("message", ex.getMessage());
        errorResponse.put("timestamp", LocalDateTime.now());

        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(errorResponse);
    }

    /**
     * 일반 예외 처리
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleException(Exception ex) {
        log.error("Unexpected Exception: ", ex);

        Map<String, Object> errorResponse = new HashMap<>();
        errorResponse.put("error", "Internal Server Error");
        errorResponse.put("message", "An unexpected error occurred");
        errorResponse.put("timestamp", LocalDateTime.now());

        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(errorResponse);
    }
}
```

**@RestControllerAdvice 설명**:
- 모든 `@RestController`의 예외를 한 곳에서 처리
- `@ExceptionHandler`: 특정 예외 타입 지정
- AOP 기반 (관심사 분리)

---

## 7. 시퀀스 다이어그램

### 7.1 Health Check API 흐름

```
Client                Controller              Service
  │                       │                      │
  │  GET /api/health      │                      │
  ├──────────────────────>│                      │
  │                       │                      │
  │                       │  getHealthStatus()   │
  │                       ├─────────────────────>│
  │                       │                      │
  │                       │                      │ [시스템 상태 체크]
  │                       │                      │
  │                       │  HealthResponse      │
  │                       │<─────────────────────┤
  │                       │                      │
  │  200 OK + JSON        │                      │
  │<──────────────────────┤                      │
  │  {status: "UP", ...}  │                      │
```

### 7.2 예외 발생 시 흐름

```
Client         Controller        Service      ExceptionHandler
  │                │                 │                │
  │  GET /api/xxx  │                 │                │
  ├───────────────>│                 │                │
  │                │  someMethod()   │                │
  │                ├────────────────>│                │
  │                │                 │ [예외 발생!]   │
  │                │                 │                │
  │                │  Exception      │                │
  │                │<────────────────┤                │
  │                │                 │                │
  │                │ handleException()                │
  │                ├─────────────────────────────────>│
  │                │                 │                │
  │                │         ErrorResponse            │
  │                │<─────────────────────────────────┤
  │  400/500 Error │                 │                │
  │<───────────────┤                 │                │
```

---

## 8. 주의사항 및 모범 사례

### 8.1 DTO vs Entity

| 구분 | DTO | Entity |
|------|-----|--------|
| **용도** | 계층 간 데이터 전달 | DB 테이블 매핑 |
| **위치** | Controller ↔ Service | Service ↔ Repository |
| **Lombok** | `@Data` 사용 가능 | `@Data` 금지 (순환 참조) |
| **변경** | 자유롭게 변경 | 신중하게 (DB 영향) |

**💡 중요**: Entity를 직접 API 응답으로 반환하지 말 것!

### 8.2 의존성 주입 방법

```java
// ❌ 나쁜 예: 필드 주입
@Autowired
private HealthService healthService;

// ✅ 좋은 예: 생성자 주입
private final HealthService healthService;

public HealthController(HealthService healthService) {
    this.healthService = healthService;
}

// ✅ 더 좋은 예: Lombok 생성자 주입
@RequiredArgsConstructor
public class HealthController {
    private final HealthService healthService;
}
```

**생성자 주입 장점**:
- 불변성 보장 (`final`)
- 순환 의존성 방지
- 테스트 용이

### 8.3 로깅 주의사항

```java
// ❌ 나쁜 예
log.info("User data: " + user.toString());  // 문자열 연산 비용

// ✅ 좋은 예
log.info("User data: {}", user);  // {} 플레이스홀더 사용
```

---

## 9. 체크리스트

- [ ] `common` 패키지 구조 생성 완료
- [ ] `HealthResponse` DTO 작성 완료
- [ ] `HealthService` 작성 및 `@Service` 어노테이션 확인
- [ ] `HealthController` 작성 및 `@RestController` 확인
- [ ] `application.yml` 설정 완료
- [ ] 애플리케이션 정상 실행 확인
- [ ] `GET /api/health` API 테스트 성공
- [ ] `GlobalExceptionHandler` 작성 완료
- [ ] 로그 출력 확인 (콘솔에서 `DEBUG` 레벨)
- [ ] Git 커밋: `"Phase 1: Implement health check API"`

---

## 10. 흔한 오류 및 해결

### 10.1 문제: 404 Not Found

**원인**: Controller 컴포넌트 스캔 실패

**해결**:
1. `CodeReviewAiApplication.java`의 패키지 확인
2. Controller 패키지가 `com.erp.codereview` 하위에 있는지 확인

### 10.2 문제: Lombok 미동작

**증상**: `@Data`, `@RequiredArgsConstructor` 인식 안 됨

**해결**:
1. `build.gradle`에 Lombok 의존성 확인:
```groovy
dependencies {
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}
```
2. Annotation Processing 활성화 확인 (Phase 0 참고)

### 10.3 문제: application.yml 값 주입 안 됨

**증상**: `@Value("${app.version}")` null

**해결**:
1. `application.yml` 파일 위치 확인: `src/main/resources/`
2. YAML 문법 확인 (들여쓰기 2칸)
3. 기본값 설정: `@Value("${app.version:1.0.0}")`

---

## 11. 다음 단계

Phase 1 완료를 축하합니다! 다음은 **[Phase 2: MongoDB 연동 및 도메인 모델](./phase-2-mongodb-domain.md)**입니다.

Phase 2에서는:
- MongoDB 설치 및 연결
- 재무/회계 도메인 모델 설계
- Spring Data MongoDB 사용
- CRUD API 구현

을 학습합니다.

---

**🎉 Phase 1 완료! 다음 Phase로 이동하세요!**

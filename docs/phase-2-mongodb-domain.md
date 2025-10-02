# Phase 2: MongoDB 연동 및 도메인 모델 설계

## 📚 학습 목표

- [ ] MongoDB 설치 및 연결 설정
- [ ] Spring Data MongoDB 이해 및 설정
- [ ] 재무/회계 도메인 모델 설계
- [ ] Document vs Entity 개념 이해
- [ ] CRUD API 구현
- [ ] Repository 패턴 학습

**예상 소요 시간**: 2 ~ 3시간

---

## 1. MongoDB 설치

### 1.1 MongoDB 설치 방법

**macOS (Homebrew)**:
```bash
# MongoDB 설치
brew tap mongodb/brew
brew install mongodb-community@7.0

# MongoDB 서비스 시작
brew services start mongodb-community@7.0

# 연결 확인
mongosh
```

**Windows (Chocolatey)**:
```bash
# MongoDB 설치
choco install mongodb

# 서비스 시작
net start MongoDB
```

**Docker 사용 (권장)** ⭐:
```bash
# MongoDB 컨테이너 실행
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=admin123 \
  mongo:7.0

# 연결 확인
docker exec -it mongodb mongosh -u admin -p admin123
```

**💡 추천**: Docker 사용 시 환경 독립성, 버전 관리 용이

### 1.2 MongoDB Compass 설치 (GUI 도구)

- 다운로드: https://www.mongodb.com/try/download/compass
- Connection String: `mongodb://localhost:27017`
- Docker 사용 시: `mongodb://admin:admin123@localhost:27017`

---

## 2. Spring Data MongoDB 설정

### 2.1 의존성 추가

**파일**: `build.gradle`

```groovy
dependencies {
    // Spring Web
    implementation 'org.springframework.boot:spring-boot-starter-web'

    // Spring Data MongoDB
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'

    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // Validation
    implementation 'org.springframework.boot:spring-boot-starter-validation'

    // Test
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**의존성 다운로드**:
```bash
./gradlew build --refresh-dependencies
```

### 2.2 MongoDB 연결 설정

**파일**: `src/main/resources/application.yml`

```yaml
server:
  port: 8080

spring:
  data:
    mongodb:
      # 로컬 설치 시
      # uri: mongodb://localhost:27017/erp-db

      # Docker 사용 시
      uri: mongodb://admin:admin123@localhost:27017/erp-db?authSource=admin

      # 상세 설정 (선택)
      database: erp-db
      auto-index-creation: true  # 인덱스 자동 생성

# 로깅
logging:
  level:
    org.springframework.data.mongodb: DEBUG  # MongoDB 쿼리 로그
    com.erp.codereview: DEBUG
```

**URI 형식 설명**:
```
mongodb://[username:password@]host[:port]/database[?options]
```

---

## 3. 재무/회계 도메인 모델 설계

### 3.1 도메인 분석

**ERP 재무/회계 핵심 엔티티**:

```
┌─────────────────┐
│   Account       │  계정 과목 (자산, 부채, 자본, 수익, 비용)
│  (계정 과목)    │
└────────┬────────┘
         │ 1
         │
         │ N
┌────────▼────────┐
│  Transaction    │  거래 (입금, 출금, 이체)
│   (거래)        │
└────────┬────────┘
         │ 1
         │
         │ N
┌────────▼────────┐
│  JournalEntry   │  분개 (차변, 대변)
│   (분개)        │
└─────────────────┘
```

**이 Phase에서 구현할 모델**: `Account` (계정 과목)

### 3.2 Account 도메인 요구사항

| 필드 | 타입 | 설명 | 예시 |
|------|------|------|------|
| `id` | String | MongoDB ObjectId | "507f1f77bcf86cd799439011" |
| `code` | String | 계정 코드 (유일) | "1010" |
| `name` | String | 계정명 | "현금" |
| `type` | Enum | 계정 유형 | ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE |
| `description` | String | 설명 | "현금 보관 계정" |
| `balance` | BigDecimal | 잔액 | 1000000.00 |
| `createdAt` | LocalDateTime | 생성일시 | 2025-10-01T10:00:00 |
| `updatedAt` | LocalDateTime | 수정일시 | 2025-10-01T12:00:00 |

---

## 4. Document 클래스 구현

### 4.1 AccountType Enum

**파일**: `src/main/java/com/erp/codereview/finance/domain/AccountType.java`

```java
package com.erp.codereview.finance.domain;

import lombok.Getter;
import lombok.RequiredArgsConstructor;

/**
 * 계정 과목 유형
 */
@Getter
@RequiredArgsConstructor
public enum AccountType {
    ASSET("자산", "현금, 예금, 매출채권 등"),
    LIABILITY("부채", "미지급금, 차입금 등"),
    EQUITY("자본", "자본금, 이익잉여금 등"),
    REVENUE("수익", "매출, 이자수익 등"),
    EXPENSE("비용", "급여, 임차료 등");

    private final String koreanName;
    private final String description;
}
```

**Enum 활용 이유**:
- 타입 안정성 (오타 방지)
- 코드 가독성 향상
- DB 저장 시 문자열로 자동 변환

### 4.2 Account Document (방법 1: 기본)

**파일**: `src/main/java/com/erp/codereview/finance/domain/Account.java`

```java
package com.erp.codereview.finance.domain;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "accounts")  // MongoDB 컬렉션명 지정
public class Account {

    @Id
    private String id;  // MongoDB ObjectId (자동 생성)

    @Indexed(unique = true)  // 유니크 인덱스
    private String code;  // 계정 코드 (예: "1010")

    private String name;  // 계정명 (예: "현금")

    private AccountType type;  // 계정 유형

    private String description;  // 설명

    private BigDecimal balance;  // 잔액

    private LocalDateTime createdAt;  // 생성일시

    private LocalDateTime updatedAt;  // 수정일시
}
```

**어노테이션 설명**:
- `@Document(collection = "accounts")`: MongoDB 컬렉션명 지정 (생략 시 클래스명 소문자)
- `@Id`: MongoDB의 `_id` 필드 매핑
- `@Indexed(unique = true)`: 유니크 인덱스 생성

**💡 주의**: Entity에 `@Data` 사용은 일반적으로 권장하지 않지만, 단순 Document는 허용

---

### 4.3 Account Document (방법 2: Builder 패턴) ⭐ 권장

**파일**: `src/main/java/com/erp/codereview/finance/domain/Account.java`

```java
package com.erp.codereview.finance.domain;

import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * 계정 과목 Document
 */
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "accounts")
public class Account {

    @Id
    private String id;

    @Indexed(unique = true)
    private String code;

    private String name;

    private AccountType type;

    private String description;

    @Builder.Default  // 기본값 설정
    private BigDecimal balance = BigDecimal.ZERO;

    @Builder.Default
    private LocalDateTime createdAt = LocalDateTime.now();

    private LocalDateTime updatedAt;

    /**
     * 잔액 업데이트
     */
    public void updateBalance(BigDecimal amount) {
        this.balance = this.balance.add(amount);
        this.updatedAt = LocalDateTime.now();
    }
}
```

**Builder 패턴 장점**:
- 불변성 (Immutable) - `@Getter`만 사용
- 가독성 높은 객체 생성
- 필수/선택 파라미터 명확

**사용 예시**:
```java
Account account = Account.builder()
        .code("1010")
        .name("현금")
        .type(AccountType.ASSET)
        .description("현금 보관 계정")
        .build();
```

---

### 4.4 Account Document (방법 3: Auditing 활용) ⭐⭐⭐ 최고 권장

**설정 클래스**: `src/main/java/com/erp/codereview/common/config/MongoConfig.java`

```java
package com.erp.codereview.common.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.mongodb.config.EnableMongoAuditing;

@Configuration
@EnableMongoAuditing  // MongoDB Auditing 활성화
public class MongoConfig {
}
```

**Document 클래스**: `src/main/java/com/erp/codereview/finance/domain/Account.java`

```java
package com.erp.codereview.finance.domain;

import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.Id;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "accounts")
public class Account {

    @Id
    private String id;

    @Indexed(unique = true)
    private String code;

    private String name;

    private AccountType type;

    private String description;

    @Builder.Default
    private BigDecimal balance = BigDecimal.ZERO;

    @CreatedDate  // 자동으로 생성 시각 설정
    private LocalDateTime createdAt;

    @LastModifiedDate  // 자동으로 수정 시각 업데이트
    private LocalDateTime updatedAt;

    public void updateBalance(BigDecimal amount) {
        this.balance = this.balance.add(amount);
    }
}
```

**Auditing 장점**:
- `createdAt`, `updatedAt` 자동 관리
- 코드 중복 제거
- 실수 방지

**💡 최종 권장**: 방법 3 (Auditing) 사용

---

## 5. Repository 구현

### 5.1 Spring Data MongoDB Repository

**파일**: `src/main/java/com/erp/codereview/finance/repository/AccountRepository.java`

```java
package com.erp.codereview.finance.repository;

import com.erp.codereview.finance.domain.Account;
import com.erp.codereview.finance.domain.AccountType;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

/**
 * 계정 과목 Repository
 */
@Repository
public interface AccountRepository extends MongoRepository<Account, String> {

    /**
     * 계정 코드로 조회
     */
    Optional<Account> findByCode(String code);

    /**
     * 계정 유형으로 조회
     */
    List<Account> findByType(AccountType type);

    /**
     * 계정명으로 검색 (부분 일치)
     */
    List<Account> findByNameContaining(String keyword);

    /**
     * 계정 코드 존재 여부 확인
     */
    boolean existsByCode(String code);
}
```

**메서드 네이밍 규칙**:

| 키워드 | 설명 | 예시 |
|--------|------|------|
| `findBy` | 조회 | `findByCode(String code)` |
| `existsBy` | 존재 여부 | `existsByCode(String code)` |
| `countBy` | 개수 | `countByType(AccountType type)` |
| `deleteBy` | 삭제 | `deleteByCode(String code)` |
| `Containing` | 부분 일치 | `findByNameContaining(String keyword)` |
| `And`, `Or` | 조건 연결 | `findByTypeAndName(...)` |

**💡 장점**: 메서드 이름만으로 쿼리 자동 생성 (구현 불필요)

---

### 5.2 커스텀 쿼리 (선택)

복잡한 쿼리는 `@Query` 어노테이션 사용:

```java
@Query("{ 'balance': { $gte: ?0 } }")
List<Account> findByBalanceGreaterThan(BigDecimal amount);
```

---

## 6. DTO 설계

### 6.1 요청 DTO

**파일**: `src/main/java/com/erp/codereview/finance/dto/AccountCreateRequest.java`

```java
package com.erp.codereview.finance.dto;

import com.erp.codereview.finance.domain.AccountType;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Pattern;
import java.math.BigDecimal;

/**
 * 계정 생성 요청 DTO
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
public class AccountCreateRequest {

    @NotBlank(message = "계정 코드는 필수입니다")
    @Pattern(regexp = "^[0-9]{4}$", message = "계정 코드는 4자리 숫자여야 합니다")
    private String code;

    @NotBlank(message = "계정명은 필수입니다")
    private String name;

    @NotNull(message = "계정 유형은 필수입니다")
    private AccountType type;

    private String description;

    private BigDecimal initialBalance;  // 초기 잔액 (선택)
}
```

**Validation 어노테이션**:
- `@NotBlank`: null, "", " " 모두 불허
- `@NotNull`: null만 불허
- `@Pattern`: 정규식 검증
- `@Min`, `@Max`: 숫자 범위 검증

---

### 6.2 응답 DTO

**파일**: `src/main/java/com/erp/codereview/finance/dto/AccountResponse.java`

```java
package com.erp.codereview.finance.dto;

import com.erp.codereview.finance.domain.Account;
import com.erp.codereview.finance.domain.AccountType;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * 계정 응답 DTO
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AccountResponse {

    private String id;
    private String code;
    private String name;
    private AccountType type;
    private String description;
    private BigDecimal balance;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    /**
     * Entity → DTO 변환 (정적 팩토리 메서드)
     */
    public static AccountResponse from(Account account) {
        return AccountResponse.builder()
                .id(account.getId())
                .code(account.getCode())
                .name(account.getName())
                .type(account.getType())
                .description(account.getDescription())
                .balance(account.getBalance())
                .createdAt(account.getCreatedAt())
                .updatedAt(account.getUpdatedAt())
                .build();
    }
}
```

**정적 팩토리 메서드 장점**:
- 생성 의도 명확 (`from`, `of` 등)
- 검증 로직 추가 가능
- 캐싱 가능

---

## 7. Service 구현

**파일**: `src/main/java/com/erp/codereview/finance/service/AccountService.java`

```java
package com.erp.codereview.finance.service;

import com.erp.codereview.common.exception.BusinessException;
import com.erp.codereview.finance.domain.Account;
import com.erp.codereview.finance.domain.AccountType;
import com.erp.codereview.finance.dto.AccountCreateRequest;
import com.erp.codereview.finance.dto.AccountResponse;
import com.erp.codereview.finance.repository.AccountRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.stream.Collectors;

/**
 * 계정 과목 서비스
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AccountService {

    private final AccountRepository accountRepository;

    /**
     * 계정 생성
     */
    @Transactional
    public AccountResponse createAccount(AccountCreateRequest request) {
        log.info("Creating account - code: {}, name: {}", request.getCode(), request.getName());

        // 중복 검증
        if (accountRepository.existsByCode(request.getCode())) {
            throw new BusinessException(
                    "이미 존재하는 계정 코드입니다: " + request.getCode(),
                    "ACCOUNT_CODE_DUPLICATED"
            );
        }

        // Entity 생성
        Account account = Account.builder()
                .code(request.getCode())
                .name(request.getName())
                .type(request.getType())
                .description(request.getDescription())
                .balance(request.getInitialBalance() != null ?
                        request.getInitialBalance() : BigDecimal.ZERO)
                .build();

        // 저장
        Account saved = accountRepository.save(account);
        log.info("Account created successfully - id: {}", saved.getId());

        return AccountResponse.from(saved);
    }

    /**
     * 전체 계정 조회
     */
    @Transactional(readOnly = true)
    public List<AccountResponse> getAllAccounts() {
        return accountRepository.findAll()
                .stream()
                .map(AccountResponse::from)
                .collect(Collectors.toList());
    }

    /**
     * ID로 계정 조회
     */
    @Transactional(readOnly = true)
    public AccountResponse getAccountById(String id) {
        Account account = accountRepository.findById(id)
                .orElseThrow(() -> new BusinessException(
                        "계정을 찾을 수 없습니다: " + id,
                        "ACCOUNT_NOT_FOUND"
                ));

        return AccountResponse.from(account);
    }

    /**
     * 계정 유형별 조회
     */
    @Transactional(readOnly = true)
    public List<AccountResponse> getAccountsByType(AccountType type) {
        return accountRepository.findByType(type)
                .stream()
                .map(AccountResponse::from)
                .collect(Collectors.toList());
    }

    /**
     * 계정 삭제
     */
    @Transactional
    public void deleteAccount(String id) {
        if (!accountRepository.existsById(id)) {
            throw new BusinessException(
                    "계정을 찾을 수 없습니다: " + id,
                    "ACCOUNT_NOT_FOUND"
            );
        }

        accountRepository.deleteById(id);
        log.info("Account deleted - id: {}", id);
    }
}
```

**@Transactional 설명**:
- `@Transactional`: 트랜잭션 관리 (MongoDB는 Replica Set에서만 지원)
- `readOnly = true`: 조회 전용 (성능 최적화)

---

## 8. Controller 구현

**파일**: `src/main/java/com/erp/codereview/finance/controller/AccountController.java`

```java
package com.erp.codereview.finance.controller;

import com.erp.codereview.finance.domain.AccountType;
import com.erp.codereview.finance.dto.AccountCreateRequest;
import com.erp.codereview.finance.dto.AccountResponse;
import com.erp.codereview.finance.service.AccountService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.util.List;

/**
 * 계정 과목 API 컨트롤러
 */
@Slf4j
@RestController
@RequestMapping("/api/accounts")
@RequiredArgsConstructor
public class AccountController {

    private final AccountService accountService;

    /**
     * 계정 생성
     */
    @PostMapping
    public ResponseEntity<AccountResponse> createAccount(
            @Valid @RequestBody AccountCreateRequest request) {
        log.info("POST /api/accounts - Creating account: {}", request.getCode());
        AccountResponse response = accountService.createAccount(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    /**
     * 전체 계정 조회
     */
    @GetMapping
    public ResponseEntity<List<AccountResponse>> getAllAccounts() {
        log.info("GET /api/accounts - Fetching all accounts");
        List<AccountResponse> accounts = accountService.getAllAccounts();
        return ResponseEntity.ok(accounts);
    }

    /**
     * ID로 계정 조회
     */
    @GetMapping("/{id}")
    public ResponseEntity<AccountResponse> getAccountById(@PathVariable String id) {
        log.info("GET /api/accounts/{} - Fetching account", id);
        AccountResponse account = accountService.getAccountById(id);
        return ResponseEntity.ok(account);
    }

    /**
     * 유형별 계정 조회
     */
    @GetMapping("/type/{type}")
    public ResponseEntity<List<AccountResponse>> getAccountsByType(
            @PathVariable AccountType type) {
        log.info("GET /api/accounts/type/{} - Fetching accounts by type", type);
        List<AccountResponse> accounts = accountService.getAccountsByType(type);
        return ResponseEntity.ok(accounts);
    }

    /**
     * 계정 삭제
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteAccount(@PathVariable String id) {
        log.info("DELETE /api/accounts/{} - Deleting account", id);
        accountService.deleteAccount(id);
        return ResponseEntity.noContent().build();
    }
}
```

**HTTP 상태 코드**:
- `201 Created`: 생성 성공
- `200 OK`: 조회 성공
- `204 No Content`: 삭제 성공
- `400 Bad Request`: 잘못된 요청
- `404 Not Found`: 리소스 없음

---

## 9. 테스트

### 9.1 API 테스트 (HTTP Client)

**파일**: `test-account.http` (IntelliJ)

```http
### 계정 생성
POST http://localhost:8080/api/accounts
Content-Type: application/json

{
  "code": "1010",
  "name": "현금",
  "type": "ASSET",
  "description": "현금 보관 계정",
  "initialBalance": 1000000
}

### 전체 계정 조회
GET http://localhost:8080/api/accounts

### ID로 조회 (위에서 받은 ID 입력)
GET http://localhost:8080/api/accounts/{{accountId}}

### 유형별 조회
GET http://localhost:8080/api/accounts/type/ASSET

### 계정 삭제
DELETE http://localhost:8080/api/accounts/{{accountId}}
```

### 9.2 curl 테스트

```bash
# 계정 생성
curl -X POST http://localhost:8080/api/accounts \
  -H "Content-Type: application/json" \
  -d '{
    "code": "1010",
    "name": "현금",
    "type": "ASSET",
    "description": "현금 보관 계정",
    "initialBalance": 1000000
  }'

# 전체 조회
curl http://localhost:8080/api/accounts
```

---

## 10. 시퀀스 다이어그램

### 10.1 계정 생성 흐름

```
Client         Controller       Service        Repository      MongoDB
  │                │               │                │             │
  │  POST /api/    │               │                │             │
  │  accounts      │               │                │             │
  ├───────────────>│               │                │             │
  │                │               │                │             │
  │                │ createAccount()│                │             │
  │                ├──────────────>│                │             │
  │                │               │ existsByCode() │             │
  │                │               ├───────────────>│             │
  │                │               │                │ find()      │
  │                │               │                ├────────────>│
  │                │               │                │ result      │
  │                │               │                │<────────────┤
  │                │               │ false          │             │
  │                │               │<───────────────┤             │
  │                │               │                │             │
  │                │               │ save()         │             │
  │                │               ├───────────────>│             │
  │                │               │                │ insert()    │
  │                │               │                ├────────────>│
  │                │               │                │ Account     │
  │                │               │                │<────────────┤
  │                │               │ Account        │             │
  │                │               │<───────────────┤             │
  │                │ Response      │                │             │
  │                │<──────────────┤                │             │
  │  201 Created   │               │                │             │
  │<───────────────┤               │                │             │
```

---

## 11. 주의사항

### 11.1 BigDecimal 사용 이유

```java
// ❌ 나쁜 예: double 사용
double balance = 1000.1 + 2000.2;  // 결과: 3000.3000000000002

// ✅ 좋은 예: BigDecimal 사용
BigDecimal balance = new BigDecimal("1000.1")
        .add(new BigDecimal("2000.2"));  // 결과: 3000.3
```

**재무/회계에서는 반드시 `BigDecimal` 사용!**

### 11.2 MongoDB ID 타입

```java
@Id
private String id;  // ObjectId → String 자동 변환
```

**주의**: `ObjectId` 타입도 사용 가능하지만, `String`이 더 범용적

### 11.3 Validation 활성화

`@Valid` 어노테이션 동작하려면:

```groovy
// build.gradle
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

---

## 12. 체크리스트

- [ ] MongoDB 설치 및 실행 확인 (`mongosh` 연결 테스트)
- [ ] Spring Data MongoDB 의존성 추가
- [ ] `application.yml`에 MongoDB URI 설정
- [ ] `AccountType` Enum 작성
- [ ] `Account` Document 작성 (Auditing 적용)
- [ ] `MongoConfig` 설정 (`@EnableMongoAuditing`)
- [ ] `AccountRepository` 인터페이스 작성
- [ ] `AccountCreateRequest`, `AccountResponse` DTO 작성
- [ ] `AccountService` 작성 (CRUD 로직)
- [ ] `AccountController` 작성 (REST API)
- [ ] API 테스트 (POST, GET, DELETE) 성공
- [ ] MongoDB Compass에서 데이터 확인
- [ ] Git 커밋: `"Phase 2: Implement Account domain and CRUD API"`

---

## 13. 다음 단계

Phase 2 완료를 축하합니다! 다음은 **[Phase 3: Spring AI 및 LLM 연동](./phase-3-spring-ai-llm.md)**입니다.

Phase 3에서는:
- Spring AI 설정
- Hugging Face API 연동
- StarCoder2 모델 사용
- 프롬프트 엔지니어링 기초

를 학습합니다.

---

**🎉 Phase 2 완료! MongoDB와 도메인 모델을 마스터했습니다!**

# Phase 3: Spring AI 및 LLM 연동

## 📚 학습 목표

- [ ] Spring AI 개념 및 아키텍처 이해
- [ ] Hugging Face API 설정
- [ ] StarCoder2 모델 연동
- [ ] 프롬프트 엔지니어링 기초
- [ ] LLM 응답 처리 및 에러 핸들링
- [ ] 간단한 코드 분석 API 구현

**예상 소요 시간**: 2 ~ 3시간

---

## 1. Spring AI 소개

### 1.1 Spring AI란?

**Spring AI**는 AI 애플리케이션 개발을 위한 Spring 공식 프레임워크입니다 (2024년 정식 출시).

**주요 기능**:
- 다양한 LLM 통합 (OpenAI, Hugging Face, Ollama 등)
- 프롬프트 템플릿 관리
- RAG 파이프라인 지원
- 벡터 데이터베이스 통합

**장점**:
- Spring 생태계와 완벽 통합
- 코드 일관성 (Spring 스타일)
- 프로덕션 레벨 안정성

### 1.2 Spring AI 아키텍처

```
┌───────────────────────────────────────────────────────┐
│                 Application Layer                     │
│              (Controller, Service)                    │
└────────────────────┬──────────────────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────────────────┐
│                  Spring AI Core                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ ChatClient   │  │PromptTemplate│  │EmbeddingCli│ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
└────────────────────┬──────────────────────────────────┘
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ OpenAI  │ │Hugging  │ │ Ollama  │
    │         │ │  Face   │ │         │
    └─────────┘ └─────────┘ └─────────┘
```

---

## 2. Hugging Face 설정

### 2.1 Hugging Face 계정 생성

1. https://huggingface.co/ 접속
2. `Sign Up` → 계정 생성
3. 프로필 → `Settings` → `Access Tokens`
4. `New token` 클릭:
   - Name: `erp-code-review`
   - Role: `read` (읽기 전용)
   - `Generate a token` 클릭
5. 토큰 복사 (예: `hf_xxxxxxxxxxxxxxxxxxxxx`)

**💡 주의**: 토큰은 한 번만 표시되므로 안전한 곳에 저장

### 2.2 StarCoder2 모델 소개

**StarCoder2**:
- Hugging Face가 개발한 코드 생성/분석 특화 모델
- 600개 이상의 프로그래밍 언어 학습
- 코드 리뷰, 버그 찾기, 리팩토링 제안에 탁월

**모델 선택**:
- `bigcode/starcoder2-15b`: 대형 모델 (고성능, 느림)
- `bigcode/starcoder2-7b`: 중형 모델 (균형) ⭐ 권장
- `bigcode/starcoder2-3b`: 소형 모델 (빠름, 정확도 낮음)

---

## 3. Spring AI 의존성 추가

### 3.1 방법 1: Spring AI BOM 사용 (권장) ⭐

**파일**: `build.gradle`

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

repositories {
    mavenCentral()
    maven { url 'https://repo.spring.io/milestone' }  // Spring AI는 아직 milestone
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.ai:spring-ai-bom:0.8.1"  // 최신 버전 확인
    }
}

dependencies {
    // Spring Boot
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
    implementation 'org.springframework.boot:spring-boot-starter-validation'

    // Spring AI
    implementation 'org.springframework.ai:spring-ai-huggingface'

    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // Test
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**BOM(Bill of Materials) 장점**:
- 버전 충돌 방지
- 일관된 버전 관리

---

### 3.2 방법 2: 직접 버전 지정

```groovy
dependencies {
    implementation 'org.springframework.ai:spring-ai-huggingface:0.8.1'
}
```

**단점**: 버전 직접 관리 필요

---

### 3.3 방법 3: OpenAI 사용 (대안)

Spring AI는 OpenAI도 지원합니다:

```groovy
dependencies {
    implementation 'org.springframework.ai:spring-ai-openai'
}
```

**장점**: 높은 성능, 간편한 API
**단점**: 유료 (API 요금 부과)

**💡 이 프로젝트에서는**: Hugging Face (무료) 사용

---

## 4. 설정 파일 작성

### 4.1 application.yml 설정

**파일**: `src/main/resources/application.yml`

```yaml
server:
  port: 8080

spring:
  data:
    mongodb:
      uri: mongodb://admin:admin123@localhost:27017/erp-db?authSource=admin

  ai:
    huggingface:
      api-key: ${HUGGINGFACE_API_KEY}  # 환경 변수에서 주입
      chat:
        model: bigcode/starcoder2-7b  # 사용할 모델
        options:
          temperature: 0.7  # 창의성 (0.0 ~ 1.0)
          max-tokens: 500   # 최대 응답 길이

# 로깅
logging:
  level:
    org.springframework.ai: DEBUG
    com.erp.codereview: DEBUG
```

**설정 설명**:
- `temperature`: 낮을수록 일관된 답변, 높을수록 창의적
- `max-tokens`: 응답 길이 제한 (토큰 = 대략 단어 수)
- `api-key`: 보안을 위해 환경 변수 사용

### 4.2 환경 변수 설정

**macOS/Linux** (`.zshrc` 또는 `.bashrc`):
```bash
export HUGGINGFACE_API_KEY="hf_xxxxxxxxxxxxxxxxxxxxx"
```

**IntelliJ Run Configuration**:
1. `Run` → `Edit Configurations`
2. `Environment variables` 입력:
   ```
   HUGGINGFACE_API_KEY=hf_xxxxxxxxxxxxxxxxxxxxx
   ```

**💡 보안 주의**: API 키는 절대 Git에 커밋하지 말 것!

---

## 5. LLM 서비스 구현

### 5.1 방법 1: ChatClient 직접 사용 (간단)

**파일**: `src/main/java/com/erp/codereview/codereview/service/LlmService.java`

```java
package com.erp.codereview.codereview.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class LlmService {

    private final ChatClient chatClient;

    /**
     * 간단한 프롬프트 실행
     */
    public String chat(String message) {
        log.info("Sending prompt to LLM: {}", message);

        ChatResponse response = chatClient.call(new Prompt(message));
        String result = response.getResult().getOutput().getContent();

        log.info("LLM response: {}", result);
        return result;
    }
}
```

**장점**: 코드 간결
**단점**: 프롬프트 관리 어려움

---

### 5.2 방법 2: PromptTemplate 사용 (권장) ⭐

**파일**: `src/main/java/com/erp/codereview/codereview/dto/CodeReviewRequest.java`

```java
package com.erp.codereview.codereview.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.validation.constraints.NotBlank;

/**
 * 코드 리뷰 요청 DTO
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
public class CodeReviewRequest {

    @NotBlank(message = "코드는 필수입니다")
    private String code;

    private String language;  // 프로그래밍 언어 (예: "Java")

    private String context;   // 추가 컨텍스트 (예: "재무 시스템")
}
```

---

**파일**: `src/main/java/com/erp/codereview/codereview/service/CodeReviewService.java`

```java
package com.erp.codereview.codereview.service;

import com.erp.codereview.codereview.dto.CodeReviewRequest;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.stereotype.Service;

import java.util.Map;

/**
 * 코드 리뷰 서비스
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class CodeReviewService {

    private final ChatClient chatClient;

    /**
     * 코드 리뷰 수행
     */
    public String reviewCode(CodeReviewRequest request) {
        log.info("Starting code review for {} code", request.getLanguage());

        // 프롬프트 템플릿 정의
        String promptTemplate = """
                You are an expert code reviewer specializing in {language} for financial systems.

                Context: {context}

                Please review the following code and provide:
                1. Potential bugs or issues
                2. Security concerns
                3. Performance improvements
                4. Best practices recommendations

                Code:
                ```{language}
                {code}
                ```

                Provide your review in a structured format.
                """;

        // 변수 바인딩
        PromptTemplate template = new PromptTemplate(promptTemplate);
        Prompt prompt = template.create(Map.of(
                "language", request.getLanguage() != null ? request.getLanguage() : "Java",
                "context", request.getContext() != null ? request.getContext() : "ERP financial system",
                "code", request.getCode()
        ));

        // LLM 호출
        ChatResponse response = chatClient.call(prompt);
        String review = response.getResult().getOutput().getContent();

        log.info("Code review completed");
        return review;
    }

    /**
     * 간단한 코드 설명 생성
     */
    public String explainCode(String code) {
        String promptTemplate = """
                Explain the following code in simple terms:

                ```
                {code}
                ```

                Focus on:
                - What the code does
                - Key logic and algorithms
                - Potential improvements
                """;

        PromptTemplate template = new PromptTemplate(promptTemplate);
        Prompt prompt = template.create(Map.of("code", code));

        ChatResponse response = chatClient.call(prompt);
        return response.getResult().getOutput().getContent();
    }
}
```

**PromptTemplate 장점**:
- 프롬프트 재사용 가능
- 변수 바인딩으로 동적 생성
- 유지보수 용이

---

### 5.3 방법 3: 커스텀 설정 클래스 (고급)

복잡한 설정이 필요한 경우:

**파일**: `src/main/java/com/erp/codereview/common/config/AiConfig.java`

```java
package com.erp.codereview.common.config;

import org.springframework.ai.huggingface.HuggingfaceChatClient;
import org.springframework.ai.huggingface.HuggingfaceChatOptions;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AiConfig {

    @Value("${spring.ai.huggingface.api-key}")
    private String apiKey;

    @Bean
    public HuggingfaceChatClient huggingfaceChatClient() {
        return new HuggingfaceChatClient(
                apiKey,
                HuggingfaceChatOptions.builder()
                        .withModel("bigcode/starcoder2-7b")
                        .withTemperature(0.7)
                        .withMaxTokens(500)
                        .build()
        );
    }
}
```

**사용 시나리오**: 여러 모델 동시 사용, 복잡한 옵션 설정

---

## 6. Controller 구현

**파일**: `src/main/java/com/erp/codereview/codereview/controller/CodeReviewController.java`

```java
package com.erp.codereview.codereview.controller;

import com.erp.codereview.codereview.dto.CodeReviewRequest;
import com.erp.codereview.codereview.service.CodeReviewService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.util.Map;

/**
 * 코드 리뷰 API 컨트롤러
 */
@Slf4j
@RestController
@RequestMapping("/api/code-review")
@RequiredArgsConstructor
public class CodeReviewController {

    private final CodeReviewService codeReviewService;

    /**
     * 코드 리뷰 수행
     */
    @PostMapping("/review")
    public ResponseEntity<Map<String, String>> reviewCode(
            @Valid @RequestBody CodeReviewRequest request) {
        log.info("POST /api/code-review/review - Reviewing code");

        String review = codeReviewService.reviewCode(request);

        return ResponseEntity.ok(Map.of(
                "review", review,
                "language", request.getLanguage() != null ? request.getLanguage() : "Java"
        ));
    }

    /**
     * 코드 설명 생성
     */
    @PostMapping("/explain")
    public ResponseEntity<Map<String, String>> explainCode(
            @RequestBody Map<String, String> request) {
        log.info("POST /api/code-review/explain - Explaining code");

        String code = request.get("code");
        if (code == null || code.isBlank()) {
            return ResponseEntity.badRequest()
                    .body(Map.of("error", "Code is required"));
        }

        String explanation = codeReviewService.explainCode(code);

        return ResponseEntity.ok(Map.of("explanation", explanation));
    }
}
```

---

## 7. 에러 핸들링

### 7.1 LLM 예외 처리

**파일**: `src/main/java/com/erp/codereview/common/exception/GlobalExceptionHandler.java` (수정)

```java
package com.erp.codereview.common.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.retry.NonTransientAiException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 기존 핸들러들...

    /**
     * Spring AI 예외 처리
     */
    @ExceptionHandler(NonTransientAiException.class)
    public ResponseEntity<Map<String, Object>> handleAiException(NonTransientAiException ex) {
        log.error("AI Service Exception: ", ex);

        Map<String, Object> errorResponse = new HashMap<>();
        errorResponse.put("error", "AI Service Error");
        errorResponse.put("message", "Failed to process AI request. Please check API key and model availability.");
        errorResponse.put("timestamp", LocalDateTime.now());

        return ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)
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
        errorResponse.put("message", ex.getMessage());
        errorResponse.put("timestamp", LocalDateTime.now());

        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(errorResponse);
    }
}
```

**주요 예외**:
- `NonTransientAiException`: API 키 오류, 모델 없음
- `TransientAiException`: 일시적 오류 (재시도 가능)

---

## 8. 테스트

### 8.1 코드 리뷰 API 테스트

**파일**: `test-codereview.http`

```http
### 코드 리뷰
POST http://localhost:8080/api/code-review/review
Content-Type: application/json

{
  "code": "public void calculateTotal(List<Invoice> invoices) {\n    double total = 0;\n    for (Invoice inv : invoices) {\n        total = total + inv.getAmount();\n    }\n    return total;\n}",
  "language": "Java",
  "context": "Financial ERP system for calculating invoice totals"
}

### 코드 설명
POST http://localhost:8080/api/code-review/explain
Content-Type: application/json

{
  "code": "public BigDecimal getBalance() {\n    return this.balance.add(BigDecimal.ZERO);\n}"
}
```

### 8.2 curl 테스트

```bash
curl -X POST http://localhost:8080/api/code-review/review \
  -H "Content-Type: application/json" \
  -d '{
    "code": "public void transfer(Account from, Account to, double amount) { from.balance -= amount; to.balance += amount; }",
    "language": "Java",
    "context": "Banking transfer function"
  }'
```

---

## 9. 프롬프트 엔지니어링 기초

### 9.1 효과적인 프롬프트 작성 원칙

1. **명확한 역할 부여**:
```
You are an expert Java developer specializing in financial systems.
```

2. **구체적인 지시**:
```
Provide 3 specific recommendations for improving this code.
```

3. **출력 형식 지정**:
```
Format your response as:
1. Bug: ...
2. Security: ...
3. Performance: ...
```

4. **컨텍스트 제공**:
```
This code is part of an ERP accounting module handling millions of transactions.
```

### 9.2 좋은 프롬프트 vs 나쁜 프롬프트

**❌ 나쁜 예**:
```
Review this code.
```

**✅ 좋은 예**:
```
You are a senior Java developer reviewing code for a financial ERP system.
Analyze the following code for:
1. Potential bugs
2. Security vulnerabilities (especially injection attacks)
3. Performance issues with large datasets
4. Compliance with Java best practices

Provide specific line-by-line feedback.
```

---

## 10. 시퀀스 다이어그램

### 10.1 코드 리뷰 API 흐름

```
Client        Controller      Service       PromptTemplate    ChatClient    Hugging Face
  │               │              │                │               │               │
  │ POST /review  │              │                │               │               │
  ├──────────────>│              │                │               │               │
  │               │ reviewCode() │                │               │               │
  │               ├─────────────>│                │               │               │
  │               │              │ create()       │               │               │
  │               │              ├───────────────>│               │               │
  │               │              │ Prompt         │               │               │
  │               │              │<───────────────┤               │               │
  │               │              │                │ call()        │               │
  │               │              │                ├──────────────>│               │
  │               │              │                │               │ API Request   │
  │               │              │                │               ├──────────────>│
  │               │              │                │               │               │
  │               │              │                │               │  [StarCoder2] │
  │               │              │                │               │               │
  │               │              │                │               │ Response      │
  │               │              │                │               │<──────────────┤
  │               │              │                │ ChatResponse  │               │
  │               │              │                │<──────────────┤               │
  │               │              │ review text    │               │               │
  │               │              │<───────────────┤               │               │
  │               │ review       │                │               │               │
  │               │<─────────────┤                │               │               │
  │ 200 OK + JSON │              │                │               │               │
  │<──────────────┤              │                │               │               │
```

---

## 11. 주의사항 및 모범 사례

### 11.1 API 키 보안

```java
// ❌ 절대 금지
String apiKey = "hf_xxxxxxxxxxxxx";  // 하드코딩

// ✅ 환경 변수 사용
@Value("${spring.ai.huggingface.api-key}")
private String apiKey;
```

### 11.2 Rate Limiting 대응

Hugging Face API는 요청 제한이 있습니다:

```java
@Service
public class RateLimitedLlmService {

    private final RateLimiter rateLimiter = RateLimiter.create(2.0);  // 초당 2회

    public String chat(String message) {
        rateLimiter.acquire();  // Rate limit 적용
        // ... LLM 호출
    }
}
```

**의존성 추가** (Guava):
```groovy
implementation 'com.google.guava:guava:32.1.3-jre'
```

### 11.3 타임아웃 설정

```yaml
spring:
  ai:
    huggingface:
      chat:
        options:
          timeout: 30000  # 30초 타임아웃
```

---

## 12. 체크리스트

- [ ] Hugging Face 계정 생성 및 API 토큰 발급
- [ ] Spring AI 의존성 추가 (`build.gradle`)
- [ ] `application.yml`에 Hugging Face 설정 추가
- [ ] 환경 변수 `HUGGINGFACE_API_KEY` 설정
- [ ] `CodeReviewRequest` DTO 작성
- [ ] `CodeReviewService` 작성 (PromptTemplate 사용)
- [ ] `CodeReviewController` 작성
- [ ] Global Exception Handler에 AI 예외 처리 추가
- [ ] 코드 리뷰 API 테스트 (`POST /api/code-review/review`) 성공
- [ ] 코드 설명 API 테스트 (`POST /api/code-review/explain`) 성공
- [ ] LLM 응답 로그 확인
- [ ] Git 커밋: `"Phase 3: Integrate Spring AI and Hugging Face"`

---

## 13. 다음 단계

Phase 3 완료를 축하합니다! 다음은 **[Phase 4: RAG 구현](./phase-4-rag-implementation.md)**입니다.

Phase 4에서는:
- RAG (Retrieval-Augmented Generation) 패턴 이해
- 임베딩(Embedding) 생성 및 저장
- MongoDB Vector Search 설정
- 컨텍스트 기반 코드 리뷰

를 학습합니다.

---

**🎉 Phase 3 완료! LLM을 Spring Boot에 통합했습니다!**

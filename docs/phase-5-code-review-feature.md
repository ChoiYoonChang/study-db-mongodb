# Phase 5: 코드 리뷰 기능 완성

## 📚 학습 목표

- [ ] 실전 코드 리뷰 시스템 구현
- [ ] 재무/회계 도메인 특화 검증 규칙
- [ ] 리뷰 결과 저장 및 이력 관리
- [ ] 코드 품질 메트릭 분석
- [ ] 통합 테스트 작성
- [ ] 성능 최적화

**예상 소요 시간**: 3 ~ 4시간

---

## 1. 프로젝트 완성 개요

Phase 5는 이전 단계들을 통합하여 완전한 AI 코드 리뷰 시스템을 구축합니다.

### 1.1 완성된 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client (REST API)                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Controller Layer                           │
│  - CodeReviewController                                         │
│  - ReviewHistoryController                                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Service Layer                             │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │AdvancedReviewSvc │  │ RAGService   │  │ ValidationSvc    │  │
│  └──────────────────┘  └──────────────┘  └──────────────────┘  │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ MetricsService   │  │HistoryService│  │ EmbeddingService │  │
│  └──────────────────┘  └──────────────┘  └──────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────┐
│   Spring AI      │ │   MongoDB    │ │ Hugging Face │
│   (RAG Pipeline) │ │   (Vector DB)│ │  StarCoder2  │
└──────────────────┘ └──────────────┘ └──────────────┘
```

---

## 2. 도메인 모델 확장

### 2.1 ReviewResult 엔티티

**파일**: `src/main/java/com/erp/codereview/review/entity/ReviewResult.java`

```java
package com.erp.codereview.review.entity;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import java.time.LocalDateTime;
import java.util.List;

/**
 * 코드 리뷰 결과
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "review_results")
public class ReviewResult {

    @Id
    private String id;

    /**
     * 리뷰 대상 코드
     */
    private String code;

    /**
     * 프로그래밍 언어
     */
    private String language;

    /**
     * 카테고리
     */
    private String category;

    /**
     * AI 리뷰 내용
     */
    private String aiReview;

    /**
     * 발견된 이슈들
     */
    private List<Issue> issues;

    /**
     * 코드 품질 점수 (0-100)
     */
    private Integer qualityScore;

    /**
     * 메트릭
     */
    private CodeMetrics metrics;

    /**
     * 사용된 RAG 컨텍스트
     */
    private String ragContext;

    /**
     * 리뷰 방식 (basic, rag)
     */
    private String reviewMethod;

    /**
     * 생성 일시
     */
    private LocalDateTime createdAt;

    /**
     * 이슈 정의
     */
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class Issue {
        private String type;        // bug, security, performance, style
        private String severity;    // critical, high, medium, low
        private String description;
        private String suggestion;
        private Integer lineNumber;
    }

    /**
     * 코드 메트릭
     */
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class CodeMetrics {
        private Integer linesOfCode;
        private Integer complexity;          // 순환 복잡도
        private Integer maintainabilityIndex;
        private List<String> patterns;       // 감지된 패턴들
    }
}
```

---

### 2.2 Repository

**파일**: `src/main/java/com/erp/codereview/review/repository/ReviewResultRepository.java`

```java
package com.erp.codereview.review.repository;

import com.erp.codereview.review.entity.ReviewResult;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;

@Repository
public interface ReviewResultRepository extends MongoRepository<ReviewResult, String> {

    /**
     * 카테고리별 리뷰 조회
     */
    List<ReviewResult> findByCategory(String category);

    /**
     * 날짜 범위로 조회
     */
    List<ReviewResult> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);

    /**
     * 품질 점수 이하 조회
     */
    List<ReviewResult> findByQualityScoreLessThan(Integer score);

    /**
     * 최근 리뷰 조회
     */
    List<ReviewResult> findTop10ByOrderByCreatedAtDesc();
}
```

---

## 3. 도메인 특화 검증 규칙

### 3.1 Financial Validation Service

**파일**: `src/main/java/com/erp/codereview/validation/service/FinancialValidationService.java`

```java
package com.erp.codereview.validation.service;

import com.erp.codereview.review.entity.ReviewResult.Issue;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.regex.Pattern;

/**
 * 재무/회계 도메인 특화 검증 서비스
 */
@Slf4j
@Service
public class FinancialValidationService {

    // 재무 관련 위험 패턴들
    private static final Pattern DOUBLE_PATTERN = Pattern.compile("\\bdouble\\b");
    private static final Pattern FLOAT_PATTERN = Pattern.compile("\\bfloat\\b");
    private static final Pattern DIVISION_PATTERN = Pattern.compile("/");
    private static final Pattern TRANSACTION_PATTERN = Pattern.compile("@Transactional");

    /**
     * 재무 코드 검증
     */
    public List<Issue> validateFinancialCode(String code, String language) {
        log.info("Validating financial code");
        List<Issue> issues = new ArrayList<>();

        // 1. BigDecimal 사용 검증
        if (language.equalsIgnoreCase("Java")) {
            issues.addAll(checkNumericPrecision(code));
        }

        // 2. Transaction 검증
        issues.addAll(checkTransactionUsage(code));

        // 3. Division by Zero 체크
        issues.addAll(checkDivisionSafety(code));

        // 4. Null 체크
        issues.addAll(checkNullSafety(code));

        log.info("Found {} validation issues", issues.size());
        return issues;
    }

    /**
     * 숫자 정밀도 검증 (BigDecimal 사용 권장)
     */
    private List<Issue> checkNumericPrecision(String code) {
        List<Issue> issues = new ArrayList<>();

        if (DOUBLE_PATTERN.matcher(code).find() || FLOAT_PATTERN.matcher(code).find()) {
            issues.add(Issue.builder()
                    .type("bug")
                    .severity("critical")
                    .description("Using primitive double/float for financial calculations")
                    .suggestion("Use BigDecimal for precise decimal arithmetic in financial calculations")
                    .build());
        }

        return issues;
    }

    /**
     * Transaction 어노테이션 검증
     */
    private List<Issue> checkTransactionUsage(String code) {
        List<Issue> issues = new ArrayList<>();

        // 계좌 이체, 잔액 변경 등의 키워드가 있는데 @Transactional이 없는 경우
        boolean hasFinancialOperation = code.contains("transfer") ||
                code.contains("balance") ||
                code.contains("withdraw") ||
                code.contains("deposit");

        boolean hasTransaction = TRANSACTION_PATTERN.matcher(code).find();

        if (hasFinancialOperation && !hasTransaction) {
            issues.add(Issue.builder()
                    .type("bug")
                    .severity("high")
                    .description("Financial operation without @Transactional annotation")
                    .suggestion("Add @Transactional to ensure atomicity and consistency")
                    .build());
        }

        return issues;
    }

    /**
     * 나눗셈 안전성 검증
     */
    private List<Issue> checkDivisionSafety(String code) {
        List<Issue> issues = new ArrayList<>();

        if (DIVISION_PATTERN.matcher(code).find()) {
            // 간단한 휴리스틱: 나눗셈이 있는데 zero check가 없으면 경고
            if (!code.contains("!= 0") && !code.contains("> 0") && !code.contains("== 0")) {
                issues.add(Issue.builder()
                        .type("bug")
                        .severity("medium")
                        .description("Division operation without zero check")
                        .suggestion("Always validate divisor is not zero before division")
                        .build());
            }
        }

        return issues;
    }

    /**
     * Null 안전성 검증
     */
    private List<Issue> checkNullSafety(String code) {
        List<Issue> issues = new ArrayList<>();

        // getter 메서드 호출이 있는데 null check가 없는 경우
        if (code.contains(".get") && !code.contains("!= null") && !code.contains("== null")) {
            issues.add(Issue.builder()
                    .type("bug")
                    .severity("medium")
                    .description("Potential NullPointerException - missing null checks")
                    .suggestion("Add null validation before accessing object properties")
                    .build());
        }

        return issues;
    }

    /**
     * 품질 점수 계산 (0-100)
     */
    public Integer calculateQualityScore(List<Issue> issues, String code) {
        int baseScore = 100;

        for (Issue issue : issues) {
            switch (issue.getSeverity()) {
                case "critical" -> baseScore -= 25;
                case "high" -> baseScore -= 15;
                case "medium" -> baseScore -= 10;
                case "low" -> baseScore -= 5;
            }
        }

        // 코드 길이 고려 (너무 짧으면 감점)
        int lines = code.split("\n").length;
        if (lines < 5) {
            baseScore -= 10;
        }

        return Math.max(0, baseScore);
    }
}
```

---

## 4. 코드 메트릭 분석

### 4.1 Metrics Service

**파일**: `src/main/java/com/erp/codereview/metrics/service/MetricsService.java`

```java
package com.erp.codereview.metrics.service;

import com.erp.codereview.review.entity.ReviewResult.CodeMetrics;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * 코드 메트릭 분석 서비스
 */
@Slf4j
@Service
public class MetricsService {

    private static final Pattern IF_PATTERN = Pattern.compile("\\bif\\b");
    private static final Pattern LOOP_PATTERN = Pattern.compile("\\b(for|while|do)\\b");
    private static final Pattern SWITCH_PATTERN = Pattern.compile("\\bswitch\\b");
    private static final Pattern METHOD_PATTERN = Pattern.compile("(public|private|protected)\\s+\\w+\\s+\\w+\\s*\\(");

    /**
     * 코드 메트릭 분석
     */
    public CodeMetrics analyzeCode(String code) {
        log.debug("Analyzing code metrics");

        return CodeMetrics.builder()
                .linesOfCode(countLinesOfCode(code))
                .complexity(calculateComplexity(code))
                .maintainabilityIndex(calculateMaintainability(code))
                .patterns(detectPatterns(code))
                .build();
    }

    /**
     * 실제 코드 라인 수 (주석, 빈 줄 제외)
     */
    private Integer countLinesOfCode(String code) {
        return (int) code.lines()
                .map(String::trim)
                .filter(line -> !line.isEmpty())
                .filter(line -> !line.startsWith("//"))
                .filter(line -> !line.startsWith("/*"))
                .filter(line -> !line.startsWith("*"))
                .count();
    }

    /**
     * 순환 복잡도 계산 (Cyclomatic Complexity)
     */
    private Integer calculateComplexity(String code) {
        int complexity = 1;  // 기본 복잡도

        // if, else if 카운트
        Matcher ifMatcher = IF_PATTERN.matcher(code);
        while (ifMatcher.find()) {
            complexity++;
        }

        // 루프 카운트
        Matcher loopMatcher = LOOP_PATTERN.matcher(code);
        while (loopMatcher.find()) {
            complexity++;
        }

        // switch case 카운트
        Matcher switchMatcher = SWITCH_PATTERN.matcher(code);
        while (switchMatcher.find()) {
            complexity++;
        }

        return complexity;
    }

    /**
     * 유지보수성 지수 (0-100)
     */
    private Integer calculateMaintainability(String code) {
        int loc = countLinesOfCode(code);
        int complexity = calculateComplexity(code);

        // 간단한 휴리스틱 기반 계산
        int score = 100;

        // 복잡도 페널티
        if (complexity > 10) score -= 20;
        else if (complexity > 7) score -= 10;
        else if (complexity > 4) score -= 5;

        // 코드 길이 페널티
        if (loc > 100) score -= 15;
        else if (loc > 50) score -= 10;

        // 메서드 개수 (많을수록 모듈화 잘됨)
        Matcher methodMatcher = METHOD_PATTERN.matcher(code);
        int methodCount = 0;
        while (methodMatcher.find()) {
            methodCount++;
        }

        if (methodCount > 1) score += 5;  // 보너스

        return Math.max(0, Math.min(100, score));
    }

    /**
     * 디자인 패턴 감지
     */
    private List<String> detectPatterns(String code) {
        List<String> patterns = new ArrayList<>();

        if (code.contains("Builder")) patterns.add("Builder Pattern");
        if (code.contains("Factory")) patterns.add("Factory Pattern");
        if (code.contains("Singleton")) patterns.add("Singleton Pattern");
        if (code.contains("@Transactional")) patterns.add("Transaction Management");
        if (code.contains("Stream") || code.contains("stream()")) patterns.add("Functional Programming");
        if (code.contains("Optional")) patterns.add("Optional Pattern");

        return patterns;
    }
}
```

---

## 5. 고급 리뷰 서비스

### 5.1 Advanced Review Service

**파일**: `src/main/java/com/erp/codereview/review/service/AdvancedReviewService.java`

```java
package com.erp.codereview.review.service;

import com.erp.codereview.codereview.dto.CodeReviewRequest;
import com.erp.codereview.metrics.service.MetricsService;
import com.erp.codereview.rag.service.RagService;
import com.erp.codereview.review.entity.ReviewResult;
import com.erp.codereview.review.entity.ReviewResult.CodeMetrics;
import com.erp.codereview.review.entity.ReviewResult.Issue;
import com.erp.codereview.review.repository.ReviewResultRepository;
import com.erp.codereview.validation.service.FinancialValidationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;

/**
 * 고급 코드 리뷰 서비스 (검증 + 메트릭 + RAG 통합)
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AdvancedReviewService {

    private final RagService ragService;
    private final FinancialValidationService validationService;
    private final MetricsService metricsService;
    private final ReviewResultRepository reviewResultRepository;

    /**
     * 종합 코드 리뷰 수행
     */
    public ReviewResult performComprehensiveReview(CodeReviewRequest request) {
        log.info("Starting comprehensive code review");

        // 1. 도메인 검증 (재무/회계 규칙)
        List<Issue> issues = validationService.validateFinancialCode(
                request.getCode(),
                request.getLanguage()
        );

        // 2. 코드 메트릭 분석
        CodeMetrics metrics = metricsService.analyzeCode(request.getCode());

        // 3. AI 리뷰 (RAG 기반)
        String aiReview = ragService.reviewCodeWithRag(request);

        // 4. 품질 점수 계산
        Integer qualityScore = validationService.calculateQualityScore(issues, request.getCode());

        // 5. 결과 저장
        ReviewResult result = ReviewResult.builder()
                .code(request.getCode())
                .language(request.getLanguage())
                .category(request.getContext() != null && request.getContext().contains("financial") ? "financial" : "general")
                .aiReview(aiReview)
                .issues(issues)
                .qualityScore(qualityScore)
                .metrics(metrics)
                .reviewMethod("comprehensive")
                .createdAt(LocalDateTime.now())
                .build();

        ReviewResult saved = reviewResultRepository.save(result);
        log.info("Saved review result with ID: {}", saved.getId());

        return saved;
    }

    /**
     * 리뷰 이력 조회
     */
    public List<ReviewResult> getRecentReviews(int limit) {
        return reviewResultRepository.findTop10ByOrderByCreatedAtDesc();
    }

    /**
     * 품질 점수 낮은 리뷰 조회
     */
    public List<ReviewResult> getLowQualityReviews(int threshold) {
        return reviewResultRepository.findByQualityScoreLessThan(threshold);
    }
}
```

---

## 6. Controller 완성

### 6.1 Review Controller (최종)

**파일**: `src/main/java/com/erp/codereview/codereview/controller/CodeReviewController.java` (최종)

```java
package com.erp.codereview.codereview.controller;

import com.erp.codereview.codereview.dto.CodeReviewRequest;
import com.erp.codereview.review.entity.ReviewResult;
import com.erp.codereview.review.service.AdvancedReviewService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.util.List;

/**
 * 코드 리뷰 API 컨트롤러 (완성)
 */
@Slf4j
@RestController
@RequestMapping("/api/code-review")
@RequiredArgsConstructor
public class CodeReviewController {

    private final AdvancedReviewService advancedReviewService;

    /**
     * 종합 코드 리뷰 (권장)
     */
    @PostMapping("/comprehensive")
    public ResponseEntity<ReviewResult> comprehensiveReview(
            @Valid @RequestBody CodeReviewRequest request) {
        log.info("POST /api/code-review/comprehensive - Comprehensive review");

        ReviewResult result = advancedReviewService.performComprehensiveReview(request);

        return ResponseEntity.ok(result);
    }

    /**
     * 최근 리뷰 조회
     */
    @GetMapping("/recent")
    public ResponseEntity<List<ReviewResult>> getRecentReviews(
            @RequestParam(defaultValue = "10") int limit) {
        log.info("GET /api/code-review/recent - Fetching recent reviews");

        List<ReviewResult> reviews = advancedReviewService.getRecentReviews(limit);

        return ResponseEntity.ok(reviews);
    }

    /**
     * 품질 낮은 코드 조회
     */
    @GetMapping("/low-quality")
    public ResponseEntity<List<ReviewResult>> getLowQualityReviews(
            @RequestParam(defaultValue = "60") int threshold) {
        log.info("GET /api/code-review/low-quality - Threshold: {}", threshold);

        List<ReviewResult> reviews = advancedReviewService.getLowQualityReviews(threshold);

        return ResponseEntity.ok(reviews);
    }

    /**
     * 리뷰 상세 조회
     */
    @GetMapping("/{id}")
    public ResponseEntity<ReviewResult> getReview(@PathVariable String id) {
        log.info("GET /api/code-review/{} - Fetching review", id);

        // ReviewResultRepository 주입 필요
        // ReviewResult review = reviewResultRepository.findById(id)...

        return ResponseEntity.ok(null);  // TODO: 구현
    }
}
```

---

## 7. 통합 테스트

### 7.1 Service 통합 테스트

**파일**: `src/test/java/com/erp/codereview/review/service/AdvancedReviewServiceTest.java`

```java
package com.erp.codereview.review.service;

import com.erp.codereview.codereview.dto.CodeReviewRequest;
import com.erp.codereview.review.entity.ReviewResult;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.assertj.core.assertions.Assertj.assertThat;

@SpringBootTest
class AdvancedReviewServiceTest {

    @Autowired
    private AdvancedReviewService advancedReviewService;

    @Test
    void testComprehensiveReview_WithFinancialCode() {
        // Given
        CodeReviewRequest request = new CodeReviewRequest();
        request.setCode("""
                public void transfer(Account from, Account to, double amount) {
                    from.balance -= amount;
                    to.balance += amount;
                }
                """);
        request.setLanguage("Java");
        request.setContext("Financial ERP system");

        // When
        ReviewResult result = advancedReviewService.performComprehensiveReview(request);

        // Then
        assertThat(result).isNotNull();
        assertThat(result.getIssues()).isNotEmpty();
        assertThat(result.getQualityScore()).isLessThan(100);
        assertThat(result.getMetrics()).isNotNull();
        assertThat(result.getMetrics().getComplexity()).isGreaterThan(0);
    }

    @Test
    void testComprehensiveReview_WithGoodCode() {
        // Given
        CodeReviewRequest request = new CodeReviewRequest();
        request.setCode("""
                @Transactional
                public void transfer(Account from, Account to, BigDecimal amount) {
                    if (from.getBalance().compareTo(amount) < 0) {
                        throw new InsufficientFundsException();
                    }
                    from.setBalance(from.getBalance().subtract(amount));
                    to.setBalance(to.getBalance().add(amount));
                }
                """);
        request.setLanguage("Java");
        request.setContext("Financial ERP system");

        // When
        ReviewResult result = advancedReviewService.performComprehensiveReview(request);

        // Then
        assertThat(result).isNotNull();
        assertThat(result.getQualityScore()).isGreaterThan(80);  // 좋은 코드
    }
}
```

---

### 7.2 API 통합 테스트

**파일**: `test-comprehensive.http`

```http
### 종합 코드 리뷰 - 나쁜 코드
POST http://localhost:8080/api/code-review/comprehensive
Content-Type: application/json

{
  "code": "public void calculateInterest(double principal, double rate) {\n    double interest = principal * rate / 100;\n    System.out.println(interest);\n}",
  "language": "Java",
  "context": "Financial interest calculation"
}

### 종합 코드 리뷰 - 좋은 코드
POST http://localhost:8080/api/code-review/comprehensive
Content-Type: application/json

{
  "code": "@Transactional\npublic BigDecimal calculateInterest(BigDecimal principal, BigDecimal rate) {\n    if (principal == null || rate == null) {\n        throw new IllegalArgumentException(\"Principal and rate must not be null\");\n    }\n    return principal.multiply(rate).divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);\n}",
  "language": "Java",
  "context": "Financial interest calculation"
}

### 최근 리뷰 조회
GET http://localhost:8080/api/code-review/recent?limit=5

### 품질 낮은 코드 조회
GET http://localhost:8080/api/code-review/low-quality?threshold=70
```

---

## 8. 성능 최적화

### 8.1 Caching 추가

**파일**: `src/main/java/com/erp/codereview/common/config/CacheConfig.java`

```java
package com.erp.codereview.common.config;

import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;

import java.time.Duration;

@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))  // 10분 TTL
                .disableCachingNullValues();

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(config)
                .build();
    }
}
```

**Embedding Service에 캐싱 적용**:

```java
@Service
public class EmbeddingService {

    @Cacheable(value = "embeddings", key = "#text")
    public List<Double> createEmbedding(String text) {
        // ... 기존 코드
    }
}
```

**build.gradle에 의존성 추가**:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
implementation 'org.springframework.boot:spring-boot-starter-cache'
```

---

### 8.2 비동기 처리

**파일**: `src/main/java/com/erp/codereview/common/config/AsyncConfig.java`

```java
package com.erp.codereview.common.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "reviewExecutor")
    public Executor reviewExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("review-");
        executor.initialize();
        return executor;
    }
}
```

**비동기 리뷰 메서드**:

```java
@Service
public class AdvancedReviewService {

    @Async("reviewExecutor")
    public CompletableFuture<ReviewResult> performAsyncReview(CodeReviewRequest request) {
        ReviewResult result = performComprehensiveReview(request);
        return CompletableFuture.completedFuture(result);
    }
}
```

---

## 9. API 문서화 (Swagger/OpenAPI)

**파일**: `src/main/java/com/erp/codereview/common/config/OpenApiConfig.java`

```java
package com.erp.codereview.common.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("ERP Code Review API")
                        .version("1.0")
                        .description("AI-powered code review system for financial ERP systems using Spring AI and RAG"));
    }
}
```

**build.gradle에 의존성 추가**:

```groovy
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.2.0'
```

**접속**: http://localhost:8080/swagger-ui/index.html

---

## 10. 최종 시퀀스 다이어그램

```
Client         Controller     AdvancedReviewSvc  ValidationSvc  MetricsSvc  RagService  Repository
  │                │                  │                │             │           │           │
  │ POST /comprehensive               │                │             │           │           │
  ├───────────────>│                  │                │             │           │           │
  │                │ performComprehensiveReview()      │             │           │           │
  │                ├─────────────────>│                │             │           │           │
  │                │                  │ validateFinancialCode()      │           │           │
  │                │                  ├───────────────>│             │           │           │
  │                │                  │                │ [Check]     │           │           │
  │                │                  │                │ - BigDecimal│           │           │
  │                │                  │                │ - @Transact │           │           │
  │                │                  │                │ - Null safe │           │           │
  │                │                  │ List<Issue>    │             │           │           │
  │                │                  │<───────────────┤             │           │           │
  │                │                  │                              │           │           │
  │                │                  │ analyzeCode()                │           │           │
  │                │                  ├─────────────────────────────>│           │           │
  │                │                  │                              │ [Analyze] │           │
  │                │                  │                              │ - LOC     │           │
  │                │                  │                              │ - Complexity          │
  │                │                  │                              │ - Patterns│           │
  │                │                  │ CodeMetrics                  │           │           │
  │                │                  │<─────────────────────────────┤           │           │
  │                │                  │                                          │           │
  │                │                  │ reviewCodeWithRag()                      │           │
  │                │                  ├─────────────────────────────────────────>│           │
  │                │                  │                                          │ [RAG]     │
  │                │                  │                                          │ - Search  │
  │                │                  │                                          │ - LLM call│
  │                │                  │ AI Review                                │           │
  │                │                  │<─────────────────────────────────────────┤           │
  │                │                  │                                                      │
  │                │                  │ calculateQualityScore()      │                       │
  │                │                  ├───────────────>│             │                       │
  │                │                  │ score          │             │                       │
  │                │                  │<───────────────┤             │                       │
  │                │                  │                                                      │
  │                │                  │ save(ReviewResult)                                   │
  │                │                  ├─────────────────────────────────────────────────────>│
  │                │                  │                                                      │ [Save]
  │                │                  │ Saved ReviewResult                                   │
  │                │                  │<─────────────────────────────────────────────────────┤
  │                │ ReviewResult     │                                                      │
  │                │<─────────────────┤                                                      │
  │ 200 OK (JSON)  │                  │                                                      │
  │<───────────────┤                  │                                                      │
```

---

## 11. 배포 준비

### 11.1 프로덕션 설정

**파일**: `src/main/resources/application-prod.yml`

```yaml
server:
  port: 8080

spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}  # 환경 변수

  ai:
    huggingface:
      api-key: ${HUGGINGFACE_API_KEY}
      chat:
        model: bigcode/starcoder2-7b
        options:
          temperature: 0.5  # 프로덕션에서는 더 일관된 응답
          max-tokens: 500
          timeout: 30000

logging:
  level:
    root: INFO
    com.erp.codereview: INFO
  file:
    name: logs/erp-codereview.log
```

### 11.2 Dockerfile

**파일**: `Dockerfile`

```dockerfile
FROM openjdk:17-jdk-slim

WORKDIR /app

COPY build/libs/*.jar app.jar

EXPOSE 8080

ENV JAVA_OPTS=""

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### 11.3 Docker Compose

**파일**: `docker-compose.yml`

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/erp-db
      - HUGGINGFACE_API_KEY=${HUGGINGFACE_API_KEY}
      - SPRING_PROFILES_ACTIVE=prod
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

---

## 12. 체크리스트

- [ ] `ReviewResult` 엔티티 및 Repository 작성
- [ ] `FinancialValidationService` 구현 (BigDecimal, @Transactional 검증)
- [ ] `MetricsService` 구현 (LOC, 복잡도, 유지보수성)
- [ ] `AdvancedReviewService` 구현 (통합 리뷰)
- [ ] Controller에 `/comprehensive` 엔드포인트 추가
- [ ] 통합 테스트 작성 및 실행
- [ ] Caching 설정 (Redis)
- [ ] 비동기 처리 설정
- [ ] Swagger/OpenAPI 문서화
- [ ] 프로덕션 설정 파일 작성
- [ ] Dockerfile 및 Docker Compose 작성
- [ ] 최종 테스트 (나쁜 코드 vs 좋은 코드)
- [ ] Git 커밋: `"Phase 5: Complete code review system with validation and metrics"`

---

## 13. 확장 아이디어

프로젝트를 더 발전시키고 싶다면:

### 13.1 Frontend 추가
- React/Next.js로 웹 UI 구현
- 코드 에디터 통합 (Monaco Editor)
- 실시간 리뷰 결과 표시

### 13.2 고급 기능
- [ ] 여러 LLM 모델 비교 (GPT-4, Claude, Gemini)
- [ ] Fine-tuning: 자체 데이터로 모델 학습
- [ ] CI/CD 통합 (GitHub Actions, Jenkins)
- [ ] 멀티 테넌시 지원 (팀별 격리)
- [ ] 코드 품질 대시보드
- [ ] Slack/Teams 알림 통합

### 13.3 다른 도메인 적용
- 의료 시스템 (HIPAA 규정 검증)
- 보안 시스템 (OWASP 규칙 검증)
- IoT 시스템 (성능 최적화 검증)

---

## 14. 학습 정리

### 14.1 배운 내용

✅ **Phase 0**: IntelliJ 환경 설정
✅ **Phase 1**: Spring Boot 기본 구조
✅ **Phase 2**: MongoDB 연동 및 도메인 모델
✅ **Phase 3**: Spring AI + LLM 연동
✅ **Phase 4**: RAG 패턴 구현
✅ **Phase 5**: 종합 코드 리뷰 시스템

### 14.2 핵심 기술

- **Spring AI**: LLM 통합 프레임워크
- **RAG**: 검색 증강 생성 패턴
- **Vector DB**: MongoDB Vector Search
- **Embedding**: 의미론적 검색
- **도메인 검증**: 재무/회계 특화 규칙
- **코드 메트릭**: 품질 측정

### 14.3 실무 적용

이 프로젝트에서 배운 패턴은:
- **코드 리뷰 자동화**
- **기술 문서 QA 시스템**
- **고객 지원 챗봇** (도메인 지식 기반)
- **스마트 검색 엔진**

에 그대로 적용 가능합니다.

---

## 15. 최종 프로젝트 구조

```
erp-code-review/
├── src/
│   ├── main/
│   │   ├── java/com/erp/codereview/
│   │   │   ├── codereview/
│   │   │   │   ├── controller/CodeReviewController.java
│   │   │   │   ├── dto/CodeReviewRequest.java
│   │   │   │   └── service/CodeReviewService.java
│   │   │   ├── document/
│   │   │   │   ├── entity/CodeSnippet.java
│   │   │   │   ├── repository/CodeSnippetRepository.java
│   │   │   │   └── service/DocumentService.java
│   │   │   ├── embedding/
│   │   │   │   └── service/EmbeddingService.java
│   │   │   ├── search/
│   │   │   │   └── service/VectorSearchService.java
│   │   │   ├── rag/
│   │   │   │   └── service/RagService.java
│   │   │   ├── review/
│   │   │   │   ├── entity/ReviewResult.java
│   │   │   │   ├── repository/ReviewResultRepository.java
│   │   │   │   └── service/AdvancedReviewService.java
│   │   │   ├── validation/
│   │   │   │   └── service/FinancialValidationService.java
│   │   │   ├── metrics/
│   │   │   │   └── service/MetricsService.java
│   │   │   └── common/
│   │   │       ├── config/
│   │   │       │   ├── AiConfig.java
│   │   │       │   ├── AsyncConfig.java
│   │   │       │   ├── CacheConfig.java
│   │   │       │   ├── OpenApiConfig.java
│   │   │       │   └── InitialDataLoader.java
│   │   │       └── exception/
│   │   │           └── GlobalExceptionHandler.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── application-prod.yml
│   └── test/
│       └── java/com/erp/codereview/
│           └── review/service/AdvancedReviewServiceTest.java
├── docs/
│   ├── INDEX.md
│   ├── phase-0-environment-setup.md
│   ├── phase-1-spring-boot-basics.md
│   ├── phase-2-mongodb-domain.md
│   ├── phase-3-spring-ai-llm.md
│   ├── phase-4-rag-implementation.md
│   └── phase-5-code-review-feature.md
├── Dockerfile
├── docker-compose.yml
├── build.gradle
└── README.md
```

---

## 🎉 축하합니다!

**Phase 5를 완료했습니다!**

이제 당신은:
- ✅ Spring AI 프레임워크를 마스터했습니다
- ✅ RAG 패턴을 실전에 적용할 수 있습니다
- ✅ 벡터 데이터베이스를 활용할 수 있습니다
- ✅ 도메인 특화 AI 시스템을 구축할 수 있습니다
- ✅ 프로덕션 레벨 코드 리뷰 시스템을 개발했습니다

**다음 단계**:
1. 이력서에 프로젝트 추가
2. GitHub에 공개하여 포트폴리오 구축
3. 실무 프로젝트에 RAG 패턴 적용
4. 다른 도메인으로 확장 (의료, 보안, IoT 등)

**Happy Coding! 🚀**

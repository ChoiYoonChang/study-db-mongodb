# Phase 4: RAG (Retrieval-Augmented Generation) 구현

## 📚 학습 목표

- [ ] RAG 패턴의 개념과 필요성 이해
- [ ] 임베딩(Embedding)의 원리 학습
- [ ] MongoDB Vector Search 설정
- [ ] Document Store 구현
- [ ] 벡터 검색 기반 컨텍스트 검색
- [ ] RAG 파이프라인 통합

**예상 소요 시간**: 3 ~ 4시간

---

## 1. RAG란 무엇인가?

### 1.1 RAG 개념

**RAG (Retrieval-Augmented Generation)**는 LLM의 한계를 극복하기 위한 패턴입니다.

**LLM의 문제점**:
- 학습 데이터 이후의 정보 부족
- 환각(Hallucination): 잘못된 정보 생성
- 도메인 특화 지식 부족

**RAG의 해결 방법**:
1. **검색(Retrieval)**: 관련 문서/코드를 데이터베이스에서 검색
2. **증강(Augmented)**: 검색된 정보를 프롬프트에 추가
3. **생성(Generation)**: 컨텍스트가 풍부한 프롬프트로 LLM 응답 생성

### 1.2 RAG vs Fine-tuning

| 비교 항목 | RAG | Fine-tuning |
|---------|-----|-------------|
| **비용** | 저렴 (추론만 필요) | 비싸다 (재학습 필요) |
| **속도** | 빠름 (즉시 적용) | 느림 (학습 시간 필요) |
| **업데이트** | 쉬움 (문서만 추가) | 어려움 (재학습) |
| **정확도** | 출처 추적 가능 | 블랙박스 |
| **적용 사례** | 문서 기반 QA | 스타일/톤 변경 |

**이 프로젝트에서는**: RAG 사용 (재무/회계 코드 예제 기반)

### 1.3 RAG 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    1. Indexing Phase                        │
│                   (사전 준비 - 한 번만)                       │
│                                                             │
│  Source Code/Docs  →  Text Splitter  →  Embedding Model    │
│                                            ↓                │
│                                      Vector Database        │
│                                       (MongoDB)             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    2. Query Phase                           │
│                   (사용자 요청 시마다)                        │
│                                                             │
│  User Query  →  Embedding  →  Vector Search                │
│                                     ↓                       │
│                               Relevant Docs                 │
│                                     ↓                       │
│                          LLM (with context)                 │
│                                     ↓                       │
│                               Response                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Embedding (임베딩) 이해

### 2.1 임베딩이란?

**Embedding**: 텍스트를 숫자 벡터로 변환하는 기술

**예시**:
```
텍스트: "Calculate invoice total"
임베딩: [0.23, -0.45, 0.89, 0.12, ..., 0.56]  (768차원 벡터)
```

**의미론적 유사도**:
- 비슷한 의미 → 벡터 간 거리가 가까움
- 다른 의미 → 벡터 간 거리가 멀음

### 2.2 임베딩 모델 선택

| 모델 | 차원 | 속도 | 정확도 | 추천 |
|-----|-----|-----|--------|------|
| `sentence-transformers/all-MiniLM-L6-v2` | 384 | 빠름 | 중간 | ⭐ 권장 |
| `sentence-transformers/all-mpnet-base-v2` | 768 | 보통 | 높음 | 고성능 |
| OpenAI `text-embedding-ada-002` | 1536 | 보통 | 매우 높음 | 유료 |

**이 프로젝트에서는**: `all-MiniLM-L6-v2` (무료, 빠름)

---

## 3. MongoDB Vector Search 설정

### 3.1 MongoDB Atlas 설정 (클라우드)

**방법 1: MongoDB Atlas 사용 (권장)** ⭐

1. https://www.mongodb.com/cloud/atlas 접속
2. Free Tier 계정 생성
3. Cluster 생성 (M0 - 무료)
4. Database Access → Add User
5. Network Access → Add IP (0.0.0.0/0 - 모든 IP 허용)
6. Connect → Drivers → Connection String 복사

**Connection String 예시**:
```
mongodb+srv://username:password@cluster0.xxxxx.mongodb.net/erp-db?retryWrites=true&w=majority
```

**application.yml 수정**:
```yaml
spring:
  data:
    mongodb:
      uri: mongodb+srv://username:password@cluster0.xxxxx.mongodb.net/erp-db?retryWrites=true&w=majority
```

### 3.2 Vector Search Index 생성

MongoDB Atlas에서:

1. Database → Browse Collections
2. `code_snippets` 컬렉션 선택
3. **Search Indexes** 탭 → **Create Search Index**
4. **JSON Editor** 선택:

```json
{
  "mappings": {
    "dynamic": true,
    "fields": {
      "embedding": {
        "type": "knnVector",
        "dimensions": 384,
        "similarity": "cosine"
      }
    }
  }
}
```

5. Index Name: `vector_index`
6. Create

**💡 주의**: Index 생성에 5~10분 소요

---

### 3.3 방법 2: 로컬 MongoDB (제한적)

로컬 MongoDB는 Vector Search를 **지원하지 않습니다**.

**대안**:
- MongoDB Atlas 사용 (무료 M0)
- 또는 다른 Vector DB (Pinecone, Weaviate)

---

## 4. Document 모델 설계

### 4.1 CodeSnippet 엔티티

**파일**: `src/main/java/com/erp/codereview/document/entity/CodeSnippet.java`

```java
package com.erp.codereview.document.entity;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import java.time.LocalDateTime;
import java.util.List;

/**
 * 코드 스니펫 문서 (RAG용)
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "code_snippets")
public class CodeSnippet {

    @Id
    private String id;

    /**
     * 코드 내용
     */
    private String code;

    /**
     * 코드 설명
     */
    private String description;

    /**
     * 프로그래밍 언어
     */
    private String language;

    /**
     * 카테고리 (예: "financial", "accounting")
     */
    private String category;

    /**
     * 태그들
     */
    private List<String> tags;

    /**
     * 임베딩 벡터 (384차원)
     */
    private List<Double> embedding;

    /**
     * 메타데이터
     */
    private String metadata;

    /**
     * 생성 일시
     */
    private LocalDateTime createdAt;

    /**
     * 수정 일시
     */
    private LocalDateTime updatedAt;
}
```

---

### 4.2 Repository

**파일**: `src/main/java/com/erp/codereview/document/repository/CodeSnippetRepository.java`

```java
package com.erp.codereview.document.repository;

import com.erp.codereview.document.entity.CodeSnippet;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface CodeSnippetRepository extends MongoRepository<CodeSnippet, String> {

    /**
     * 카테고리로 검색
     */
    List<CodeSnippet> findByCategory(String category);

    /**
     * 언어로 검색
     */
    List<CodeSnippet> findByLanguage(String language);

    /**
     * 태그 포함 검색
     */
    List<CodeSnippet> findByTagsContaining(String tag);
}
```

---

## 5. Embedding Service 구현

### 5.1 방법 1: Spring AI EmbeddingClient 사용 (권장) ⭐

**파일**: `src/main/java/com/erp/codereview/embedding/service/EmbeddingService.java`

```java
package com.erp.codereview.embedding.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.ai.embedding.EmbeddingRequest;
import org.springframework.ai.embedding.EmbeddingResponse;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

/**
 * 임베딩 생성 서비스
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class EmbeddingService {

    private final EmbeddingClient embeddingClient;

    /**
     * 텍스트를 임베딩 벡터로 변환
     */
    public List<Double> createEmbedding(String text) {
        log.debug("Creating embedding for text: {}", text.substring(0, Math.min(50, text.length())));

        EmbeddingRequest request = new EmbeddingRequest(List.of(text), null);
        EmbeddingResponse response = embeddingClient.call(request);

        return response.getResults().get(0).getOutput().stream()
                .map(Float::doubleValue)
                .collect(Collectors.toList());
    }

    /**
     * 여러 텍스트를 한 번에 임베딩
     */
    public List<List<Double>> createEmbeddings(List<String> texts) {
        log.debug("Creating embeddings for {} texts", texts.size());

        EmbeddingRequest request = new EmbeddingRequest(texts, null);
        EmbeddingResponse response = embeddingClient.call(request);

        return response.getResults().stream()
                .map(result -> result.getOutput().stream()
                        .map(Float::doubleValue)
                        .collect(Collectors.toList()))
                .collect(Collectors.toList());
    }
}
```

**application.yml에 추가**:
```yaml
spring:
  ai:
    embedding:
      client: huggingface
      huggingface:
        model: sentence-transformers/all-MiniLM-L6-v2
```

**build.gradle에 의존성 추가**:
```groovy
implementation 'org.springframework.ai:spring-ai-huggingface-embedding'
```

---

### 5.2 방법 2: 외부 API 직접 호출

Hugging Face Inference API 직접 호출:

```java
@Service
public class HuggingFaceEmbeddingService {

    @Value("${spring.ai.huggingface.api-key}")
    private String apiKey;

    private final RestTemplate restTemplate = new RestTemplate();

    public List<Double> createEmbedding(String text) {
        String url = "https://api-inference.huggingface.co/models/sentence-transformers/all-MiniLM-L6-v2";

        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + apiKey);
        headers.setContentType(MediaType.APPLICATION_JSON);

        Map<String, Object> requestBody = Map.of("inputs", text);

        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(requestBody, headers);
        ResponseEntity<List> response = restTemplate.postForEntity(url, entity, List.class);

        return (List<Double>) response.getBody();
    }
}
```

**장점**: 직접 제어
**단점**: 에러 처리, 재시도 로직 직접 구현

---

### 5.3 방법 3: 로컬 모델 사용 (고급)

DJL(Deep Java Library)로 로컬 모델 실행:

```groovy
// build.gradle
implementation 'ai.djl:api:0.24.0'
implementation 'ai.djl.huggingface:tokenizers:0.24.0'
```

```java
// 로컬 임베딩 (네트워크 불필요)
@Service
public class LocalEmbeddingService {
    // DJL 모델 로드 및 추론
}
```

**장점**: 빠름, 무료, 네트워크 불필요
**단점**: 메모리 사용량 높음

---

## 6. Vector Search 구현

### 6.1 Vector Search Service

**파일**: `src/main/java/com/erp/codereview/search/service/VectorSearchService.java`

```java
package com.erp.codereview.search.service;

import com.erp.codereview.document.entity.CodeSnippet;
import com.erp.codereview.embedding.service.EmbeddingService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.aggregation.Aggregation;
import org.springframework.data.mongodb.core.aggregation.AggregationOperation;
import org.springframework.stereotype.Service;

import java.util.List;

/**
 * 벡터 검색 서비스
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class VectorSearchService {

    private final MongoTemplate mongoTemplate;
    private final EmbeddingService embeddingService;

    /**
     * 의미론적 유사도 기반 검색
     *
     * @param query 검색 쿼리
     * @param limit 결과 개수
     * @return 유사한 코드 스니펫들
     */
    public List<CodeSnippet> searchSimilar(String query, int limit) {
        log.info("Vector search for query: {}", query);

        // 1. 쿼리를 임베딩으로 변환
        List<Double> queryEmbedding = embeddingService.createEmbedding(query);

        // 2. MongoDB Vector Search Aggregation
        AggregationOperation vectorSearch = context -> {
            return context.getMappedObject(org.bson.Document.parse("""
                {
                    "$vectorSearch": {
                        "index": "vector_index",
                        "path": "embedding",
                        "queryVector": %s,
                        "numCandidates": 100,
                        "limit": %d
                    }
                }
                """.formatted(queryEmbedding.toString(), limit)));
        };

        // 3. 점수 계산 (유사도)
        AggregationOperation addScore = context -> {
            return context.getMappedObject(org.bson.Document.parse("""
                {
                    "$addFields": {
                        "score": { "$meta": "vectorSearchScore" }
                    }
                }
                """));
        };

        Aggregation aggregation = Aggregation.newAggregation(
                vectorSearch,
                addScore
        );

        // 4. 실행
        List<CodeSnippet> results = mongoTemplate.aggregate(
                aggregation,
                "code_snippets",
                CodeSnippet.class
        ).getMappedResults();

        log.info("Found {} similar code snippets", results.size());
        return results;
    }

    /**
     * 카테고리 필터링과 함께 검색
     */
    public List<CodeSnippet> searchSimilarByCategory(String query, String category, int limit) {
        log.info("Vector search with category filter: {}", category);

        List<Double> queryEmbedding = embeddingService.createEmbedding(query);

        AggregationOperation vectorSearch = context -> {
            return context.getMappedObject(org.bson.Document.parse("""
                {
                    "$vectorSearch": {
                        "index": "vector_index",
                        "path": "embedding",
                        "queryVector": %s,
                        "numCandidates": 100,
                        "limit": %d,
                        "filter": { "category": "%s" }
                    }
                }
                """.formatted(queryEmbedding.toString(), limit, category)));
        };

        AggregationOperation addScore = context -> {
            return context.getMappedObject(org.bson.Document.parse("""
                {
                    "$addFields": {
                        "score": { "$meta": "vectorSearchScore" }
                    }
                }
                """));
        };

        Aggregation aggregation = Aggregation.newAggregation(vectorSearch, addScore);

        return mongoTemplate.aggregate(
                aggregation,
                "code_snippets",
                CodeSnippet.class
        ).getMappedResults();
    }
}
```

---

## 7. Document Indexing (문서 색인)

### 7.1 Document Service

**파일**: `src/main/java/com/erp/codereview/document/service/DocumentService.java`

```java
package com.erp.codereview.document.service;

import com.erp.codereview.document.entity.CodeSnippet;
import com.erp.codereview.document.repository.CodeSnippetRepository;
import com.erp.codereview.embedding.service.EmbeddingService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;

/**
 * 문서 관리 서비스
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class DocumentService {

    private final CodeSnippetRepository codeSnippetRepository;
    private final EmbeddingService embeddingService;

    /**
     * 코드 스니펫 저장 (임베딩 포함)
     */
    public CodeSnippet saveCodeSnippet(CodeSnippet snippet) {
        log.info("Saving code snippet: {}", snippet.getDescription());

        // 1. 임베딩 생성 (code + description 결합)
        String textForEmbedding = snippet.getCode() + "\n" + snippet.getDescription();
        List<Double> embedding = embeddingService.createEmbedding(textForEmbedding);

        // 2. 임베딩 설정
        snippet.setEmbedding(embedding);
        snippet.setCreatedAt(LocalDateTime.now());
        snippet.setUpdatedAt(LocalDateTime.now());

        // 3. 저장
        CodeSnippet saved = codeSnippetRepository.save(snippet);
        log.info("Saved code snippet with ID: {}", saved.getId());

        return saved;
    }

    /**
     * 여러 스니펫 일괄 저장
     */
    public List<CodeSnippet> saveAll(List<CodeSnippet> snippets) {
        log.info("Batch saving {} code snippets", snippets.size());

        snippets.forEach(snippet -> {
            String textForEmbedding = snippet.getCode() + "\n" + snippet.getDescription();
            List<Double> embedding = embeddingService.createEmbedding(textForEmbedding);
            snippet.setEmbedding(embedding);
            snippet.setCreatedAt(LocalDateTime.now());
            snippet.setUpdatedAt(LocalDateTime.now());
        });

        return codeSnippetRepository.saveAll(snippets);
    }

    /**
     * 모든 스니펫 조회
     */
    public List<CodeSnippet> findAll() {
        return codeSnippetRepository.findAll();
    }

    /**
     * ID로 조회
     */
    public CodeSnippet findById(String id) {
        return codeSnippetRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("CodeSnippet not found: " + id));
    }
}
```

---

## 8. RAG 통합: 컨텍스트 기반 코드 리뷰

### 8.1 RAG Service

**파일**: `src/main/java/com/erp/codereview/rag/service/RagService.java`

```java
package com.erp.codereview.rag.service;

import com.erp.codereview.codereview.dto.CodeReviewRequest;
import com.erp.codereview.document.entity.CodeSnippet;
import com.erp.codereview.search.service.VectorSearchService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * RAG (Retrieval-Augmented Generation) 서비스
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class RagService {

    private final VectorSearchService vectorSearchService;
    private final ChatClient chatClient;

    /**
     * RAG 기반 코드 리뷰
     */
    public String reviewCodeWithRag(CodeReviewRequest request) {
        log.info("Starting RAG-based code review");

        // 1. 유사한 코드 예제 검색 (Vector Search)
        String searchQuery = request.getCode() + " " + request.getContext();
        List<CodeSnippet> similarSnippets = vectorSearchService.searchSimilarByCategory(
                searchQuery,
                "financial",
                3  // 상위 3개
        );

        log.info("Found {} similar code examples", similarSnippets.size());

        // 2. 검색된 예제를 컨텍스트로 변환
        String context = buildContext(similarSnippets);

        // 3. RAG 프롬프트 생성
        String promptTemplate = """
                You are an expert code reviewer for financial ERP systems.

                Use the following examples of good practices as reference:

                {context}

                Now, review this code:

                Language: {language}
                Context: {userContext}

                Code to review:
                ```{language}
                {code}
                ```

                Provide:
                1. Issues found (bugs, security, performance)
                2. Comparison with the reference examples
                3. Specific recommendations
                4. Code improvement suggestions

                Format your response clearly.
                """;

        PromptTemplate template = new PromptTemplate(promptTemplate);
        Prompt prompt = template.create(Map.of(
                "context", context,
                "language", request.getLanguage() != null ? request.getLanguage() : "Java",
                "userContext", request.getContext() != null ? request.getContext() : "Financial ERP",
                "code", request.getCode()
        ));

        // 4. LLM 호출
        ChatResponse response = chatClient.call(prompt);
        String review = response.getResult().getOutput().getContent();

        log.info("RAG-based review completed");
        return review;
    }

    /**
     * 검색된 스니펫을 컨텍스트 문자열로 변환
     */
    private String buildContext(List<CodeSnippet> snippets) {
        if (snippets.isEmpty()) {
            return "No reference examples available.";
        }

        return snippets.stream()
                .map(snippet -> String.format("""
                        Example %d:
                        Description: %s
                        Code:
                        ```%s
                        %s
                        ```
                        """,
                        snippets.indexOf(snippet) + 1,
                        snippet.getDescription(),
                        snippet.getLanguage(),
                        snippet.getCode()))
                .collect(Collectors.joining("\n---\n"));
    }
}
```

---

## 9. Controller 업데이트

**파일**: `src/main/java/com/erp/codereview/codereview/controller/CodeReviewController.java` (수정)

```java
package com.erp.codereview.codereview.controller;

import com.erp.codereview.codereview.dto.CodeReviewRequest;
import com.erp.codereview.codereview.service.CodeReviewService;
import com.erp.codereview.rag.service.RagService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api/code-review")
@RequiredArgsConstructor
public class CodeReviewController {

    private final CodeReviewService codeReviewService;
    private final RagService ragService;  // 추가

    /**
     * 기본 코드 리뷰 (RAG 없이)
     */
    @PostMapping("/review")
    public ResponseEntity<Map<String, String>> reviewCode(
            @Valid @RequestBody CodeReviewRequest request) {
        log.info("POST /api/code-review/review - Basic review");

        String review = codeReviewService.reviewCode(request);

        return ResponseEntity.ok(Map.of(
                "review", review,
                "method", "basic"
        ));
    }

    /**
     * RAG 기반 코드 리뷰 (권장)
     */
    @PostMapping("/review-rag")
    public ResponseEntity<Map<String, String>> reviewCodeWithRag(
            @Valid @RequestBody CodeReviewRequest request) {
        log.info("POST /api/code-review/review-rag - RAG-based review");

        String review = ragService.reviewCodeWithRag(request);

        return ResponseEntity.ok(Map.of(
                "review", review,
                "method", "rag"
        ));
    }
}
```

---

## 10. 샘플 데이터 로드

### 10.1 InitialDataLoader

**파일**: `src/main/java/com/erp/codereview/common/config/InitialDataLoader.java`

```java
package com.erp.codereview.common.config;

import com.erp.codereview.document.entity.CodeSnippet;
import com.erp.codereview.document.service.DocumentService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class InitialDataLoader {

    private final DocumentService documentService;

    @Bean
    public CommandLineRunner loadSampleData() {
        return args -> {
            // 기존 데이터가 있으면 스킵
            if (!documentService.findAll().isEmpty()) {
                log.info("Sample data already exists, skipping...");
                return;
            }

            log.info("Loading sample financial code snippets...");

            List<CodeSnippet> samples = List.of(
                    CodeSnippet.builder()
                            .code("""
                                    public BigDecimal calculateInvoiceTotal(List<InvoiceItem> items) {
                                        return items.stream()
                                            .map(InvoiceItem::getAmount)
                                            .reduce(BigDecimal.ZERO, BigDecimal::add);
                                    }
                                    """)
                            .description("Calculate total amount from invoice items using BigDecimal for precision")
                            .language("Java")
                            .category("financial")
                            .tags(List.of("invoice", "calculation", "bigdecimal"))
                            .build(),

                    CodeSnippet.builder()
                            .code("""
                                    @Transactional
                                    public void transferFunds(Account from, Account to, BigDecimal amount) {
                                        if (from.getBalance().compareTo(amount) < 0) {
                                            throw new InsufficientFundsException();
                                        }
                                        from.setBalance(from.getBalance().subtract(amount));
                                        to.setBalance(to.getBalance().add(amount));
                                        accountRepository.saveAll(List.of(from, to));
                                    }
                                    """)
                            .description("Thread-safe fund transfer with transaction management and balance validation")
                            .language("Java")
                            .category("financial")
                            .tags(List.of("transaction", "transfer", "validation"))
                            .build(),

                    CodeSnippet.builder()
                            .code("""
                                    public void recordJournalEntry(JournalEntry entry) {
                                        BigDecimal debitTotal = entry.getDebits().stream()
                                            .map(Debit::getAmount)
                                            .reduce(BigDecimal.ZERO, BigDecimal::add);

                                        BigDecimal creditTotal = entry.getCredits().stream()
                                            .map(Credit::getAmount)
                                            .reduce(BigDecimal.ZERO, BigDecimal::add);

                                        if (debitTotal.compareTo(creditTotal) != 0) {
                                            throw new UnbalancedEntryException("Debits must equal credits");
                                        }

                                        journalRepository.save(entry);
                                    }
                                    """)
                            .description("Accounting journal entry validation ensuring debits equal credits (double-entry bookkeeping)")
                            .language("Java")
                            .category("accounting")
                            .tags(List.of("journal", "double-entry", "validation"))
                            .build()
            );

            documentService.saveAll(samples);
            log.info("Loaded {} sample code snippets", samples.size());
        };
    }
}
```

---

## 11. 테스트

### 11.1 RAG API 테스트

**파일**: `test-rag.http`

```http
### RAG 기반 코드 리뷰
POST http://localhost:8080/api/code-review/review-rag
Content-Type: application/json

{
  "code": "public void transfer(Account from, Account to, double amount) {\n    from.balance -= amount;\n    to.balance += amount;\n}",
  "language": "Java",
  "context": "Banking fund transfer"
}

### 벡터 검색 테스트 (별도 엔드포인트 추가 시)
POST http://localhost:8080/api/search/similar
Content-Type: application/json

{
  "query": "calculate invoice total",
  "limit": 3
}
```

### 11.2 예상 응답

```json
{
  "review": "**Issues Found:**\n\n1. **Bug**: Using primitive `double` for money - causes precision errors\n2. **Security**: No balance validation - allows negative balance\n3. **Performance**: Not thread-safe - race condition possible\n\n**Reference Examples:**\n\nExample 2 shows the correct approach:\n- Uses `BigDecimal` for precision\n- Validates balance before transfer\n- Uses `@Transactional` for atomicity\n\n**Recommendations:**\n\n```java\n@Transactional\npublic void transfer(Account from, Account to, BigDecimal amount) {\n    if (from.getBalance().compareTo(amount) < 0) {\n        throw new InsufficientFundsException();\n    }\n    from.setBalance(from.getBalance().subtract(amount));\n    to.setBalance(to.getBalance().add(amount));\n    accountRepository.saveAll(List.of(from, to));\n}\n```",
  "method": "rag"
}
```

---

## 12. 시퀀스 다이어그램

```
User         Controller    RagService    VectorSearch   EmbeddingService   MongoDB    ChatClient
 │               │             │              │                │              │           │
 │ POST /review-rag           │              │                │              │           │
 ├──────────────>│             │              │                │              │           │
 │               │ reviewCodeWithRag()        │                │              │           │
 │               ├────────────>│              │                │              │           │
 │               │             │ searchSimilar()               │              │           │
 │               │             ├─────────────>│                │              │           │
 │               │             │              │ createEmbedding()            │           │
 │               │             │              ├───────────────>│              │           │
 │               │             │              │                │ [Embed query]│           │
 │               │             │              │<───────────────┤              │           │
 │               │             │              │  Vector Search (aggregation)  │           │
 │               │             │              ├──────────────────────────────>│           │
 │               │             │              │      Similar docs             │           │
 │               │             │              │<──────────────────────────────┤           │
 │               │             │ CodeSnippets │                │              │           │
 │               │             │<─────────────┤                │              │           │
 │               │             │                                                          │
 │               │             │ buildContext()                │              │           │
 │               │             ├──────────┐   │                │              │           │
 │               │             │          │   │                │              │           │
 │               │             │<─────────┘   │                │              │           │
 │               │             │                                                          │
 │               │             │ create Prompt with context    │              │           │
 │               │             ├──────────┐   │                │              │           │
 │               │             │          │   │                │              │           │
 │               │             │<─────────┘   │                │              │           │
 │               │             │                                              │  call()   │
 │               │             │                                              ├──────────>│
 │               │             │                                              │ [LLM]     │
 │               │             │                                              │<──────────┤
 │               │             │ review       │                │              │           │
 │               │<────────────┤              │                │              │           │
 │ 200 OK (review)             │              │                │              │           │
 │<──────────────┤             │              │                │              │           │
```

---

## 13. 주의사항 및 Best Practices

### 13.1 임베딩 차원 일치

```java
// ❌ 잘못된 설정
MongoDB Index: dimensions: 768
Embedding Model: all-MiniLM-L6-v2 (384차원)

// ✅ 올바른 설정
MongoDB Index: dimensions: 384
Embedding Model: all-MiniLM-L6-v2 (384차원)
```

### 13.2 검색 결과 개수

```java
// Too many: 느림, 프롬프트 길어짐
searchSimilar(query, 20);

// ✅ 적절한 개수: 3~5개
searchSimilar(query, 3);
```

### 13.3 컨텍스트 크기 제한

```java
private String buildContext(List<CodeSnippet> snippets) {
    int maxTokens = 1000;  // LLM 토큰 제한 고려

    StringBuilder context = new StringBuilder();
    for (CodeSnippet snippet : snippets) {
        String snippetText = formatSnippet(snippet);
        if (context.length() + snippetText.length() < maxTokens * 4) {  // ~4 chars per token
            context.append(snippetText);
        }
    }
    return context.toString();
}
```

---

## 14. 체크리스트

- [ ] MongoDB Atlas 계정 생성 및 클러스터 설정
- [ ] Vector Search Index 생성 (`vector_index`, 384차원)
- [ ] `CodeSnippet` 엔티티 및 Repository 작성
- [ ] `EmbeddingService` 구현 (Hugging Face API 연동)
- [ ] `VectorSearchService` 구현 (MongoDB aggregation)
- [ ] `DocumentService` 구현 (임베딩 포함 저장)
- [ ] `RagService` 구현 (검색 + LLM 통합)
- [ ] `InitialDataLoader`로 샘플 데이터 로드
- [ ] Controller에 `/review-rag` 엔드포인트 추가
- [ ] RAG API 테스트 성공 (`POST /api/code-review/review-rag`)
- [ ] 기본 리뷰 vs RAG 리뷰 결과 비교
- [ ] Git 커밋: `"Phase 4: Implement RAG with MongoDB Vector Search"`

---

## 15. 다음 단계

Phase 4 완료를 축하합니다! 다음은 **[Phase 5: 코드 리뷰 기능 구현](./phase-5-code-review-feature.md)**입니다.

Phase 5에서는:
- 실전 코드 리뷰 시스템 구현
- 재무/회계 도메인 특화 검증 규칙
- 리뷰 결과 저장 및 이력 관리
- 통합 테스트 및 성능 최적화

를 학습합니다.

---

**🎉 Phase 4 완료! RAG 패턴을 성공적으로 구현했습니다!**

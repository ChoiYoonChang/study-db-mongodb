# Spring AI + RAG 코드 리뷰 프로젝트 학습 가이드

## 📚 프로젝트 개요

이 프로젝트는 **Spring AI**, **Hugging Face StarCoder2**, **RAG(Retrieval-Augmented Generation)**를 활용한 AI 기반 코드 리뷰 시스템을 단계별로 학습하고 구현하는 toy project입니다.

**주제**: 회사 ERP 시스템의 재무/회계 분야 코드 리뷰 자동화

## 🎯 학습 목표

- [ ] Spring AI의 기본 개념과 활용법 이해
- [ ] LLM(Large Language Model)과의 연동 방법 학습
- [ ] RAG 패턴을 활용한 컨텍스트 기반 AI 응답 구현
- [ ] MongoDB를 활용한 벡터 검색 및 데이터 저장
- [ ] 실무에 적용 가능한 코드 리뷰 자동화 시스템 구축

## 📋 Phase별 학습 로드맵

### Phase 0: 환경 설정 및 IntelliJ 세팅
**문서**: [phase-0-environment-setup.md](./phase-0-environment-setup.md)

- [ ] IntelliJ IDEA 설치 및 프로젝트 생성
- [ ] 개발 환경 설정 (JDK, Gradle 등)
- [ ] 필수 플러그인 설치 및 설정
- [ ] 디버깅 환경 구축

**예상 소요 시간**: 30분 ~ 1시간

---

### Phase 1: Spring Boot 프로젝트 기본 구조
**문서**: [phase-1-spring-boot-basics.md](./phase-1-spring-boot-basics.md)

- [ ] Spring Boot 프로젝트 구조 이해
- [ ] 기본 REST API 엔드포인트 구현
- [ ] ERP 도메인 개요 및 패키지 구조 설계
- [ ] 기본 예외 처리 및 로깅 설정

**예상 소요 시간**: 1 ~ 2시간

---

### Phase 2: MongoDB 연동 및 도메인 모델 설계
**문서**: [phase-2-mongodb-domain.md](./phase-2-mongodb-domain.md)

- [ ] MongoDB 설치 및 연결 설정
- [ ] 재무/회계 도메인 모델 설계
- [ ] Spring Data MongoDB 설정
- [ ] CRUD 기본 기능 구현
- [ ] 도메인 모델 테스트 작성

**예상 소요 시간**: 2 ~ 3시간

---

### Phase 3: Spring AI 기본 설정 및 LLM 연동
**문서**: [phase-3-spring-ai-llm.md](./phase-3-spring-ai-llm.md)

- [ ] Spring AI 의존성 추가 및 설정
- [ ] Hugging Face API 연동
- [ ] StarCoder2 모델 설정
- [ ] 기본 프롬프트 엔지니어링
- [ ] LLM 응답 처리 및 에러 핸들링

**예상 소요 시간**: 2 ~ 3시간

---

### Phase 4: RAG (Retrieval-Augmented Generation) 구현
**문서**: [phase-4-rag-implementation.md](./phase-4-rag-implementation.md)

- [ ] RAG 패턴 개념 이해
- [ ] 임베딩(Embedding) 생성 및 저장
- [ ] 벡터 검색 구현 (MongoDB Vector Search)
- [ ] 컨텍스트 기반 프롬프트 생성
- [ ] RAG 파이프라인 통합

**예상 소요 시간**: 3 ~ 4시간

---

### Phase 5: 코드 리뷰 기능 구현
**문서**: [phase-5-code-review-feature.md](./phase-5-code-review-feature.md)

- [ ] 코드 분석 로직 구현
- [ ] AI 기반 코드 리뷰 생성
- [ ] 재무/회계 도메인 특화 검증 규칙
- [ ] 리뷰 결과 저장 및 조회 API
- [ ] 통합 테스트 작성

**예상 소요 시간**: 3 ~ 4시간

---

## 🏗️ 프로젝트 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client (REST API)                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Controller Layer                           │
│  - CodeReviewController                                         │
│  - FinanceDataController                                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Service Layer                             │
│  ┌─────────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ CodeReviewService│  │ RAGService   │  │ EmbeddingService │  │
│  └─────────────────┘  └──────────────┘  └──────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────┐
│   Spring AI      │ │   MongoDB    │ │ Hugging Face │
│   (RAG Pipeline) │ │   (Vector DB)│ │  StarCoder2  │
└──────────────────┘ └──────────────┘ └──────────────┘
```

## 🛠️ 기술 스택

### Backend
- **Spring Boot** 3.x
- **Spring AI** (최신 버전)
- **Spring Data MongoDB**
- **Java** 17 이상

### AI/ML
- **Hugging Face API**
- **StarCoder2** (코드 생성 및 리뷰 특화 모델)
- **Sentence Transformers** (임베딩 생성)

### Database
- **MongoDB** (벡터 검색 지원)

### Tools
- **IntelliJ IDEA**
- **Gradle**
- **Git**

## 📖 학습 방법

1. **순차적 학습**: Phase 0부터 순서대로 진행하는 것을 권장합니다.
2. **체크박스 활용**: 각 phase의 체크박스를 활용하여 진행 상황을 관리하세요.
3. **직접 타이핑**: 코드를 복사/붙여넣기보다는 직접 타이핑하면서 학습하세요.
4. **실험**: 제시된 3가지 구현 방법을 직접 비교하며 장단점을 체험하세요.
5. **디버깅**: 각 단계마다 디버깅을 통해 동작 방식을 이해하세요.

## 🔍 각 Phase에서 배울 내용

각 phase별 문서에는 다음 내용이 포함됩니다:

- ✅ **학습 목표 및 체크리스트**
- 📁 **파일 구조 및 위치**
- 💻 **전체 소스코드** (누락 없이)
- 🔀 **3가지 구현 방법** 및 장단점 비교
- 📊 **다이어그램** (Sequence, Flow 등)
- 📝 **문법 설명** 및 주의사항
- 🐛 **흔한 에러 및 해결 방법**
- 🧪 **테스트 코드**

## 💡 추가 학습 리소스

- [Spring AI 공식 문서](https://docs.spring.io/spring-ai/reference/)
- [Hugging Face Documentation](https://huggingface.co/docs)
- [MongoDB Vector Search](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/)
- [RAG 패턴 설명](https://www.pinecone.io/learn/retrieval-augmented-generation/)

## 🚀 최신 기술 트렌드 (2025)

이 프로젝트에서 다루는 최신 기술들:

1. **Spring AI**: 2024년 정식 출시된 Spring의 AI 통합 프레임워크
2. **RAG 패턴**: LLM의 환각(Hallucination) 문제를 해결하는 핵심 패턴
3. **벡터 데이터베이스**: MongoDB의 벡터 검색 기능 (Atlas Vector Search)
4. **도메인 특화 AI**: 일반적인 AI가 아닌 특정 도메인(재무/회계)에 특화된 AI

## 🎓 학습 완료 후 확장 가능한 부분

- [ ] 다양한 LLM 모델 비교 (GPT-4, Claude, Gemini 등)
- [ ] 웹 UI 추가 (React, Next.js)
- [ ] 실시간 코드 리뷰 (WebSocket)
- [ ] CI/CD 파이프라인 통합
- [ ] 멀티 테넌시 지원
- [ ] 코드 품질 메트릭 추적
- [ ] Fine-tuning: 재무/회계 도메인으로 모델 미세조정

## 📞 문제 해결 및 참고사항

각 phase에서 어려움을 겪을 경우:
1. 해당 phase 문서의 "흔한 에러 및 해결 방법" 섹션 참고
2. 디버깅 방법을 활용하여 단계별로 확인
3. 제공된 테스트 코드를 실행하여 검증

---

**준비되셨나요? [Phase 0: 환경 설정](./phase-0-environment-setup.md)부터 시작하세요!** 🚀

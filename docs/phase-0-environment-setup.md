# Phase 0: 환경 설정 및 IntelliJ 세팅

## 📚 학습 목표

- [ ] IntelliJ IDEA 설치 및 기본 설정
- [ ] JDK 17 설치 및 환경 변수 설정
- [ ] Gradle 프로젝트 생성 및 이해
- [ ] 필수 IntelliJ 플러그인 설치
- [ ] 디버깅 환경 구축
- [ ] Git 초기 설정

**예상 소요 시간**: 30분 ~ 1시간

---

## 1. IntelliJ IDEA 설치

### 1.1 다운로드 및 설치

**macOS 기준**:
```bash
# Homebrew를 사용하는 경우
brew install --cask intellij-idea-ce

# 또는 공식 웹사이트에서 다운로드
# https://www.jetbrains.com/idea/download/
```

**Windows 기준**:
1. https://www.jetbrains.com/idea/download/ 접속
2. Community Edition 또는 Ultimate Edition 다운로드
3. 설치 프로그램 실행

### 1.2 IntelliJ Edition 선택 가이드

| Edition | 장점 | 단점 | 추천 대상 |
|---------|------|------|-----------|
| **Community** | 무료, Spring Boot 개발 가능 | 일부 고급 기능 제한 | 개인 학습, 오픈소스 |
| **Ultimate** | 모든 기능, Spring 전용 도구 | 유료 (30일 무료 체험) | 업무용, 전문 개발 |

**💡 이 프로젝트에서는**: Community Edition으로 충분합니다.

---

## 2. JDK 설치

### 2.1 JDK 17 설치 (권장)

**macOS**:
```bash
# Homebrew 사용
brew install openjdk@17

# 환경 변수 설정 (zsh 사용 시)
echo 'export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# 설치 확인
java -version
```

**Windows**:
```bash
# Chocolatey 사용
choco install openjdk17

# 또는 수동 설치
# https://adoptium.net/temurin/releases/
```

### 2.2 JDK 버전 선택 이유

- **JDK 17**: LTS 버전, Spring Boot 3.x와 최적 호환성
- **JDK 21**: 최신 LTS이지만, 일부 라이브러리 호환성 이슈 가능
- **JDK 11**: 구버전, Spring Boot 3.x는 17 이상 필요

---

## 3. IntelliJ에서 Spring Boot 프로젝트 생성

### 3.1 방법 1: Spring Initializr (웹) - ⭐ 추천

**장점**:
- 시각적으로 직관적
- 최신 버전 자동 제안
- 의존성 검색 편리

**단점**:
- 네트워크 필요
- IntelliJ에서 재확인 필요

**단계**:
1. https://start.spring.io/ 접속
2. 다음 설정 선택:
   ```
   Project: Gradle - Groovy
   Language: Java
   Spring Boot: 3.2.x (최신 stable)

   Project Metadata:
   - Group: com.erp
   - Artifact: code-review-ai
   - Name: code-review-ai
   - Package name: com.erp.codereview
   - Packaging: Jar
   - Java: 17
   ```
3. Dependencies 추가:
   - Spring Web
   - Spring Data MongoDB
   - Lombok
4. Generate 클릭 → ZIP 다운로드
5. IntelliJ에서 `File` → `Open` → 압축 푼 폴더 선택

### 3.2 방법 2: IntelliJ 내장 Spring Initializr

**장점**:
- IDE 내에서 완결
- 프로젝트 생성 후 바로 작업 가능

**단점**:
- Ultimate Edition에서만 완전 지원
- Community Edition은 기능 제한적

**단계** (Ultimate Edition):
1. `File` → `New` → `Project`
2. `Spring Initializr` 선택
3. 위와 동일한 설정 입력
4. `Create` 클릭

### 3.3 방법 3: Gradle 수동 설정

**장점**:
- 완전한 제어 가능
- 학습 효과 높음

**단점**:
- 초기 설정 복잡
- 오류 가능성 높음

**단계**:
1. `File` → `New` → `Project`
2. `Gradle` 선택 → JDK 17 선택
3. `build.gradle` 수동 작성 (Phase 1에서 상세 설명)

**💡 권장**: 처음 학습할 때는 **방법 1 (Spring Initializr 웹)**을 사용하세요.

---

## 4. 필수 플러그인 설치

### 4.1 플러그인 설치 방법

`File` → `Settings` (macOS: `Preferences`) → `Plugins` → `Marketplace`

### 4.2 필수 플러그인

#### 1. **Lombok** ⭐⭐⭐
- **용도**: 보일러플레이트 코드 자동 생성 (`@Data`, `@Builder` 등)
- **설치 후 필수 설정**:
  ```
  Settings → Build, Execution, Deployment → Compiler → Annotation Processors
  → ✅ Enable annotation processing
  ```

#### 2. **Rainbow Brackets** ⭐⭐
- **용도**: 괄호 색상 구분으로 가독성 향상
- **추천 이유**: 중첩된 코드 블록 파악 용이

#### 3. **GitToolBox** ⭐⭐
- **용도**: Git 상태 실시간 표시, Blame 정보
- **추천 이유**: 협업 시 유용

#### 4. **Key Promoter X** ⭐
- **용도**: 마우스 사용 시 대응 단축키 알림
- **추천 이유**: 단축키 학습에 효과적

#### 5. **SonarLint** ⭐⭐⭐
- **용도**: 코드 품질 실시간 분석
- **추천 이유**: 코드 작성 중 버그 및 스멜 감지

### 4.3 선택적 플러그인

| 플러그인 | 용도 | 추천 대상 |
|----------|------|-----------|
| **Tabnine** | AI 코드 자동완성 | AI 보조 원하는 경우 |
| **MongoDB Plugin** | MongoDB GUI 통합 | DB 직접 조회 필요 시 |
| **CodeGlance** | 미니맵 표시 | 긴 파일 작업 시 |
| **Material Theme UI** | IDE 테마 변경 | 시각적 개선 원할 때 |

---

## 5. IntelliJ 개발 환경 설정

### 5.1 코드 스타일 설정

`Settings` → `Editor` → `Code Style` → `Java`

**권장 설정**:
```
- Tab size: 4
- Indent: 4
- Continuation indent: 8
- ✅ Use tab character: OFF (스페이스 사용)
```

**💡 팁**: Google Java Style Guide 적용
```
Settings → Editor → Code Style → Java
→ ⚙️ 톱니바퀴 아이콘 → Import Scheme
→ IntelliJ IDEA code style XML
→ intellij-java-google-style.xml 다운로드 후 적용
```
다운로드: https://github.com/google/styleguide/blob/gh-pages/intellij-java-google-style.xml

### 5.2 자동 Import 설정

`Settings` → `Editor` → `General` → `Auto Import`

```
✅ Add unambiguous imports on the fly
✅ Optimize imports on the fly
```

**장점**: import 문 자동 정리, 미사용 import 제거

### 5.3 파일 인코딩 설정

`Settings` → `Editor` → `File Encodings`

```
Global Encoding: UTF-8
Project Encoding: UTF-8
Default encoding for properties files: UTF-8
✅ Transparent native-to-ascii conversion
```

**이유**: 한글 깨짐 방지, 크로스 플랫폼 호환성

---

## 6. 디버깅 환경 구축

### 6.1 디버그 설정 생성

1. 우클릭 `src/main/java/com/erp/codereview/CodeReviewAiApplication.java`
2. `Debug 'CodeReviewAiApplication.main()'` 클릭

### 6.2 디버그 포트 설정 (고급)

`Run` → `Edit Configurations` → `Spring Boot` → `CodeReviewAiApplication`

**환경 변수 추가**:
```
VM options: -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5005
```

**사용 시나리오**: 원격 디버깅, Docker 컨테이너 디버깅

### 6.3 유용한 디버깅 단축키

| 단축키 (macOS) | 단축키 (Windows/Linux) | 기능 |
|----------------|------------------------|------|
| `Cmd + F8` | `Ctrl + F8` | 브레이크포인트 설정/해제 |
| `Ctrl + D` | `F9` | 디버그 시작 |
| `F8` | `F8` | Step Over (다음 줄) |
| `F7` | `F7` | Step Into (메서드 내부) |
| `Shift + F8` | `Shift + F8` | Step Out (메서드 빠져나가기) |
| `Cmd + F8` | `Ctrl + F8` | 조건부 브레이크포인트 |

### 6.4 디버깅 모범 사례

1. **Breakpoint 종류**:
   - **Line Breakpoint**: 특정 줄에서 멈춤
   - **Method Breakpoint**: 메서드 진입 시 멈춤
   - **Exception Breakpoint**: 예외 발생 시 멈춤

2. **Evaluate Expression** 활용:
   - 디버그 중 `Alt + F8`로 변수 값 실시간 확인

3. **Watch 설정**:
   - 변수 우클릭 → `Add to Watches`
   - 변수 값 변화 추적

---

## 7. Gradle 이해하기

### 7.1 Gradle이란?

**빌드 자동화 도구**: 프로젝트의 의존성, 컴파일, 테스트, 패키징을 자동화

**Maven vs Gradle 비교**:

| 구분 | Gradle | Maven |
|------|--------|-------|
| 설정 파일 | `build.gradle` (Groovy/Kotlin DSL) | `pom.xml` (XML) |
| 가독성 | 높음 | 낮음 (XML 장황함) |
| 빌드 속도 | 빠름 (증분 빌드) | 느림 |
| 최신 트렌드 | ⭐⭐⭐ | ⭐⭐ |

**💡 추천**: 새 프로젝트는 Gradle 사용

### 7.2 `build.gradle` 기본 구조

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
}

group = 'com.erp'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**섹션 설명**:
- `plugins`: 사용할 플러그인 선언
- `repositories`: 의존성 다운로드 저장소 (Maven Central 등)
- `dependencies`: 프로젝트에 필요한 라이브러리

### 7.3 Gradle 주요 명령어

```bash
# 프로젝트 빌드
./gradlew build

# 애플리케이션 실행
./gradlew bootRun

# 테스트 실행
./gradlew test

# 의존성 목록 확인
./gradlew dependencies

# 빌드 캐시 삭제
./gradlew clean
```

**IntelliJ에서 실행**:
우측 `Gradle` 패널 → `Tasks` → 해당 task 더블클릭

---

## 8. Git 초기 설정

### 8.1 Git 저장소 초기화

```bash
# 프로젝트 루트에서
git init

# 사용자 정보 설정
git config user.name "Your Name"
git config user.email "your.email@example.com"
```

### 8.2 `.gitignore` 설정

**이미 생성된 경우**: Spring Initializr가 자동 생성
**수동 생성**: 루트 디렉토리에 `.gitignore` 파일 생성

```gitignore
# Gradle
.gradle/
build/
!gradle/wrapper/gradle-wrapper.jar

# IntelliJ
.idea/
*.iml
*.iws
out/

# Application
application-local.yml
application-secret.yml

# OS
.DS_Store
Thumbs.db

# Logs
logs/
*.log
```

### 8.3 첫 커밋

```bash
git add .
git commit -m "Initial commit: Spring Boot project setup"
```

---

## 9. 실행 및 확인

### 9.1 프로젝트 실행

**방법 1: IntelliJ 실행 버튼**
1. `CodeReviewAiApplication.java` 열기
2. 클래스 옆 녹색 ▶️ 아이콘 클릭
3. `Run 'CodeReviewAiApplication.main()'` 선택

**방법 2: Gradle 명령어**
```bash
./gradlew bootRun
```

### 9.2 정상 실행 확인

터미널에 다음과 같은 로그가 보이면 성공:
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.0)

... Tomcat started on port(s): 8080 (http) ...
... Started CodeReviewAiApplication in 2.5 seconds ...
```

**테스트**:
```bash
# 터미널에서
curl http://localhost:8080

# 브라우저에서
http://localhost:8080
```

**예상 결과**: `Whitelabel Error Page` (정상 - 아직 API가 없음)

---

## 10. 자주 발생하는 문제 및 해결

### 10.1 문제: JDK 버전 불일치

**증상**:
```
java: error: invalid source release: 17
```

**원인**: IntelliJ가 다른 JDK 버전 사용 중

**해결**:
```
File → Project Structure → Project Settings → Project
→ SDK: 17
→ Language level: 17

File → Project Structure → Modules → study-mongodb-v1
→ Language level: 17
```

### 10.2 문제: Lombok 동작 안 함

**증상**:
```
java: cannot find symbol
  symbol:   method builder()
```

**해결**:
1. Lombok 플러그인 설치 확인
2. `Settings` → `Build` → `Compiler` → `Annotation Processors`
   → ✅ `Enable annotation processing` 체크
3. `File` → `Invalidate Caches` → `Invalidate and Restart`

### 10.3 문제: Gradle 빌드 실패

**증상**:
```
Could not resolve all dependencies
```

**해결**:
```bash
# Gradle 캐시 삭제
rm -rf ~/.gradle/caches/

# 프로젝트 빌드 재시도
./gradlew clean build --refresh-dependencies
```

### 10.4 문제: 포트 이미 사용 중

**증상**:
```
Web server failed to start. Port 8080 was already in use.
```

**해결 1**: 포트 변경 (`src/main/resources/application.yml`)
```yaml
server:
  port: 8081
```

**해결 2**: 기존 프로세스 종료
```bash
# macOS/Linux
lsof -ti:8080 | xargs kill -9

# Windows
netstat -ano | findstr :8080
taskkill /PID <PID번호> /F
```

---

## 11. 체크리스트

완료한 항목에 ✅ 표시하세요:

- [ ] IntelliJ IDEA 설치 완료
- [ ] JDK 17 설치 및 확인 (`java -version`)
- [ ] Spring Boot 프로젝트 생성 완료
- [ ] Lombok 플러그인 설치 및 활성화
- [ ] Rainbow Brackets, SonarLint 플러그인 설치
- [ ] 코드 스타일 설정 (Google Java Style 권장)
- [ ] 자동 Import 설정 활성화
- [ ] 파일 인코딩 UTF-8로 설정
- [ ] 디버그 환경 테스트 (브레이크포인트 설정 후 실행)
- [ ] Gradle 명령어 테스트 (`./gradlew build`)
- [ ] Git 저장소 초기화 및 첫 커밋
- [ ] 애플리케이션 정상 실행 확인 (`http://localhost:8080`)

---

## 📚 다음 단계

환경 설정이 완료되었습니다! 다음은 **[Phase 1: Spring Boot 프로젝트 기본 구조](./phase-1-spring-boot-basics.md)**로 이동하세요.

Phase 1에서는:
- 프로젝트 패키지 구조 설계
- 첫 REST API 구현
- ERP 도메인 기초 설계

를 학습합니다.

---

**🎉 수고하셨습니다! 다음 Phase에서 뵙겠습니다!**

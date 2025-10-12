# Violin 프로젝트 개발 컨벤션 및 규칙

## 프로젝트 구조 규칙

### 디렉토리 명명 규칙
- **서비스 디렉토리**: kebab-case 사용 (예: `login-app`, `kubernetes`)
- **Java 패키지**: com.api 기본 패키지 구조
- **설정 파일**: 서비스별 독립적인 설정 관리

### 버전 관리
- **공통 버전**: 모든 서비스 `0.0.1-SNAPSHOT`
- **Spring Boot**: `3.3.3` 통일
- **Spring Cloud**: `2023.0.5` 통일
- **Java**: `21` LTS 버전 사용

## 코딩 컨벤션

### Java/Spring Boot 규칙

#### 빌드 도구
```gradle
// 공통 플러그인 구성
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.3'
    id 'io.spring.dependency-management' version '1.1.6'
}

// Java 버전 통일
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

#### 의존성 관리
- **Lombok**: 모든 서비스에서 사용하여 보일러플레이트 코드 제거
- **Spring Cloud BOM**: 버전 호환성 보장
- **Micrometer**: 모니터링 메트릭 표준화

#### 공통 의존성 패턴
```gradle
dependencies {
    // 공통 Spring Boot 스타터
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    
    // 코드 간소화
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    
    // 모니터링 (필수)
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp'
    
    // 서비스 디스커버리 (Gateway, Kubernetes 서비스)
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
}
```

### Frontend (React/TypeScript) 규칙

#### 패키지 관리
- **React**: `18.3.1` 버전 고정
- **TypeScript**: `^4.7.4` 사용
- **Chakra UI**: `2.6.1` 버전으로 UI 통일

#### 상태 관리
- **Zustand**: 경량 상태 관리 라이브러리 사용
- Redux 대신 Zustand로 복잡성 최소화

#### 코드 품질
```json
{
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test"
  }
}
```

## 아키텍처 규칙

### 마이크로서비스 패턴
1. **단일 책임 원칙**: 각 서비스는 하나의 비즈니스 도메인 담당
2. **독립적 배포**: 서비스별 독립적인 빌드 및 배포
3. **데이터베이스 분리**: 서비스별 독립적인 데이터 저장소

### 서비스 간 통신
- **동기 통신**: OpenFeign 사용
- **비동기 통신**: RabbitMQ 메시지 큐 활용
- **서비스 디스커버리**: Eureka 필수 등록

### API 설계 규칙
- **RESTful API**: HTTP 메서드 의미에 맞는 사용
- **Swagger/OpenAPI**: API 문서화 필수
- **JWT 인증**: 모든 보호된 엔드포인트에 적용

## 보안 규칙

### 인증/인가
```java
// JWT 토큰 설정 예시
implementation("io.jsonwebtoken:jjwt:0.9.1")
implementation("javax.xml.bind:jaxb-api:2.3.1")
```

### 데이터 보호
- **Redis**: 세션 및 캐시 데이터 암호화
- **MariaDB**: 민감 정보 암호화 저장
- **HTTPS**: 모든 외부 통신 암호화

## 모니터링 및 로깅 규칙

### 필수 모니터링 구성
```gradle
// 모든 서비스에 필수 포함
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'io.micrometer:micrometer-tracing-bridge-otel'
implementation 'io.opentelemetry:opentelemetry-exporter-otlp'
```

### 로깅 표준
- **구조화된 로깅**: JSON 형태로 로그 출력
- **분산 트레이싱**: Tempo를 통한 요청 추적
- **메트릭 수집**: Prometheus 표준 메트릭

## 배포 및 운영 규칙

### Kubernetes 배포
- **Helm Charts**: 모든 인프라 구성요소는 Helm으로 관리
- **ConfigMap/Secret**: 환경별 설정 분리
- **Health Check**: 모든 서비스에 헬스 체크 엔드포인트 구현

### 환경 관리
```yaml
# 공통 서비스 설정 패턴
service:
  type: NodePort  # 개발 환경
  port: 80
  targetPort: 3000
```

### 버전 관리
- **Git Flow**: feature/develop/main 브랜치 전략
- **Semantic Versioning**: 의미있는 버전 관리
- **Docker 태그**: 버전별 이미지 태깅

## 테스트 규칙

### 테스트 전략
```gradle
// 공통 테스트 의존성
testImplementation 'org.springframework.boot:spring-boot-starter-test'
testRuntimeOnly 'org.junit.platform:junit-platform-launcher'

tasks.named('test') {
    useJUnitPlatform()
}
```

### 테스트 범위
- **Unit Test**: 각 서비스별 단위 테스트
- **Integration Test**: 서비스 간 통합 테스트
- **E2E Test**: 전체 워크플로우 테스트

## 문서화 규칙

### API 문서화
- **Swagger UI**: 모든 REST API 문서화
- **README**: 각 서비스별 실행 방법 명시
- **Architecture Decision Records**: 주요 기술 결정 사항 기록

### 코드 문서화
- **JavaDoc**: 공개 API에 대한 문서화
- **TypeScript**: 인터페이스 및 타입 정의
- **주석**: 비즈니스 로직에 대한 설명

## 성능 및 최적화 규칙

### 캐싱 전략
- **Redis**: 자주 조회되는 데이터 캐싱
- **HTTP 캐싱**: 정적 리소스 캐시 헤더 설정
- **데이터베이스**: 인덱스 최적화

### 리소스 관리
- **Connection Pool**: 데이터베이스 연결 풀 최적화
- **Memory**: JVM 힙 메모리 튜닝
- **CPU**: 비동기 처리를 통한 CPU 효율성 향상
# Violin 프로젝트 개선 및 보강 필요 사항

## 🔴 Critical - 즉시 보강 필요

### 1. **통합 빌드 시스템 부재**
**현황**: 각 서비스가 독립적인 Gradle 프로젝트로 분리되어 있음
- discovery, gateway, kubernetes, user 각각 별도 빌드
- 루트에 통합 빌드 설정 없음

**문제점**:
- 전체 프로젝트 빌드/테스트가 번거로움
- 의존성 버전 관리가 분산됨
- CI/CD 파이프라인 구성 복잡

**개선 방안**:
```
violin/
├── build.gradle (루트 프로젝트)
├── settings.gradle (서브프로젝트 포함)
├── gradle.properties (공통 버전 관리)
└── buildSrc/ (공통 빌드 로직)
```

### 2. **Gateway 라우팅 설정 미비**
**현황**: gateway의 application-local.yml에 단 하나의 라우트만 정의
```yaml
routes:
  - id: spring-cloud-service
    uri: lb://SPRING-CLOUD-SERVICE
```

**문제점**:
- user 서비스 (9090) 라우팅 없음
- kubernetes 서비스 (9091) 라우팅 없음
- JWT 필터 설정 없음
- CORS 설정 없음

**필요한 라우팅**:
- `/auth/**` → user 서비스
- `/k8s/**` → kubernetes 서비스
- JWT 인증 필터 체인
- Rate Limiting 설정

### 3. **환경별 설정 관리 부재**
**현황**: application-local.yml만 존재
- dev, staging, prod 환경 설정 없음
- 하드코딩된 IP 주소 (192.168.64.3)
- 민감 정보 평문 저장 (password: admin123)

**개선 방안**:
- application-{profile}.yml 구조화
- ConfigMap/Secret 활용
- Spring Cloud Config Server 도입 검토

### 4. **데이터베이스 설정 불일치**
**현황**: 
- kubernetes 서비스: H2 사용 (개발용)
- user 서비스: H2 사용 (개발용)
- mariadb Helm 차트는 있지만 연동 안됨

**문제점**:
- 프로덕션 DB 설정 없음
- 데이터 영속성 보장 안됨
- 마이그레이션 전략 없음

**개선 방안**:
- Flyway/Liquibase 도입
- 환경별 DB 설정 분리
- MariaDB 연동 설정

## 🟡 Important - 조속히 보강 필요

### 5. **API 문서화 부재**
**현황**: 
- Swagger 의존성은 kubernetes 서비스에만 있음
- 실제 Swagger UI 설정 확인 필요
- API 명세서 없음

**개선 방안**:
- 모든 서비스에 SpringDoc OpenAPI 적용
- API 문서 자동 생성
- Postman Collection 제공

### 6. **보안 설정 미흡**
**현황**:
- JWT secret이 평문으로 설정 파일에 노출 (`secret: qkrdudwn`)
- Redis 비밀번호 평문 노출
- HTTPS 설정 없음

**개선 방안**:
- Kubernetes Secret 활용
- Vault 같은 Secret 관리 도구 도입
- 환경변수로 민감 정보 주입

### 7. **로깅 전략 불명확**
**현황**:
- logback-spring.xml 파일 존재
- Loki Stack Helm 차트 있음
- 실제 연동 상태 불명확

**개선 방안**:
- 구조화된 로깅 (JSON 포맷)
- 로그 레벨 환경별 설정
- Loki 연동 확인 및 문서화

### 8. **테스트 코드 부족**
**현황**:
- 테스트 의존성은 있음
- 실제 테스트 코드 확인 필요

**개선 방안**:
- Unit Test 작성
- Integration Test 작성
- E2E Test 시나리오 정의

### 9. **모니터링 연동 미완성**
**현황**:
- Prometheus, Grafana, Tempo Helm 차트 존재
- Actuator 의존성 있음
- 실제 메트릭 수집 확인 필요

**개선 방안**:
- Prometheus ServiceMonitor 설정
- Grafana 대시보드 구성
- Alert 규칙 정의

## 🟢 Nice to Have - 추후 개선 고려

### 10. **CI/CD 파이프라인 부재**
**필요 사항**:
- GitHub Actions / Jenkins 파이프라인
- Docker 이미지 빌드 자동화
- Kubernetes 배포 자동화
- 테스트 자동화

### 11. **API Gateway 고급 기능**
**추가 고려 사항**:
- Rate Limiting
- Circuit Breaker
- Request/Response 로깅
- API 버저닝

### 12. **K8S 리소스 관리 확장**
**현재 지원**: Pod, Deployment, Service, Node, PV, PVC, Namespace

**추가 필요**:
- ConfigMap, Secret
- Ingress
- StatefulSet, DaemonSet
- HPA (Horizontal Pod Autoscaler)
- NetworkPolicy
- RBAC (Role, RoleBinding)

### 13. **프론트엔드 빌드 최적화**
**개선 사항**:
- 환경변수 관리 (.env 파일 구조화)
- 빌드 최적화 (Code Splitting)
- Docker 멀티스테이지 빌드

### 14. **개발 환경 표준화**
**필요 사항**:
- Docker Compose로 로컬 개발 환경 구성
- Skaffold/Tilt 같은 K8S 개발 도구
- 개발자 온보딩 가이드

### 15. **에러 처리 표준화**
**개선 사항**:
- 전역 Exception Handler
- 표준화된 에러 응답 포맷
- 에러 코드 체계 정의

## 우선순위 제안

### Phase 1 (즉시 착수)
1. Gateway 라우팅 설정 완성
2. 환경별 설정 파일 구조화
3. 민감 정보 Secret 처리
4. 통합 빌드 시스템 구축

### Phase 2 (1-2주 내)
5. MariaDB 연동 및 마이그레이션 도구
6. API 문서화 (Swagger)
7. 로깅 전략 수립 및 Loki 연동
8. 기본 테스트 코드 작성

### Phase 3 (1개월 내)
9. 모니터링 대시보드 구성
10. CI/CD 파이프라인 구축
11. Docker Compose 개발 환경
12. K8S 리소스 관리 확장

### Phase 4 (장기)
13. API Gateway 고급 기능
14. 프론트엔드 최적화
15. 에러 처리 표준화
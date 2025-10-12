# Violin - Multi K8S Management Platform 기능 분석

## 프로젝트 개요
**Violin**은 클라우드 플랫폼에 종속되지 않는 Multi Kubernetes Management Platform으로, 마이크로서비스 아키텍처를 기반으로 구축된 사이드 프로젝트입니다.

## 아키텍처 구성

### 마이크로서비스 구조
```
Violin Platform
├── Frontend (React + TypeScript)
│   └── login-app/ - Horizon UI 기반 웹 인터페이스
├── Backend Services (Spring Boot + Java 21)
│   ├── discovery/ - Eureka 서비스 디스커버리
│   ├── gateway/ - Spring Cloud Gateway (API 게이트웨이)
│   ├── kubernetes/ - K8S 리소스 관리 서비스
│   └── user/ - 사용자 관리 서비스
└── Infrastructure
    ├── grafana/ - 모니터링 대시보드
    ├── prometheus/ - 메트릭 수집
    ├── tempo/ - 분산 트레이싱
    ├── loki-stack/ - 로그 수집
    ├── mariadb/ - 관계형 데이터베이스
    ├── redis/ - 캐시 및 세션 관리
    └── rabbitmq/ - 메시지 큐
```

## 핵심 기능 분석

### 1. 인증 및 보안
- **JWT Token 기반 인증**: JSON Web Token을 활용한 stateless 인증
- **Redis 세션 관리**: 분산 환경에서의 세션 공유
- **API Gateway 보안**: 중앙화된 인증/인가 처리

### 2. Kubernetes 관리
- **Fabric8 K8S Client**: Kubernetes API와의 통신
- **Kubeconfig 연동**: 다중 클러스터 관리
- **리소스 CRUD**: Pod, Service, Deployment 등 기본 리소스 관리
- **CRD 지원**: Custom Resource Definition 등록 및 관리
- **Helm Release**: Helm 차트 기반 애플리케이션 배포

### 3. 모니터링 및 관찰성
- **Prometheus 연동**: 메트릭 수집 및 임계치 설정
- **Grafana 대시보드**: 실시간 모니터링 및 시각화
- **Tempo 분산 트레이싱**: 마이크로서비스 간 요청 추적
- **Micrometer**: Spring Boot 애플리케이션 메트릭
- **OpenTelemetry**: 표준화된 관찰성 데이터 수집

### 4. 데이터 관리
- **MariaDB**: 영구 데이터 저장
- **Redis**: 캐싱 및 임시 데이터
- **RabbitMQ**: 비동기 메시지 처리

## 기술 스택 상세

### Backend (Spring Boot 3.3.3 + Java 21)
- **Spring Cloud Gateway**: API 라우팅 및 필터링
- **Spring Cloud Netflix Eureka**: 서비스 디스커버리
- **Spring Data JPA**: 데이터 액세스 레이어
- **Spring Boot Actuator**: 운영 모니터링
- **Lombok**: 코드 간소화
- **Resilience4j**: 회복력 패턴 (Retry 등)

### Frontend (React 18.3.1 + TypeScript)
- **Horizon UI**: Chakra UI 기반 관리자 템플릿
- **Zustand**: 경량 상태 관리
- **React Router**: SPA 라우팅
- **Axios**: HTTP 클라이언트
- **ApexCharts**: 차트 및 시각화

### Infrastructure
- **Helm Charts**: Kubernetes 애플리케이션 패키징
- **Docker**: 컨테이너화
- **Gradle**: 빌드 도구

## 현재 구현 상태

### ✅ 완료된 기능
- JWT 기반 로그인 시스템
- Kubeconfig를 통한 클러스터 연동
- Fabric8을 이용한 K8S 기본 리소스 조회
- 상태 대시보드
- 마이크로서비스 기본 구조
- 모니터링 스택 구성

### 🔨 개발 중인 기능
- 리소스 생성, 수정, 삭제 (CRUD)
- Prometheus 연동 및 메트릭 조회
- K8S API 서버 감사 로그
- 최적의 Pod 리소스 계산
- CRD 등록 및 Helm Release 관리

## 서비스별 역할

### Discovery Service
- Eureka 서버 역할
- 마이크로서비스 등록 및 발견
- 서비스 헬스 체크

### Gateway Service
- API 게이트웨이 역할
- JWT 토큰 검증
- 라우팅 및 로드 밸런싱
- CORS 처리

### Kubernetes Service
- K8S 클러스터 관리
- 리소스 CRUD 작업
- Prometheus 메트릭 연동
- Fabric8 클라이언트 활용

### Login App (Frontend)
- 사용자 인터페이스
- 대시보드 및 모니터링 화면
- K8S 리소스 관리 UI
- 실시간 상태 표시

## 배포 및 운영

### Helm Charts
- Grafana, Prometheus, Tempo 등 모니터링 스택
- 표준화된 Kubernetes 배포
- 환경별 설정 관리

### 모니터링
- Grafana 대시보드를 통한 시각화
- Prometheus 메트릭 수집
- Tempo 분산 트레이싱
- Spring Boot Actuator 헬스 체크
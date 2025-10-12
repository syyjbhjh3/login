# Violin 프로젝트 소스 코드 이슈 및 기능 개발 제안

## 🔴 소스 코드 레벨 이슈 (즉시 수정 필요)

### 1. **ClusterController의 잘못된 HTTP 메서드 사용**
**위치**: `ClusterController.java`

**문제**:
```java
@GetMapping("/detail/{clusterId}")
public ResultDTO detail(@RequestBody KubernetesDTO kubernetesDTO) {
    return clusterService.detail(kubernetesDTO);
}
```
- GET 요청에 `@RequestBody` 사용 (HTTP 스펙 위반)
- `@PathVariable`로 clusterId를 받아야 함

**수정 필요**:
```java
@GetMapping("/detail/{clusterId}")
public ResultDTO detail(@PathVariable UUID clusterId) {
    return clusterService.detail(clusterId);
}
```

### 2. **DELETE 메서드가 실제로 삭제하지 않음**
**위치**: `ClusterServiceImpl.delete()`

**문제**:
```java
public ResultDTO delete(KubernetesDTO kubernetesDTO) {
    // ... 엔티티를 찾고
    clusterRepository.save(deleteEntity);  // ❌ save를 호출
    return new ResultDTO<>(Status.SUCCESS, Message.CLUSTER_DELETE_SUCCESS.message);
}
```
- `save()`를 호출하는데 삭제 성공 메시지 반환
- Soft Delete인지 Hard Delete인지 불명확

**수정 필요**:
```java
// Hard Delete
clusterRepository.delete(deleteEntity);
// 또는 Soft Delete
deleteEntity.setStatus(Status.DELETED);
clusterRepository.save(deleteEntity);
```

### 3. **JWT 토큰 검증 로직 누락**
**위치**: `Gateway` 서비스

**문제**:
- Gateway에 JWT 검증 필터가 없음
- JWT 라이브러리 의존성은 있지만 실제 검증 코드 없음
- 모든 요청이 인증 없이 통과됨

**필요한 구현**:
```java
// JwtAuthenticationFilter.java (Gateway에 필요)
public class JwtAuthenticationFilter implements GatewayFilter {
    // JWT 검증 로직
    // User 서비스와 JWT secret 공유 필요
}
```

### 4. **UserClusterClientManager의 메모리 누수 위험**
**위치**: `UserClusterClientManager.java`

**문제**:
```java
public KubernetesClient getClusterClient(UUID clusterId) {
    // 매번 새로운 클라이언트 생성
    KubernetesClient newClient = new KubernetesClientBuilder()
            .withConfig(kubeconfigData)
            .build();
    return newClient;  // close() 호출 안됨
}
```
- 매번 새 클라이언트 생성하지만 닫지 않음
- Connection Pool 고갈 위험

**개선 방안**:
```java
// 클라이언트 캐싱 + try-with-resources 패턴
private final Map<UUID, KubernetesClient> clientCache = new ConcurrentHashMap<>();

public KubernetesClient getClusterClient(UUID clusterId) {
    return clientCache.computeIfAbsent(clusterId, id -> {
        String kubeconfigData = clusterRepository.findByClusterId(id).getKubeConfigData();
        return new KubernetesClientBuilder().withConfig(kubeconfigData).build();
    });
}
```

### 5. **비밀번호 암호화 로직의 취약점**
**위치**: `UserServiceImpl.signUp()`

**문제**:
```java
if (dto.getType().equals("1")) {  // 문자열 비교
    salt = encrypt.getSalt();
    encrytPassword = encrypt.getEncrypt(dto.getPassword(), salt);
}
```
- Type이 "1"이 아니면 암호화 안함 (OAuth 사용자)
- 하지만 일반 사용자도 암호화 안될 수 있음

**개선 필요**:
```java
if (Type.NORMAL.equals(dto.getType())) {  // Enum 사용
    // 암호화 로직
}
```

### 6. **GlobalExceptionHandler의 중복 처리**
**위치**: `GlobalExceptionHandler.java`

**문제**:
```java
@ExceptionHandler(Exception.class)
public ResultDTO handleException(Exception e) { ... }

@ExceptionHandler(RuntimeException.class)
public ResultDTO handleException(RuntimeException e) { ... }
```
- `RuntimeException`은 `Exception`의 하위 클래스
- 중복 처리되며 우선순위 불명확

**개선**:
```java
@ExceptionHandler(IllegalArgumentException.class)
public ResultDTO handleIllegalArgument(IllegalArgumentException e) {
    return new ResultDTO<>(Status.BAD_REQUEST, e.getMessage());
}

@ExceptionHandler(Exception.class)
public ResultDTO handleGenericException(Exception e) {
    log.error("Unexpected error", e);
    return new ResultDTO<>(Status.ERROR, "Internal server error");
}
```

### 7. **CORS 설정이 모든 컨트롤러에 중복**
**위치**: 모든 Controller

**문제**:
```java
@CrossOrigin("*")  // 모든 컨트롤러에 반복
@RestController
```
- 보안 위험 (모든 Origin 허용)
- 중복 코드

**개선**:
```java
// WebConfig.java에서 전역 설정
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("http://localhost:3000")  // 특정 Origin만
                .allowedMethods("GET", "POST", "PUT", "DELETE");
    }
}
```

### 8. **ResultDTO의 제네릭 타입 불일치**
**위치**: 여러 Service 메서드

**문제**:
```java
return new ResultDTO(Status.SUCCESS, Message.JOIN_SUCCESS.message);  // Raw type
```
- 제네릭 타입 명시 안함
- 타입 안정성 저하

**수정**:
```java
return new ResultDTO<>(Status.SUCCESS, Message.JOIN_SUCCESS.message);
```

## 🟡 설계 및 구조 개선 사항

### 9. **트랜잭션 관리 부재**
**문제**: 
- Service 메서드에 `@Transactional` 없음
- 데이터 일관성 보장 안됨

**개선**:
```java
@Transactional
public ResultDTO create(KubernetesDTO kubernetesDTO) {
    // ...
}
```

### 10. **DTO ↔ Entity 변환 로직 분산**
**문제**:
- DTO에 `toEntity()` 메서드
- 책임 분리 원칙 위반

**개선**:
```java
// Mapper 클래스 도입
@Component
public class ClusterMapper {
    public ClusterEntity toEntity(KubernetesDTO dto) { ... }
    public KubernetesDTO toDto(ClusterEntity entity) { ... }
}
```

### 11. **비즈니스 로직이 Controller에 노출**
**문제**:
- Controller에서 Header 값 직접 추출
- Service 레이어로 이동 필요

### 12. **로깅 전략 불일치**
**문제**:
- 일부는 `log.error()`, 일부는 `System.out.println()`
- 로그 레벨 불일치

## 🟢 기능 개발 제안

### Phase 1: 핵심 기능 완성 (1-2주)

#### 1. **K8S 리소스 생성/수정/삭제 (CRUD 완성)**
**현황**: 조회만 가능
**필요 기능**:
```java
// DeploymentController
@PostMapping
public ResultDTO create(@RequestBody DeploymentDTO dto) {
    // Deployment 생성
}

@PutMapping("/{namespace}/{name}")
public ResultDTO update(@PathVariable String namespace, 
                       @PathVariable String name,
                       @RequestBody DeploymentDTO dto) {
    // Deployment 수정 (replicas, image 등)
}

@DeleteMapping("/{namespace}/{name}")
public ResultDTO delete(@PathVariable String namespace, 
                       @PathVariable String name) {
    // Deployment 삭제
}
```

**우선순위 리소스**:
1. Deployment (replicas 조정, 이미지 업데이트)
2. Service (NodePort, LoadBalancer 생성)
3. ConfigMap/Secret (환경 설정)
4. Namespace (격리)

#### 2. **Pod 로그 조회 기능**
```java
@GetMapping("/namespaces/{namespace}/pods/{pod}/logs")
public ResultDTO getLogs(
    @PathVariable String namespace,
    @PathVariable String pod,
    @RequestParam(required = false) String container,
    @RequestParam(defaultValue = "100") int tailLines
) {
    // Pod 로그 스트리밍
}
```

#### 3. **Pod 실행 명령 (Exec)**
```java
@PostMapping("/namespaces/{namespace}/pods/{pod}/exec")
public ResultDTO execCommand(
    @PathVariable String namespace,
    @PathVariable String pod,
    @RequestBody ExecCommandDTO command
) {
    // kubectl exec 기능
}
```

#### 4. **리소스 이벤트 조회**
```java
@GetMapping("/namespaces/{namespace}/events")
public ResultDTO getEvents(@PathVariable String namespace) {
    // K8S 이벤트 조회 (Warning, Normal)
}
```

### Phase 2: 모니터링 및 관찰성 (2-3주)

#### 5. **Prometheus 메트릭 연동**
```java
@GetMapping("/metrics/pod/{namespace}/{pod}")
public ResultDTO getPodMetrics(
    @PathVariable String namespace,
    @PathVariable String pod,
    @RequestParam String timeRange
) {
    // CPU, Memory 사용량 조회
}
```

**필요 구현**:
- Prometheus API 클라이언트
- PromQL 쿼리 빌더
- 시계열 데이터 포맷팅

#### 6. **리소스 사용량 대시보드**
- 클러스터별 CPU/Memory 사용률
- Node별 리소스 현황
- Namespace별 리소스 할당량

#### 7. **알람 설정 기능**
```java
@PostMapping("/alerts")
public ResultDTO createAlert(@RequestBody AlertRuleDTO rule) {
    // CPU > 80%, Memory > 90% 등
}
```

### Phase 3: 고급 기능 (3-4주)

#### 8. **YAML 에디터 및 Apply**
```java
@PostMapping("/apply")
public ResultDTO applyYaml(@RequestBody String yaml) {
    // kubectl apply -f 기능
}
```

#### 9. **Helm Chart 관리**
```java
@PostMapping("/helm/install")
public ResultDTO installChart(@RequestBody HelmChartDTO chart) {
    // Helm install
}

@GetMapping("/helm/releases")
public ResultDTO listReleases() {
    // Helm list
}
```

#### 10. **리소스 권장 사항 (VPA 연동)**
```java
@GetMapping("/recommendations/pod/{namespace}/{pod}")
public ResultDTO getRecommendations(
    @PathVariable String namespace,
    @PathVariable String pod
) {
    // 최적 CPU/Memory 권장
}
```

#### 11. **멀티 클러스터 비교 뷰**
- 여러 클러스터의 리소스 상태 한눈에 비교
- 클러스터 간 워크로드 마이그레이션

#### 12. **RBAC 관리**
```java
@GetMapping("/rbac/roles")
public ResultDTO listRoles() {
    // Role, ClusterRole 조회
}

@PostMapping("/rbac/rolebinding")
public ResultDTO createRoleBinding(@RequestBody RoleBindingDTO dto) {
    // RoleBinding 생성
}
```

### Phase 4: 사용자 경험 개선 (4-5주)

#### 13. **실시간 리소스 모니터링 (WebSocket)**
```java
@MessageMapping("/watch/pods/{namespace}")
public void watchPods(@DestinationVariable String namespace) {
    // K8S Watch API 연동
    // 실시간 Pod 상태 변경 푸시
}
```

#### 14. **리소스 검색 및 필터링**
```java
@GetMapping("/search")
public ResultDTO search(
    @RequestParam String query,
    @RequestParam(required = false) String resourceType,
    @RequestParam(required = false) String namespace
) {
    // 전체 리소스 검색
}
```

#### 15. **사용자 권한 관리 (RBAC)**
- Admin, Developer, Viewer 역할
- 클러스터별 접근 권한
- 팀 단위 관리

#### 16. **감사 로그 (Audit Log)**
```java
@GetMapping("/audit/logs")
public ResultDTO getAuditLogs(
    @RequestParam String userId,
    @RequestParam String action,
    @RequestParam String timeRange
) {
    // 누가, 언제, 무엇을 했는지 추적
}
```

#### 17. **백업 및 복원**
```java
@PostMapping("/backup/cluster/{clusterId}")
public ResultDTO backupCluster(@PathVariable UUID clusterId) {
    // 클러스터 리소스 백업 (YAML)
}

@PostMapping("/restore")
public ResultDTO restore(@RequestBody BackupDTO backup) {
    // 백업에서 복원
}
```

## 📊 우선순위 매트릭스

| 기능 | 중요도 | 난이도 | 우선순위 |
|------|--------|--------|----------|
| K8S CRUD 완성 | 높음 | 중 | 1 |
| Gateway JWT 검증 | 높음 | 중 | 2 |
| Pod 로그 조회 | 높음 | 낮음 | 3 |
| 메모리 누수 수정 | 높음 | 중 | 4 |
| 리소스 이벤트 | 중 | 낮음 | 5 |
| Prometheus 연동 | 중 | 높음 | 6 |
| YAML Apply | 중 | 중 | 7 |
| WebSocket 실시간 | 낮음 | 높음 | 8 |
| Helm 관리 | 낮음 | 높음 | 9 |

## 🎯 다음 스프린트 제안 (2주)

### Week 1
1. ClusterController HTTP 메서드 수정
2. Delete 로직 수정
3. K8S 리소스 CRUD 구현 (Deployment)
4. Pod 로그 조회 기능

### Week 2
5. Gateway JWT 검증 필터 구현
6. UserClusterClientManager 메모리 누수 수정
7. 리소스 이벤트 조회
8. 전역 예외 처리 개선

이렇게 진행하면 2주 안에 핵심 기능이 완성되고, 실제 사용 가능한 K8S 관리 플랫폼이 될 것 같습니다!
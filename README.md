# appcenter-server-metric-monitoring

앱센터 서버의 **중앙 모니터링 스택**. 각 서비스는 메트릭을 노출하기만 하면 되고, 수집, 시각화는 이 스택이 담당한다.

| 구성 | 주소 |
|---|---|
| Grafana | https://monitoring.inuappcenter.kr |
> **Grafana 계정(아이디, 비밀번호)은 [@xunssoie](https://github.com/xunssoie)에게 문의.**

<br>

아래 예시의 `your-application`, `your-server-prod`, `your-network` 등은 각자 서비스 값으로 바꿔 쓴다.

<br>

---

### 1. 애플리케이션에서 할 일

#### 1-1. 메트릭 노출

```gradle
// build.gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'io.micrometer:micrometer-registry-prometheus'
```

```yaml
# application-prod.yml
management:
  server:
    port: 8081
  endpoints:
    web:
      exposure:
        include: health,prometheus
  metrics:
    tags:
      application: your-application     # 서비스, 환경 단위로 유일하게
    distribution:
      percentiles-histogram:
        http.server.requests: true      # p95/p99를 보려면 필수
      slo:
        http.server.requests: 100ms,200ms,500ms,1s,2s,5s
```

<br>

#### 1-2. 네트워크 이름을 고정한다

중앙 스택이 `external`로 참조할 이름이므로 배포 경로에 따라 흔들리면 안 된다. **배포 스크립트가 네트워크를 만들고, compose는 `external`로 참조한다.**

```yaml
# docker-compose-prod.yml
networks:
  your-network:
    external: true
    name: your-network
```

<br>

#### 1-3. 배포 스크립트에서 `down`을 쓰지 않는다

`down`은 네트워크를 지우고, 그러면 중앙 Prometheus의 스크랩이 조용히 끊긴다. 서비스 단위로 갱신한다.

```bash
# .github/workflows/cd-prod.yml (배포 스크립트)
docker-compose -f docker-compose-prod.yml pull your-server-prod
docker-compose -f docker-compose-prod.yml up -d --no-deps --force-recreate your-server-prod
```

> ⚠️ **`--remove-orphans` 옵션은 절대 붙이지 X**

<br>

---

### 2. 본 레포에서 수정해야 할 것

새 서비스를 붙일 때 손대는 파일은 두 개다. 수정 후 커밋/푸시를 진행하면 자동으로 CD가 돌아 서버에 반영된다.

#### 2-1. `prometheus/prometheus.yml` - 스크랩 대상 추가

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'your-service-prod'                 # {서비스}-{환경}
    static_configs:
      - targets: ['your-server-prod:8081']        # 컨테이너 이름 : 컨테이너 내부 포트
    metrics_path: '/actuator/prometheus'
```

`targets`에는 컨테이너 이름을 쓴다. 호스트 IP나 publish된 포트가 아니다.

<br>

#### 2-2. `docker-compose.yml` - 서비스 네트워크에 조인

Prometheus가 서비스 네트워크에 조인해야 컨테이너 이름으로 접근할 수 있다. Loki를 조회하려면 Grafana도 조인한다.

```yaml
# docker-compose.yml
services:
  prometheus:
    networks:
      - monitoring
      - gravit-prod
      - gravit-dev
      - your-network        # 추가

  grafana:
    networks:
      - monitoring
      - your-network        # 서비스의 Loki를 조회할 때만

networks:
  monitoring:
    driver: bridge
  gravit-prod:
    external: true
    name: prod_gravit-prod
  your-network:             # 추가
    external: true
    name: your-network
```

<br>

---

### 3. 검증

#### 앱이 메트릭을 내보내는가

```bash
docker run --rm --network your-network curlimages/curl -s \
  http://your-server-prod:8081/actuator/prometheus | head -3

# 성공 예시
# HELP application_ready_time_seconds Time taken for the application to be ready to service requests
# TYPE application_ready_time_seconds gauge
application_ready_time_seconds{application="your-application",...} 5.779
```

<br>

#### Prometheus가 서비스 네트워크에 붙어 있는가

```bash
docker inspect appcenter-prometheus -f '{{json .NetworkSettings.Networks}}'

# 성공 예시 - 네트워크 목록에 your-network가 있어야 한다
{"appcenter-server-metric-monitoring_monitoring":{...},"your-network":{...}}
```

마지막으로 Prometheus UI(`서버:9095`) → **Status → Targets**에서 해당 job이 `UP`인지 확인한다.

<br>

---

### 4. 컨벤션

| 항목 | 규칙 | 예 |
|---|---|---|
| `job_name` | `{서비스}-{환경}` | `your-service-prod` |
| `application` 태그 | 서비스, 환경 단위로 유일하게 | `your-application` |
| 컨테이너 이름 | `container_name`으로 고정 | `your-server-prod` |
| 네트워크 이름 | `name:` 명시 또는 사전 생성으로 고정 | `your-network` |


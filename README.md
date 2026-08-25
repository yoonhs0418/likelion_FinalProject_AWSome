# AWSome — 하이브리드/멀티 클라우드 배포·운영 인프라

> 과제로 주어진 금융 서비스 애플리케이션을 **On-premise(OpenShift) + AWS 하이브리드 환경에 배포하고 운영**한 인프라 구축 프로젝트입니다.
>
> 🏆 **멋쟁이사자처럼 클라우드 엔지니어링 스쿨 1기 최종 프로젝트 [대상] 수상**

- **기간** : 2024. 09. 30 ~ 2024. 11. 01 (사전 셋업 2024. 09. 27 ~ 09. 30)
- **인원** : 5명 (박종찬, 손채린, 윤현석, 정소은, 최건우)
- **스택** : `AWS` `GCP(GKE)` `OpenShift` `Kubernetes` `Istio` `ArgoCD` `GitHub Actions` `Prometheus` `Grafana` `ELK Stack` `KEDA` `Nexus` `Python(Boto3)` `Slack API` `Node.js`

---

## 0. 이 저장소를 읽기 전에

**이 저장소는 애플리케이션 개발 산출물이 아니라 배포·운영 인프라 산출물입니다.**

- 배포 대상인 금융 서비스 앱(인증·자산관리·부동산 조회)은 **과제로 주어진 것**이고,
  프로젝트의 목표는 그 앱을 **하이브리드 클라우드 위에서 안정적으로 굴러가게 만드는 것**이었습니다.
- 따라서 이 저장소에 담긴 것은 **Kubernetes 매니페스트, HPA/PromQL 룰, CI/CD 파이프라인,
  운영 자동화 스크립트**이며, 앱 코드는 배포 대상으로서 함께 보관되어 있을 뿐입니다.
- **실제 작업의 대부분은 커밋으로 남지 않습니다.** OpenShift 콘솔, AWS 콘솔,
  Site-to-Site VPN 설정, ArgoCD·Kibana·Grafana UI, 그리고 클러스터 안에서 직접 수행한 작업이
  결과물의 중심이기 때문입니다. **커밋 수로 작업량을 판단할 수 없는 저장소**입니다.
  (히스토리는 자격증명 노출을 정리하기 위해 2026년에 한 번 리셋되었습니다. [11장](#11-보안-관련-안내))

---

## 1. 담당 역할

| 이름 | 역할 | 담당 영역 |
|:-----|:-----|:----------|
| **윤현석** | **PL** | **ELK 로그 파이프라인 구축 · OpenShift 클러스터 구축 · Site-to-Site VPN** |
| 박종찬 | <!-- TODO --> | <!-- TODO --> |
| 손채린 | <!-- TODO --> | <!-- TODO --> |
| 정소은 | <!-- TODO --> | <!-- TODO --> |
| 최건우 | <!-- TODO --> | <!-- TODO --> |

<!--
TODO: 나머지 4명의 역할(PM / CI/CD / Monitoring / Infra)을 이름과 1:1로 확정해서 채울 것.
      이전 README는 "박종찬, 손채린, 윤현석, 정소은, 최건우 — PM / PL / CI/CD / Monitoring / Infra"
      처럼 이름과 역할을 나란히 나열해 윤현석=CI/CD 로 오독되게 되어 있었음.
      발표자료 기준으로 윤현석은 PL(ELK / OpenShift / VPN)이 맞음.
-->

이 README는 **PL(윤현석) 관점**에서 작성되었으며,
본인 담당 영역인 [ELK 로그 파이프라인](#6-관찰성--elk-로그-파이프라인)을 가장 자세히 다룹니다.

---

## 2. 요구사항과 실제 구성

| 구분 | 과제 요구사항 | 실제 구현 |
|:-----|:--------------|:----------|
| 배포 환경 | 하이브리드 클라우드 (On-prem + AWS) | 하이브리드 **+ GCP GKE를 더한 멀티 클라우드** |
| 배포 대상 앱 | Rust + Node.js | Node.js(Express/EJS) 앱을 실제 배포 대상으로 운영 |
| 배포 방식 | — | GitHub Actions(CI) + **ArgoCD(GitOps CD)** |
| 확장 | — | HPA v2 커스텀 메트릭 + **KEDA** + **Cluster Autoscaler** |

요구사항은 **하이브리드까지**였습니다.
여기서 한 단계 더 나아가 **GCP GKE를 세 번째 런타임으로 붙이고 Route 53으로 트래픽을 분산**해
멀티 클라우드로 확장한 것이 이 프로젝트의 차별점이자 수상 포인트입니다.

---

## 3. 아키텍처

```
                          Route 53 (트래픽 분산)
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
        ┌────────────────────────┐      ┌────────────────────────┐
        │  AWS (ap-northeast-3)  │      │  GCP  GKE              │
        │  Kubernetes Cluster    │      │  Kubernetes Cluster    │
        │  Istio Gateway / VS    │      │                        │
        │  frontend / backend    │      │  frontend / backend    │
        │  HPA v2 (1~5) + KEDA   │      │                        │
        │  Cluster Autoscaler    │      │  (Cloud SQL → RDS 이관)│
        │  RDS(MySQL) · S3       │      │                        │
        └───────────┬────────────┘      └────────────────────────┘
                    │ Site-to-Site VPN
                    ▼
        ┌────────────────────────────────────────────┐
        │  On-premise (OpenShift)                    │
        │  Nexus Private Registry                    │
        │  Redis (Session Store)                     │
        └────────────────────────────────────────────┘

  관찰성 : kube-prometheus-stack + Grafana
           ELK (Filebeat/Metricbeat → Logstash → Elasticsearch → Kibana)
  운영   : ArgoCD (GitOps)  ·  Slack ChatOps (Python/Boto3)
```

### 왜 하이브리드인가
금융권은 규제·보안상 핵심 데이터와 코어 시스템을 온프레미스에 두는 경우가 많고,
트래픽 변동이 큰 대고객 채널은 클라우드의 탄력성이 유리합니다.
이 제약을 반영해 **코어(프라이빗 레지스트리·세션 스토어)는 온프레미스(OpenShift),
변동 부하는 AWS**로 분리했고, 두 구간을 **Site-to-Site VPN**으로 연결했습니다.

### 왜 멀티 클라우드까지 갔는가
하이브리드만으로는 **AWS 리전 단위 장애**에 대응할 수 없습니다.
GKE에 동일한 워크로드를 올리고 **Route 53으로 GKE ↔ AWS 트래픽을 분산**해
한쪽이 죽어도 서비스가 유지되는 구성을 만들었습니다.
데이터 계층은 **Cloud SQL → RDS로 마이그레이션**해 한쪽으로 통일했고,
이미지 등 정적 자산은 **S3**로 분리했습니다.

---

## 4. CI/CD — GitHub Actions + ArgoCD

파이프라인 전체가 [`ci-cd/docker-action.yml`](ci-cd/docker-action.yml) 에 정의되어 있습니다.
(원래 위치는 `Rust-Tool/.github/workflows/` 였으며, 아래 8장의 이유로 해당 디렉터리를 정리하면서 옮겼습니다.)

**CI (self-hosted runner)**

1. Docker 이미지 빌드
2. **Trivy 취약점 스캔** — `CRITICAL,HIGH` / `--ignore-unfixed` 로 OS·라이브러리 취약점 검사
3. 컨테이너를 실제로 띄운 뒤 **`/health` 엔드포인트 헬스체크 게이트**
   (5초 간격 10회 재시도, 실패 시 파이프라인 중단)
4. 통과한 이미지만 **Nexus 프라이빗 레지스트리**로 push

**CD (GitOps)**

```bash
argocd app sync this-is-my-app --auth-token $ARGOCD_TOKEN --server $ARGOCD_SERVER --insecure --force
kubectl rollout restart deployment/backend-server -n bank-namespace
```

헬스체크를 통과하지 못한 이미지는 레지스트리에 올라가지 않으므로,
**ArgoCD가 동기화하는 시점에는 기동이 검증된 이미지만 남습니다.**
파이프라인 종료 시에는 테스트 컨테이너와 dangling 이미지를 정리해
self-hosted runner에 잔여물이 쌓이지 않게 했습니다.

---

## 5. 커스텀 메트릭 오토스케일링 (PromQL + HPA v2)

CPU 사용률은 후행 지표라서, 트래픽이 급증하면 CPU가 오르기 전에 이미 응답 지연이 발생합니다.
이벤트성 트래픽에 선제 대응하기 위해 **PromQL 레코딩 룰을 직접 작성**하고
Prometheus Adapter를 통해 HPA v2의 `Object` 메트릭으로 연결했습니다.

### 5-1. Istio 메트릭을 오토스케일링 입력으로

앱 코드를 한 줄도 고치지 않고 요청 단위 지표를 얻기 위해,
**Istio 사이드카가 이미 수집하고 있는 `istio_requests_total` 을 스케일링 입력으로 사용**했습니다.
수집 경로는 [`istio-monitoring.yaml`](Node-Kubernetes/horizontal-pod-autoscalers/istio-monitoring.yaml) 에서
`ServiceMonitor`(istiod) + `PodMonitor`(envoy `/stats/prometheus`, 15초 간격)로 구성했습니다.

### 5-2. PromQL 레코딩 룰 5종

[`prometheus-rule.yaml`](Node-Kubernetes/horizontal-pod-autoscalers/prometheus-rule.yaml) —
frontend/backend 각각 정의했으며, 아래는 backend 기준입니다.

**RPS — 레플리카 1개당 초당 요청 수**

```promql
sum(rate(istio_requests_total{reporter="destination", destination_app="backend"}[5m]))
/ sum(kube_deployment_status_replicas_available{deployment="backend-server", namespace="bank-namespace"})
```

> 전체 RPS가 아니라 **가용 레플리카 수로 나눈 값**을 기록합니다.
> 이미 스케일아웃된 상태에서 총량만 보면 계속 늘리게 되므로, 파드당 부하로 정규화했습니다.

**CPU — 요청량(request) 대비 사용률**

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="bank-namespace", pod=~"backend-server-.*"}[5m]))
/ sum(kube_pod_container_resource_requests{namespace="bank-namespace", pod=~"backend-server-.*", resource="cpu"})
```

**응답 시간 — 평균 지연(ms)**

```promql
sum(rate(istio_request_duration_milliseconds_sum{namespace="bank-namespace", destination_app="backend"}[5m]))
/ clamp_min(sum(rate(istio_requests_total{namespace="bank-namespace", destination_app="backend"}[5m])), 1)
```

> 분모에 `clamp_min(..., 1)` 을 둔 이유: 트래픽이 없는 구간에서 분모가 0이 되면
> 결과가 `NaN` 이 되고, HPA는 메트릭을 읽지 못해 스케일링 판단 자체가 멈춥니다.

**메모리 — working set**

```promql
sum(container_memory_working_set_bytes{namespace="bank-namespace", pod=~"backend-server-.*"})
```

**네트워크 — 송·수신 합산**

```promql
sum(rate(container_network_receive_bytes_total{namespace="bank-namespace", pod=~"backend-server-.*"}[5m]))
+ sum(rate(container_network_transmit_bytes_total{namespace="bank-namespace", pod=~"backend-server-.*"}[5m]))
```

### 5-3. HPA v2 — 5개 메트릭 동시 적용

[`bank-app-back-hpa.yaml`](Node-Kubernetes/horizontal-pod-autoscalers/bank-app-back-hpa.yaml) /
[`bank-app-front-hpa.yaml`](Node-Kubernetes/horizontal-pod-autoscalers/bank-app-front-hpa.yaml) —
`minReplicas: 1`, `maxReplicas: 5`, 5개 지표를 모두 `type: Object` 로 걸어
**가장 먼저 임계값을 넘는 지표가 스케일아웃을 트리거**합니다.

| 메트릭 | backend 임계값 | frontend 임계값 |
|:-------|:---------------|:----------------|
| `*_cpu_usage_percentage` | 0.8 | 0.8 |
| `*_memory_usage` | 540Mi | 540Mi |
| `*_requests_per_second` | 5 | 5 |
| `*_response_time` | 5 (ms) | 5 (ms) |
| `*_network_traffic` | 50000 (B/s) | 100000 (B/s) |

메모리 임계값 540Mi는 [`bank-limitrange.yaml`](Node-Kubernetes/horizontal-pod-autoscalers/bank-limitrange.yaml) 의
컨테이너 기본 요청량 600Mi 대비 **90% 지점**으로, OOM 이전에 스케일아웃이 걸리도록 잡은 값입니다.
프론트엔드 네트워크 임계값을 백엔드의 2배로 둔 것은 정적 자산 전송량 차이를 반영한 것입니다.

### 5-4. 파드 위 계층 — KEDA · Cluster Autoscaler

- **KEDA** : 이벤트 기반 스케일링을 담당해 HPA만으로 다루기 어려운 워크로드를 보완
- **Cluster Autoscaler** : 파드가 늘어 노드 자원이 부족해지면 **노드 자체를 확장**
  (HPA만 있으면 `Pending` 파드가 쌓인 채 멈춥니다)

<!--
TODO: KEDA ScaledObject의 트리거 종류·임계값, Cluster Autoscaler 노드 그룹 min/max를 확인해서 채울 것.
      두 리소스 모두 클러스터/콘솔에서 직접 구성해 이 저장소에 매니페스트가 없음.
-->

---

## 6. 관찰성 — ELK 로그 파이프라인

> **본인(PL, 윤현석) 담당 영역이자 1인 1주제 발표 주제입니다.**

### 6-1. 왜 필요했는가

워크로드가 여러 클러스터(AWS · GKE · OpenShift)로 흩어지면 `kubectl logs` 로는 장애를 추적할 수 없습니다.
파드는 스케일인되면 로그와 함께 사라지고, 같은 요청이 어느 클러스터에서 처리됐는지도 알 수 없습니다.
**로그를 파드 바깥에 모아 한 곳에서 질의할 수 있는 계층**이 필요했습니다.

### 6-2. 구성

```
Filebeat   (DaemonSet, 각 노드의 컨테이너 로그 수집)
Metricbeat (노드/파드 메트릭 수집)
        │
        ▼
   Logstash       (파싱 · 필드 매핑 · 정규화)
        │
        ▼
 Elasticsearch    (StatefulSet, 인덱싱/저장)
        │
        ▼
    Kibana        (대시보드 · 질의)
```

실제 매니페스트와 Helm values는 별도 저장소에 있습니다 →
**[yoonhs0418/elk_backup](https://github.com/yoonhs0418/elk_backup)**
(Elastic 8.5.1. 어느 파일이 직접 작성분이고 어느 것이 upstream 원본인지 그쪽 README에 구분해 두었습니다.)

### 6-3. Beats 출력을 Logstash 경유로 전환

Filebeat/Metricbeat가 Elasticsearch로 **직접** 보내던 구성을 **Logstash 경유**로 바꿨습니다.
(Helm 차트 기본값이 `output.elasticsearch` 이므로, 이를 주석 처리하고 `output.logstash` 로 교체)

```yaml
# filebeat/values.yaml, metricbeat/values.yaml 공통
output.logstash:
  hosts: "team1-logstash...:5044"
```

Logstash를 중간에 둔 이유는 파이프라인 한 곳에서 입력을 분기해 가공하기 위해서입니다.

```ruby
filter {
  if [agent][type] == "metricbeat" {
    mutate { add_field => { "metric_beat_processed" => "true" } }
  } else if [agent][type] == "filebeat" {
    if [parsed][log][level] == "ERROR" {
      mutate { add_field => { "log_severity" => "critical" } }
    } else if [parsed][log][level] == "WARN" {
      ...
```

메트릭(metricbeat)은 로그 레벨 분류가 무의미하므로 태깅만 하고 통과시키고,
로그(filebeat)만 레벨을 `log_severity` 로 정규화합니다.
두 종류의 입력을 한 곳에서 다르게 처리하려면 수집기와 저장소 사이에 계층이 필요했습니다.

### 6-4. Logstash — 차트 대신 매니페스트 직접 작성

Elasticsearch/Filebeat/Kibana/Metricbeat는 Helm 차트(`elastic/helm-charts` v8.5.1)를 썼지만,
**Logstash만 차트를 쓰지 않고 ConfigMap 2개 + StatefulSet + Service를 직접 작성**했습니다.
(같은 릴리스에 logstash 차트가 존재함에도 그렇게 했습니다.)

차트를 쓴 나머지에서도 기본값을 그대로 쓸 수 없어 상당 부분을 덮어썼습니다.

| 대상 | 차트 기본값 | 변경 | 이유 |
|:--|:--|:--|:--|
| Elasticsearch | `antiAffinity: hard` | `soft` | 노드 수가 부족해 `hard` 면 파드가 `Pending` |
| Elasticsearch | tolerations 없음 | control-plane `NoSchedule` 허용 | 마스터 노드에도 배치 |
| Elasticsearch | requests 1000m/2Gi | 500m/1Gi | 요청량을 낮춰 스케줄 가능하게 |
| Kibana | `initialDelaySeconds: 10`, `successThreshold: 3` | `60`, `1` | 기동이 느려 readiness가 반복 실패 |
| 전체 | `elasticsearch-master-certs` | 자체 발급 인증서 Secret | 차트 자동 생성 인증서 대신 직접 발급분 마운트 |

**배운 점** — 차트의 편의는 기본값이 내 환경과 맞을 때만 성립합니다.
맞지 않으면 `values.yaml` 표면만 봐서는 무엇이 문제인지 알 수 없고,
결국 템플릿과 실제 생성되는 리소스를 열어봐야 원인이 잡힙니다.

### 6-5. TLS 검증 범위를 나눈 판단

전부 끄는 대신, **데이터가 흐르는 경로의 검증은 지키고 브라우저 접속 경로만 완화**했습니다.

| 경로 | 설정 | 상태 |
|:--|:--|:--|
| Logstash → Elasticsearch (데이터) | `cacert` 마운트 + `ssl_certificate_verification => true` | **검증 켬** |
| Logstash → Elasticsearch (모니터링) | `ssl.verification_mode: "certificate"` | **검증 켬** |
| Kibana → Elasticsearch | `verificationMode: "none"` | 검증 끔 |

Logstash 쪽은 CA 인증서를 컨테이너에 마운트한 뒤 검증을 켰습니다.
즉 "검증을 켰다"가 아니라 **CA를 넣어 검증이 실제로 통과하도록 만든** 것입니다.

<!-- TODO(본인 확인): Kibana만 검증을 끈 것이 의도적 판단이었는지, 시간에 쫓긴 임시 조치였는지.
     후자면 그대로 쓸 것. 의도적이었다고 쓰려면 근거가 필요함. -->

### 6-6. 남은 결함 — 레벨 매핑은 동작하지 않았다 (2026년 재검토 중 발견)

6-3의 `log_severity` 매핑은 **당시에 동작하지 않았습니다.** 이유가 둘 있고, 각각 독립적입니다.

**(1) 파싱할 JSON이 없었다.**
Filebeat는 `decode_json_fields` 로 `message` 안의 JSON을 꺼내려 하는데,
배포 대상인 Node 앱은 JSON 로거(winston·pino 등)를 쓰지 않고
`console.log` / `console.error` 로 평문만 출력합니다. (백엔드·프론트 합쳐 56곳, 전부 평문)
입력이 `type: container` 이므로 `message` 에는 앱이 찍은 평문 그대로가 들어갑니다.
꺼낼 JSON이 없으니 어떤 필드도 만들어지지 않습니다.

**(2) 필드 경로가 어긋나 있었다.**
설령 JSON 로그였더라도, `decode_json_fields` 의 `target: ""` 은 디코딩 결과를
이벤트 **루트**에 펼칩니다. 즉 경로는 `[log][level]` 이 됩니다.
그런데 파이프라인 필터는 `[parsed][log][level]` 을 참조합니다 — `target: "parsed"` 를 가정한 경로입니다.
저장소 전체에서 `parsed` 라는 필드를 만드는 곳은 **어디에도 없습니다.**

결과적으로 filebeat 이벤트는 전부 `else` 분기로 떨어져
`log_severity` 는 항상 `"low"` 로만 기록됐을 것입니다.

당시에는 인지하지 못했고, 2년 뒤 저장소를 정리하며 파일을 대조하다 발견했습니다.
**그래서 이 README에는 "필드 매핑 문제를 해결했다"고 적지 않습니다.**
의도한 설계와 실제 동작이 어긋난 채로 끝난 부분입니다.

> 고치려면 Filebeat 쪽을 `target: "parsed"` 로 바꾸거나 필터를 `[log][level]` 로 맞추고,
> 그 전에 **앱이 구조화 로그를 출력하도록** 만드는 것이 먼저입니다.
> 평문 로그에 파싱 파이프라인을 붙인 것 자체가 순서가 뒤바뀐 설계였습니다.

### 6-7. 메트릭 관찰성

로그(ELK)와 별개로 **kube-prometheus-stack + Grafana**로 메트릭 관찰성을 구성했습니다.
Prometheus는 오토스케일링의 데이터 소스([5장](#5-커스텀-메트릭-오토스케일링-promql--hpa-v2))를 겸합니다.

<!-- TODO: Grafana 대시보드 구성(어떤 패널을 상시로 봤는지)과 알림 규칙 유무를 확인해서 채울 것. -->

---

## 7. ChatOps — Slack 기반 인프라 운영 (Python/Boto3)

AWS 콘솔에 접속하지 않고 Slack에서 인프라를 제어하는 챗봇입니다. ([`Python-SlackBot/`](Python-SlackBot))

- **조회** : `/all-ec2` `/all-rds` `/all-asg` `/all-instance` —
  EC2·RDS·Auto Scaling Group 상태를 CloudWatch 지표(CPU·RAM·네트워크)와 함께 조회
- **제어** : EC2/RDS 시작·중지, ASG `desired capacity` 조정
- **예약** : `/예약 YYYY-MM-DD HH:MM` 으로 시작·중지 스케줄 등록,
  `/예약-목록` 으로 조회·취소 (야간·주말 자동 중지로 실습 비용 절감)
- **IAM** : 클러스터 노드/마스터 role에 필요한 정책 일괄 부착 (`IAMPolicyManager`)
- **UX** : 슬래시 커맨드 → 버튼 UI 제시 → 선택 시 백그라운드 스레드로 실행.
  작업 중복 실행 방지 가드(`is_working`)를 두고, 완료되면 채널에 결과를 회신합니다.

---

## 8. 디렉토리 구조

```
├── Node-Frontend/        # [배포 대상] Node.js(Express) + EJS 프론트엔드
├── Node-Backend/         # [배포 대상] Node.js(Express) REST API (인증 / 자산관리 / 부동산 / 챗봇)
├── Node-Kubernetes/      # ★ K8s 매니페스트 (Kustomize)
│   ├── deployments/                 # Deployment (Istio sidecar 포함, Secret 참조)
│   ├── horizontal-pod-autoscalers/  # HPA v2, PrometheusRule, Prometheus Adapter,
│   │                                #   ServiceMonitor/PodMonitor, LimitRange
│   ├── istio/                       # Gateway, VirtualService
│   ├── namespaces/                  # bank-namespace
│   ├── network-policies/            # NetworkPolicy
│   ├── secrets/                     # Secret 템플릿 (.example.yaml만 커밋)
│   └── services/                    # Service
├── Python-SlackBot/      # ★ ChatOps 챗봇 (Flask, Boto3, Slack SDK)
└── ci-cd/                # ★ GitHub Actions
    └── docker-action.yml         #   빌드 → Trivy → 헬스체크 → Nexus push → ArgoCD sync
```

> 과제 요구사항에 있던 **Rust 기반 컴포넌트는 별도 담당자 작업이므로 이 저장소에서 제외**했습니다.
> ELK 관련 매니페스트는 [별도 저장소](https://github.com/yoonhs0418/elk_backup)에 있습니다.

---

## 9. 로컬에서 매니페스트 확인하기

```bash
kubectl kustomize Node-Kubernetes/
```

Secret은 커밋되지 않습니다. 배포 시에는 템플릿을 복사해 값을 채워 사용합니다.

```bash
cp Node-Kubernetes/secrets/bank-backend-secret.example.yaml Node-Kubernetes/secrets/bank-backend-secret.yaml
```

값을 입력한 뒤 적용합니다.

```bash
kubectl apply -f Node-Kubernetes/secrets/bank-backend-secret.yaml
```

---

## 10. 알려진 한계

정직하게 남겨둡니다.

- [`network-policies/network-policy.yaml`](Node-Kubernetes/network-policies/network-policy.yaml) 은
  **`allow-all` 정책**입니다. 네트워크 격리를 실제로 적용한 것이 아니라 자리만 잡아둔 상태입니다.
- `deployments/kustomization.yaml` 이 참조하는 `frontend-deployment.yaml` 과
  `secrets/` 하위 kustomization이 저장소에 없어 `kubectl kustomize` 가 그대로는 실행되지 않습니다.
  <!-- TODO: frontend-deployment.yaml 원본이 남아 있으면 복구할 것. -->
- 6-6에 적은 대로 **로그 레벨 매핑(`log_severity`)은 실제로 동작하지 않았습니다.**
  앱이 평문 로그를 찍는데 JSON 파싱 파이프라인을 붙였고, 필드 경로도 어긋나 있었습니다.
- KEDA · Cluster Autoscaler · GKE · Route 53 · ELK 구성은 **콘솔과 클러스터에서 직접 수행**되어
  이 저장소에 매니페스트가 남아 있지 않습니다.

---

## 11. 보안 관련 안내

초기 커밋에 RDS 엔드포인트와 DB/Redis 자격증명이 평문으로 포함되어 있었습니다.
현재는 **Kubernetes Secret 참조 방식으로 전환**했고,
노출된 값이 히스토리에 남지 않도록 **저장소 히스토리를 리셋**했습니다.
해당 자격증명은 모두 폐기된 실습용 값입니다.

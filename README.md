# 장애 대응형 비동기 파일 처리 플랫폼

대규모 파일 처리 서비스를 가정하여, 사용자가 업로드한 파일의 **썸네일 생성·포맷 변환·데이터 분석** 등의 작업을 비동기로 처리하는 플랫폼을 구축합니다.

파일 처리 작업은 종류에 따라 처리 시간과 CPU·메모리 사용량이 다르고, 특정 작업에만 요청이 집중될 수 있습니다. 또한 작업을 처리하는 Worker가 장애를 일으키거나 새로운 버전을 배포하는 과정에서도 작업 처리가 중단되지 않아야 합니다.

이에 따라 작업 유형별 Worker를 독립적으로 운영하고, **Queue의 작업량에 따라 필요한 Worker만 자동으로 확장하며 장애 발생 시 자동 복구할 수 있는 구조**를 설계합니다. 또한 Retry / DLQ를 통해 실패한 작업을 격리하고, 무중단 배포 및 Rollback을 지원합니다.

전체 시스템은 Spring Boot와 Docker를 기반으로 구축하고, Kubernetes와 KEDA를 활용하여 Worker의 배포·확장·복구를 자동화합니다. 이후 실제 부하와 장애를 주입하여 시스템의 확장성과 복원력을 검증합니다.

<br>

## 1. 서비스 시나리오

사용자가 파일을 업로드하면 파일 처리에 필요한 작업이 Queue에 등록되고, 작업 유형별 Worker가 이를 비동기로 처리합니다.

    사용자
      │
      ▼
    Upload API
      │
      ▼
     Queue
      │
      ├──────────────┬──────────────┐
      ▼              ▼              ▼
    Thumbnail      Converter      Analyzer
    Worker         Worker         Worker
      │              │              │
      └──────────────┼──────────────┘
                     ▼
               Object Storage

각 Worker는 서로 다른 처리 특성을 가집니다.

| Worker | 처리 특성 |
|---|---|
| Thumbnail | 낮은 CPU 사용량 / 짧은 처리 시간 |
| Converter | 높은 CPU 사용량 / 긴 처리 시간 |
| Analyzer | 긴 처리 시간 / 작업량 변동성이 큼 |

특정 작업에 요청이 집중되는 경우 해당 Worker만 독립적으로 확장합니다.

    Converter Queue

    작업량 증가
        ↓
    Queue Backlog 증가
        ↓
    Converter Worker
    2 Pods → 5 Pods → 10 Pods
        ↓
    처리량 증가
        ↓
    Backlog 감소
        ↓
    Worker Scale-in

<br>

## 2. 핵심 요구사항

### Scalability

- Normal: **10~50 jobs/sec**
- Peak: **최대 1,000 jobs/sec**
- Queue backlog에 따른 Worker 자동 Scale-out / Scale-in
- Workload별 독립적인 확장

### Fault Tolerance

- Worker 장애 시 자동 복구
- 장애로 인한 작업 유실 방지
- 일시적인 작업 실패에 대한 자동 Retry
- 반복 실패 작업은 DLQ로 격리

### Deployment

- 서비스 운영 중 무중단 배포
- 비정상 버전 배포 시 Rollback
- 배포 중에도 기존 작업 처리 지속

### Processing

- 정상 상황에서 **95% 이상의 작업을 30초 이내 처리**
- API Availability **≥ 99.9%**

> 위 수치는 실제 서비스가 아닌 시스템 설계 및 부하 테스트를 위한 가정입니다.

<br>

## 3. 장애 시나리오

| 상황 | 기대 동작 |
|---|---|
| Worker Failure | 장애 Worker 자동 복구 |
| Traffic Spike | Queue backlog 기반 Worker Scale-out |
| 특정 Workload 급증 | 해당 Worker Pool만 독립적으로 확장 |
| 작업 반복 실패 | Retry 후 DLQ 격리 |
| Deployment Failure | 이전 버전으로 Rollback |

<br>

## 4. Architecture

    Client
      │
      ▼
    Spring Boot API
      │
      ▼
    SQS
      │
      ├─────────────┬─────────────┐
      ▼             ▼             ▼
    Thumbnail     Converter     Analyzer
    Worker        Worker        Worker
      │             │             │
      └─────────────┼─────────────┘
                    ▼
              Object Storage
                    │
               PostgreSQL

    ┌─────────────────────────────────┐
    │       Kubernetes Cluster        │
    │                                 │
    │  KEDA      Prometheus  Grafana  │
    └─────────────────────────────────┘

각 Worker는 Kubernetes의 독립적인 Deployment로 운영합니다.

Queue backlog가 증가하면 KEDA가 이를 감지하여 해당 Worker의 Replica 수를 조정합니다.

    SQS
     │
     │ Queue Backlog 증가
     ▼
    KEDA
     │
     ▼
    Kubernetes Deployment
     │
     ├── Worker Pod
     ├── Worker Pod
     ├── Worker Pod
     └── ...

<br>

## 5. Technology Stack

| Category | Technology |
|---|---|
| Backend | Java, Spring Boot |
| Container | Docker |
| Orchestration | Kubernetes |
| Autoscaling | KEDA |
| Message Queue | AWS SQS |
| Database | PostgreSQL |
| Object Storage | Amazon S3 |
| Monitoring | Prometheus, Grafana |
| CI/CD | GitHub Actions |
| Load Testing | k6 |

<br>

## 6. 검증

실제 부하와 장애를 주입하여 시스템의 확장성과 복원력을 검증합니다.

### Load Test

- Queue 처리량
- Worker Replica Count
- Processing Throughput
- P95 Latency
- Error Rate

### Failure Test

- Worker Pod 강제 종료
- 특정 Worker의 반복적인 작업 실패
- Queue backlog 급증
- 비정상 버전 배포

### Recovery

- Worker Recovery Time
- Job Loss
- Retry 성공률
- DLQ 발생량
- 서비스 중단 여부

실험 결과는 실제 측정값을 기준으로 기록합니다.

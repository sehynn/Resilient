# Kubernetes 기반의 장애 대응형 비동기 작업 처리 플랫폼

대규모 파일 처리 서비스를 가정합니다.

사용자가 파일을 업로드하면 썸네일 생성, 포맷 변환, 데이터 분석과 같은 여러 비동기 작업이 발생합니다. 각 작업은 필요한 CPU·메모리와 처리 시간이 서로 다르며, 특정 시점에는 특정 작업에만 요청이 집중될 수 있습니다.

따라서 모든 작업을 동일한 서버에서 처리하는 대신, 작업 유형별 Worker를 독립적으로 운영하고 **작업량에 따라 필요한 Worker를 자동으로 늘리거나 줄일 수 있는 환경**이 필요합니다.

또한 Worker 장애나 배포 실패가 발생하더라도 작업 처리가 중단되지 않아야 합니다. 이를 위해 **Kubernetes를 기반으로 Worker의 배포·복구·확장을 자동화하고, Queue backlog를 기준으로 Workload별 Worker를 독립적으로 Scale-out**하는 시스템을 구축합니다.

구축 이후 실제 트래픽 급증과 Worker 장애를 주입하여 **자동 확장, 장애 복구, 작업 유실 방지 및 시스템 복원력**을 검증합니다.

<br>
<br>

## 1. 서비스 시나리오

### 대규모 파일 처리 플랫폼

```text
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
```

파일 하나가 업로드되면 여러 종류의 작업이 Queue에 등록되고, 각 Worker가 해당 작업을 비동기로 처리합니다.

각 Worker는 서로 다른 특성을 가집니다.

| Worker | 특성 |
|---|---|
| Thumbnail | 낮은 CPU 사용량 / 짧은 처리 시간 |
| Converter | 높은 CPU 사용량 / 긴 처리 시간 |
| Analyzer | 긴 처리 시간 / 높은 작업 변동성 |

또한 트래픽 상황에 따라 특정 작업만 급격하게 증가할 수 있습니다.

```text
Normal

Thumbnail    ███
Converter    ███
Analyzer     ███


Traffic Spike

Thumbnail    ███
Converter    ███████████████████
Analyzer     ███
```

따라서 **Worker를 하나의 서버에서 함께 확장하는 것이 아니라 Workload별로 독립적인 Worker Pool을 구성하고 필요한 Worker만 확장**합니다.

<br>

## 2. 왜 Kubernetes인가?

이 시스템에서는 다음과 같은 운영 문제가 발생할 수 있습니다.

### Worker Scale-out

Converter 작업이 급증하면 Converter Worker만 빠르게 확장해야 합니다.

```text
Converter Queue
      │
      ▼
  Queue Backlog 증가
      │
      ▼
Worker 2 → 5 → 10
```

### Worker Failure

Worker가 비정상 종료되더라도 새로운 Worker가 자동으로 실행되어야 합니다.

```text
Worker Pod
    │
    X  Failure
    │
    ▼
Kubernetes
    │
    ▼
Replacement Pod
```

### Deployment

서비스가 실행 중인 상태에서 새로운 Worker 버전을 배포해야 하며, 배포된 버전에 문제가 발생하면 기존 버전으로 복구할 수 있어야 합니다.

```text
v1
 │
 ▼
Rolling Update
 │
 ▼
v2
 │
 ├── Failure
 │
 ▼
Rollback
 │
 ▼
v1
```

이처럼 **Worker의 수명주기, 자동 확장, 장애 복구, 배포를 지속적으로 관리해야 하기 때문에 컨테이너 오케스트레이션 플랫폼인 Kubernetes를 사용합니다.**

<br>
<br>

## 3. 요구사항

### Traffic

- Normal: **10~50 jobs/sec**
- Peak: **최대 1,000 jobs/sec**

> 부하 테스트 및 시스템 설계를 위한 가정입니다.

### Scalability

- Queue backlog 증가 시 Worker 자동 Scale-out
- Workload별 독립적인 Scale-out
- 작업량 감소 시 Worker Scale-in

### Availability

- API Availability **≥ 99.9%**
- 단일 Worker 장애가 전체 서비스 장애로 이어지지 않아야 함
- 장애 Worker 자동 복구

### Processing

- 정상 상황에서 **95% 이상의 작업을 30초 이내 처리**
- 작업 실패 시 Retry
- 반복 실패 작업은 DLQ로 격리
- 서비스 운영 중 무중단 배포 및 Rollback

<br>
<br>

## 4. 핵심 장애 시나리오

| 상황 | 대응 |
|---|---|
| Worker Failure | Kubernetes Self-healing |
| Traffic Spike | KEDA 기반 Worker Autoscaling |
| Worker 반복 실패 | Retry / DLQ |
| 특정 Workload 급증 | Worker Pool 독립 확장 |
| Deployment Failure | Rolling Update / Rollback |

<br>
<br>

## 5. 검증

실제 부하와 장애를 주입하여 시스템의 확장성과 복원력을 검증합니다.

- Queue Backlog
- Worker Replica Count
- Processing Throughput
- P95 Latency
- Error Rate
- Recovery Time
- Job Loss

실험 결과는 실제 측정값을 기준으로 기록합니다.

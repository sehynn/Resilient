# Kubernetes 기반 장애 대응형 비동기 작업 처리 플랫폼

- 대규모 파일 처리 서비스를 가정하여, 파일 업로드 이후 발생하는 썸네일 생성, 포맷 변환, 데이터 분석 등의 작업을 비동기로 처리하는 플랫폼을 구축합니다.
- 작업량이 급격히 증가하거나 특정 Worker에 장애가 발생하는 상황에서도 안정적으로 서비스를 운영할 수 있도록, 작업 유형별 Worker를 Kubernetes로 독립 운영하고 Queue 기반 자동 확장, 장애 자동 복구, Retry 및 DLQ를 구현합니다.
- 구현 이후 실제 부하와 장애를 주입하여 시스템의 확장성과 복원력을 검증합니다.
<br>
<br>

## 1. 서비스 시나리오

대규모 파일 처리 플랫폼을 가정하였습니다. 

사용자가 파일을 업로드하면 작업 유형에 따라 비동기 작업이 생성되고, 각 Worker가 독립적으로 처리합니다. 

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

각 Worker는 서로 다른 처리 특성을 가지므로 **Workload별 독립적인 Worker Pool**로 구성하고, 작업량에 따라 개별적으로 확장해야 합니다. 
<br>
<br>

## 2. 요구사항

### Traffic

- Normal: **10~50 jobs/sec**
- Peak: **최대 1,000 jobs/sec**

### Availability

- API Availability **≥ 99.9%**
- 단일 Worker 장애가 전체 서비스 장애로 이어지지 않아야 함
- 장애 Worker 자동 복구

### Processing

- 정상 상황에서 **95% 이상의 작업을 30초 이내 처리**
- Queue backlog 증가 시 Worker 자동 Scale-out
- 작업 실패 시 Retry
- 반복 실패 작업은 DLQ로 격리
- 서비스 운영 중 무중단 배포 및 Rollback
 
<br>
<br>

## 3. 핵심 장애 시나리오

| 상황 | 대응 |
|---|---|
| Worker Failure | Kubernetes Self-healing |
| Traffic Spike | KEDA 기반 Worker Autoscaling |
| Worker 반복 실패 | Retry / DLQ |
| 특정 Workload 급증 | Worker Pool 독립 확장 |
| Deployment Failure | Rolling Update / Rollback |
<br>
<br>

## 4. 검증

실제 부하 및 장애를 주입하여 시스템의 복원력과 확장성을 검증합니다.

- Queue Backlog
- Worker Replica Count
- Processing Throughput
- P95 Latency
- Error Rate
- Recovery Time
- Job Loss

실험 결과는 실제 측정값을 기준으로 기록합니다.

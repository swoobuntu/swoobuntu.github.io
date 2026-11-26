---
title: "Bank of Anthos 멀티클라우드 재해복구(DR)"
kicker: "KDT 최종 프로젝트"
summary: "AWS가 통째로 사라져도 은행 서비스를 GCP에서 이어가려면 무엇이 필요하고, 그때 무엇을 잃는가."
description: "AWS EKS 주 사이트와 GCP 대기 사이트를 잇는 크로스클라우드 재해복구 — DMS CDC, HA VPN, Terraform, OpenTelemetry, 감사 체인"
meta:
  - { label: "기간", value: "2026.08.31 – 10.02 (5주)" }
  - { label: "과정", value: "KDT 멀티클라우드 엔지니어 부트캠프 4기 · 최종 프로젝트" }
  - { label: "담당 파트", value: "Database(DR) · Observability · Governance" }
  - { label: "핵심 기술", value: "Terraform · EKS · DMS · Cloud SQL · OpenTelemetry" }
repos:
  - { label: "프로젝트 문서 저장소", url: "https://github.com/swoobuntu/multicloud-dr-bank-of-anthos" }
---

## 프로젝트 요약

| 항목 | 내용 |
|---|---|
| 목적 | 은행 마이크로서비스를 AWS 주 사이트와 GCP 대기 사이트에 걸쳐 운영하고, 클라우드 단위 장애에 대비한 재해복구를 구축 · 시연 |
| 대상 | 구글 샘플 은행 앱 Bank of Anthos (Java 3종 · Python 3종), PostgreSQL 2개 DB (accounts · ledger) |
| 구조 | AWS 서울(EKS · RDS) 주 사이트 → AWS DMS CDC → HA VPN → GCP 오사카(GKE · Cloud SQL) 대기 사이트 |
| 담당 파트 | **Database(DR)** · **Observability** · **Governance** |

| 담당 파트 | 설계 | 구축 |
|---|---|---|
| [Database(DR)](#part-dr) | Pilot Light, 비동기 CDC, 복제 도구 선정, RTO · RPO 기준 | Terraform 루트 분리, GCP 대기 사이트 · HA VPN · DMS, 복제 감시 알람 |
| [Observability](#part-obs) | 앱 무수정 계측, 출구 하나 · 마스킹 지점 하나, 두 클라우드 같은 구성 | OTel 수집기 3종, 배포 순서(sync-wave), 수집기 마스킹, 대시보드 |
| [Governance](#part-gov) | 변경 행위자 추적, 배포 경로 권한 경계, Git 밖 시크릿 | CloudTrail · 감사 버킷 · EKS audit 감사 체인, Argo CD 권한 축소, ESO |

**설계 기준** — "클라우드 하나가 통째로 사라져도 건너갔다가 돌아올 수 있는가", "앱 코드를 고치지 않는가", "민감정보가 나가는 출구가 하나인가"

## 전체 구조

두 구성을 모두 구축했습니다. ①로 서비스를 먼저 완성한 뒤, 은행권 결제 서비스처럼 **서버의 인터넷 아웃바운드를 제한하는 환경**에서도 운영할 수 있도록 CI/CD와 관측을 다시 설계한 것이 ②이며, PoC 단계까지 진행했습니다.

| 구분 | ① NAT-GW 기반 구성 | ② 아웃바운드 제한 구성 (PoC) |
|---|---|---|
| 목적 | 멀티클라우드 DR 서비스 구축 | 인터넷 아웃바운드 없이 배포 · 운영 |
| 배포 방식 | **Pull** — Argo CD가 GitHub을 조회 | **Push** — GitHub Actions가 ECR · SQS에 전달 |
| 인터넷 출구 | 프라이빗 서브넷 → AZ별 NAT-GW | 없음 — AWS 서비스는 VPC 엔드포인트로 |
| 관측 | OpenTelemetry → WhaTap (외부 SaaS) | Prometheus · Grafana · Tempo · Loki (VPC 내부) |

> 그림의 파란 상자와 빨간 상자는 두 구성이 서로 다른 부분입니다. 그림을 누르면 원본 크기로 열립니다.

### ① NAT-GW 기반 구성

[![아키텍처 ① — NAT-GW 기반, Argo CD GitOps, OpenTelemetry · WhaTap 관측](/images/final-architecture-1.png)](/images/final-architecture-1.png)

| 단계 | 주체 | 동작 |
|---|---|---|
| 1 | GitHub Actions | 테스트 · 빌드 후 `sha-<커밋>` 불변 태그로 ECR · Artifact Registry에 push |
| 2 | GitHub Actions | gitops 저장소의 오버레이 이미지 태그를 커밋 |
| 3 | Argo CD (EKS · GKE) | 같은 gitops 저장소를 조회해 두 클러스터에 동기화 (`automated` · `prune` · `selfHeal`) |

- **같은 Git, 두 클러스터** — base 매니페스트 하나에 `overlays/{eks,gke}`가 이미지 경로 · DB 목적지 · 시크릿 공급자 차이만 덮습니다. Git이 두 클라우드 밖에 있어 한쪽 장애에도 배포 기준이 남습니다.
- **한계** — 클러스터에서 인터넷으로 나가는 경로가 항상 열려 있어, 아웃바운드를 제한하는 환경에는 맞지 않습니다.

### ② 아웃바운드 제한 구성 (PoC)

[![아키텍처 ② — 아웃바운드 제한 구성, Argo Events · Workflows, Prometheus · Grafana · Tempo · Loki 관측](/images/final-architecture-2.png)](/images/final-architecture-2.png)

**배경** — 금융권처럼 서버가 인터넷으로 나가는 경로를 막는 환경에서는 ①의 두 가지가 동작하지 않습니다. 클러스터의 Argo CD가 GitHub을 조회하는 것과, 관측 데이터를 외부 SaaS로 보내는 것입니다.

| 단계 | 주체 | 동작 |
|---|---|---|
| 1 | GitHub Actions | OIDC로 AWS 권한을 받아 매니페스트 렌더링 · 우리 ECR 외 이미지가 있으면 중단 |
| 2 | GitHub Actions | Helm 차트로 묶어 ECR에 push(OCI) · 배포 메시지(커밋 · 차트 버전 · digest)를 SQS로 전송 |
| 3 | Argo Events (EKS) | VPC 엔드포인트로 SQS 메시지를 받아 Argo Workflows 실행 |
| 4 | Argo Workflows (EKS) | ECR의 차트 digest가 메시지와 같은지 검증 → 차트 pull → `helm upgrade --atomic` 배포 |

- **아웃바운드 0 · Pull → Push** — NAT-GW를 없애고, 외부(GitHub)가 AWS 안의 ECR · SQS에 넣어 두면 클러스터는 VPC 엔드포인트로 꺼내기만 합니다.
- **배포 대상 고정** — 우리 ECR 이미지만 허용하고, 차트 digest가 메시지와 다르면 배포하지 않습니다. 실패하면 자동으로 되돌리고, 동시 배포는 잠금으로 막습니다.
- **관측 내재화** — 외부 SaaS 대신 VPC 안의 Prometheus · Grafana · Tempo · Loki로 지표 · 트레이스 · 로그를 모읍니다.
- **대가** — SQS, Argo Events · Workflows, VPC 엔드포인트, 자체 관측 스택까지 직접 운영할 구성 요소가 늘어납니다.

## 담당 파트 ① Database(DR) {#part-dr}

### 설계

**출발점 — 실패 도메인이 두 개뿐이다.** 연결이 끊기면 대기 사이트는 주 DB가 죽은 것인지 네트워크만 끊긴 것인지 구분할 수 없습니다. 다수결(쿼럼)로 판정하려면 독립된 장애 단위가 세 곳 이상 필요하므로, 두 클라우드로는 **RPO 0과 자동 전환을 함께 가질 수 없습니다.** 둘 다 노리면 양쪽이 동시에 쓰기를 받는 split-brain이 생기고, 은행 잔액처럼 합칠 수 없는 데이터가 갈라집니다.

| 결정 | 비교한 대안 | 근거 | 감수한 것 |
|---|---|---|---|
| **Active-Standby (Pilot Light)** | Backup & Restore, Warm Standby, Active-Active | 5주 · 5인 · 교육용 예산. GCP에는 DB 복제본과 최소 GKE만 상시 유지 | 전환 시간(RTO) |
| **관리형 DB + 비동기 CDC** | 설계안 9종(Active-Standby 5 · Active-Active 4)을 RPO · 전환 방식 · 앱 수정 여부로 비교 | 두 사이트로 성립, RPO 수 초 수준, 앱 수정 없음. RPO 0 · 자동 전환안은 세 번째 사이트가 필요 | RPO 0 불가, 승격은 수동 |
| **동기 복제는 리전 안에서만** | 클라우드 간 동기 복제 | 클라우드 간 동기면 송금마다 VPN 왕복, VPN이 끊기면 멀쩡한 AWS의 쓰기까지 멈춤. AZ 장애는 RDS Multi-AZ(동기)가 처리 | 장애 순간의 복제 지연 = RPO |
| **복제 도구 AWS DMS** | GCP Database Migration Service | GCP DMS는 승격이 비가역이라 돌아오는 경로가 없음. AWS DMS는 Cloud SQL을 소스로도 지원 | 무과금 · 스키마 충실도. 시퀀스 미복제로 전환 시 `setval` 보정 |
| **AWS → GCP 단방향** | 양방향 복제 | 쓰기 지점이 하나라 split-brain이 생기지 않음 | 페일백 때 GCP에만 쓰인 데이터 처리 |
| **사설 경로 HA VPN + BGP** | DB 공인 노출, 전용선 | DB를 인터넷에 열지 않음. 터널 4개 + BGP로 한 터널 장애 시 자동 우회 | 경로 디버깅 지점 증가 |
| **DNS 이중 위임, 전환은 승인 후** | Route 53 단독 + 헬스체크 자동 장애조치 | AWS 장애 때 Route 53까지 영향받는 상황 회피. DB 승격이 수동이라 DNS만 자동으로 넘어가면 GCP 앱이 쓸 DB가 없음 | 두 존을 손으로 맞춰야 함 |

**Pilot Light 구성**

| 구분 | PRIMARY · AWS 서울 | PILOT LIGHT · GCP 오사카 |
|---|---|---|
| 클러스터 | EKS — 서비스 파드 운영, 정상 트래픽 처리 | GKE — 시스템 노드 1대 상시, 앱 노드 풀 평상시 0대 |
| DB | RDS PostgreSQL 16 (Multi-AZ) — 읽기 · 쓰기, DMS 원본 | Cloud SQL PostgreSQL 16 (단일 영역) — DMS 복제 대상, 앱 쓰기 차단 |
| 전환 | ① 감지 · 알림 · 전환 요청(자동) → ② 사람 승인 → ③ DMS 중지 · 시퀀스 보정 → ④ GitOps로 GKE 앱 기동 → ⑤ DNS 전환 | |

**RTO · RPO 기준**

| 지표 | 목표 | 산정 근거 |
|---|---|---|
| RTO | 30분 | 장애 탐지 · DR 선언 최대 10분 + 대기 클러스터 구동 최대 15분 + DNS 전환 최대 5분(TTL 60초) |
| RPO | 5분 | 비동기 복제의 RPO는 설정값이 아니라 **장애 순간의 복제 지연**입니다. 5분은 "DMS CDC면 수 분"이라는 범주에서 고른 값이고, 실측 뒤 알람 기준을 다시 세웠습니다 |

### 구축

| 구축 항목 | 내용 |
|---|---|
| Terraform 루트 분리 | 한 사람의 변경이 전체 state를 잠그던 단일 루트를 컴포넌트 루트로 분리 — AWS 8(core · eks · rds · cache · ops · db-bootstrap · observability · dms), GCP 4(core · cloudsql · gke · ops), 연결 1(net-link) |
| 루트 간 의존 1단 | 각 루트는 remote state로 core만 읽음. RDS 인바운드를 EKS 보안그룹 대신 프라이빗 서브넷 대역으로 허용해 DB 루트가 클러스터 루트를 몰라도 되게 함. state는 S3 key 분리 + `use_lockfile` |
| 두 클라우드 VPN을 한 루트에서 | `net-link`가 AWS · GCP 프로바이더를 함께 써서, GCP VPN 주소 → AWS 고객 게이트웨이, AWS 터널 주소 · 공유키 → GCP 터널로 한 번의 plan/apply에 맞춤 |
| 네트워크 | AWS VGW ↔ GCP HA VPN 터널 4개 · BGP 세션 4개. Cloud SQL이 구글 관리 VPC에 있어 PSA 피어링에 커스텀 경로 내보내기(`exportCustomRoutes`)를 켜야 AWS 대역으로 응답 |
| 주 DB | RDS PostgreSQL 16 Multi-AZ, 논리 복제 활성화(`rds.logical_replication=1`, 재부팅 필요) |
| 복제 | AWS DMS Full Load + CDC, 태스크 2개(accounts: users · contacts / ledger: transactions), DB별 복제 슬롯 2개, 대상 스키마를 미리 만들고 `DO_NOTHING` 적재 |
| 복제 감시 | 복제 지연 알람 120초 × 2회(5분 목표용) · 60초 × 1회(3분 목표용), 복제 슬롯 WAL 적체 1GB 알람, DMS 이벤트 구독(실패 · 상태 변경) |
| 장애 공지 수신 | AWS Health 이벤트를 서울 · 오리건 두 리전의 EventBridge → SNS로 수신 (리전 장애 때 그 리전의 알림 경로도 끊기는 문제 대비) |
| 대시보드 | CloudWatch DR DB · DMS · 감사 3종 — 클러스터가 죽어도 남는 화면 |

[![CloudWatch DR 운영 대시보드 — RDS 용량 · 부하, DMS CDC 지연, 알람](/images/final/obs-cloudwatch-dr.png)](/images/final/obs-cloudwatch-dr.png)

> CloudWatch DR 운영 대시보드. DMS CDC 지연을 RPO의 대리 지표로 봅니다.

[![DR 절차 — CloudWatch 경보 → EventBridge → Lambda → Slack · PR → 승인 후 GitHub Actions로 GCP 기동](/images/final/dr-procedure.png)](/images/final/dr-procedure.png)

> DR 절차. 하위 경보(ALB · RDS · DMS)가 모두 울려야 복합 경보가 울리고, 알림 · PR 생성 → 승인 후 GCP 기동으로 이어집니다. 감지 · 승인 자동화는 팀원이 구현했습니다.

### 문제 해결 — 복제는 멀쩡한데 소스 디스크가 차오른다

- **문제** — 복제 지연은 0초인데, 소스 DB의 복제 슬롯이 붙잡은 변경 로그(WAL)가 **최대 9GB**까지 불어났습니다.
- **조사** — 지표를 10분 단위로 쪼개 보니 5분마다 64MB씩 일정하게 늘고 있었습니다. RDS가 유휴 상태에서도 5분마다(`archive_timeout=300`) WAL 세그먼트를 넘기는데, DMS 하트비트가 기본값으로 꺼져 있어 트래픽이 없으면 슬롯이 앞으로 나아가지 못한 것입니다.
- **변경** — DMS 소스 엔드포인트에 하트비트를 켜고 주기를 5분으로 맞췄습니다.
- **결과** — 적용 약 5분 뒤 **5.25GB → 0.06GB**로 떨어졌고, 이후 WAL 세그먼트 1~2개 수준으로 유지됐습니다. 재발 감지를 위해 슬롯 적체 1GB 알람을 추가했습니다.

### 문제 해결 — RPO 알람이 목표보다 늦게 울린다

- **문제** — 기존 복제 지연 알람(180초 × 3회)은 복제가 완전히 멈추면 약 6분 뒤에 울려, 5분 목표를 넘었습니다.
- **변경** — DMS 지표가 1분 단위로 나오는 것을 확인하고, 5분 목표용 120초 × 2회와 3분 목표용 60초 × 1회를 함께 두었습니다. 목표 확정은 부하 상태 실측 뒤로 미뤘습니다.

### 장애 주입 시연 — 기록으로 다시 읽기

시연 뒤 CloudTrail · CloudWatch · GCP Cloud Monitoring 기록을 시각순으로 맞춰 사건을 재구성했습니다.

| 시각 | 사건 |
|---|---|
| 21:05 | RDS 중지 요청(`StopDBInstance`) |
| 21:10 | 기본 NACL 규칙 삭제로 네트워크 차단 주입 |
| 21:14 | ALB 정상 대상 없음 알람 |
| 21:20 | 복제 지연 알람 — 이론상 약 2분인데 실측 약 15분 (원인 미확정) |
| 22:30 ~ 35 | Cloud SQL 첫 쓰기 관측 → **RTO 상한 약 85~90분** |

- DNS 전환 · GKE 앱 준비 시각을 남기지 않아 정확한 RTO는 낼 수 없었고, RPO는 측정점이 없어 미측정입니다.
- 페일백은 GCP에만 쓰인 데이터를 폐기하기로 정한 뒤 RDS를 다시 띄웠지만, DMS와 파드가 DB에 닿지 않았습니다. 원인을 **시연 때 지운 기본 NACL 규칙**(서브넷 8개 전체에 걸린 단일 규칙)으로 특정했고, 철거 일정 때문에 완주하지 못했습니다.

### 남은 한계

| 구분 | 한계 | 개선안 |
|---|---|---|
| 단일 장애점 | DMS 복제 인스턴스 단일 AZ, Cloud SQL 단일 영역, 서브넷 전체에 걸린 기본 NACL 하나 | DMS · 대기 DB 고가용성, 서브넷 계층별 NACL |
| DB 접근 범위 | RDS 인바운드를 서브넷 대역으로 허용해 경계가 넓어짐 | NACL 또는 보안그룹 참조로 좁히기 (보류된 채 종료) |
| state 위치 | GCP 인프라의 Terraform state가 AWS S3에 있어, AWS 장애 때 GCP를 Terraform으로 다루기 어려움 | DR 사이트 state를 GCS 등 다른 곳에 분리 |
| 코드 불일치 | DMS를 콘솔로 재구축해 코드와 실물이 다름 | DMS를 코드로 일원화 |

## 담당 파트 ② Observability {#part-obs}

### 설계

**전제** — 앱 코드에 관측 설정이 없는 상태에서, 앱을 고치지 않고 두 클라우드에서 같은 방식으로 봅니다. 서비스 6개를 거치는 요청은 트레이스가 없으면 어디서 느려졌는지 알 수 없습니다.

| 안 | 내용 | 판단 |
|---|---|---|
| **OpenTelemetry 전면** | 자동계측 → 수집기 → WhaTap(OTLP) · S3 | **채택** — 마스킹 지점이 하나, EKS · GKE 대칭, 목적지는 exporter만 바꾸면 됨 |
| 혼합 | 워크로드 계층을 WhaTap 자체 APM 에이전트로 | 기각 — WhaTap으로 가는 경로가 수집기를 거치지 않아 마스킹 지점이 둘로 갈림. 자바 에이전트 동시 부착 시 `JAVA_TOOL_OPTIONS` 충돌 |
| Prometheus + Grafana (초기안) | 지표 중심 pull 방식 | 트레이스 · 로그는 다른 도구가 필요해, 세 신호를 한 파이프라인으로 받는 수집기를 택함 |
| 제외 | X-Ray, CloudWatch 관측 애드온 | X-Ray는 GCP에서 못 씀, 애드온은 자동 주입과 충돌 |

**계층별 수집 대상**

| 계층 | 수집 대상 | 대시보드 |
|---|---|---|
| 애플리케이션 | 트레이스 · 앱 로그 · 런타임 지표 | WhaTap |
| 노드 · 컨테이너 | 노드 자원 지표 · 컨테이너 로그 | WhaTap |
| 클러스터 | 클러스터 상태 지표 · 경고 이벤트 | WhaTap |
| AWS 관리형 | ALB · RDS · Valkey · DMS · VPN | CloudWatch |
| GCP 관리형 | Cloud NAT · Cloud Load Balancing · Cloud SQL · VPN | Cloud Monitoring |

- **출구 하나, 마스킹 지점 하나** — 클러스터 밖으로 나가는 관측 데이터는 전부 gateway 수집기를 지나므로, 금융 데이터 마스킹을 그 한 곳에서 합니다. 외부 SaaS로 한 번 보낸 데이터는 회수할 수 없기 때문입니다.
- **WhaTap 쿠버네티스 에이전트 병행** — WhaTap 쿠버네티스 화면을 OTLP만으로 채우는 실험을 했지만, 제품이 자기 에이전트 연결을 추적해야 화면을 그려서 실패했습니다. 그래서 쿠버네티스 에이전트를 함께 두었고, 클러스터 지표 중복 수집은 제품 구조상 감수했습니다.

[![관측 스택 구성도 — 앱 파드 자동계측 → otel-agent · otel-cluster → otel-gateway(마스킹 → 배치) → WhaTap · S3 · CloudWatch](/images/final/obs-stack.png)](/images/final/obs-stack.png)

### 구축

| 구축 항목 | 내용 |
|---|---|
| 자동계측 | OpenTelemetry 오퍼레이터의 Instrumentation 리소스 + 파드 어노테이션(`inject-java` · `inject-python`) 한 줄. **앱 코드 수정 · 이미지 재빌드 없이** 6종 계측 |
| 수집기 3종 | `otel-agent` DaemonSet(노드 지표 · 컨테이너 로그), `otel-cluster` Deployment 1개(클러스터 상태 · 이벤트, 중복 방지), `otel-gateway` Deployment 2개(유일한 출구) |
| gateway 처리 순서 | `memory_limiter`(과부하 시 먼저 거절) → 쿠버네티스 속성 부착 → **마스킹(redaction · transform)** → 배치 → WhaTap · S3. 재시도 · 전송 큐 설정 |
| 목적지 | WhaTap은 gateway 옆 `whatap-otel` 사이드카(베타 — 지표 · 로그 수신을 명시적으로 켜야 함), 트레이스 · 로그는 마스킹 후 자체 계정 S3에도 보관 |
| 배포 순서 | Argo CD sync-wave로 플랫폼을 앱보다 먼저 — AppProject(-30) → external-secrets(-28) → cert-manager(-25) → 시크릿 · 네임스페이스(-22) → OTel 오퍼레이터(-20) → WhaTap 오퍼레이터(-15) → 앱(-10 / 0). 주입 웹훅보다 앱이 먼저 뜨면 계측이 붙지 않음 |
| 시크릿 전달 | WhaTap 라이선스를 External Secrets Operator로 Secrets Manager에서 주입 (Governance 참고) |
| 대시보드 | WhaTap 애플리케이션 · 쿠버네티스(EKS · GKE 나란히), CloudWatch DR, GCP Cloud Monitoring 인프라 |

[![WhaTap 애플리케이션 대시보드](/images/final/obs-whatap-apm.png)](/images/final/obs-whatap-apm.png)

> WhaTap 애플리케이션 대시보드. 수집기를 거친 트레이스로 TPS · 응답시간 · Apdex를 봅니다.

[![EKS Primary와 GKE DR의 노드 · 파드 상태](/images/final/obs-whatap-k8s.png)](/images/final/obs-whatap-k8s.png)

> 쿠버네티스 대시보드. EKS 주 사이트와 GKE 대기 사이트를 나란히 놓아, 평상시 대기 사이트가 최소 구성으로 유지되는지 봅니다.

[![GCP Cloud Monitoring — Cloud SQL 디스크 · 메모리 · CPU, VPN 터널 4개 송수신](/images/final/obs-gcp-monitoring.png)](/images/final/obs-gcp-monitoring.png)

> GCP 대기 사이트 대시보드. Cloud SQL의 쓰기 연산과 VPN 터널 4개의 트래픽으로 복제가 흐르는지 봅니다.

### 문제 해결 — 관측 스택 첫 배포가 통째로 Missing

첫 동기화에서 관측 Application이 OutOfSync/Missing으로 멈췄고, 차트 원본의 스키마와 CRD를 대조해 원인 세 개를 찾았습니다.

| 구분 | 원인 | 조치 |
|---|---|---|
| 차트 스키마 | OTel 오퍼레이터 차트 values가 스키마 검증(`additionalProperties: false`) 대상이라, 없는 키 하나로 렌더링 자체가 실패 | 지원되는 키로 교체 |
| 차트 버전 | WhaTap Helm 저장소 주소 404, 구버전 차트는 CRD에 없는 필드를 조용히 버림 | 저장소 주소 수정, 차트 1.9.11로 고정 |
| 권한 경계 | WhaTap 에이전트 CRD가 클러스터 범위라 AppProject의 네임스페이스 허용 목록에 걸림. Argo CD는 묶음 중 하나가 막히면 전부 적용하지 않아 15개 리소스가 모두 Missing | AppProject 클러스터 리소스 허용 목록에 명시 |

> 배운 기준 — 차트 values는 스키마로 검증되거나 조용히 무시됩니다. 문서보다 차트 원본의 스키마와 CRD를 봅니다.

### 문제 해결 — 마스킹이 있다고 생각했던 곳에 없었다

- **문제** — 발표 자료에는 "앱 필드 기반 마스킹"이 적혀 있었지만, 앱 코드에 마스킹 로직이 없었고 오히려 회원가입 요청을 통째로 debug 로그에 남기고 있었습니다.
- **원인** — 수집기 규칙에도 구멍이 있었습니다. 라우팅번호 규칙 누락, JWT 치환이 로그에만 있고 트레이스에는 없음, 값 치환이 URL 경로 하나만 봄, 키 필터가 로그 본문 문자열에는 닿지 않음. 값을 통째로 지우는 `redaction`이 부분 마스킹(`transform`)보다 먼저 실행되는 **순서 문제**도 있었습니다.
- **변경** — 속성 맵 전체와 본문 문자열을 훑도록 규칙을 고치고, 처리 순서를 바로잡았습니다. 계좌 10자리 · 라우팅 9자리는 끝 6자리만 가려 조사용 단서를 남기고, JWT는 치환, 가입 정보(이름 · 생일 · 주소 · SSN)는 제거합니다.
- **검증** — 실제 로그 형태 4종을 재현해 규칙을 적용했고, **남은 민감값 0개**를 확인했습니다(재현한 4종 기준).
- **남은 한계** — 문자에 바로 붙은 숫자(`acct1011226111`)는 놓칩니다. 근본 해결은 앱이 요청을 debug 로그에 찍지 않게 고치는 것이라 앱 담당에 인계했습니다.

[![민감 데이터 처리 — 로그 원문 → redaction · transform → WhaTap에 남는 값](/images/final/masking-flow.png)](/images/final/masking-flow.png)

> 수집기 마스킹 흐름 (예시 값). 원문에 있던 비밀번호 · 주민번호 · 계좌번호가 대시보드에는 가려진 상태로 남습니다.

## 담당 파트 ③ Governance {#part-gov}

### 설계

**출발점** — 팀원 5명이 코드 서버 한 대를 리눅스 계정까지 공유해 썼고, 그 서버의 IAM 역할에 클러스터 관리자 권한(`AmazonEKSClusterAdminPolicy`)이 붙어 있었습니다. 사고가 나면 **누가, 어디서, 무엇을 바꿨는지** 답할 수 없는 상태였습니다.

| 설계 원칙 | 내용 |
|---|---|
| 두 기록을 엮는다 | CloudTrail은 AWS API 호출을, EKS audit은 쿠버네티스 API 호출을 기록합니다. kubectl로 HPA를 고쳐도 CloudTrail에는 신원 확인 한 줄만 남으므로 둘 다 필요합니다 |
| IP가 아니라 IAM ARN으로 | kubectl 토큰은 EKS 안의 인증 모듈이 STS에 대신 제출하므로, CloudTrail에 찍힌 IP는 작업자가 아니라 **EKS 제어부 IP**입니다. 그래서 audit의 요청자 ARN(세션 이름 = 인스턴스 ID)으로 CloudTrail과 연결합니다 |
| 기록은 지울 수 없게 | 감사 기록은 객체 잠금 버킷에 두고, 감사 버킷 자체의 접근도 기록합니다 |
| 배포 경로는 좁게 | 앱 배포 경로(Argo CD)로는 클러스터 권한을 얻을 수 없게 합니다 |
| 시크릿 값은 Git 밖에 | Git에는 "무엇을 어디로"만 두고, 값은 Secrets Manager에서 읽습니다 |

[![감사 체인 — AWS 관리 영역 → CloudTrail → S3, EKS 제어부 audit · authenticator → CloudWatch Logs → 변경 내역 분석](/images/final/audit-flow.png)](/images/final/audit-flow.png)

> 발표 자료의 감사 체인 그림. 그림에 적힌 보존 기간(730일 · 2년)과 달리, 실제로 적용한 값은 아래 구축 표와 같습니다.

### 구축

| 구축 항목 | 내용 |
|---|---|
| CloudTrail 트레일 | 멀티 리전 · 글로벌 서비스 이벤트 포함, KMS 암호화, 로그 파일 무결성 검증 |
| 이벤트 선택기 3개 | ① 관리 이벤트 전체 ② 감사 버킷 자체의 객체 접근 전부 ③ 텔레메트리 보관 버킷의 삭제만 — 데이터 이벤트는 건수 과금이라 범위를 좁힘 |
| 감사 버킷 (S3) | 객체 잠금(Governance) 1일, 90일 뒤 Glacier, 365일 뒤 삭제 |
| EKS 제어부 로그 | audit · authenticator 등 5종을 CloudWatch Logs로, 보존 30일. secret · configmap은 본문 없이(Metadata), HPA · pod · role은 변경 본문까지(RequestResponse) 기록 |
| 조회 | Logs Insights 저장 질의 7개 — 클러스터 변경 행위자, RBAC 거부, IAM ↔ 쿠버네티스 사용자 매핑, 송금 앱 변경 이력 등 |
| Argo CD 권한 축소 | 관측 Application을 별도 AppProject로 분리하고, 앱 프로젝트를 가져올 저장소 5 → 1, 배포 대상 7 → 1, 클러스터 리소스 8종 → 1종(Namespace)으로 축소 |
| 시크릿 (ESO) | Secrets Store CSI는 파드가 볼륨을 마운트해야 Secret이 생겨, 환경변수로만 읽는 수집기와 다른 네임스페이스의 에이전트에 값을 줄 수 없었음 → External Secrets Operator로 교체. 컨트롤러는 액세스 키 없이 Pod Identity로 읽고, IAM은 관측 경로만, 사용 가능한 네임스페이스는 2곳으로 제한 |

[![EKS 감사 대시보드 — 클러스터 변경 행위자, RBAC 거부, IAM 인증 주체별 활동량](/images/final/audit-eks-dashboard.png)](/images/final/audit-eks-dashboard.png)

> EKS 감사 대시보드. 행위자 ARN과 인증 메시지는 계정 식별값이 있어 가렸습니다.

### 결과 — GitOps를 우회한 변경을 감사로 잡아내다

[![송금 앱 변경 이력 — Argo CD 컨트롤러의 HPA 변경 사이에 관리 서버 역할의 직접 변경이 섞여 있음](/images/final/audit-eks-changes.png)](/images/final/audit-eks-changes.png)

송금 앱 변경 이력에서 대부분의 HPA 변경은 Argo CD 컨트롤러가 했지만, 공용 코드 서버 역할이 **HPA를 직접 패치한 기록 3건**이 섞여 있었습니다(행위자는 가림). 이 변경으로 Argo CD 추적 어노테이션까지 지워졌습니다.
이를 계기로 위의 Argo CD 권한 축소를 적용했습니다. 시연 뒤에는 같은 방식으로 CloudTrail 기록(`StopDBInstance`, `DeleteNetworkAclEntry`)을 읽어 장애 타임라인과 페일백 실패 원인을 재구성했습니다.

### 한계

| 구분 | 한계 | 이유 · 개선안 |
|---|---|---|
| 추적 단위 | 사람이 아니라 서버 단위까지만 특정 | 리눅스 계정 공유. 사람별 SSO 세션과 Access Entry로 바꾸면 audit에 개인이 남음 |
| 서버 권한 | 코드 서버 역할의 ClusterAdmin은 그대로 | Argo CD 경로는 줄였지만 서버 역할 자체는 줄이지 못함. 클러스터 변경은 GitOps만, 사람 권한은 읽기 위주로 |
| 기록 시점 | 트레일 로깅은 시연 다음 날부터 | 시연 구간은 CloudTrail 기본 이벤트 기록(90일)으로 재구성 |

## 결과와 검증

| 구분 | 내용 |
|---|---|
| 구현 | AWS · GCP 양 사이트, HA VPN, DMS CDC, GitOps 배포, OpenTelemetry 관측, 수집기 마스킹, 복제 · 장애 알림, 감사 체인 |
| 시연에서 확인 | 실제 데이터의 GCP 복제, 장애 주입 후 GCP에서 쓰기 처리 |
| 측정 | 슬롯 적체 최대 9GB → 0.06GB, 마스킹 재현 로그 잔여 민감값 0개, RTO 상한 약 85~90분 |
| 미검증 | RTO · RPO 목표 달성, 부하 상태의 평상시 복제 지연, 페일백 완료 |

> 교육 과정의 시연 환경이며 실제 사용자 트래픽은 없었습니다.

## 회고

- **RPO는 설정이 아니라 측정이다.** 처음에는 "CDC라 사실상 유실이 없다"고 생각했지만, 비동기 복제의 RPO는 장애 순간의 지연입니다. 목표를 정하기 전에 감지 가능한 최소 시간과 평상시 지연부터 재야 한다고 판단을 바꿨습니다.
- **"켜져 있다"와 "동작한다"는 다르다.** 마스킹이 된다고 생각한 곳, 알람이 제때 울린다고 생각한 곳을 실제 데이터로 다시 확인해 구멍을 찾았습니다.
- **유지할 선택** — 되돌아올 수 있는 복제 도구, 마스킹 지점의 단일화, 컴포넌트 루트, Argo CD 권한 경계
- **바꿀 선택** — DMS를 코드로 일원화, DMS와 대기 DB에 고가용성 확보, 시연 전에 측정점(DNS 전환 · 앱 준비 시각) 정하기

> 팀 프로젝트이며, CI · GitOps 스크립트 · HPA · DR 감지와 승인 자동화는 팀원이 구현했습니다.
> 코드 작성 일부에 AI 도구(Claude Code)를 활용했고, 문제 정의 · 설계 판단 · 검증은 직접 했습니다.

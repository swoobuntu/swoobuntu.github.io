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
| 목적 | 은행 마이크로서비스를 AWS 주 사이트와 GCP 대기 사이트에 걸쳐 운영하고, 클라우드 단위 장애에 대비한 재해복구를 구축·시연 |
| 대상 | 구글 샘플 은행 앱 Bank of Anthos (Python 3종, Java 3종), PostgreSQL 2개 DB (accounts · ledger) |
| 담당 파트 | **Database(DR)** · **Observability** · **Governance** |

| 담당 파트 | 설계 | 구축 |
|---|---|---|
| [Database(DR)](#part-dr) | Pilot Light DR, 복제 도구 선정, RTO · RPO 기준 | GCP 대기 사이트 · HA VPN · DMS Terraform 루트, 복제 감시 대시보드 |
| [Observability](#part-obs) | 계층별 수집 대상, 앱 무수정 계측, 마스킹 단일 지점 | OTel 수집기 3종, 수집기 마스킹, 양 클라우드 대시보드 |
| [Governance](#part-gov) | 고객정보 보호 · 운영자 행위 · 서비스 접근 추적 | CloudTrail · 감사 버킷 · EKS audit 감사 체인 |

**판단 기준** — "건너갔다가 돌아올 수 있는가", "앱 코드를 고치지 않는가", "민감정보가 나가는 출구가 하나인가"

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
| 1 | GitHub Actions | 앱 이미지를 빌드해 ECR에 push |
| 2 | Argo CD (EKS) | NAT-GW를 거쳐 GitHub의 GitOps 매니페스트를 조회하고 변경분을 동기화 |
| 3 | EKS 노드 | ECR에서 이미지를 받아 서비스 기동 |

- **표준 GitOps** — 클러스터가 Git 저장소를 바라보는 Argo CD 기본 방식이라 구성이 단순합니다.
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

**Pilot Light DR** — 데이터 계층과 최소 시스템 노드만 GCP에 상시 유지하고, 장애가 나면 애플리케이션 용량을 늘려 넘어갑니다.

| 구분 | PRIMARY · AWS 서울 | PILOT LIGHT · GCP 오사카 |
|---|---|---|
| 클러스터 | EKS — 서비스 파드 운영, 정상 트래픽 처리 | GKE — 시스템 노드 1대 상시, 앱 노드 풀 평상시 0대 |
| DB | RDS PostgreSQL (Multi-AZ) — 읽기 · 쓰기, DMS 원본 | Cloud SQL — DMS 복제 대상, 앱 쓰기 차단 |
| 전환 순서 | ① DR 선언 → ② GKE 앱 노드 풀 확장 → ③ DNS 레코드 전환 (Route 53 + Cloud DNS) | |

**DB 동기화** — RDS의 데이터를 **AWS DMS Full Load + CDC**로 Cloud SQL에 계속 반영합니다. accounts · ledger DB마다 태스크를 두고, VPN 너머 Cloud SQL의 사설 IP로 복제합니다.
전환할 때는 **단일 쓰기 원칙**을 지킵니다. AWS 쓰기를 막고, DMS를 멈춰 정합성을 확인한 뒤에 GCP 앱의 쓰기를 엽니다.

**복제 도구 선정 — 되돌아올 수 있는가**

- GCP DMS는 서비스 요금이 없고 스키마도 더 온전히 옮기지만, 승격이 비가역이라 GCP에서 AWS로 돌아오는 경로가 도구 안에 없습니다.
- DR은 왕복이 전제이고 시연도 반복해야 해, Cloud SQL을 소스로도 지원하는 **AWS DMS**를 택했습니다.
- 대신 대기 DB가 늘 쓰기 가능하고, DMS가 시퀀스를 복제하지 않아 전환 때 시퀀스 보정이 필요합니다.

**왜 비동기 복제인가** — 두 곳이 동시에 쓰기를 받으려면 합의(쿼럼)가 필요하고, 쿼럼은 독립된 장애 단위가 최소 3개 있어야 성립합니다. 실패 도메인이 AWS · GCP 두 개뿐이라 비동기 복제를 받아들이고 그 지연을 감시하는 쪽을 택했습니다.

**RTO · RPO 기준**

| 지표 | 목표 | 산정 근거 |
|---|---|---|
| RTO | 30분 | 장애 탐지 · DR 선언 최대 10분 + 대기 클러스터 구동 최대 15분 + DNS 전환 최대 5분(TTL 60초) |
| RPO | 60초 | CDC 지연(`CDCLatencyTarget`) 상한. 계획 전환은 쓰기 동결 후 잔여분을 반영해 0 |

### 구축

| 구축 항목 | 내용 |
|---|---|
| Terraform 컴포넌트 루트 | 단일 루트를 컴포넌트별(AWS 8 · GCP 4 · VPN 1)로 나누고 서로 직접 참조하지 않게 리팩터. 예: RDS 인바운드를 EKS 보안그룹이 아닌 프라이빗 서브넷 대역으로 허용 |
| GCP 대기 사이트 | VPC · GKE(시스템 풀 + 0대 앱 풀) · Cloud SQL 루트 작성 |
| AWS ↔ GCP 네트워크 | HA VPN 터널 4개 + BGP. Cloud SQL은 구글 관리 VPC에 피어링으로 붙어 있어, 피어링에 경로를 따로 내보내 VPN 너머에서 닿게 함 |
| 복제 | RDS 파라미터 그룹에 논리 복제 활성화, DMS 복제 인스턴스 · 엔드포인트 · 태스크(accounts · ledger) 구성 |
| 복제 감시 | RDS · DMS CDC 지연 · 디스크 · ALB 5xx · 알람을 한 화면에 모은 DR 운영 대시보드 |

[![CloudWatch DR 운영 대시보드 — RDS 용량 · 부하, DMS CDC 지연, 알람](/images/final/obs-cloudwatch-dr.png)](/images/final/obs-cloudwatch-dr.png)

> CloudWatch DR 운영 대시보드. DMS CDC 지연을 RPO의 대리 지표로 봅니다.

[![DR 절차 — CloudWatch 경보 → EventBridge → Lambda → Slack · PR → 승인 후 GitHub Actions로 GCP 기동](/images/final/dr-procedure.png)](/images/final/dr-procedure.png)

> DR 절차. 경보 → 알림 · PR 생성 → 승인 후 GCP 기동으로 이어지며, 감지 자동화는 팀원이 구현했습니다.

### 문제 해결 — 복제는 멀쩡한데 소스 디스크가 차오른다

- **문제** — 복제 지연은 0초인데, 소스 DB의 복제 슬롯이 붙잡은 변경 로그(WAL)가 **최대 9GB**까지 불어났습니다.
- **조사** — 지표를 10분 단위로 쪼개 보니 5분마다 64MB씩 늘고 있었습니다. RDS가 유휴 상태에서도 5분마다 로그 세그먼트를 전환하는데, 트래픽이 없어 슬롯이 앞으로 나아가지 못한 것입니다.
- **변경 · 결과** — DMS 소스 엔드포인트에 하트비트를 켜고 주기를 5분으로 맞췄고, 약 5분 뒤 **0.1GB 미만**으로 떨어져 그 수준에서 유지됐습니다.

### 문제 해결 — 장애 시연을 기록으로 다시 읽다

- **문제** — 장애 주입 시연(DB 중지, 네트워크 차단) 후 GCP에서 쓰기를 받는 데까지는 갔지만, 목표를 지켰는지 말할 근거가 없었습니다.
- **조사** — CloudTrail · CloudWatch · GCP Cloud Monitoring 기록을 시각순으로 맞춰 사건을 재구성했습니다.
- **결과** — **RTO 목표 미달 가능성과 RPO 측정점 부재**를 도출했습니다. 페일백 중 모든 계층이 동시에 끊긴 원인은 시연 때 지운 기본 NACL 규칙으로 특정했습니다.

## 담당 파트 ② Observability {#part-obs}

### 설계

**계층별 수집 대상**

| 계층 | 수집 대상 | 대시보드 |
|---|---|---|
| 애플리케이션 | 트레이스 · 앱 로그 · 런타임 지표 | WhaTap |
| 노드 · 컨테이너 | 노드 자원 지표 · 컨테이너 로그 | WhaTap |
| 클러스터 | 클러스터 상태 지표 · 이벤트 | WhaTap |
| AWS 관리형 | ALB · RDS · Valkey · DMS · VPN | CloudWatch |
| GCP 관리형 | Cloud NAT · Cloud Load Balancing · Cloud SQL · VPN | Cloud Monitoring |

- **앱을 고치지 않는 계측** — 앱 · 클러스터 계층을 모두 OpenTelemetry로 봅니다. 목적지를 바꿀 때 수집기 설정만 고치면 됩니다.
- **마스킹 지점은 하나** — 클러스터를 나가는 데이터가 전부 게이트웨이 수집기를 지나므로, 금융 데이터 마스킹을 그 한 곳에서 합니다. 외부 SaaS로 한 번 보낸 데이터는 회수할 수 없기 때문입니다.
- **같은 구성, 두 클라우드** — EKS와 GKE에 같은 수집기 구성과 같은 마스킹 규칙을 배포합니다.

[![관측 스택 구성도 — 앱 파드 자동계측 → otel-agent · otel-cluster → otel-gateway(마스킹 → 배치) → WhaTap · S3 · CloudWatch](/images/final/obs-stack.png)](/images/final/obs-stack.png)

### 구축

| 구축 항목 | 내용 |
|---|---|
| 자동계측 | OpenTelemetry Operator의 Instrumentation CR + 파드 어노테이션 한 줄로 Java 3종 · Python 3종에 에이전트 주입 |
| 수집기 3종 | `otel-agent`(노드 · 컨테이너 지표 · 로그), `otel-cluster`(클러스터 상태 · 이벤트), `otel-gateway` ×2(마스킹 2겹 → 배치 → 내보내기) |
| 목적지 분기 | 트레이스 · 지표 · 로그 → WhaTap 사이드카, 트레이스 · 로그 → S3 텔레메트리 아카이브, 송금 감사 로그 → S3 감사 경로 |
| 마스킹 | ① 키 이름으로 값 통째 제거(password · ssn · 이름) ② 키가 없는 본문 문자열 안쪽 마스킹(계좌 · 라우팅번호는 끝 6자리만 가림) |
| 대시보드 | WhaTap 애플리케이션 · 쿠버네티스(EKS · GKE 나란히), CloudWatch DR 운영, GCP Cloud Monitoring 인프라 |

[![민감 데이터 처리 — 로그 원문 → redaction · transform → WhaTap에 남는 값](/images/final/masking-flow.png)](/images/final/masking-flow.png)

> 수집기 마스킹 흐름 (예시 값). 원문에 있던 비밀번호 · 주민번호 · 계좌번호가 대시보드에는 가려진 상태로 남습니다.

[![WhaTap 애플리케이션 대시보드](/images/final/obs-whatap-apm.png)](/images/final/obs-whatap-apm.png)

> WhaTap 애플리케이션 대시보드. 수집기를 거친 트레이스로 TPS · 응답시간 · Apdex를 봅니다.

[![EKS Primary와 GKE DR의 노드 · 파드 상태](/images/final/obs-whatap-k8s.png)](/images/final/obs-whatap-k8s.png)

> 쿠버네티스 대시보드. EKS 주 사이트와 GKE 대기 사이트를 나란히 놓아, 평상시 대기 사이트가 최소 구성으로 유지되는지 봅니다.

[![GCP Cloud Monitoring — Cloud SQL 디스크 · 메모리 · CPU, VPN 터널 4개 송수신](/images/final/obs-gcp-monitoring.png)](/images/final/obs-gcp-monitoring.png)

> GCP 대기 사이트 대시보드. Cloud SQL의 쓰기 연산과 VPN 터널 4개의 트래픽으로 복제가 흐르는지 봅니다.

### 문제 해결 — 마스킹이 있다고 생각했던 곳에 없었다

- **문제** — 발표 자료에는 "앱 필드 기반 마스킹"이 적혀 있었지만 실제로는 동작하지 않았습니다. 앱은 오히려 회원가입 요청을 통째로 디버그 로그에 남기고 있었습니다.
- **원인** — 수집기 규칙에도 구멍이 세 개 있었습니다. 라우팅번호 규칙 누락, JWT 치환이 로그에만 있고 트레이스에는 없음, 값 치환이 URL 경로 하나만 봄. 값을 통째로 지우는 단계가 부분 마스킹보다 먼저 실행되는 **순서 문제**도 있었습니다.
- **변경** — 속성 맵 전체를 훑도록 바꾸고 본문용 규칙을 추가했습니다.
- **검증** — 실제 로그 형태 4종을 재현해 규칙을 적용했고, **남은 민감값 0개**를 확인했습니다.

## 담당 파트 ③ Governance {#part-gov}

### 설계

**목표** — 계좌 · 인증정보 노출을 줄이고, 송금 시스템의 접근과 변경 이력을 남깁니다. **고객정보 보호 + 운영자 행위 추적 + 서비스 접근 추적**의 세 가지를 하나의 감사 체인으로 엮었습니다.

- **두 기록을 엮어야 하는 이유** — 팀이 관리 서버 한 대를 공용 계정으로 함께 썼고, 그 서버 역할이 클러스터 관리자 권한을 갖고 있었습니다. 실제 로그를 열어 보니 kubectl 인증 과정에서 CloudTrail에 남는 IP는 사용자가 아니라 EKS 제어부의 IP였고, 진짜 클라이언트 IP는 EKS audit 로그에만 있었습니다.
- **연결 키** — CloudTrail의 AWS 신원 정보와 EKS 리소스 작업을 **시간 · IAM ARN · 사용자명 · Source IP**로 연결해 인스턴스까지 특정합니다.
- **변조 방지** — 감사 로그는 지울 수 없게 보관합니다.

[![감사 체인 — AWS 관리 영역 → CloudTrail → S3(KMS · 수명주기 · 객체 잠금), EKS 제어부 audit · authenticator → CloudWatch Logs → 변경 내역 분석](/images/final/audit-flow.png)](/images/final/audit-flow.png)

### 구축

| 구축 항목 | 내용 |
|---|---|
| CloudTrail | 관리 이벤트 전체(Read + Write) · 글로벌 이벤트, 감사 · 보관 버킷 데이터 이벤트 |
| 감사 버킷 (S3) | KMS 암호화, 객체 잠금(Governance), 90일 뒤 Glacier로 옮기는 수명주기 |
| EKS 제어부 로그 | audit · authenticator 로그를 CloudWatch Logs(`/aws/eks/boa-eks/cluster`)로 수집 |
| 감사 대시보드 | Logs Insights 쿼리로 클러스터 변경 행위자 · RBAC 거부 · IAM ↔ 쿠버네티스 사용자 매핑 · 송금 앱 변경 이력을 위젯화 |

[![EKS 감사 대시보드 — 클러스터 변경 행위자, RBAC 거부, IAM 인증 주체별 활동량](/images/final/audit-eks-dashboard.png)](/images/final/audit-eks-dashboard.png)

> EKS 감사 대시보드. 행위자 ARN과 인증 메시지는 계정 식별값이 있어 가렸습니다.

### 결과 — GitOps를 우회한 변경을 감사로 잡아내다

[![송금 앱 변경 이력 — Argo CD 컨트롤러의 HPA 변경 사이에 관리 서버 역할의 직접 변경이 섞여 있음](/images/final/audit-eks-changes.png)](/images/final/audit-eks-changes.png)

송금 앱 변경 이력에서 대부분의 HPA 변경은 Argo CD 컨트롤러가 했지만, 8번째 줄은 **관리 서버 역할이 직접** HPA를 고친 기록이었습니다(행위자는 가림).
이 변경으로 Argo CD 추적 표시가 지워진 것을 찾아냈습니다. **GitOps를 우회한 변경을 감사 체인으로 잡아낸 사례**입니다.

## 결과와 검증

| 구분 | 내용 |
|---|---|
| 구현 | AWS · GCP 양 사이트, VPN, DMS CDC, GitOps 배포, OpenTelemetry 관측, 수집기 마스킹, 복제 · 장애 알림, 감사 체인 |
| 시연에서 확인 | 실제 데이터의 GCP 복제, 장애 주입 후 GCP에서 쓰기 처리 |
| 측정 | 슬롯 적체 최대 9GB → 0.1GB 미만, 마스킹 재현 로그 잔여 민감값 0개 |
| 미검증 | RTO · RPO 목표 달성, 부하 상태의 평상시 복제 지연, 페일백 완료 |

> 교육 과정의 시연 환경이며 실제 사용자 트래픽은 없었습니다.

## 회고

- **RPO는 설정이 아니라 측정이다.** 처음에는 "CDC라 사실상 유실이 없다"고 생각했지만, 비동기 복제의 RPO는 장애 순간의 지연입니다. 목표를 정하기 전에 감지 가능한 최소 시간과 평상시 지연부터 재야 한다고 판단을 바꿨습니다.
- **유지할 선택** — 되돌아올 수 있는 복제 도구, 마스킹 지점의 단일화, 컴포넌트 루트, Argo CD 권한 경계
- **바꿀 선택** — DMS를 코드와 콘솔 중 한쪽으로 통일, DMS와 대기 DB에 고가용성 확보, 시연 전에 측정점 정하기

> 팀 프로젝트이며, GitOps 스크립트 · CI · 오토스케일링 · DR 감지 자동화는 팀원이 구현했습니다.
> 코드 작성 일부에 AI 도구(Claude Code)를 활용했고, 문제 정의 · 설계 판단 · 검증은 직접 했습니다.

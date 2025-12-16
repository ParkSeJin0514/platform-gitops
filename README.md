# 🚀 Platform GitOps

ArgoCD 기반 GitOps 저장소 - 플랫폼 컴포넌트 및 애플리케이션 배포

## 🏛️ 아키텍처
```
ArgoCD (root-app)
    │
    ├── platform-apps.yaml
    │   │
    │   ├── [Wave 1] platform-infra
    │   │   ├── alb-controller
    │   │   ├── efs-csi-driver
    │   │   └── external-secrets
    │   │
    │   ├── [Wave 5] karpenter          ★ Karpenter Controller
    │   │   └── Karpenter Controller
    │   │
    │   ├── [Wave 6] karpenter-config   ★ Karpenter 설정
    │   │   ├── EC2NodeClass
    │   │   └── NodePool
    │   │
    │   └── [Wave 10] platform-ingress
    │       └── argocd-ingress
    │
    └── petclinic-app.yaml
        └── [Wave 15] petclinic         ★ VPC CNI 준비 후 배포
```

## 📁 디렉토리 구조
```
├── apps/                      # ArgoCD Application 정의
│   ├── platform-apps.yaml     # 플랫폼 컴포넌트 (Sync Wave 적용)
│   └── petclinic-app.yaml     # PetClinic 애플리케이션 (Wave 15)
│
├── platform/                  # 플랫폼 컴포넌트 매니페스트
│   ├── alb-controller/        # AWS Load Balancer Controller
│   ├── efs-csi-driver/        # EFS CSI Driver
│   ├── external-secrets/      # External Secrets Operator
│   ├── karpenter/             # ★ Karpenter Controller (Helm)
│   ├── karpenter-config/      # ★ NodePool & EC2NodeClass
│   └── argocd-ingress/        # ArgoCD ALB Ingress
│
└── applications/              # 애플리케이션 매니페스트
    └── petclinic/             # PetClinic 서비스들
```

## 🧩 플랫폼 컴포넌트

| 컴포넌트 | 설명 | Sync Wave |
|---------|------|-----------|
| ALB Controller | AWS ALB/NLB Ingress 관리 | 1 (먼저) |
| EFS CSI Driver | EFS 볼륨 마운트 | 1 (먼저) |
| External Secrets | AWS Secrets Manager 연동 | 1 (먼저) |
| **Karpenter** | 노드 자동 프로비저닝 | **5** |
| **Karpenter Config** | NodePool, EC2NodeClass | **6** |
| ArgoCD Ingress | ArgoCD 웹 UI ALB 노출 | 10 (나중) |
| **PetClinic** | 애플리케이션 | **15** (VPC CNI 준비 후) |

## ⚡ Sync Wave 배포 순서

```
Wave 1: platform-infra (인프라 기반)
├── alb-controller      ✅ ALB Controller 설치
├── efs-csi-driver      ✅ EFS CSI Driver 설치
└── external-secrets    ✅ External Secrets 설치
        ↓
Wave 5: karpenter (노드 오토프로비저너)
└── karpenter           ✅ Karpenter Controller 설치
        ↓
Wave 6: karpenter-config (Karpenter 설정)
├── EC2NodeClass        ✅ EC2 인스턴스 설정
└── NodePool            ✅ 노드 풀 규칙 정의
        ↓
Wave 10: platform-ingress (외부 노출)
└── argocd-ingress      ✅ ALB 자동 생성!
        ↓
Wave 15: petclinic (애플리케이션)  ★ NEW
└── petclinic           ✅ VPC CNI IP 풀 준비 완료 후 배포
```

> **Wave 15의 필요성**: EKS 클러스터 생성 직후 VPC CNI(aws-node)의 IP 풀 준비에 30초~2분 소요. 이전에 Pod 배포 시 `failed to assign an IP address` 에러 발생 가능.

## 🎯 Karpenter 설정

### EC2NodeClass (`platform/karpenter-config/templates/ec2nodeclass.yaml`)

| 설정 | 값 | 설명 |
|------|-----|------|
| AMI | AL2023 | Amazon Linux 2023 EKS 최적화 |
| Role | `petclinic-kr-karpenter-node` | Terraform에서 생성 |
| Subnet | `karpenter.sh/discovery` 태그 | Private EKS Subnet |
| Security Group | `karpenter.sh/discovery` 태그 | EKS 클러스터 SG |
| Volume | 50GB gp3, 암호화 | EBS 루트 볼륨 |
| IMDS | IMDSv2 필수 | 보안 강화 |

### NodePool (`platform/karpenter-config/templates/nodepool-general.yaml`)

| 설정 | 값 | 설명 |
|------|-----|------|
| 용량 타입 | **Spot 우선**, ON_DEMAND fallback | 비용 최적화 |
| 인스턴스 카테고리 | t (버스터블) | t3 시리즈 |
| 인스턴스 크기 | medium, large, xlarge, 2xlarge | 2~8 vCPU |
| CPU 제한 | 100 vCPU | 최대 프로비저닝 |
| Memory 제한 | 200Gi | 최대 프로비저닝 |
| 통합 정책 | WhenEmptyOrUnderutilized | 빈/저사용 노드 종료 |
| 통합 대기 | 1분 | 빠른 스케일다운 |

## 💰 비용 최적화 전략

### Spot Instance 우선 사용

```yaml
# NodePool 설정
requirements:
  - key: karpenter.sh/capacity-type
    operator: In
    values:
      - spot        # 우선 사용 (60-70% 저렴)
      - on-demand   # fallback (Spot 불가 시)
```

### 자동 노드 통합

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 1m
  budgets:
    - nodes: "20%"  # 동시에 20%까지만 중단
```

### 예상 비용 절감

| 시나리오 | ON_DEMAND | Spot 혼합 | 절감 |
|---------|-----------|-----------|------|
| t3.medium 3대 | ~$115/월 | ~$40/월 | **~65%** |

## 🚀 사용법

ArgoCD에서 root-app이 이 저장소를 감시하여 자동 배포
```bash
# ArgoCD 동기화 상태 확인
argocd app list

# 수동 동기화
argocd app sync platform-infra
argocd app sync karpenter
argocd app sync karpenter-config
argocd app sync platform-ingress
argocd app sync petclinic
```

## 🔍 Karpenter 모니터링

```bash
# 전체 상태 확인
kubectl get ec2nodeclasses,nodepools,nodeclaims

# Karpenter 로그 확인
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter -f

# NodePool 상태 확인
kubectl get nodepools

# NodeClaim 상태 확인 (프로비저닝 중인 노드)
kubectl get nodeclaims

# EC2NodeClass 확인
kubectl get ec2nodeclasses

# Karpenter가 프로비저닝한 노드 확인
kubectl get nodes -l karpenter.sh/nodepool=general
```

## 🔐 IRSA 설정 필요

`platform/` 하위 values.yaml에서 IRSA Role ARN 설정:
```yaml
# platform/karpenter/values.yaml
karpenter:
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/petclinic-kr-karpenter-controller
```

## 🔧 트러블슈팅

### 1. Karpenter Pod가 Pending 상태

**증상:**
```
karpenter-xxx   0/1   Pending   0   5m
```

**원인:** Managed Node Group에 `karpenter.sh/nodepool` 레이블이 있어 Node Affinity 충돌

**해결:**
```bash
kubectl label nodes --all karpenter.sh/nodepool-
```

### 2. EC2NodeClass가 Ready=Unknown

**증상:**
```
kubectl get ec2nodeclasses
NAME      READY
default   Unknown
```

**원인:** Karpenter Controller IAM Role에 Instance Profile 관리 권한 부족

**해결:** `platform-dev`의 `karpenter.tf`에서 IAM 정책 확인:
```hcl
# 필요한 권한
Action = [
  "iam:AddRoleToInstanceProfile",
  "iam:CreateInstanceProfile",
  "iam:DeleteInstanceProfile",
  "iam:GetInstanceProfile",
  "iam:RemoveRoleFromInstanceProfile",
  "iam:TagInstanceProfile"
]
```

### 3. NodeClaim 생성되지만 노드 안 뜸

**증상:**
```
AuthFailure.ServiceLinkedRoleCreationNotPermitted
```

**원인:** Spot Service-Linked Role 없음

**해결:**
```bash
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com
```

### 4. ArgoCD karpenter-config Unknown 상태

**증상:**
```
parse error: unexpected ".0" in operand
```

**원인:** Helm 템플릿에서 배열 접근 문법 오류

**잘못된 코드:**
```yaml
{{ .Values.blockDeviceMappings.0.ebs.volumeSize }}
```

**올바른 코드:**
```yaml
{{ (index .Values.blockDeviceMappings 0).ebs.volumeSize }}
# 또는 values.yaml 구조 단순화
{{ .Values.ebs.volumeSize }}
```

### 5. Pod가 ContainerCreating에서 멈춤

**증상:**
```
failed to assign an IP address to container
```

**원인:** VPC CNI의 IP 풀 준비 미완료 (클러스터 부트스트랩 직후)

**해결:**
- Pod 삭제 후 재생성: `kubectl delete pod <pod-name>`
- Sync Wave 15 설정으로 예방 (이 저장소에 적용됨)

## ⚠️ 주의사항

### Karpenter 설치 순서
1. **Terraform Apply 먼저** - Karpenter IAM, SQS 등 AWS 리소스 생성
2. **ArgoCD Sync** - Karpenter Controller → Config 순서로 자동 배포

### 시스템 노드 보호
- Karpenter Controller는 **Managed Node Group**에서 실행
- `karpenter.sh/nodepool` 레이블이 없는 노드에만 배치
- Karpenter가 자신을 실행하는 노드를 삭제하지 않음

### Spot 중단 대응
- SQS Queue로 2분 전 알림 수신
- 자동으로 새 노드 프로비저닝
- Pod 안전하게 이동

## 🔗 연관 저장소

| 저장소 | 설명 |
|--------|------|
| **platform-dev** | Terraform/Terragrunt 인프라 코드 (Karpenter IAM 포함) |
| **petclinic-gitops** | PetClinic 애플리케이션 GitOps |
| **petclinic-dev** | PetClinic 소스 코드 + CI/CD |
# rhel.ha-cluster_pacemaker_on_aws

AWS에서 RHEL 기반 고가용성(HA) 클러스터를 자동으로 구성하는 솔루션입니다.

## 개요

이 프로젝트는 두 가지 주요 구성 요소로 이루어져 있습니다:
1. **CloudFormation 템플릿** (`deployment.yaml`) - AWS 인프라 구성
2. **SSM 문서** (`ha-cluster-setup-document.json`) - HA 클러스터 자동 설정

## 아키텍처

### 인프라 구성
- **VPC**: 10.0.0.0/16 CIDR 블록
- **서브넷**: 
  - SubnetB (ap-northeast-2b): 10.0.1.0/24
  - SubnetD (ap-northeast-2d): 10.0.2.0/24
- **EC2 인스턴스**: RHEL 9.6 (t3.medium) 2대
- **보안 그룹**: 클러스터 내부 통신 허용
- **IAM 역할**: STONITH 및 Elastic IP 관리 권한

### HA 클러스터 구성
- **Pacemaker/Corosync** 기반 클러스터
- **STONITH**: `fence_aws` 사용
- **Ansible**: rhel-system-roles로 자동 구성
- **선택적 Dummy 리소스**: 테스트용

## 사전 요구사항

- AWS CLI 설치 및 구성
- 적절한 AWS 권한 (EC2, IAM, SSM)
- 서울 리전 (ap-northeast-2) 사용

## 설치 및 사용법

### 1단계: 인프라 배포

```bash
# CloudFormation 스택 생성
aws cloudformation create-stack \
    --stack-name rhel-ha-cluster \
    --template-body file://deployment.yaml \
    --parameters ParameterKey=SameSubnet,ParameterValue=false \
    --capabilities CAPABILITY_NAMED_IAM \
    --region ap-northeast-2
```

#### 파라미터
- `SameSubnet`: 
  - `false` (기본값): 서로 다른 AZ에 배치
  - `true`: 같은 서브넷에 배치

### 2단계: SSM 문서 생성

```bash
# SSM 문서 생성
aws ssm create-document \
    --name "HA-Cluster-Setup" \
    --document-type "Command" \
    --document-format "JSON" \
    --content file://ha-cluster-setup-document.json \
    --region ap-northeast-2
```

### 3단계: HA 클러스터 구성

***[주의] HA 클러스터 설정 실행은 Node 1 에서만 SSM 문서 "명령 실행" 을 수행합니다.***   
   
```bash
# 인스턴스 정보 확인
aws cloudformation describe-stacks \
    --stack-name rhel-ha-cluster \
    --query 'Stacks[0].Outputs' \
    --region ap-northeast-2

# HA 클러스터 설정 실행 > Node 1 로 사용될 인스턴스 한대에 대상으로 SSM 문서 명령 실행
aws ssm send-command \
    --document-name "HA-Cluster-Setup" \
    --instance-ids "i-xxxxxxxxx" \
    --parameters '{
        "Node1InstanceId":"i-xxxxxxxxx",
        "Node1PrivateIP":"10.0.1.10",
        "Node2InstanceId":"i-yyyyyyyyy",
        "Node2PrivateIP":"10.0.2.20",
        "ClusterPassword":"secure-password",
        "ClusterName":"my-ha-cluster",
        "DeployDummyResource":"false"
    }' \
    --region ap-northeast-2
```

## 파라미터 설명

### CloudFormation 파라미터
| 파라미터 | 기본값 | 설명 |
|----------|--------|------|
| SameSubnet | false | 인스턴스 배치 방식 선택 |

### SSM 문서 파라미터
| 파라미터 | 필수 | 기본값 | 설명 |
|----------|------|--------|------|
| Node1InstanceId | ✓ | - | 노드1 인스턴스 ID |
| Node1PrivateIP | ✓ | - | 노드1 프라이빗 IP |
| Node2InstanceId | ✓ | - | 노드2 인스턴스 ID |
| Node2PrivateIP | ✓ | - | 노드2 프라이빗 IP |
| ClusterPassword | | redhat | 클러스터 패스워드 |
| ClusterName | | fast-aws-rh-cluster | 클러스터 이름 |
| DeployDummyResource | | false | 테스트용 Dummy 리소스 배포 |

## 주요 기능

### 자동화된 구성
- **시스템 업데이트**: 최신 패키지로 자동 업데이트
- **사용자 생성**: haadm 계정 자동 생성
- **인스턴스 연결**: Session Manager 를 사용하도록 설정
- **필수 패키지**: rhel-system-roles, AWS CLI, 기타 도구 설치

### HA 클러스터 설정
- **방화벽/SELinux**: 자동 비활성화
- **부팅 시 시작**: 비활성화 (수동 시작)
- **클라우드 에이전트**: 자동 설치
- **STONITH 설정**: `fence_aws` 사용

### 네트워크 구성
- **호스트 파일**: 자동 업데이트 (`/etc/hosts`)
- **Ansible 인벤토리**: 동적 생성

## 파일 구조

```
pcs_automation/
├── deployment.yaml                 # CloudFormation 템플릿
├── ha-cluster-setup-document.json  # SSM 문서
└── README.md                       # 이 파일
```

## 생성되는 파일들

SSM 문서 실행 시 `/usr/local/ha_cluster/`에 다음 파일들이 생성됩니다:

- `inventory.yml`: Ansible 인벤토리
- `update-hosts.yaml`: 호스트 파일 업데이트 플레이북
- `fast-aws-playbook.yaml`: HA 클러스터 배포 플레이북
- `group_vars/${CLUSTER_NAME}.yml`: 클러스터 설정 변수

## 보안 고려사항

- **IAM 권한**: 최소 권한 원칙 적용
- **보안 그룹**: VPC 내부 통신만 허용
- **IMDS**: v2 사용으로 보안 강화

## 문제 해결

### 일반적인 문제
1. **SSM 에이전트 연결 실패**: IAM 역할 확인
2. **Ansible 실행 실패**: SSH 연결성 확인
3. **STONITH 실패**: IAM 권한 확인

### 로그 확인
```bash
# SSM 명령 실행 상태 확인
aws ssm get-command-invocation \
    --command-id "command-id" \
    --instance-id "i-xxxxxxxxx" \
    --region ap-northeast-2

# 클러스터 상태 확인
sudo pcs status
sudo pcs config
```

## 정리

```bash
# CloudFormation 스택 삭제
aws cloudformation delete-stack \
    --stack-name rhel-ha-cluster \
    --region ap-northeast-2

# SSM 문서 삭제
aws ssm delete-document \
    --name "HA-Cluster-Setup" \
    --region ap-northeast-2
```

## 기여

버그 리포트나 기능 요청은 이슈를 통해 제출해 주세요.

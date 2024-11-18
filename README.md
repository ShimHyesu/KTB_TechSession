# KTB_TechSession
2024.11.19_카카오테크 부트캠프 기술 세미나_클라우드 - EKS 설정법
<br><br><br>

## 실습 개요

본 실습에서는 Terraform과 eksctl을 활용하여 AWS EKS 클러스터를 구축하고, AWS Load Balancer Controller를 통해 NLB 및 ALB를 실습합니다.
<br><br>


## 실습 진행 단계

### 1. Terraform을 활용한 EKS 클러스터 구성
Terraform을 사용하여 VPC, 서브넷 등을 포함한 클러스터 네트워크를 자동으로 생성합니다.

<img width="1398" alt="image" src="https://github.com/user-attachments/assets/de6ecc48-7758-43fc-a8f9-2bf5f528977a">
<br>

### 2. eks-node-role 역할 생성
```bash
# 역할 생성
aws iam create-role --role-name eks-node-role --assume-role-policy-document file://trust-policy.json

# 정책 연결
aws iam attach-role-policy --role-name eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam attach-role-policy --role-name eks-node-role --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```
<br>

### 3. eks-cluster-config.yaml 파일 수정
- vpc, subnet id
- managedNodeGroups.iam.instanceRoleARN
<br><br>

### 4. 클러스터 생성
[eksctl 설치 가이드](https://eksctl.io/installation/)를 참고
```bash
eksctl create cluster -f eks-cluster-config.yaml
```
<br>

### 5. OIDC를 활성화하고, eksctl update addon 명령어를 사용하여 애드온을 업데이트
```bash
# OIDC 활성화
eksctl utils associate-iam-oidc-provider --cluster eks-cluster --region ap-northeast-2 --approve

# vpc-cni 애드온 업데이트
eksctl update addon --name vpc-cni --cluster eks-cluster --region ap-northeast-2 --force
```
<br>

### 6. helm 을 통한 AWS LB Controller 설치
[helm 설치 가이드](https://helm.sh/docs/intro/install/)를 참고

```bash
# Helm Chart Repository 추가 및 업데이트
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# Helm Chart - AWS Load Balancer Controller 설치
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --namespace kube-system \
    --set clusterName=${CLUSTER_NAME} \
    --set serviceAccount.create=false \
    --set region=${CLUSTER_REGION} \
    --set vpcId=${CLUSTER_VPC} \
    --set serviceAccount.name=aws-load-balancer-controller
```
<br>

### 7. NLB 실습
- 콘솔에서 NLB가 생성된 것 확인
- 등록된 대상 → Pod의 ip인것 확인
- 트래픽 확인
```bash
kubectl apply -f service-nlb.yaml
```
```bash
# NLB 도메인 변수 선언
NLB=$(kubectl get svc svc-nlb-type -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")

# 확인
echo $NLB

# 웹 접속 확인 (100회 - 카운팅)
for i in {1..100}; do curl -s $NLB | grep Hostname ; done | sort | uniq -c | sort -nr
## 약 5:5의 비율로 로드밸런싱 수행되는것 확인

# 파드 3대로 조정
kubectl scale deployment deploy-echo --replicas=3

# 웹 접속 확인 (100회 - 카운팅)
for i in {1..100}; do curl -s $NLB | grep Hostname ; done | sort | uniq -c | sort -nr
## 1/3씩 로드밸런싱 수행되는것 확인
```
<br>

### 8. ALB 실습
- 콘솔에서 ALB 생성된것 확인
- 해당 DNS로 접속 → 마리오 게임
```bash
kubectl apply -f service-alb.yaml
```

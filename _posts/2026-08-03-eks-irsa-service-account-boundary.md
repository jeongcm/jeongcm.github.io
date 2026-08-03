---
title: "EKS IRSA를 ServiceAccount 단위 권한 경계로 이해하기"
date: 2026-08-03 09:12:00 +0900
categories: [data-engineering]
tags: [EKS, IRSA, Kubernetes, IAM, data-platform]
---

데이터 플랫폼을 Kubernetes 위에서 운영하면 Pod가 S3, DynamoDB, MSK, Secrets Manager 같은 AWS 리소스에 접근해야 하는 순간이 자주 생긴다. 이때 node IAM role에 넓은 권한을 붙이면 같은 node의 여러 workload가 권한 경계를 공유하게 된다. EKS의 IAM Roles for Service Accounts, 즉 IRSA는 Kubernetes `ServiceAccount`와 AWS IAM role을 연결해 Pod 단위에 가까운 최소 권한 경계를 만들기 위한 방식이다.

AWS EKS 문서에 따르면 IRSA는 Pod의 컨테이너가 AWS SDK나 AWS CLI로 API를 호출할 때 사용할 임시 자격 증명을 제공한다. 애플리케이션에 access key를 배포하거나 EC2 instance profile에 모든 권한을 몰아넣는 대신, 특정 Kubernetes service account에 IAM role ARN을 annotation으로 연결한다. 그러면 해당 service account를 사용하는 Pod만 그 role을 통해 AWS API를 호출할 수 있다.

<figure class="post-diagram">
  <svg viewBox="0 0 880 340" role="img" aria-labelledby="irsa-title irsa-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="irsa-title">EKS IRSA trust flow</title>
    <desc id="irsa-desc">A pod uses a Kubernetes service account token. AWS STS validates it through the EKS OIDC provider and returns temporary credentials for the IAM role.</desc>
    <defs>
      <marker id="arrow-irsa" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
        <path d="M1,1 L9,5 L1,9 Z" fill="#475569" />
      </marker>
      <style>
        .outer { fill: #f8fafc; stroke: #cbd5e1; stroke-width: 2; rx: 8; }
        .k8s { fill: #eef6ff; stroke: #3b82f6; stroke-width: 2; rx: 8; }
        .aws { fill: #fff7ed; stroke: #f97316; stroke-width: 2; rx: 8; }
        .iam { fill: #ecfdf5; stroke: #10b981; stroke-width: 2; rx: 8; }
        .label { font: 700 16px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #0f172a; }
        .small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #475569; }
        .line { stroke: #475569; stroke-width: 3; fill: none; marker-end: url(#arrow-irsa); }
      </style>
    </defs>
    <rect x="24" y="28" width="832" height="284" class="outer" />
    <text x="54" y="64" class="label">IRSA narrows AWS permissions to a Kubernetes ServiceAccount</text>
    <rect x="58" y="126" width="170" height="96" class="k8s" />
    <text x="143" y="158" text-anchor="middle" class="label">Pod</text>
    <text x="143" y="184" text-anchor="middle" class="small">uses ServiceAccount</text>
    <text x="143" y="206" text-anchor="middle" class="small">projected OIDC token</text>
    <path d="M228 174 H320" class="line" />
    <rect x="320" y="112" width="190" height="124" class="k8s" />
    <text x="415" y="146" text-anchor="middle" class="label">EKS OIDC issuer</text>
    <text x="415" y="172" text-anchor="middle" class="small">cluster-specific provider</text>
    <text x="415" y="196" text-anchor="middle" class="small">signing key rotation</text>
    <path d="M510 174 H602" class="line" />
    <rect x="602" y="96" width="204" height="152" class="aws" />
    <text x="704" y="132" text-anchor="middle" class="label">AWS STS</text>
    <text x="704" y="158" text-anchor="middle" class="small">AssumeRoleWithWebIdentity</text>
    <text x="704" y="182" text-anchor="middle" class="small">trust policy checks sub/aud</text>
    <text x="704" y="206" text-anchor="middle" class="small">temporary credentials</text>
    <path d="M704 248 V282 H142 V224" class="line" />
  </svg>
  <figcaption>IRSA의 신뢰 경계는 service account token, cluster OIDC provider, IAM role trust policy가 함께 맞을 때 성립한다.</figcaption>
</figure>

핵심은 OIDC federation이다. EKS는 cluster마다 public OIDC discovery endpoint를 제공하고, Kubernetes의 projected service account token을 AWS IAM이 검증할 수 있게 한다. IAM role의 trust policy는 `sts:AssumeRoleWithWebIdentity`를 허용하면서 `aud`를 `sts.amazonaws.com`으로, `sub`를 `system:serviceaccount:<namespace>:<service-account>`로 제한한다. 이 조건이 좁을수록 같은 cluster 안에서도 workload별 권한 분리가 쉬워진다.

다음은 설명을 위한 개념 예시다. 실제 account id, OIDC provider, bucket ARN은 환경에 맞게 바꿔야 한다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: batch-s3-reader
  namespace: data-jobs
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/batch-s3-reader
```

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/CLUSTER_ID"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.ap-northeast-2.amazonaws.com/id/CLUSTER_ID:aud": "sts.amazonaws.com",
      "oidc.eks.ap-northeast-2.amazonaws.com/id/CLUSTER_ID:sub": "system:serviceaccount:data-jobs:batch-s3-reader"
    }
  }
}
```

운영에서 자주 놓치는 지점은 세 가지다. 첫째, AWS SDK가 IRSA를 사용하려면 기본 credential chain을 따라야 한다. 코드에서 다른 credential provider를 앞에 고정해 두면 IRSA를 설정해도 기존 자격 증명이 계속 쓰일 수 있다. 둘째, IMDS 접근을 제한하지 않으면 Pod가 node role credential에도 접근할 수 있는 구성이 남을 수 있다. AWS 문서는 `hostNetwork: true` Pod는 항상 IMDS 접근이 가능하지만, IRSA가 활성화된 경우 SDK와 CLI는 IRSA credential을 사용한다고 설명한다. 셋째, cluster OIDC provider는 cluster마다 설정해야 하며, EKS VPC endpoint만 있는 폐쇄망에서는 OIDC issuer URL 조회가 막혀 provider 생성이 실패할 수 있다.

2026년 현재 EKS 문서는 Pod에 AWS 권한을 주는 기본 선택지로 EKS Pod Identity와 IRSA를 함께 설명한다. AWS best practices는 특별히 IRSA가 필요한 경우가 아니라면 EKS Pod Identity를 권장한다고 정리한다. 다만 IRSA는 OIDC federation과 `AssumeRoleWithWebIdentity`를 직접 사용하므로 cross-account 접근을 비교적 명확하게 설계할 수 있고, 기존 controller나 data job chart에서 이미 널리 쓰인다.

설계 기준은 단순하다. S3 읽기 batch job, ExternalDNS, AWS Load Balancer Controller, Spark driver처럼 AWS API 권한이 필요한 workload마다 service account를 분리하고, IAM policy는 필요한 action과 resource로 줄인다. namespace 전체에 wildcard trust를 열기보다 `sub` 조건을 구체적으로 두고, 여러 workload가 같은 role을 공유해야 할 때만 의식적으로 범위를 넓힌다. IRSA는 secret을 없애는 기능이 아니라, Kubernetes workload identity와 AWS IAM 권한을 검증 가능한 계약으로 묶는 장치다.

## 참고 링크

- [Amazon EKS - IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Amazon EKS - Assign IAM roles to Kubernetes service accounts](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html)
- [Amazon EKS - Use IRSA with the AWS SDK](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-minimum-sdk.html)
- [Amazon EKS best practices - Identity and Access Management](https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html)
- [Amazon EKS - Authenticate to another account with IRSA](https://docs.aws.amazon.com/eks/latest/userguide/cross-account-access.html)

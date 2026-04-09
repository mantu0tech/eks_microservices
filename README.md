# Microservices Deployment on Amazon EKS with Fargate & ALB Ingress

A hands-on practical demonstrating how to deploy two microservices (`/` and `/about`) on **Amazon EKS using AWS Fargate**, routed via an **AWS Application Load Balancer (ALB) Ingress Controller**, with Docker images pulled directly from **Docker Hub** — all managed through the CLI.

---

## Architecture

```
<img width="934" height="664" alt="image" src="https://github.com/user-attachments/assets/edfc5bf2-e462-4356-bdaf-4b6de409f8bc" />

```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Container Orchestration | Amazon EKS |
| Compute | AWS Fargate (serverless) |
| Load Balancer | AWS ALB via Ingress Controller |
| Routing | Kubernetes Ingress |
| Config Management | Kubernetes ConfigMap |
| Container Registry | Docker Hub |
| CLI Tools | `eksctl`, `kubectl`, `aws cli`, `helm` |
| OS | Ubuntu (EC2) |

---

## Prerequisites

- AWS account with sufficient IAM permissions
- EC2 instance (Ubuntu) with:
  - `eksctl`
  - `kubectl`
  - `aws cli` (configured with `aws configure`)
  - `helm`
- Docker images pushed to Docker Hub

---

## Project Structure

```
.
├── README.md
└── app.yaml          # All K8s manifests in one file
    ├── Namespace
    ├── ConfigMap
    ├── Deployment (home)
    ├── Service (home)
    ├── Deployment (about)
    ├── Service (about)
    └── Ingress (ALB)
```

---

## Step-by-Step Deployment

### 1. Create EKS Cluster with Fargate

```bash
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --fargate \
  --alb-ingress-access

aws eks update-kubeconfig --region us-east-1 --name my-cluster
```

### 2. Create Fargate Profile for App Namespace

```bash
eksctl create fargateprofile \
  --cluster my-cluster \
  --name app-profile \
  --namespace app \
  --region us-east-1
```

### 3. Install AWS Load Balancer Controller

```bash
# Associate OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --region us-east-1 \
  --approve

# Create IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Create service account (replace with your AWS account ID)
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve \
  --region us-east-1

# Install via Helm
helm repo add eks https://aws.github.io/eks-charts && helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccountName=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$(aws eks describe-cluster --name my-cluster \
    --query "cluster.resourcesVpcConfig.vpcId" --output text)
```

### 4. Apply All Kubernetes Manifests

```bash
kubectl apply -f app.yaml
```

The `app.yaml` includes:

- **Namespace** — `app`
- **ConfigMap** — shared env vars injected into both containers
- **Deployments** — one for home (`/`), one for about (`/about`), both pulling from Docker Hub
- **Services** — ClusterIP services for each deployment
- **Ingress** — ALB with path-based routing, `target-type: ip` (required for Fargate)

### 5. Get the Load Balancer URL

```bash
kubectl get ingress app-ingress -n app
```

Wait ~2 minutes, then access:

```
http://<ALB-ADDRESS>/        → Home page
http://<ALB-ADDRESS>/about   → About page
```

---

## Key Manifest — Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: app
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip   # required for Fargate
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: home-service
                port:
                  number: 80
          - path: /about
            pathType: Prefix
            backend:
              service:
                name: about-service
                port:
                  number: 80
```

> **Why `target-type: ip`?** Fargate does not support node-level target groups. The ALB must target pod IPs directly.

---

## ConfigMap Usage

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: app
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
```

Referenced in deployments via `envFrom`:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

---

## Cleanup

```bash
# Delete all K8s resources
kubectl delete -f app.yaml

# Uninstall Helm chart
helm uninstall aws-load-balancer-controller -n kube-system

# Delete IAM service account
eksctl delete iamserviceaccount \
  --cluster my-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --region us-east-1

# Delete Fargate profile
eksctl delete fargateprofile \
  --cluster my-cluster \
  --name app-profile \
  --region us-east-1

# Delete the cluster (also removes VPC, subnets, etc.)
eksctl delete cluster --name my-cluster --region us-east-1
```

here is the output 

<img width="939" height="537" alt="image" src="https://github.com/user-attachments/assets/005d2eef-5845-4418-bd0e-d1fb2d9c2541" />
<img width="934" height="577" alt="image" src="https://github.com/user-attachments/assets/6c14ee8a-62f2-4306-9bb0-a83ddf3ff84c" />


---

## Lessons Learned

- `target-type: ip` is mandatory in the Ingress annotation when running on Fargate — without it the ALB health checks fail silently.
- Fargate profiles must match the namespace of the pods — pods in an unmatched namespace will not schedule.
- The AWS Load Balancer Controller must be installed via Helm with IRSA (IAM Roles for Service Accounts) set up correctly before the Ingress resource creates an actual ALB.
- ConfigMaps are a clean way to decouple environment-specific config from the container image.

---

## Connect

Feel free to open an issue or connect on [LinkedIn](#) if you have questions or feedback!

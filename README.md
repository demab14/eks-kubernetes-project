# EKS Kubernetes Project

![Build and Deploy to EKS](https://github.com/demab14/eks-kubernetes-project/actions/workflows/deploy.yml/badge.svg)

A production-grade containerized application deployed on Amazon EKS with automated CI/CD, horizontal autoscaling and CloudWatch monitoring.

## Architecture
```
Developer ── git push ──► GitHub Actions ──► Docker Hub (image:<git-sha>)
                               │
                               └── OIDC ──► AWS EKS (eu-west-2) ── helm upgrade

User → AWS Load Balancer → Kubernetes Service → Pods (1–5, managed by HPA) → nginx container
```

## Tech Stack
- AWS EKS (Kubernetes cluster, 3 x t3.medium nodes)
- Docker + Docker Hub
- Helm (deployment management)
- GitHub Actions (CI/CD with AWS OIDC, no long-lived keys)
- HPA (horizontal pod autoscaling)
- AWS CloudWatch Container Insights (monitoring)

## Project Phases
1. Environment setup
2. EKS cluster creation (3 x t3.medium nodes)
3. Docker app build and Helm deployment
4. CI/CD pipeline with GitHub Actions
5. Horizontal Pod Autoscaler configuration
6. Load testing with siege
7. CloudWatch Container Insights monitoring

## CI/CD Pipeline (GitHub Actions)

Workflow: [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml)

| Trigger | Jobs that run |
|---|---|
| Pull request to `main` | `validate` only |
| Push to `main` / manual run | `validate` → `deploy` |

**`validate`**
- `helm lint` on the chart
- Renders the chart and validates it plus `hpa.yaml` against the Kubernetes schema with `kubeconform`
- Builds the Docker image to catch Dockerfile errors (not pushed)

**`deploy`** (runs in the `production` environment, so it can be protected with a required reviewer)
1. Builds and pushes the image to Docker Hub tagged with the commit SHA (immutable, traceable to a commit) and `latest`
2. Authenticates to AWS with **OIDC** by assuming an IAM role, so no AWS access keys are stored in GitHub
3. Runs `helm upgrade --install` with `image.tag=<commit SHA>` and waits for the release to become healthy
4. Applies the HPA and verifies the rollout with `kubectl rollout status`

### Required setup

GitHub repository secrets:

| Secret | Purpose |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub account |
| `DOCKERHUB_TOKEN` | Docker Hub access token (not your password) |
| `AWS_ROLE_ARN` | IAM role GitHub Actions assumes via OIDC |

AWS side:
1. Add the GitHub OIDC identity provider to IAM (`token.actions.githubusercontent.com`, audience `sts.amazonaws.com`).
2. Create an IAM role whose trust policy only allows this repo's `main` branch:
   ```json
   "Condition": {
     "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
     "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:demab14/eks-kubernetes-project:*" }
   }
   ```
   Give it `eks:DescribeCluster`.
3. Grant the role access inside the cluster with an EKS access entry:
   ```bash
   aws eks create-access-entry --cluster-name enterprise-cluster --region eu-west-2 \
     --principal-arn <AWS_ROLE_ARN>
   aws eks associate-access-policy --cluster-name enterprise-cluster --region eu-west-2 \
     --principal-arn <AWS_ROLE_ARN> \
     --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
     --access-scope type=cluster
   ```
   For a real production setup, scope this down to a namespace-level policy.

## Autoscaling (HPA)

The Horizontal Pod Autoscaler ([`hpa.yaml`](hpa.yaml)) watches the CPU usage of the `eks-project` pods and adds or removes pods to keep average usage around the target.

| Setting | Value |
|---|---|
| Target | `Deployment/eks-project` |
| Min replicas | 1 |
| Max replicas | 5 |
| Scale-out trigger | average CPU above 50% of the request |
| CPU request / limit per pod | 300m / 500m |

Because each pod requests 300m CPU, the HPA adds pods once average usage goes above about 150m per pod. The CPU **request** in `values.yaml` is what makes this work: without requests, the HPA has nothing to calculate a percentage against and reports `<unknown>`.

**How it behaves**
- **Scale out:** under load, CPU climbs past 50%, and the HPA increases replicas (up to 5). New pods are added behind the same Service and Load Balancer, so users see no change.
- **Scale in:** when load drops, the HPA waits for its stabilisation window (5 minutes by default) before removing pods, which prevents flapping.

### Prerequisite: metrics-server
The HPA reads CPU from the Kubernetes Metrics API, so metrics-server must be installed:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl top pods   # should return CPU/memory, not an error
```

### Load test with siege
```bash
# Get the Load Balancer address
LB=$(kubectl get svc eks-project -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Terminal 1: watch the autoscaler react
kubectl get hpa eks-hpa --watch

# Terminal 2: 200 concurrent users for 5 minutes
siege -c 200 -t 5M http://$LB/
```

What to expect:
1. `TARGETS` in `kubectl get hpa` rises above `50%`.
2. `REPLICAS` increases step by step towards 5 (`kubectl get pods -w` shows new pods starting).
3. After siege stops, CPU falls, and after the stabilisation window replicas scale back down to 1.

### Results
<!-- Add screenshots from your own load test run -->
| Before load | During load | After cool-down |
|---|---|---|
| ![](docs/hpa-before.png) | ![](docs/hpa-during.png) | ![](docs/hpa-after.png) |

CloudWatch Container Insights view of the same test:

![](docs/cloudwatch-cpu.png)

## Monitoring
CloudWatch Container Insights collects CPU, memory, network and pod-count metrics for the cluster, nodes and pods. Use it alongside `kubectl get hpa` to confirm that scaling decisions line up with actual resource usage.

## Cleanup
```bash
helm uninstall eks-project
kubectl delete -f hpa.yaml
eksctl delete cluster --name enterprise-cluster --region eu-west-2
```

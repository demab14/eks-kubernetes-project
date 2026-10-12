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
| Pull request or push to `main` | `validate` only |
| Manual run (**Actions → Run workflow**) | `validate` → `deploy` |

Deploys are manual on purpose. The cluster is only running while I'm demoing or testing, so automatic deploys on every push would fail whenever it's switched off.

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
The HPA reads CPU from the Kubernetes Metrics API, so metrics-server must be installed. Recent `eksctl` versions install it automatically as an EKS add-on. To install it by hand:
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

## Results (live run, October 2026)

### 1. Pipeline: build → push → deploy in 45 seconds
A manual run of the workflow passed every step end to end:
```
✓ Lint and validate manifests in 15s
✓ Build, push and deploy in 45s
  ✓ Log in to Docker Hub
  ✓ Build and push image
  ✓ Configure AWS credentials (OIDC, no static keys)
  ✓ Update kubeconfig
  ✓ Deploy with Helm
  ✓ Apply Horizontal Pod Autoscaler
  ✓ Verify rollout
✓ Run Build and Deploy to EKS completed with 'success'
```

### 2. Running in the cluster
```
$ kubectl get pods,svc,hpa
pod/eks-project-5dd6f98d4c-vg6x8   1/1   Running

service/eks-project   LoadBalancer   ...   <elb-hostname>.eu-west-2.elb.amazonaws.com   80:30328/TCP

horizontalpodautoscaler.autoscaling/eks-hpa   Deployment/eks-project   cpu: 0%/50%   1   5   1
```
The HPA reports real CPU figures (`0%/50%` rather than `<unknown>`), which confirms metrics-server is feeding it.

### 3. Autoscaling under load

**Before load:** 1 replica, CPU at 0%.
```
NAME      REFERENCE                TARGETS       MINPODS   MAXPODS   REPLICAS
eks-hpa   Deployment/eks-project   cpu: 0%/50%   1         5         1
```

**During load:** the HPA saw CPU go above the 50% target and **added a second pod automatically**:
```
NAME                           READY   STATUS    RESTARTS   AGE
eks-project-5dd6f98d4c-vg6x8   1/1     Running   0          11m
eks-project-5dd6f98d4c-vwgbl   1/1     Running   0          116s   ← created by the HPA

NAME      REFERENCE                TARGETS        MINPODS   MAXPODS   REPLICAS
eks-hpa   Deployment/eks-project   cpu: 19%/50%   1         5         2
```
With the traffic split across 2 pods, CPU per pod settled at about 10–25%, below the target. So the HPA held at 2 replicas and didn't over-provision.

**After load:** once the load stopped, the HPA waited for its 5-minute stabilisation window and then scaled back down to 1 replica.

### What I learned from the load test
- **External load testing was network-bound.** Running `siege` from my laptop only got pod CPU to about 9% and then hit connection timeouts. My own connection was the bottleneck, not the cluster.
- **In-cluster load isolates the app.** I generated traffic from busybox pods inside the cluster instead:
  ```bash
  kubectl create deployment load-generator --image=busybox --replicas=10 \
    -- /bin/sh -c "while true; do wget -q -O- http://eks-project > /dev/null; done"
  ```
  This pushed CPU over the threshold and triggered the scale-out.
- **Static nginx is cheap to serve.** Increasing the load generators from 10 to 25 didn't raise CPU further, because each busybox `wget` loop sends only one request at a time. Scaling up to the 5-replica maximum would need a heavier load tool (for example `hey` or `k6`) or a more CPU-intensive app.

<!-- Optional: add screenshots to docs/ and uncomment
| Before | During | After |
|---|---|---|
| ![](docs/hpa-before.png) | ![](docs/hpa-during.png) | ![](docs/hpa-after.png) |
-->

## Monitoring
CloudWatch Container Insights collects CPU, memory, network and pod-count metrics for the cluster, nodes and pods. Use it alongside `kubectl get hpa` to confirm that scaling decisions line up with actual resource usage.

## Cleanup
```bash
helm uninstall eks-project
kubectl delete -f hpa.yaml
eksctl delete cluster --name enterprise-cluster --region eu-west-2
```

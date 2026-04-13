# EKS Kubernetes Project

A production-grade containerized application deployed on Amazon EKS with autoscaling and CloudWatch monitoring.

## Architecture
User → AWS Load Balancer → Kubernetes Service → Pods → nginx Container

## Tech Stack
- AWS EKS (Kubernetes cluster)
- Docker + Docker Hub
- Helm (deployment management)
- HPA (autoscaling)
- AWS CloudWatch (monitoring)

## Project Phases
1. Environment setup
2. EKS cluster creation (3 x t3.medium nodes)
3. Docker app build and Helm deployment
4. Horizontal Pod Autoscaler configuration
5. Load testing with siege
6. CloudWatch Container Insights monitoring

## Cleanup
```bash
kubectl delete all --all
eksctl delete cluster --name enterprise-cluster --region eu-west-2
```

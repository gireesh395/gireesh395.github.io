---
title: "Breaking and Securing My First DevSecOps Pipeline on AWS"
date: 2026-10-30T14:00:00+02:00
draft: false
tags: ["DevSecOps", "Kubernetes", "AWS", "Terraform", "GitHub Actions"]
summary: "How I built a 4-ring automated CI/CD pipeline, hit weird AWS ENI pod limits on t3.micro nodes, and learned cloud security by intentionally breaking my own builds."
---

Most tutorials make DevSecOps sound like magic: plug in a few GitHub Actions, add a couple of scanners, and boom—your app is secure. 

In reality, building a secure delivery pipeline means dealing with endless false alarms, weird cloud constraints, and realizing that simple configuration choices can crash an entire cluster. 

I built **Project Shield** to move beyond the textbook theory and build a real, functioning delivery pipeline on AWS that enforces security automatically—without developer roadblocks. Here is what I built, what broke, and what I learned along the way.

---

### The Strategy: A 4-Ring Defense Grid

I wanted a setup where security wasn’t an afterthought checked right before launch. The pipeline had to "shift left"—catching leaks, vulnerable libraries, and infrastructure bugs the second code hit GitHub.

I structured the CI/CD pipeline into four sequential stages:

```text
[ Git Push / PR ]
       │
       ├── RING 1: Secret Leaks & SAST (Gitleaks + Semgrep)
       ├── RING 2: Container CVE Auditing (Aqua Trivy)
       ├── RING 3: Infrastructure-as-Code Checks (Checkov)
       │
       ▼ (Runs only if Rings 1-3 pass cleanly)
[ RING 4: Build & Deploy to Amazon EKS ]

If anything fails in an upstream ring, the pipeline terminates immediately. No bad container ever gets pushed to Docker Hub, and no insecure manifest ever touches the cluster.

Hardening the Workload Before It Runs
A pipeline is useless if the application running inside the pod has excessive privileges. I started with a minimal Flask service and hardened it from the Dockerfile up to the Kubernetes manifest.

First, I built on Alpine and stripped root execution entirely:
FROM python:3.11-alpine
WORKDIR /workspace

# Create an isolated, non-root system user
RUN addgroup -S securegroup && adduser -S secureuser -G securegroup
COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ ./app

RUN chown -R secureuser:securegroup /workspace
USER secureuser
EXPOSE 5000
CMD ["python", "app/server.py"]

Next, in the Kubernetes deployment spec (k8s-deployment.yaml), I locked down the Linux runtime controls:
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 10001
  capabilities:
    drop:
    - ALL
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "250m"
    memory: "256Mi"

    Locking the filesystem to read-only ensures that even if someone manages to pop a shell inside the container, they cannot drop scripts or modify runtime binaries. Dropping ALL kernel capabilities ensures basic utilities can't be abused for network sniffing or privilege escalation.

The Fun Part: Intentionally Breaking the Pipeline
You can’t trust a security tool until you watch it fail on bad code. To prove the pipeline actually worked, I created branch after branch with deliberate security violations:

Leaking Fake AWS Keys: I pushed a test commit containing synthetic high-entropy AWS credentials. Ring 1 triggered immediately—Gitleaks scanned the commit delta, failed the run, and stopped execution before any build steps started.

Injecting Known CVEs: I added old versions of requests 2.20.0 and PyYAML 5.3 to requirements.txt. Aqua Trivy caught the critical vulnerabilities and blocked the image from moving forward.

Taming the Noise: Not every alert requires a code rewrite. Some upstream packages have open CVEs with no official fix available. Setting --ignore-unfixed and using a .trivyignore file allowed me to silence unfixable noise while keeping critical, actionable alerts loud and clear.

Provisioning the Cluster with Terraform
With the pipeline tested, I automated the AWS infrastructure using Terraform modules in the Paris region (eu-west-3).

provider "aws" {
  region = "eu-west-3"
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  name    = "shield-vpc"
  cidr    = "10.0.0.0/16"

  azs             = ["eu-west-3a", "eu-west-3b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = true
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  version         = "~> 20.0"
  cluster_name    = "shield-eks-cluster"
  cluster_version = "1.30"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  eks_managed_node_groups = {
    nodes = {
      min_size       = 1
      max_size       = 3
      desired_size   = 2
      instance_types = ["t3.micro"]
      ami_type       = "AL2023_x86_64_STANDARD"
    }
  }
}

Roadblocks & Real-World Lessons
1. The t3.micro Pod Limit Trap
To keep cloud costs minimal, I spun up the worker nodes using t3.micro instances. Everything looked clean until I scaled the deployment to 3 replicas—and watched pods get stuck in an endless Pending state.

I dug into the node logs and hit a fundamental AWS networking rule: each EC2 instance size has an Elastic Network Interface (ENI) cap on how many IP addresses it can assign to pods. On a t3.micro, that limit is 4 pods per node.

Between AWS system pods (aws-node, kube-proxy, and CoreDNS), three slots were already spoken for. The node literally ran out of IPs for application containers. The fix was recalculating my cluster density and tuning deployment replicas down to 2.

2. Disciplined Cloud FinOps
Managed Kubernetes clusters and NAT gateways cost real money every hour they sit idle. I treated the cloud environment as ephemeral:

Spun up the cluster with Terraform.

Validated pipeline runs and end-to-end rollouts.

Ran terraform destroy when testing finished.
module.eks.module.eks_managed_node_group["nodes"].aws_eks_node_group.this[0]: Still destroying...
module.eks.module.eks_managed_node_group["nodes"].aws_launch_template.this[0]: Destroying...
Destruction complete after 8m25s
Resources: 54 destroyed.

Tearing down 54 cloud resources cleanly when testing is done is just as important as setting them up.

What’s Next
Building Project Shield gave me a deep appreciation for the friction points between DevOps velocity and security engineering. For the next iteration, I’m planning to:

Switch to AWS IAM OIDC: Eliminate long-lived AWS keys in GitHub Secrets and use temporary, short-lived OIDC assume-role tokens.

Full GitOps with ArgoCD: Move the deploy step out of GitHub Actions entirely and let an in-cluster ArgoCD operator handle continuous sync and drift correction.

Lock Down Endpoints: Move the EKS API endpoint completely off the public internet behind an internal management bastion.
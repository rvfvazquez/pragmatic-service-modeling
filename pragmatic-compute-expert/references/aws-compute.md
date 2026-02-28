# AWS Compute Reference: Lambda, ECS, EKS (Fargate & EC2)

## Table of Contents
1. [AWS Lambda](#aws-lambda)
2. [AWS ECS](#aws-ecs)
3. [AWS EKS](#aws-eks)
4. [Infrastructure Layer: Fargate vs EC2](#infrastructure-layer-fargate-vs-ec2)
5. [Decision Matrix](#decision-matrix)
6. [Architecture Combination Patterns](#architecture-combination-patterns)

---

## AWS Lambda

**Type**: Serverless function-as-a-service (FaaS)
**Execution model**: Event-triggered, stateless, ephemeral (invocation duration up to 15 minutes)
**Scaling**: Automatic, instant burst scaling (up to 1,000 concurrent executions by default, soft limit adjustable)
**Runtime**: Managed runtimes (Node.js, Python, Java, Go, .NET, Ruby) or custom runtime via container image
**Cold starts**: Yes — milliseconds to seconds depending on runtime and package size (mitigated with Provisioned Concurrency)
**State**: Stateless by nature — state must live in DynamoDB, S3, ElastiCache, etc.
**Connections**: Short-lived — avoid persistent DB connections (use RDS Proxy or DynamoDB)
**Max package size**: 50MB zipped (250MB unzipped), or up to 10GB via container image
**Pricing**: Pay per invocation + duration (GB-seconds) — extremely cost-effective for intermittent workloads

### When Lambda wins
- Processing is triggered by events: API Gateway, SQS, SNS, S3, EventBridge, Kinesis, DynamoDB Streams
- Tasks are short-lived (seconds to a few minutes) and stateless
- Traffic is unpredictable or highly variable — no need to pay for idle capacity
- Startup wants maximum developer velocity with minimum infrastructure overhead
- Scheduled tasks (cron jobs via EventBridge Scheduler)
- Glue code between AWS services — fan-out, transformation, routing

### When Lambda struggles
- Long-running processes (> 15 minutes) — requires chunking or Step Functions orchestration
- Workloads requiring persistent in-memory state or long-lived connections
- Heavy compute tasks (video encoding, ML inference) — limited CPU/memory (max 10GB RAM, 6 vCPU)
- Cold start sensitivity for latency-critical synchronous APIs — consider Provisioned Concurrency
- Large dependency packages — startup time increases; use Lambda layers or container images
- Very high and steady throughput — at millions of req/sec, ECS/EKS on EC2 becomes more cost-effective

### Key design considerations
- **Concurrency limits**: Default 1,000 concurrent. Set reserved concurrency to protect downstream services.
- **Provisioned Concurrency**: Eliminates cold starts for latency-critical paths — adds fixed cost.
- **Lambda layers**: Share common dependencies across functions.
- **Step Functions**: Orchestrate multi-step workflows instead of chaining Lambda calls.
- **Power Tuning**: Use AWS Lambda Power Tuning tool to optimize memory/cost tradeoff.

---

## AWS ECS

**Type**: Container orchestration service — manages Docker containers
**Control plane**: AWS-managed (no cluster masters to operate)
**Workload types**: Services (long-running), Tasks (one-off jobs)
**Scaling**: Application Auto Scaling based on CPU, memory, or custom CloudWatch metrics
**Networking**: VPC-native with AWS VPC CNI — each task gets its own ENI (awsvpc mode)
**Service discovery**: Cloud Map or load balancer DNS
**Ecosystem**: Deep AWS integration — IAM task roles, Secrets Manager, X-Ray, CloudWatch Logs
**Complexity**: Lower than EKS — no Kubernetes knowledge required
**Infrastructure options**: Fargate (serverless containers) or EC2 (self-managed nodes)

### When ECS wins
- Team has Docker experience but little or no Kubernetes expertise
- Workload runs as long-lived containers (web APIs, background workers, streaming processors)
- Simpler deployment and operational model is preferred over Kubernetes flexibility
- Startup or small team wanting container benefits without Kubernetes overhead
- Migrating from single-instance EC2 to containers — ECS is a natural first step
- AWS-native integration is a priority (ALB, ACM, Secrets Manager, IAM roles per task)

### When ECS struggles
- Complex microservices requiring advanced scheduling, custom operators, or CRDs
- Multi-cloud or hybrid cloud requirements (Kubernetes is more portable)
- Organization already has Kubernetes expertise and tooling (Helm, Argo CD, etc.)
- Need for advanced networking policies, service meshes at cluster level, or Kubernetes ecosystem tools

### Key design considerations
- **Task definition**: Defines container image, CPU, memory, environment variables, secrets, and logging
- **Service auto-scaling**: Configure target tracking on CPU/memory or custom metrics
- **Blue/green deployments**: Native with CodeDeploy integration or rolling updates
- **Spot integration**: ECS can mix Fargate Spot with regular Fargate for cost savings

---

## AWS EKS

**Type**: Managed Kubernetes service
**Control plane**: AWS-managed Kubernetes masters (HA, automatically updated)
**Workload types**: Pods, Deployments, StatefulSets, DaemonSets, Jobs, CronJobs
**Scaling**: Cluster Autoscaler or Karpenter (node-level) + HPA/KEDA (pod-level)
**Networking**: VPC CNI (aws-node), supports Calico, Cilium for network policies
**Ecosystem**: Full Kubernetes ecosystem — Helm, Argo CD, Istio, Prometheus, Grafana, Karpenter, KEDA
**Infrastructure options**: Fargate profiles (serverless pods) or managed/self-managed node groups on EC2
**Complexity**: Higher than ECS — requires Kubernetes expertise

### When EKS wins
- Enterprise with existing Kubernetes expertise and tooling investment
- Large, complex microservices platform with many services requiring independent scaling
- Multi-cloud or hybrid cloud strategy — portability matters (Kubernetes runs anywhere)
- Advanced requirements: custom schedulers, operators, service mesh (Istio/Linkerd), KEDA for event-driven scaling
- Platform team exists to manage and abstract Kubernetes for development teams
- GitOps workflow with Argo CD or Flux is a requirement
- Fine-grained resource management: namespaces, RBAC, resource quotas per team

### When EKS struggles
- Small team without Kubernetes expertise — operational overhead is significant
- Simple workloads that don't need Kubernetes flexibility — ECS is simpler and cheaper to operate
- Tight time-to-market — ECS gets you to production faster
- Cost-conscious startups — EKS control plane has a fixed cost ($0.10/hr per cluster)

### Key design considerations
- **Managed node groups**: AWS manages node lifecycle (updates, replacements) — use over self-managed
- **Karpenter**: Modern node provisioner — faster and smarter than Cluster Autoscaler
- **KEDA**: Event-driven autoscaling for pods — scale on SQS depth, Kafka lag, custom metrics
- **Fargate profiles**: Serverless pods without managing nodes — good for burst workloads or isolated namespaces
- **Add-ons**: EKS Add-ons for CoreDNS, kube-proxy, VPC CNI, EBS CSI — managed by AWS

---

## Infrastructure Layer: Fargate vs EC2

This decision applies when ECS or EKS is chosen.

### AWS Fargate

**Model**: Serverless containers — no node/instance management
**Billing**: Pay per vCPU + GB memory per second of task/pod runtime
**Startup**: Slightly slower task startup vs EC2 (cold start of container image pull)
**Isolation**: Each task runs in its own micro-VM (stronger security isolation)
**Operations**: Zero node management — no patching, scaling node pools, or AMI updates
**Spot**: Fargate Spot available (up to 70% discount, can be interrupted)

#### Choose Fargate when:
- Small to medium team — operational simplicity is more valuable than cost optimization
- Variable, unpredictable workload — pay only for what you use
- Security/compliance requires strong workload isolation
- Batch or background jobs that run intermittently
- Getting started with containers — no infrastructure expertise yet
- EKS workloads that need quick isolation (Fargate profiles per namespace)

#### Fargate limitations:
- Cannot run privileged containers or host networking mode (awsvpc only for ECS)
- Slightly higher per-unit cost vs EC2 at large, steady-state scale
- No persistent volumes on Fargate (EFS supported; local ephemeral storage only)
- Startup time slightly higher than pre-warmed EC2 instances

---

### EC2 (for ECS or EKS)

**Model**: You manage EC2 instances as container hosts (or use EKS managed node groups)
**Billing**: Pay per instance — Reserved Instances (up to 72% savings) or Spot (up to 90% savings)
**Startup**: Faster container startup (instances are pre-warmed)
**Flexibility**: Full control — instance type, GPU, custom AMI, host-level networking
**Operations**: Requires node/patch management (mitigated with EKS managed node groups and Karpenter)

#### Choose EC2 when:
- Large, steady-state workload — Reserved Instances make EC2 significantly cheaper than Fargate
- GPU workloads (ML inference, video encoding) — Fargate doesn't support GPU instances
- Workloads needing very fast container startup (no image pull cold start)
- High-memory or compute-intensive workloads where large instance types are more cost-effective
- Spot instances for batch/background workloads with interruption tolerance (up to 90% savings)
- Need for DaemonSets in EKS (not supported on Fargate) — monitoring agents, log collectors

#### EC2 best practices for containers:
- **EKS Managed Node Groups**: AWS handles node updates and replacement — much less operational burden
- **Karpenter**: Provision right-sized nodes on demand; consolidate and terminate idle nodes — better than Cluster Autoscaler
- **Mixed instance types**: Use Karpenter or ASG with mixed policies to combine Spot and On-Demand
- **Reserved Instances**: Commit to 1–3 years for baseline capacity; Spot for burst

---

## Decision Matrix

| Criterion | Lambda | ECS | EKS |
|-----------|--------|-----|-----|
| Execution model | Event-driven, short-lived | Long-running containers | Long-running pods (Kubernetes) |
| Max runtime | 15 minutes | Unlimited | Unlimited |
| State | Stateless | Stateless or stateful (EFS/EBS) | Stateless or stateful |
| Team expertise | Low (managed) | Medium (Docker) | High (Kubernetes) |
| Operational complexity | Very low | Low-Medium | Medium-High |
| Scaling | Instant, automatic | ASG / App Auto Scaling | Karpenter + HPA/KEDA |
| Cold starts | Yes (mitigated with Provisioned Concurrency) | Low (pre-warmed instances) | Low (pre-warmed nodes) |
| Cost model | Pay per invocation + duration | Pay per task/node | Pay per pod/node + cluster fee |
| Best for | Startups, event pipelines, glue code | Small-medium containers, AWS-native | Enterprise microservices platform |
| Multi-cloud portability | No | No | Yes (Kubernetes) |
| GPU support | No | Yes (EC2) | Yes (EC2) |

| Criterion | Fargate | EC2 |
|-----------|---------|-----|
| Operational overhead | None | Medium (mitigated with managed node groups) |
| Cost (steady, large) | Higher | Lower (Reserved + Spot) |
| Cost (variable/spiky) | Lower (pay per use) | Higher (idle capacity) |
| Startup time | Slightly slower | Faster (pre-warmed) |
| Security isolation | Stronger (micro-VM) | Shared kernel on node |
| GPU support | No | Yes |
| Best for | Variable workloads, small teams, getting started | Large stable workloads, cost optimization, GPU |

---

## Architecture Combination Patterns

### Pattern 1: Lambda (API layer) + ECS (background workers)
**Use case**: Synchronous API calls handled by Lambda (auto-scaling, no idle cost), long-running or CPU-intensive background processing on ECS Fargate.
**Integration**: Lambda writes jobs to SQS; ECS workers poll SQS and process asynchronously.
**Example**: Image upload API (Lambda) → SQS → ECS task for image processing and ML inference.

### Pattern 2: EKS (platform) + Lambda (event glue)
**Use case**: Core microservices on EKS for complex, long-lived workloads. Lambda for lightweight event-driven integrations (S3 triggers, EventBridge rules, Kinesis consumers).
**Example**: Enterprise platform on EKS, with Lambda handling notification dispatch and data transformation pipelines.

### Pattern 3: ECS Fargate (services) + EC2 Spot (batch)
**Use case**: Customer-facing services on Fargate (reliability, simplicity). Batch/analytics workloads on EC2 Spot for massive cost savings.
**Integration**: ECS supports mixed capacity providers — define a strategy with Fargate as base and EC2 Spot for burst.

### Common anti-patterns
- **Lambda for long-running, CPU-intensive tasks**: Timeout and cost issues. Move to ECS/EKS.
- **EKS for a small team with 3 services**: Kubernetes overhead isn't justified. Use ECS.
- **Fargate for large steady-state 24/7 workloads**: EC2 Reserved is significantly cheaper at scale.
- **Ignoring Karpenter**: Cluster Autoscaler is slower and less cost-efficient. Karpenter should be the default for new EKS clusters.

# DevOps & Cloud Engineering Mock Interview Q&A

This repository contains a comprehensive summary of a senior-level DevOps technical interview. The topics cover production incident response, Kubernetes operations, CI/CD security, and Cloud Infrastructure automation.

---

## 📋 Table of Contents
- [Round 1: Kubernetes Operations & Zero-Downtime Deployments](#round-1-kubernetes-operations--zero-downtime-deployments)
- [Round 2: CI/CD Pipelines, GitOps, & Observability](#round-2-cicd-pipelines-gitops--observability)
- [Round 3: Infrastructure as Code (Terraform) & Cloud Security (AWS IRSA)](#round-3-infrastructure-as-code-terraform--cloud-security-aws-irsa)

---

## Round 1: Kubernetes Operations & Zero-Downtime Deployments

### Q1: What is the difference between `livenessProbe`, `readinessProbe`, and `startupProbe`? What happens when they fail during a deployment rollout?

**Answer:**
* **Startup Probe:** Checks if the application inside the container has fully initialized. While the startup probe is running, both `livenessProbe` and `readinessProbe` are disabled to prevent slow-starting applications from being killed prematurely.
* **Readiness Probe:** Checks if the application is ready to accept HTTP traffic (e.g., database connection pool initialized, cache loaded).
  * *On Failure:* The Pod IP is immediately removed from the Kubernetes Service `Endpoints` object and Ingress target groups. The Pod receives **0% traffic**, but the container is **not restarted**. The deployment rollout pauses while remaining healthy pods handle traffic.
* **Liveness Probe:** Checks if the application container process is healthy and actively running (e.g., detecting deadlocks or frozen processes).
  * *On Failure:* The `kubelet` kills the container process and restarts it according to its `restartPolicy`. If it fails repeatedly, the Pod enters a `CrashLoopBackOff` state and halts the deployment.

---

### Q2: How do you configure a Deployment strategy using `maxSurge` and `maxUnavailable` to guarantee zero-downtime rollouts?

**Answer:**
To guarantee zero capacity reduction and zero packet loss during a deployment rollout, configure the `RollingUpdate` strategy as follows:

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Creates new v2 pods FIRST before terminating old v1 pods
      maxUnavailable: 0    # Guarantees that active capacity never drops below 100% of target replicas
maxSurge: 25%: Forces Kubernetes to provision new, updated Pods first and verify their readiness before terminating old Pods.

maxUnavailable: 0: Ensures that active serving capacity never drops below the desired replica count during the update.

Q3: How do you prevent HTTP 502/504 errors for active in-flight requests when a Pod is terminated during a rolling update?
Answer:
When a Pod is terminated, a race condition occurs: the API server sends a SIGTERM signal to the container at the exact same millisecond it initiates Pod IP removal from load balancer target groups. Network propagation for load balancer deregistration takes several seconds, leading to HTTP 502/504 errors.

Solution: Use a preStop lifecycle hook alongside terminationGracePeriodSeconds:

YAML
spec:
  terminationGracePeriodSeconds: 45
  containers:
  - name: my-app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 15"]
preStop (sleep 15): Pauses container shutdown for 15 seconds, allowing Cloud Load Balancers/Ingress controllers to deregister the Pod IP from active target groups.

SIGTERM Handling: After preStop finishes, SIGTERM is sent to the process to stop accepting new requests and complete active, in-flight work.

terminationGracePeriodSeconds: Serves as a hard deadline (e.g., 45s) ensuring preStop and graceful application drain complete before SIGKILL is issued.

Round 2: CI/CD Pipelines, GitOps, & Observability
Q4: How does ArgoCD handle syncs and rollbacks in a GitOps workflow? Why is git revert preferred over manual UI rollbacks?
Answer:

ArgoCD Sync Mechanics: ArgoCD continuously monitors the Git repository (desired state) and compares it against the live Kubernetes cluster state. If autoSync and selfHeal are enabled, any drift in the cluster is automatically overwritten to match Git.

Rollbacks (git revert vs. UI): Manual UI rollbacks temporarily sync the cluster to a previous commit. However, if Git remains pointing to the bad commit, ArgoCD will mark the application Out-of-Sync or overwrite the manual rollback during the next auto-sync cycle. Executing git revert creates a new Git commit, preserving Git as the immutable single source of truth.

Q5: How does Horizontal Pod Autoscaler (HPA) scale Pods using custom application metrics (e.g., HTTP requests per second)?
Answer:
The custom metrics auto-scaling pipeline follows a 4-step pull-based flow:

Application (OpenTelemetry) 
Scraped by

​
 Prometheus Server 
Queried by

​
 Prometheus Adapter 
Exposed to

​
 HPA Controller
Metrics Scraping: The application exposes metrics via an OpenTelemetry /metrics endpoint, which is scraped by Prometheus Server.

Prometheus Adapter: Installed in the cluster, it executes PromQL queries against Prometheus Server and registers metrics under the custom.metrics.k8s.io API.

HPA Evaluation: The HPA Controller polls the custom metrics API every 15 seconds. If traffic exceeds the configured threshold (e.g., >60 req/sec per pod), HPA calculates and updates the replicas field on the Deployment to scale out Pods.

Round 3: Infrastructure as Code (Terraform) & Cloud Security (AWS IRSA)
Q6: How do you handle a crashed terraform apply job that leaves a residual lock in DynamoDB and potential state drift?
Answer:

Verify Process Status: Inspect CI/CD runner processes and check AWS CloudTrail logs for active API calls under the execution role to confirm no background terraform process is actively modifying infrastructure.

Release State Lock: Forcefully clear the lock using the lock ID from the error message:

Bash
terraform force-unlock <LOCK_ID>
Reconcile State Drift:

Run terraform refresh or terraform plan to compare real-world infrastructure against the state file.

If resources were partially created in AWS but omitted from state, use terraform import <resource_address> <aws_id> to import them.

Apply missing infrastructure using targeted execution: terraform apply -target=<resource_name>.

Q7: How does IAM Roles for Service Accounts (IRSA) work under the hood in AWS EKS? How does it integrate with the Secrets Store CSI Driver?
Answer:
IRSA provides keyless, short-lived authentication for EKS Pods accessing AWS resources:

[ Pod Container ] ──(OIDC Token)──> [ AWS STS ] ──(Temp Credentials)──> [ AWS Secrets Manager ]
Core Configuration:

An IAM OIDC Identity Provider is configured for the EKS cluster.

An IAM Role is created with a Trust Policy restricted to the Kubernetes ServiceAccount via OIDC sub claims.

The ServiceAccount is annotated with eks.amazonaws.com/role-arn.

Runtime Execution:

The EKS Mutating Webhook injects a projected OIDC JWT token into /var/run/secrets/kubernetes.io/serviceaccount/token and sets environment variables (AWS_ROLE_ARN).

The AWS SDK / Secrets Store CSI Driver sends the JWT token to AWS STS via AssumeRoleWithWebIdentity.

STS validates the token against the OIDC provider and issues short-lived AWS credentials.

Secrets Store CSI Driver Integration: The CSI driver uses these temporary STS credentials to fetch secrets from AWS Secrets Manager and mounts them as an ephemeral, in-memory tmpfs volume inside the Pod container.


Question: Multi-Region Disaster Recovery (RTO/RPO & Active-Passive Architecture)
You are designing the disaster recovery strategy for a mission-critical financial API running on AWS (EKS, Aurora PostgreSQL, and S3). The business requires an RPO (Recovery Point Objective) < 1 minute and an RTO (Recovery Time Objective) < 15 minutes.

Database & Storage Layer: How do you configure AWS Aurora PostgreSQL and Amazon S3 across a primary region (e.g., us-east-1) and a secondary DR region (e.g., us-west-2) to meet the sub-1-minute RPO?


1. AWS Aurora PostgreSQL: Use Aurora Global Database
Instead of setting up manual standby PostgreSQL instances or logical replication (which adds overhead), use Amazon Aurora Global Database:

How it Works:
Instead of saying generic "cross-region replication for Aurora," specifically name Amazon Aurora Global Database. Standard RDS uses logical replication (which is slower), whereas Aurora Global Database uses dedicated physical storage-level replication across AWS regions to keep latency under 1 second.

RPO Metrics: Cross-region replication latency is typically under 1 second, easily satisfying your sub-1-minute RPO requirement.

Failover Mechanics: If us-east-1 experiences a catastrophic outage, you can promote the us-west-2 secondary cluster to read-write mode in less than 1 minute (unplanned failover), with zero data loss for committed transactions.

2. Amazon S3: Use S3 Cross-Region Replication (CRR)
To replicate object storage (logs, media, assets, backups) from us-east-1 to us-west-2:

How it Works: Enable S3 Cross-Region Replication (CRR) on the source bucket. CRR requires S3 Versioning to be enabled on both source and destination buckets.

Sub-1-Minute RPO Guarantee (S3 RTC): Standard CRR replicates objects asynchronously, usually within a few minutes. To strictly enforce your sub-1-minute RPO SLA, enable S3 Replication Time Control (S3 RTC).

S3 RTC SLA: AWS guarantees that 99.9% of objects are replicated within 15 minutes (and typically within seconds for standard file sizes), backed by a Service Level Agreement (SLA) and CloudWatch metrics (ReplicationLatency) to monitor RPO drift in real-time.

Question:-How do you configure AWS Route 53 (Health Checks, Routing Policies like Failover vs. Geolocation) to automatically or semi-automatically shift live user traffic from us-east-1 to us-west-2 during a regional outage?
How AWS Route 53 Failover Works Under the Hood
1. Route 53 Failover Routing Policy
You create a Failover Routing Policy in Route 53 for your domain (e.g., api.yourcompany.com):

Primary Record: Points to the ALB / Ingress Controller IP in us-east-1.

Secondary Record: Points to the ALB / Ingress Controller IP in us-west-2.

2. Route 53 Health Checks
You configure a Route 53 Health Check that monitors an endpoint on your primary region (e.g., [https://api-us-east-1.yourcompany.com/healthz](https://api-us-east-1.yourcompany.com/healthz)).

In-Flight Validation: The health check probes your application's /healthz route (which checks database connectivity, core pod health, and ingress responsiveness).

3. The Automatic Failover Sequence
Outage Occurs: us-east-1 suffers a major infrastructure or regional failure.

Health Check Fails: Route 53 Health Check fails 3 consecutive times (typically within 30 seconds).

DNS Failover Executed: Route 53 automatically stops returning the IP address of us-east-1 in DNS lookup queries and begins returning the IP address of the secondary ingress in us-west-2.
When a user types your URL (e.g., api.company.com) into a browser, their computer doesn't know which IP address that domain belongs to. It has to ask a DNS Server (like AWS Route 53) to translate that domain name into a numeric IP address.

To save time and prevent asking Route 53 every single second, the user's browser, computer, and Internet Service Provider (ISP) save (cache) that IP address locally for a set amount of time.

That saved duration is controlled by the TTL (Time to Live).

The Problem with High TTL During an Outage
Imagine your primary region (us-east-1) IP is 1.1.1.1 and your TTL is set to 24 hours (86,400 seconds):

Normal State: A user visits your app. Their browser asks Route 53, gets 1.1.1.1, and remembers: "Send all traffic for api.company.com to 1.1.1.1 for the next 24 hours."

Outage Occurs: us-east-1 goes down. Route 53 detects the failure within 30 seconds and changes the domain's record to point to your DR region (us-west-2) at 2.2.2.2.

The Cache Trap: Even though Route 53 updated its records in 30 seconds, your user's browser is still using the saved 1.1.1.1 IP address because their 24-hour TTL timer hasn't expired!

Result: The user experiences an outage for hours until their local cache expires and requests a fresh IP from Route 53.

Why "Low TTL Enforcement" Solves This
By setting your DNS record's TTL to 60 seconds or lower (e.g., 10–60 seconds):

You force browsers, operating systems, and ISPs to discard their saved IP address every 60 seconds and ask Route 53 for a fresh lookup.

During a Failover: When Route 53 switches traffic from us-east-1 (1.1.1.1) to us-west-2 (2.2.2.2), clients will pick up the new, healthy IP address within 60 seconds or less, minimizing downtime for live users.

Low TTL Enforcement: To prevent client-side DNS caching from delaying failover, set the DNS record TTL (Time to Live) to 60 seconds or lower.

Pro Tip for Senior Interviews: Route 53 ARC (Application Recovery Controller)
To sound like an enterprise architect, mention Route 53 Application Recovery Controller (ARC) Routing Controls:

"For critical financial systems, fully automated DNS failovers can sometimes trigger false positives if a simple health check flap occurs. To prevent split-brain scenarios, we use Route 53 Application Recovery Controller (ARC) Routing Controls. ARC uses manual or automated safety-check rules (routing control switches) to safely execute cross-region traffic shifting without relying solely on simple HTTP health checks."

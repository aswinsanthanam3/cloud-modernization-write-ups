# ADR-2025-001: AWS Lambda vs EKS Automode

*Architecture Decision Record*

**When and How to Use AWS Lambda Functions Over the Default EKS Automode Compute Platform**

-----

|Field         |Value                                           |
|--------------|------------------------------------------------|
|**Status**    |ACCEPTED                                        |
|**Date**      |March 10, 2026                                  |
|**Authors**   |Platform Engineering Team                       |
|**Reviewers** |Engineering Leadership, SRE, Security           |
|**Supersedes**|None                                            |
|**Tags**      |compute, serverless, lambda, eks, infrastructure|

-----

## 1. Context

Our company has standardized on AWS EKS Automode as the default compute platform for all production workloads. EKS Automode manages node lifecycle automatically, provisions capacity on demand, and provides a consistent Kubernetes-native operational model across teams. This ADR defines the bounded contexts under which AWS Lambda functions are a better architectural choice, and governs how teams should evaluate, adopt, and operate Lambda where EKS Automode is the default.

This decision is necessary because Lambda and EKS Automode represent fundamentally different execution models with distinct tradeoffs across latency, throughput, concurrency limits, and observability. Without a clear policy, teams risk defaulting to Lambda for workloads better served by EKS, or avoiding Lambda for tasks where it is demonstrably superior.

-----

## 2. Decision Drivers

- **Operational consistency:** EKS Automode is the default. Lambda adoption must be intentional and justified.
- **Latency SLOs:** Customer-facing services often require sub-50ms p99 response times that Lambda cold starts can violate.
- **Cost efficiency:** Lambda’s pay-per-invocation model is economically superior for sporadic, low-frequency workloads.
- **Concurrency and scale:** Lambda has hard per-region concurrency soft limits and a burst ramp rate that can bottleneck sudden traffic spikes.
- **Observability requirements:** Our SRE team mandates distributed tracing continuity, structured logs, and Prometheus-compatible metrics. Lambda requires additional tooling to meet this bar.
- **Developer experience:** Lambda lowers the barrier for event-driven integrations without requiring teams to manage Kubernetes manifests.

-----

## 3. Decision

AWS Lambda is an approved compute option as an **exception** to the EKS Automode default, subject to the criteria defined in this document. Lambda **MUST** be used only when the workload characteristics below are met, and teams **MUST NOT** migrate existing EKS-hosted services to Lambda without formal re-evaluation against these criteria.

### Decision Matrix

|Workload Characteristic|Use Lambda                        |Use EKS Automode                |Rationale                                              |
|-----------------------|----------------------------------|--------------------------------|-------------------------------------------------------|
|Latency                |< 200ms p99 acceptable            |< 50ms p99 required             |Cold starts preclude Lambda for sub-50ms SLOs          |
|Duration               |< 15 min, bursty                  |Long-running (> 15 min)         |Lambda 15-min hard timeout                             |
|Invocation pattern     |Sporadic / event-driven           |Steady, high-concurrency        |EKS handles sustained load more cost-effectively       |
|Throughput             |Up to ~10K TPS burst              |Sustained high TPS              |Lambda TPS = min(10x concurrency, concurrency/duration)|
|Concurrency            |Default 1000 / region (soft limit)|Unlimited (node-bound)          |EKS scales without per-region soft limits              |
|Observability          |Acceptable with ADOT/X-Ray        |Full OTEL stack, Prometheus     |EKS allows richer, lower-overhead telemetry            |
|Execution environment  |Stateless, ephemeral              |Stateful, persistent connections|Lambda cannot maintain open DB connections natively    |
|Deployment complexity  |Simple, single-function           |Complex multi-service workload  |Lambda has lower ops overhead for isolated tasks       |
|Cost model             |Pay-per-invocation, low baseline  |Running cost, always-on         |Lambda cheaper at low invocation rates                 |

-----

## 4. Detailed Factor Analysis

### Factor Comparison Summary

|Factor                |Lambda                                                                                                                  |EKS Automode                                                                                                             |Winner                                                                      |
|----------------------|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
|**Latency (p99)**     |1.2–7s cold start (Java: up to 7s). Warm: 5–50ms                                                                        |Container startup < 5s once; warm p99 < 10ms consistently                                                                |EKS (latency-sensitive) / Lambda (tolerable cold starts OK)                 |
|**Throughput**        |Scales to ~10K TPS per account per region. TPS = concurrency / duration                                                 |No hard TPS limit; HPA scales linearly                                                                                   |EKS for sustained throughput; Lambda for bursty spikes                      |
|**Concurrency Limits**|Default 1000 concurrent executions / region (soft limit). Burst limit: 1000 new envs / 10 sec                           |No Lambda-style concurrency cap; bounded by node capacity which autoscales                                               |EKS — Lambda limits require quota increase requests and burst ramp-up delays|
|**Observability**     |CloudWatch + X-Ray. Ephemeral nature complicates trace continuity. Log tailing latency common. No infrastructure metrics|Full Prometheus/Grafana stack. OpenTelemetry first-class. Persistent sidecar agents. Infrastructure + app metrics unified|EKS — richer, more predictable observability pipeline                       |

-----

### 4.1 Latency

Lambda cold start latency is the most significant barrier to use in latency-sensitive paths. Measured p95 cold start times by runtime (2026 benchmarks) are:

- **Node.js v20:** 1.2–2.8 seconds cold, ~120ms warm
- **Python 3.12:** 2.1–3.5 seconds cold
- **Java 21 (standard):** 4–7 seconds cold; SnapStart reduces this to 90–140ms
- **ARM64 (Graviton2):** 45–65% reduction across all runtimes

EKS Automode pod startup is a one-time cost per deployment event. Once running, container response times are consistent sub-10ms for warm paths, without the per-invocation cold start risk. For workloads with p99 SLOs below 50ms, Lambda is disqualified unless Provisioned Concurrency is configured — which substantially increases cost and does not eliminate cold starts beyond the reserved pool.

> **Rule:** Lambda SHOULD NOT be used for synchronous, customer-facing APIs with p99 SLO requirements below 200ms unless SnapStart or Provisioned Concurrency is explicitly configured, reviewed by Platform Engineering, and the cost impact is accepted.

-----

### 4.2 Throughput

Lambda throughput is governed by the formula:

```
TPS = min(10 × concurrency, concurrency / function_duration_in_seconds)
```

With the default concurrency limit of 1,000 and a 100ms function duration, an account can sustain approximately 10,000 TPS. For high-throughput, low-latency functions this is generally sufficient for bursty workloads.

However, Lambda throughput degrades predictably with longer function durations. A function averaging 1 second per invocation is limited to 1,000 TPS at the default concurrency limit. EKS pods face no analogous concurrency-per-invocation constraint — horizontal pod autoscaling (HPA) can distribute load across an unbounded number of replicas.

> **Rule:** Lambda is appropriate for throughput requirements up to approximately 5,000–10,000 TPS with short-duration functions. Sustained throughput above these thresholds SHOULD use EKS Automode unless a quota increase has been pre-approved and load tested.

-----

### 4.3 Concurrency Limits

AWS Lambda enforces three distinct concurrency-related limits in every region:

- **Account-level concurrent execution limit:** 1,000 by default (soft limit, increasable via AWS Support).
- **Burst limit:** 1,000 new execution environments per 10 seconds. Traffic spikes exceeding this ramp rate will be throttled, returning HTTP 429 (`TooManyRequestsException`) to callers.
- **Reserved concurrency:** When set, acts as a hard cap per function and can inadvertently starve other functions if misconfigured.

EKS Automode has no analogous concurrency cap. Kubernetes HPA scales pods based on CPU/memory/custom metrics and is bounded only by available node capacity, which Automode provisions automatically. There is no burst ramp rate — pods can scale from zero to cluster maximum within the time it takes to schedule and start containers.

> **Rule:** Teams relying on Lambda for sudden, large-scale traffic spikes (e.g., a flash sale, viral event, or batch trigger) MUST account for the burst ramp rate and pre-warm via Provisioned Concurrency or request a concurrency limit increase well in advance.

-----

### 4.4 Observability

Lambda’s ephemeral, stateless execution model creates structural observability challenges that diverge from our standard EKS observability stack:

- **Trace continuity:** Each Lambda invocation is isolated. Cold starts and execution environment recycling can break distributed trace propagation unless OpenTelemetry context is explicitly propagated across all invocation boundaries.
- **Log latency:** CloudWatch Logs ingestion is not real-time. During high-load incidents, log processing delays have been observed causing Lambda functions using blocking log drivers to fail their health checks.
- **No sidecar pattern:** Lambda does not support persistent sidecar processes. This prevents the Datadog Agent, Fluent Bit, or Prometheus exporters from running as co-located processes. AWS Distro for OpenTelemetry (ADOT) Lambda layers provide a workaround but add cold start overhead.
- **Infrastructure metrics gap:** Lambda provides no access to underlying infrastructure metrics (CPU steal, network I/O, host saturation). EKS exposes full node and pod-level metrics via Prometheus.

> **Rule:** Teams adopting Lambda MUST configure ADOT or an equivalent OpenTelemetry layer, and MUST ensure trace context propagation is validated end-to-end during acceptance testing. Lambda workloads are NOT exempt from our distributed tracing SLO requirements.

-----

## 5. Real-World Lambda Incidents and Outages

The following publicly documented incidents illustrate production risks associated with AWS Lambda and directly inform the guardrails in this ADR.

-----

### Incident 1: AWS Lambda Outage, US-EAST-1 — June 13, 2023

**Sources:** [AWS Post-Event Summary](https://aws.amazon.com/message/061323/) · [ThousandEyes Outage Analysis](https://www.thousandeyes.com/blog/aws-outage-analysis-june-13-2023)

Starting at 11:49 AM PDT, customers experienced increased error rates and latencies for Lambda function invocations in us-east-1. The root cause was a failure in Lambda’s internal capacity management subsystem. Over 104 AWS services were impacted, including API Gateway, AWS Management Console, EKS, Amazon Connect, and EventBridge. The outage lasted approximately 3 hours and 48 minutes. Downstream customers including The Boston Globe, the New York MTA, and The Associated Press experienced full or partial service outages.

**Relevance to this ADR:** Demonstrates that Lambda is not isolated from AWS-wide control plane failures and that its cellular architecture can amplify blast radius when capacity subsystems fail. Workloads with hard availability SLOs should not be exclusively Lambda-dependent.

-----

### Incident 2: Amazon Kinesis Failure Cascades to Lambda — July 30, 2024

**Sources:** [AWS Post-Event Summary](https://aws.amazon.com/premiumsupport/technology/pes/) · [StatusGator AWS Outage History](https://statusgator.com/blog/aws-outage-history/)

A routine Kinesis architecture upgrade exposed a flaw in handling a large number of low-throughput shards, causing Kinesis’ cell management system to misinterpret host health and trigger an overload in shard redistribution. This cascaded to Lambda: customers were unable to view real-time logs, and Lambda functions using blocking log drivers failed with errors. The outage lasted nearly seven hours.

**Relevance to this ADR:** Directly validates Observability Risk identified in Section 4.4 — CloudWatch Logs disruption silences Lambda observability at exactly the moment it is most needed. EKS deployments using sidecar-based log forwarding are insulated from this failure mode.

-----

### Incident 3: AWS US-EAST-1 DynamoDB DNS Cascade — October 19, 2025

**Sources:** [Medium RCA by Leela Kumili](https://medium.com/@leela.kumili/aws-outage-root-cause-analysis-bd88ffcab160) · [The New Stack Analysis](https://thenewstack.io/a-cascade-of-failures-a-breakdown-of-the-massive-aws-outage/)

A race condition in DynamoDB’s internal DNS management layer caused valid DNS records to be deleted, triggering a 14+ hour service disruption. Lambda invocation failures spiked significantly as EC2 control-plane requests were delayed. The incident cascaded across EC2, Lambda, NLB, ECS/EKS, and dozens of downstream services.

**Relevance to this ADR:** Lambda, as a higher-level managed service, inherits failure modes from multiple AWS primitives (DNS, control plane, IAM, CloudWatch). EKS workloads with pre-warmed node pools and local service discovery are more insulated from control-plane DNS failures than Lambda, which must resolve cold-start environments on every new invocation.

-----

### Incident 4: Lambda as Root Cause of Multi-Service Degradation — June 2023

**Source:** [Network Computing — “Amazon Outage: AWS Lambda Function Issue Causes East Coast Service Outages”](https://www.networkcomputing.com/network-security/amazon-outage-aws-lambda-function-issue-causes-east-coast-service-outages)

Following the June 2023 Lambda outage, industry analysts noted that Lambda’s growing role as a backbone for other AWS services created a cascading failure pattern. Two in three companies had adopted Lambda functions at the time, meaning a Lambda control-plane failure had disproportionate blast radius versus historical EC2-level outages. The analysis concluded that unlike previous cloud outages caused by configuration errors or network misroutes, this incident may have been caused by capacity exhaustion — a risk directly connected to Lambda’s account-level concurrency model.

**Relevance to this ADR:** Reinforces why concurrency limits must be understood and planned for before relying on Lambda in any critical path, and why EKS Automode is the safer default for high-availability workloads.

-----

### Incident 5: Kinesis Outage Causes Lambda Log Blindness — July 2024

**Source:** [StatusGator AWS Outage History](https://statusgator.com/blog/aws-outage-history/)

The July 2024 Kinesis outage resulted in CloudWatch Logs suffering delayed log processing, which in turn caused Lambda logs to go missing during the incident. Lambda functions with blocking log drivers failed, and ECS tasks were terminated due to failed health checks. This occurred across CloudWatch Logs, Amazon Data Firehose, Amazon S3 event notifications, AWS Lambda, ECS, Amazon Redshift, and AWS Glue.

**Relevance to this ADR:** Lambda’s tight coupling to CloudWatch for observability creates a single point of failure for runtime visibility — one that does not exist in EKS deployments using independent sidecar-based log forwarding. This incident is a direct real-world case for the observability requirements mandated in Section 4.4.

-----

## 6. Consequences

### Positive

- Teams have a clear, consistent framework for choosing Lambda, reducing ad hoc or tribal decisions.
- Lambda workloads are bounded to use cases where its economics and simplicity genuinely win.
- Observability and concurrency requirements prevent Lambda from being used as a silent failure point in critical paths.
- Event-driven integrations (S3 triggers, SQS consumers, scheduled tasks) benefit from Lambda’s native event source mapping without adding EKS complexity.

### Negative / Risks

- Additional approval overhead for Lambda adoption may slow iteration for teams with genuinely appropriate use cases.
- Teams must invest in ADOT or equivalent observability tooling to meet the Lambda observability requirements defined here.
- Provisioned Concurrency costs may increase Lambda’s total cost of ownership for latency-sensitive workloads, partially closing the cost gap with EKS.

-----

## 7. Compliance Criteria

A Lambda workload is **compliant** with this ADR if it meets ALL of the following:

1. It is event-driven, sporadic, or scheduled — not a synchronous, persistent service.
1. Its p99 latency SLO is either above 200ms, or Provisioned Concurrency / SnapStart is configured and documented.
1. Its expected peak TPS has been calculated using `TPS = min(10 × concurrency, concurrency / duration)` and does not exceed the current concurrency limit.
1. OpenTelemetry (via ADOT or equivalent) is configured, and end-to-end trace continuity has been validated in a staging environment.
1. The workload has no persistent state, long-lived connections, or stateful memory requirements.
1. A Platform Engineering review is completed before deploying Lambda in any critical-path or customer-facing flow.

-----

## 8. Alternatives Considered

### Alternative A: Lambda as Default (Rejected)

Using Lambda as the default compute option was rejected because it inverts the operational model. Lambda requires bespoke observability tooling, has hard concurrency and TPS ceilings, and is more tightly coupled to AWS managed service failures (as evidenced by the outage history in Section 5). EKS Automode provides a more predictable, portable, and observable foundation for the majority of workloads.

### Alternative B: Prohibit Lambda Entirely (Rejected)

A blanket prohibition on Lambda was rejected because Lambda provides genuine economic and operational advantages for event-driven workloads with low invocation frequency, short durations, and bursty patterns. Requiring all such workloads to run on EKS would increase cost and operational overhead for teams building lightweight integrations, scheduled jobs, or infrastructure automation.

### Alternative C: AWS Fargate (Deferred)

Fargate was evaluated as a middle ground — container-based but serverless. However, Fargate does not integrate with EKS Automode’s node management and costs more than EKS Automode at scale. It was deferred for future evaluation in the context of workloads that need container isolation without Kubernetes overhead.

-----

## 9. References

- AWS Post-Event Summary: Lambda US-EAST-1, June 13, 2023 — https://aws.amazon.com/message/061323/
- ThousandEyes: AWS Outage Analysis June 13, 2023 — https://www.thousandeyes.com/blog/aws-outage-analysis-june-13-2023
- StatusGator: History of AWS Outages — https://statusgator.com/blog/aws-outage-history/
- Data Center Dynamics: AWS US-EAST-1 Lambda Outage — https://www.datacenterdynamics.com/en/news/aws-us-east-1-lamda-outage-causes-issues-globally/
- Network Computing: Amazon Outage Lambda Root Cause — https://www.networkcomputing.com/network-security/amazon-outage-aws-lambda-function-issue-causes-east-coast-service-outages
- The New Stack: Cascade of Failures AWS Outage — https://thenewstack.io/a-cascade-of-failures-a-breakdown-of-the-massive-aws-outage/
- AWS Compute Blog: Understanding Lambda Invoke Throttle Limits — https://aws.amazon.com/blogs/compute/understanding-aws-lambdas-invoke-throttle-limits/
- InfoQ: The Right Way of Tracing AWS Lambda Functions — https://www.infoq.com/articles/tracing-aws-lambda-functions/
- Grafana Labs: AWS Lambda OpenTelemetry Serverless Observability — https://grafana.com/blog/aws-lambda-opentelemetry-and-grafana-cloud-a-guide-to-serverless-observability-considerations/
- AgileSoftLabs: AWS Lambda Cold Start 7 Proven Fixes 2026 — https://www.agilesoftlabs.com/blog/2026/02/aws-lambda-cold-start-7-proven-fixes
- AWS re:Post: Resolve Lambda Throttling Errors — https://repost.aws/knowledge-center/lambda-troubleshoot-throttling

-----

## Sign-off

|Role                     |Signature|Date|
|-------------------------|---------|----|
|Platform Engineering Lead|         |    |
|SRE Lead                 |         |    |
|Engineering VP           |         |    |

-----

*INTERNAL USE ONLY — Platform Engineering*
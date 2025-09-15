# **Infrastructure Design Scenario**

## Objective

_Define a repeatable consistent architecture for the facilitation of a production application as well as defining the necessary tooling and workflow processes for its successful deployment_

## The Applications

- A front-end application
- Varied architectures including a backend API, async handlers, a message bus, and background workers.
  - An assumption of the application's usage being a payments process as well as an email notification service to facilitate some of the requirements around the varied architectures.
- An Internal Admin tool only available to the support team.

## The Tools

- **Github Actions**
  - Handles Builds, Tests and Triggers ArgoCD Deployments
- **EKS**
- **Docker**
- **Helm**
- **ArgoCD**
  - Deployed in the management account
- **Argo Rollouts**
  - Used to facilitate canary deployments

## AWS Accounts and Access Model

The environment will use four AWS accounts under a single AWS Organization:

- **Management Account** – Centralized governance, billing, and global policy enforcement.

- **Test Account** – Used for PR Env testing and Dev Deploys.

- **Pre-prod Account** – Staging environment mirroring production for performance and release validation.

- **Production Account** – Isolated, least-privilege environment for live workloads.

This access model provides granular control of user onboarding and offboarding by integrating with IAM Identity Center (AWS SSO) or external identity providers such as Okta. Permission sets can be centrally defined and assigned per account, reducing operational overhead while maintaining strict boundaries.

From the Management Account, global governance is enforced using Service Control Policies (SCPs) and Resource Control Policies (RCPs) to apply baseline security and compliance standards consistently across all accounts.

## AWS Account Layers

A secure, multi-account AWS architecture with management and application accounts. The management account provides centralized identity, governance, and shared networking via Transit Gateway and VPN, while application accounts host EKS workloads, isolated load balancers, CloudFront distribution, and core managed services, all provisioned through Terraform.

### Network and Compute Layer

<img src="https://github.com/dblqr/arch/blob/main/files/arch-Network-Architecture.png?raw=true">

- A VPC with three public and private subnets distributed across multiple Availability Zones.
- An External Load Balancer in front of the public subnets for internet-facing services, and Internal Load Balancer in front of the private subnets for internal-only access.
- An EKS cluster deployed only in the private subnets.
- CloudFront is used to expose external applications, providing TLS termination, caching, and global edge distribution for frontend traffic.
- Internal access to applications behind the internal load balancer provided through a VPN (e.g., Tailscale) that connects via network access from the management account.
- A dedicated NAT Gateway per AZ to avoid the single point of failure created by a shared NAT Gateway across all AZs.
- S3, DynamoDB, SQS, OpenSearch (or similar), and an RDS with a replica integrated with the Payments API for persistence, messaging, and relational workloads.

All services run in private subnets across three Availability Zones. The NAT Gateways provide controlled outbound internet access for EKS nodes and workloads, while VPC Interface Endpoints enable private connectivity to AWS APIs. This design ensures internet-facing services remain isolated and internal systems communicate securely.

The base AWS account infrastructure is created in terraform and is hosted in a repository that is responsible for the deployment of the shared infrastructure as well as the payments application infrastructure.

### Application Layer

<img src="https://github.com/dblqr/arch/blob/main/files/Kubernetes-Traffic.png?raw=true">

External users access the system through CloudFront which acts as a CDN serving static assets from S3 and routes dynamic requests to the public ALB.

All traffic goes through an ALB which forwards to an HTTP gateway exposed via an ingress controller in EKS on port 443. External users come through the public internet facing ALB. Internal users go through an internal ALB which is routable through a VPN.

Although the diagram does not show multiple instances, each deployment should contain multiple replicas spread out across multiple AZ’s with podAntiAffinity.

All applications running in EKS must have a readiness and liveness check to allow for self healing in the event of an application error.

Each application should, if necessary, have an IAM role + policy which makes short lived AWS credentials available to pods utilizing the AWS SDK.

The applications will follow the [12 factor pattern](https://12factor.net/) which allows for configuration via environment variables which are mounted via ConfigMaps.

### The Management Account Layer

<img src="https://github.com/dblqr/arch/blob/main/files/arch-Multi-Account.png?raw=true">

The management account hosts a dedicated VPC with an EKS cluster and a hosted VPN for access for internal tools that has network access to the other VPCs in the test, pre-prod, and production accounts.

We run our instance ArgoCD from the management account and deploys are pushed out using the network access that the management account has. This gives us a single pane of glass as the management account produces deployments, while workload accounts’ clusters consume them.

Cross-account IAM roles in each workload account will be used for delegated access, ensuring centralized governance and streamlined user lifecycle management across the organization.

## Application Architecture

<img src="https://github.com/dblqr/arch/blob/main/files/arch-Application.png?raw=true">

External users access the system through CloudFront, which serves static frontend assets from S3 and routes dynamic requests to the public ALB. Traffic is then forwarded to the EKS cluster via the public ingress controller, where the Payments Web and Payments API services run.

Internal staff are able to connect over VPN into the internal ALB, which targets the internal ingress controller and exposes the Admin Web tool securely which has access to the payments API and web application.

When a payment is submitted by a user, the Payments API validates the request and records the transaction in DynamoDB (`status = processing`) archiving the raw payload to S3 and triggering its processing using eventbridge.

An `S3 -> SQS -> Lambda` pipeline then moves the work into background processing. The Lambda worker consumes messages from SQS, updates DynamoDB to mark the payment as complete, and then the payments service persists transactional data into RDS (with read replicas for scale and reporting), and indexes the transaction into OpenSearch for fast search and reporting.

If the Lambda Function fails at any point, retry logic is built into the system so the message stays in the queue until re-processed. After max attempts it goes to a DLQ for later replay.

SES also handles email notifications such as receipts or alerts when the process completes or fails.

## Other Architectural Considerations

### Isolation

- Workloads have their own namespace to avoid sharing secrets and to allow for implementing simple network policies.
- Workloads can utilize mTLS to ensure that both the client and server in the request agree on who they are sending/receiving data from.
- RBAC permissions will be set up to limit the scope of service accounts, users, and applications.

### Disaster recovery

- All stateful datastores use AWS snapshots or AWS backup for DR
- Kubernetes cluster state is defined in GitOps and stateless services can be restored into any new cluster.
- All S3 Buckets replicate data to a replication bucket and versioning is enabled on all objects.

### Redundancy

- Pod anti affinities can be used to ensure that workloads are spread across multiple AZ’s.
- Node groups and tolerations could be used to ensure that secure/unsecure workloads do not run on the same infrastructure.
- Look at circuit breaking in service mesh (if in use) for resilience
  https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/
- The production database will have a multi AZ set up with a replica to allow for redundancy or multi region if necessary.

## Secrets Management in EKS

All secrets will be stored in a repository using [SOPS](https://github.com/getsops/sops) which encrypts the data using [AWS KMS keys](https://github.com/getsops/sops?tab=readme-ov-file#using-sops-yaml-conf-to-select-kms-pgp-and-age-for-new-files) per environment. The secrets are created in terraform and only Admin accounts will be able to decrypt and add SOPS secrets for production.

The secrets being located in AWS Secrets Manager, enables you to use open source tooling such as External Secrets Operator, or read them at the application runtime.

For added security, the application could reach directly to Secrets Manager to fetch secrets at runtime, using an IRSA role and a service account. This pattern allows for applications secrets to be pulled into memory versus the applications container environment.

## The Deployment Process

**<center>_Build -> Test -> Deploy -> Observe -> Promote_</center>**

<img src="https://github.com/dblqr/arch/blob/main/files/arch-CI_CD Process.png?raw=true">

For local development, the developers have the ability to run the application stack locally either using docker or docker compose. Unit and functional tests can run locally against the application.

When a PR is created, a temporary environment spins up with app dependencies and infra dependencies, running integration and E2E tests against that created environment. This allows developers the ability to integrate with other services like the full run of Payments API process without touching upper environments.

The application on every PR and commit, a build pipeline will run making sure that the docker image builds successfully

```yaml
name: build
on:
  pull_request:
    paths:
      - "src/**"

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

env:
  APP_REPO_NAME: "us-east1-docker.pkg.dev/app-12345/app/app"

permissions:
  contents: read
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@08c6903cd8c0fde910a37f88322edcfb5dd907a8 # ratchet:actions/checkout@v5
        with:
          fetch-depth: 0

      - name: Authenticate to Google Cloud
        id: auth
        uses: google-github-actions/auth@7c6bc770dae815cd3e89ee6cdf493a5fab2cc093 # ratchet:google-github-actions/auth@v3
        with:
          token_format: access_token
          workload_identity_provider: "projects/12345678910/locations/global/workloadIdentityPools/github-com/providers/github-com-oidc"
          service_account: "app-repo@app-repo.iam.gserviceaccount.com"

      - name: Login to GAR
        uses: docker/login-action@184bdaa0721073962dff0199f1fb9940f07167d1 # ratchet:docker/login-action@v3
        with:
          registry: us-east1-docker.pkg.dev
          username: oauth2accesstoken
          password: ${{ steps.auth.outputs.access_token }}

      - name: Set up QEMU
        uses: docker/setup-qemu-action@29109295f81e9208d7d86ff1c6c12d2833863392 # ratchet:docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@e468171a9de216ec08956ac3ada2f0791b6bd435 # ratchet:docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@263435318d21b8e681c14492fe198d362a7d2c83 # ratchet:docker/build-push-action@v6
        with:
          push: false
          context: src/app
          cache-from: type=registry,ref=${{ env.METRICS_SERVER_REPO_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.METRICS_SERVER_REPO_NAME }}:buildcache,mode=max
```

Once approved and tested, PRs are queued to merge into main sequentially using a merge queue. Once the merge takes place a deploy occurs to the test environment and the PR env is spun down. Once the application is available smoke and E2E tests run.

If dev tests pass, the build is promoted to pre-prod for smoke, E2E, performance, and regression testing.

Only after all tests pass and approved, the build is promoted to production. The deployment uses a canary strategy with automated rollback if analysis detects issues based upon specified metrics.>

## Deployments With Helm Charts and ArgoCD

<img src="https://github.com/dblqr/arch/blob/main/files/arch-ArgoCD.png?raw=true">

To facilitate the deployment of the application we are utilizing application helm charts which live alongside the application and are also treated as a versioned artifact.

When a build and release for the application takes place, the helm chart is versioned and released and pushed up to a chart repository (ECR in the example). This allows use to promote the same artifact through the entire CI/CD pipeline.

From there, the CI/CD deployment process triggers an API call to ArgoCD to deploy the specific version of the helm chart which contains the released application. The deployment targets the specific environment based on where the deployment process is in the merge queue.

As a part of the deployment process we will also utilize pre/post hooks for certain actions like database migrations or running post scripts.

## Canary Releases using ArgoCD Rollouts

<img src="https://github.com/dblqr/arch/blob/main/files/arch-ArgoCD Rollout.png?raw=true">

To facilitate a canary release with rollbacks in Production or other environments we will utilize ArgoCD Rollouts. Due to the usage of ArgoCD for our deployment mechanism, using rollouts is a logical next step to expand the functionality of `release -> deploy -> observe` to expand the base functionality of ArgoCD and Kubernetes ability to repair and keep applications healthy.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: example-rollout
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.15.4
          ports:
            - containerPort: 80
  minReadySeconds: 30
  revisionHistoryLimit: 3
  strategy:
    canary: # Indicates that the rollout should use the Canary strategy
      maxSurge: "25%"
      maxUnavailable: 0
      steps:
        - setWeight: 25 # sets 25% of traffic over 30 minutes
        - pause:
            duration: 30m
        - setWeight: 50 # sets 50% of traffic over 15 minutes
        - pause:
            duration: 15m
        - setWeight: 75 # sets 75% traffic over 5 minutes
        - pause:
            duration: 5m
```

In this example rollout, we are deploying 10 replicas of Nginx using a canary strategy. The new versions are introduced gradually by shifting traffic to the canary deployment in stages:

- 25% for 30 minutes
- 50% for 15 minutes
- 75% for 5 minutes

Once all gates have been met successfully, the rollout continues the deployment.

To measure whether a gate is successful you utilize an analysis template to measure its success.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: app-dd-slo
spec:
  metrics:
    - name: p95_latency
      interval: 1m
      successCondition: result < 0.35 # requests in seconds of duration
      failureLimit: 1
      provider:
        datadog:
          query: |
            avg:trace.http.request.duration{service:app,env:prod}.rollup(p95)
    - name: error_rate
      interval: 1m
      successCondition: result < 0.05 # < 5% errors
      failureLimit: 1
      provider:
        datadog:
          query: |
            sum:trace.http.error{service:app,env:prod}.rollup(sum) /
            sum:trace.http.requests{service:app,env:prod}.rollup(sum)
```

For example, by using a Datadog provider to gather metrics, we can monitor the rollout of the application by checking the latency of the application or other actionable metrics as well like error rates, before moving to the next gate.

Rollouts can also be expanded to smoke testing for usage in lower environments vs using ArgoCD post sync hooks.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: app-smoke-tests
spec:
  provider:
    job:
      spec:
        template:
          spec:
            restartPolicy: Never
            containers:
              - name: check
                image: curlimages/curl:8
                args: ["-sfS", "http://app-canary.prod.svc/healthz"]
```

## The Observability Layer

Throughout the entire application release process, as well as at an AWS account and infrastructure level, observability, monitoring and alerting are baked into the release lifecycle.

### Application Logging and Monitoring

#### Application Monitoring

Throughout the applications lifecycle we want to monitor for specific KPI's, for example:

- **Latency**
  - How long requests take.
- **Error rate**
  - How often they fail.
- **Throughput**
  - How many requests are processed.
- **Saturation**
  - How close the system is to resource limits.

These metrics are collected from many sources across the stack like EKS, ALBs, RDS, DynamoDB, SQS, etc. and should be aggregated in monitoring and alerting tools like Cloudwatch, Datadog or visualization tools like Grafana.

#### Application Logs

We would be collecting and aggregating application logs from many different sources like application logs/container logs from EKS, load balancer logs from the ALBS or K8s Ingresses, or CloudFront logs centralized via CloudWatch Logs or a log pipeline to Datadog or similar.

Alerts and monitors should be set up for actionable remediation alongside runbooks versus causing alert fatigue.

As an example, we _should_ alert on if there is increased latency in the application or if we have an anomalous increase in 5XX errors.

### Infrastructure Monitoring and Observability

To allow for confidence in all our integrated systems we need to make sure that for all of our different types of shared infrastructure we have valid and actionable monitoring and alerting to go to the right teams and owners that can take action upon them.

- **Access logging**

  - Actively monitoring IAM Identity center logins.
  - GuardDuty or a cloud SIEM of your choice is enabled in all environments for intrusion detection.
  - Alerting on failed AWS API attempts.
  - Root AWS user activity monitored and not in use.
  - 2FA enabled for the root user.

- **Network monitoring**

  - WAF set up at network access layers.
  - VPC flow logs enabled and monitored.
  - Forwarding Cloudfront logs and monitoring for increases in errors or failed access attempts.

- **AWS Infrastructure**

  - **EKS**
    - Monitoring and alerting set up for all EKS clusters.
    - Application logs are exported to Datadog or similar like ELK.
    - K8s audit logging is enabled with alerts set for failed access to internal resources like secrets and logs.
    - Alerting on failed attempts of pod or service account creation.
    - Monitoring on node health and ready states.
    - Pod scheduling failures.
    - Monitoring on application and shared resource pressure (CPU, memory, disk).
  - **RDS instances**
    - RDS Instance monitoring and alerting for CPU and Memory utilization.
    - Is storage autoscaling enabled for our RDS instances with monitors for these events.
    - Alerting on replication lag.
    - Alerts and monitors for max connections and long running queries.
  - **S3**
    - S3 access logs and eventbridge logging set up.
    - Policies in place for bucket visibility i.e public vs private buckets.
    - Object lifecycle policies are set up.
    - Bucket objects KMS encrypted.
  - **SQS**
    - Alerts set up for Queue length i.e `ApproximateNumberOfMessagesVisible`.
    - Alerts on thresholds for the Oldest message in the queue.
    - Monitoring and alerting on if messages are in the DLQ.
  - **DynamoDB**
    - Monitoring latency and throttling events on read/write events.
  - **Lambda**
    - Forwarding Lambda logs and invocation event logs.
    - Monitoring and alerting set up for invocation errors and throttling.
    - Baseline alerts set up for Lambda execution durations.

### Deployment Monitoring

Throughout the CI/CD process observability should be baked into the build and deployment lifecycles.

#### Builds

- If a build fails in CI/CD it should be blocked as a part of required repository checks.
- Tests should also follow the same pattern.
- Failed build logs should be triaged for patterns if failures are persistent and prioritized for fixes.

#### Deployments

- Github actions pipelines should alert the specific applications alert channel if the release process fails or if a deployment fails.
- ArgoCD post syncs will alert specific channels on success of deployments or sync failures.
- If E2E tests fail the applications alert channel should also be alerted.
- If an ArgoCD Rollout fails, that failure should also alert the requisite channels.

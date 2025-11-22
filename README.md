DevOps Design Take-Home Assignment

This repository contains the design documentation for a secure, scalable, and highly available web application architecture deployed on Amazon Web Services (AWS), along with a modern CI/CD pipeline.

The design covers the following components:

UI (Frontend) Architecture: Secure, scalable hosting for a stateless frontend application (e.g., React/Vue).

API (Backend) Architecture: Scalable, secure deployment for a containerized backend service (e.g., Python/Node.js).

Database Architecture: Resilient, high-availability relational database (PostgreSQL).

CI/CD Pipeline: Automated workflow using Jenkins, GitHub, and Argo CD.

1. UI (Frontend) Architecture Design

Diagram Reference: 01 PA2.pdf

The initial diagram in 01 PA2.pdf suggests using an ALB + EC2/Auto Scaling Group setup. For a purely static, stateless UI (like a built React application), a Serverless CDN-based approach offers better performance, resilience, and cost-effectiveness. The revised design focuses on the standard AWS S3 + CloudFront pattern.

Component

Explanation

Route 53

Primary DNS service. Uses a simple A record aliased to the CloudFront distribution.

Amazon S3

Hosting. The compiled, static UI assets (HTML, CSS, JS) are stored in an S3 bucket. The bucket is configured as a static website host (or configured for Object Lock if necessary).

CloudFront

CDN & Caching. Serves as the Content Delivery Network and Origin Shield. It caches the static content at Edge Locations globally, drastically reducing latency and load on the S3 bucket.

Origin Access Control (OAC)

Security. Ensures that CloudFront is the only service that can access the S3 bucket, preventing direct public access to the raw files (S3 is not public, only CloudFront is).

Application Load Balancer (ALB)

Not used in this design. The static UI is served directly via CloudFront.

Scaling Approach for UI

Horizontal Scaling: Handled inherently by CloudFront. By serving content from globally distributed Edge Locations, the system can handle massive spikes in traffic without scaling the origin infrastructure (S3).

Availability Zones: S3 and CloudFront are highly distributed global services by nature, providing multi-AZ and multi-region resilience for static asset delivery.

Traffic Routing: Route 53 directs users to the nearest CloudFront Edge Location, which serves the cached content.

Maintenance: Deployments are near-instantaneous. New static assets are uploaded to S3, and a CloudFront Invalidation is triggered to clear the old cache, forcing the CDN to fetch the new version immediately.

2. API (Backend) Architecture Design

Diagram Reference: 02 PA2.pdf

The design uses Amazon API Gateway to handle routing and request throttling before passing traffic to a highly scalable containerized service running on AWS Fargate.

Component

Explanation

Amazon API Gateway

Traffic Entry & Routing. Acts as the single entry point. It provides request validation, throttling, and routing capabilities based on URL paths (/service 1, /service 2, etc.).

Amazon ECS (Fargate)

Where API Runs & Horizontal Scaling. The backend service is containerized (e.g., Docker) and deployed using AWS Fargate (Serverless compute for containers). Fargate eliminates the need to manage EC2 instances, simplifying the operation.

Auto Scaling

Scaling. ECS Service Auto Scaling monitors metrics (like CPU Utilization, Request Count) and automatically increases (or decreases) the number of Fargate tasks (containers) running the API.

VPC & Private Subnets

Security. Fargate tasks run within private subnets, ensuring they are not directly accessible from the public internet. All traffic must pass through the API Gateway.

AWS Secrets Manager

Secrets Management. Used to securely store sensitive data (e.g., Database credentials, third-party API keys). Fargate tasks are granted permission via an IAM Role to retrieve secrets at runtime, preventing hardcoded secrets in containers.

Secure Service-to-Database Communication

The Fargate tasks are in a Private Subnet.

The RDS database is also in a Private Subnet.

Communication is secured using Security Groups, where the RDS Security Group only allows inbound traffic (PostgreSQL port 5432) from the Security Group associated with the Fargate tasks. This ensures VPC-internal, private, and firewall-protected communication.

3. Database Architecture Design

Diagram Reference: 03 PH2.pdf

The design utilizes Amazon RDS for PostgreSQL in a Multi-AZ configuration to ensure high availability and resilience.

Feature

Explanation

High Availability (HA)

Multi-AZ Deployment. The database instance runs in a primary Availability Zone (AZ 'a'), with an automatically provisioned synchronous standby replica in a second AZ (AZ 'b'). In case of a failure in AZ 'a', RDS performs an automatic failover to the standby in AZ 'b'.

Read Scaling

Read Replicas. Separate dedicated Read Replicas are deployed in additional AZs. The application will be configured to route read-heavy traffic to these replicas, reducing the load on the primary writer instance.

Vertical Scaling

Instance Type Adjustment. When CPU, memory, or IOPS exceed acceptable thresholds, the instance class (e.g., upgrading from db.t3.medium to db.r6g.large) can be modified via a single RDS command. This requires a brief downtime during the application of the change.

Backup Plan

Automated Snapshots. RDS performs daily automated snapshots, retained for 7 days.

Disaster Recovery

Point-in-Time Recovery (PITR). Enabled by Binary Logging (WAL), allowing recovery to any specific second within the retention window (e.g., up to 7 days).

Migration Strategy

Schema Versioning. We use a tool like Liquibase or Flyway to manage database schema changes. Migration scripts are version-controlled, and the CI/CD pipeline runs the required schema migrations before deploying the new API version. This ensures the database is compatible with the new code version.

4. CI/CD Pipeline Design

Diagram Reference: 04 PH2.pdf

The CI/CD pipeline is designed for a containerized application, utilizing Jenkins for orchestration (CI) and Argo CD for GitOps-based continuous deployment (CD).

Stage

Tool

Description & Health Check

1. Trigger

GitHub Webhook

Triggered by a Pull Request (PR) merge into the main branch (for deployment) or a push to the feature branch (for test runs).

2. Code Quality & Security

Jenkins + SonarQube

Runs static code analysis (SAST) and unit tests. PRs are blocked if the Quality Gate fails.

3. Container Build & Scan

Jenkins + Docker + Aqua Trivy

Builds the Docker image based on the new source code. Aqua Trivy scans the final image for OS and application dependencies vulnerabilities. If a critical vulnerability is found, the pipeline stops (NO in the diagram), and a REPORT email notification is sent.

4. Artifact Registry

Amazon ECR

On successful build and scan, the new, immutable Docker image is tagged and pushed to the Elastic Container Registry (ECR).

5. Deployment (CD)

Argo CD

Argo CD monitors the Git repository's deployment manifest (GitOps). A successful image push triggers an update to the image tag in the Git manifest, and Argo CD automatically syncs the new version to the target environment.

6. Health Checks & Verification

Prometheus & Grafana

After deployment, Prometheus monitors the /health endpoint of the new service. If the service returns a 200 OK within a grace period, the deployment is considered successful. Argo CD can manage rollout strategies like Blue/Green or Canary deployments for zero-downtime updates.

Promotion Strategy (Dev → Stage → Prod)

The promotion strategy follows a controlled flow to ensure stability:

Develop (Dev): Every merge to main deploys automatically to the Dev environment (full CI/CD flow).

Staging (Stage): A successful Dev deployment is automatically tagged. A manual approval step in Jenkins is required to promote the same artifact (ECR image tag) to the Staging environment.

Production (Prod): After successful integration and UAT (User Acceptance Testing) in Staging, a second, highly restricted Manual Approval is required. Argo CD then syncs the same, tested artifact to the Production environment. This ensures artifact immutability across all environments.

5. System Load Handling

The entire architecture is designed to handle high load by focusing on distributing compute and serving static content globally.

Component

Load Handling Mechanism

Frontend

CloudFront absorbs the vast majority of requests and traffic spikes via global caching, drastically reducing load on the origin (S3).

API Entry

API Gateway provides throttling and caching layers, protecting the downstream Fargate tasks from uncontrolled burst traffic.

Backend API

ECS Fargate Auto Scaling dynamically adds or removes containers in seconds, ensuring linear scaling of the compute layer to match the request volume.

Database

Multi-AZ ensures immediate failover. Read Replicas offload read-heavy traffic from the Primary Writer, allowing the system to scale reads independently of writes.

System Resiliency

Running components across multiple Availability Zones protects the system against single data-center failures.

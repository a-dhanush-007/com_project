# **1. UI (Frontend) Architecture**

UI Frontend Architecture – Explanation

The frontend UI is deployed in an AWS VPC across multiple Availability Zones to ensure high availability. The UI files (HTML, CSS, JS) are stored in Amazon S3, which provides a durable and scalable storage layer.
Traffic from users is routed through an Internet Gateway → VPC → ALB, which distributes traffic across multiple EC2 instances in an Auto Scaling Group.

The Auto Scaling Group ensures horizontal scaling, automatically increasing or decreasing EC2 instances based on traffic.
S3 is used for static assets and can be optionally combined with a CDN like CloudFront to improve global performance.

### **How the UI is hosted**

The UI (React/Angular/Vue) is hosted in:

* **Amazon S3** static website hosting
* Distributed globally using **Amazon CloudFront (CDN)**

### **How horizontal scaling works**

* S3 is automatically scalable
* CloudFront caches static assets at 300+ edge locations
* No servers needed → completely stateless + auto-scale 

### **How traffic is routed**

```
User → Route 53 → CloudFront → S3 (Static Content) OR Internet Gateway → VPC → ALB
```

### **How CDN improves performance**

* Caches JS, CSS, images, videos
* Reduces latency
* Reduces S3 load
* Provides DDoS protection (via AWS Shield)

### **How Availability Zones are used**

* Static Content are stored redundantly across **multiple AZs** within S3
* CloudFront edge locations provide global high availability
* The UI remains available even if one AZ fails

---

# **2. API (Backend) Architecture **

### **Where the API runs**

The backend API runs on:

* **AWS ECS Fargate** (serverless containers)
  OR
* **AWS EC2 Auto Scaling Group**

### **How horizontal scaling works**

* ECS Fargate auto-scales based on:

  * CPU usage
  * Memory usage
  * Request count
* Tasks spread across multiple private subnets in different AZs

### **How secrets are stored securely**

* **AWS Secrets Manager** stores:

  * DB credentials
  * API keys
  * Tokens
* ECS retrieves secrets via **IAM Task Role**
* No hardcoding of secrets

### **How API communicates with Database**

* ECS → RDS via private subnet (no internet exposure)
* Strict **security group rules** (only backend SG → DB SG)
* TLS-encrypted database connections

### **How traffic enters the backend**

Two options:

* **Application Load Balancer (ALB)** → ECS
* **API Gateway** → Lambda/ECS

Typical path:

```
User → CloudFront → ALB → ECS Backend → RDS
```

---

# **3. Database Architecture **

### **DB technology**

* **Amazon RDS (PostgreSQL/MySQL/MariaDB)**

---

### **How scaling is handled**

#### **1. Read Replicas (Horizontal Scaling)**

* Multiple read replicas handle read-heavy workloads
* Application routes read traffic to replicas
* Primary DB handles writes

#### **2. Vertical Scaling**

* Increase instance size (CPU/RAM) anytime
* Switch to high IOPS storage (io1/io2)

#### **3. Multi-AZ Design (High Availability)**

* Primary in **AZ-1**, synchronous standby in **AZ-2**
* Automatic failover if primary goes down
* No manual intervention required

---

### **Backup Plan**

* Automated daily snapshots
* Manual snapshots for major releases
* Multi-day retention (7–35 days)
* PITR (Point-In-Time Recovery) enabled

---

### **Migration Strategy**

* Version-controlled schema using **Flyway** or **Liquibase**
* Migrations run automatically in CI/CD pipeline
* Rollback supported using reversible migrations
* Staging DB used for testing new schema before production rollout

---

# **4. CI/CD Pipeline (GitHub Actions/Jenkins) **

### **Trigger Events**

* **Push to `dev` branch** → deploy to Dev
* **Pull Request to `main`** → run tests
* **Merge to `main`** → deploy to Staging
* **Manual approval** → deploy to Prod

---

### **Build Stage (concept only)**

#### **Frontend**

* `npm install`
* `npm run build`
* Compress & prepare frontend bundle

#### **Backend**

* Install dependencies
* Build application
* Build Docker image (concept only)

---

### **Testing Workflow**

* Unit tests (frontend/backend)
* Integration tests against test API
* SonarQube static analysis (code quality)
* Trivy container scan (vulnerabilities)

If any step fails → pipeline stops.

---

### **Deployment Steps**

#### **Frontend**

1. Upload build artifacts to **S3**
2. Invalidate **CloudFront** cache
3. Update versioning tag

#### **Backend**

1. Push Docker image → **AWS ECR**
2. ArgoCD pulls new image
3. Deploy to **AWS EKS** (or ECS)
4. Rolling update ensures zero downtime
5. Auto rollback on failure

---

### **Basic Health Checks**

* ALB health checks for ECS tasks
* `/health` endpoint for API
* CloudWatch alarms (CPU, memory, errors)
* Kubernetes liveness & readiness probes

---

### **Promotion Strategy**

```
DEV → STAGING → PRODUCTION
```

* Dev: automatic deployment
* Staging: tested by QA
* Prod: manual approval required
* ArgoCD syncs changes automatically


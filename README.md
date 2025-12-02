# aws-architecture-design
1. Overview
This document presents the complete architecture design for a three-tier application deployed on AWS, along with a secure, automated CI/CD pipeline.
The system consists of:
•	Frontend (UI) – hosted on S3 and served via CloudFront
•	Backend (API) – containerized services running on ECS Fargate
•	Database – Amazon RDS PostgreSQL with Multi-AZ high availability
•	CI/CD Pipeline – GitHub Actions workflow for automated build, test, and deployment
The architecture is scalable, fault-tolerant, secure, and aligned with AWS best practices.

2.Architecture Workflow 
2.1 User Request Workflow (Frontend Delivery)
1.	The user opens the application in a browser.
2.	The domain is resolved by Amazon Route 53.
3.	Route 53 sends the request to CloudFront.
4.	CloudFront checks for cached UI content at its edge location:
o	If cached → response is returned instantly.
o	If NOT cached → CloudFront fetches UI files from the S3 bucket using OAI/OAC.
5.	The user receives the static frontend with low latency and high availability.

2.2 API Workflow (Backend Processing)
After the UI loads:
1.	The browser triggers API calls to the backend URL (e.g., /api/login).
2.	The request enters the Application Load Balancer (ALB) located in the public subnet.
3.	ALB forwards the request to ECS Fargate Tasks running in private subnets.
4.	Fargate processes:
o	authentication
o	backend logic
o	data validation
o	data modification
Security:
•	Backend tasks run without public IPs
•	Only ALB can communicate with ECS tasks
•	API endpoints are not exposed directly to the internet

2.3 Database Workflow (RDS PostgreSQL)
1.	ECS tasks retrieve database credentials from AWS Secrets Manager.
2.	Backend securely connects to RDS PostgreSQL inside the private data subnet.
3.	Primary DB handles all reads/writes.
4.	Data is synchronously replicated to the standby DB in the second Availability Zone.
________________________________________
3. Component Design & Implementation
3.1 UI (Frontend) Architecture
Strategy
Static website hosting using Amazon S3 with global delivery through CloudFront CDN.
Hosting
•	UI build artifacts (HTML, CSS, JavaScript) stored in a private S3 bucket.
•	Bucket access restricted using Origin Access Identity (OAI/OAC).
Routing & Caching
•	CloudFront caches static files at edge locations for low-latency content delivery.
•	Route 53 routes domain traffic to the CloudFront distribution.
Scaling & Availability
•	S3 supports unlimited storage and auto-scaling traffic.
•	CloudFront offloads global traffic through its edge network.
•	Highly durable and highly available by design (11 nines durability for S3).
________________________________________
3.2 API (Backend) Architecture
Strategy
Containerized backend services managed by ECS Fargate (serverless compute).
Traffic Entry
•	Public-facing Application Load Balancer (ALB) receives HTTPS traffic.
•	ALB routes traffic to ECS tasks running inside private subnets.
Compute & Scaling
•	ECS Fargate tasks automatically scale using Application Auto Scaling policies:
o	CPU > 70%
o	Memory > 75%
o	Requests per target threshold
Security & Networking
•	ECS tasks do not have public IPs.
•	Outbound traffic is routed through NAT Gateways.
•	Secrets like DB credentials and API keys stored in AWS Secrets Manager and injected at runtime.
•	IAM Task Roles ensure least-privilege access.
Database Connectivity
•	ECS connects securely to RDS PostgreSQL via a private endpoint.
•	Optional RDS Proxy for connection pooling and reliability.
________________________________________
3.3 Database Architecture (RDS PostgreSQL)
Strategy
Fully managed relational database with Multi-AZ high availability.
High Availability
•	Primary database in one Availability Zone.
•	Synchronous replication to Standby DB in second AZ.
•	Automatic failover ensures minimal downtime.
Scaling
•	Vertical scaling for increased CPU/memory.
•	Read Replicas (optional) used for read-heavy workloads such as analytics.
Backup & Recovery
•	Automated backups retained for 7–30 days.
•	Point-in-Time Recovery (PITR) enabled.
•	Manual snapshots for long-term retention.
Migration Strategy
•	Schema migrations handled via Flyway/Liquibase/Prisma.
•	Migrations executed as a pre-deployment task during CI/CD.
•	Ensures DB schema matches deployed application version.

3.4 Network Security & Isolation Strategy 
While subnets provide network separation, we implement Security Group Chaining to enforce strict traffic flow:
•	ALB Security Group: Allows Inbound HTTPS (443) from 0.0.0.0/0 (Internet).
•	ECS Task Security Group: Denies all direct internet access. Allows Inbound traffic on port 8080 only from the ALB Security Group ID.
•	RDS Security Group: Allows Inbound traffic on port 5432 only from the ECS Task Security Group ID.
Additional Protections:
•	AWS WAF (Web Application Firewall): Attached to CloudFront to block common exploits (SQL Injection, XSS) and rate-limit bad actors.
•	S3 Block Public Access: Enabled at the account level to prevent accidental data leaks.

3.5 Observability & Monitoring 
To ensure system reliability, we utilize the AWS CloudWatch suite:
•	Logging: ECS Fargate containers stream stdout/stderr logs directly to Amazon CloudWatch Logs via the awslogs driver.
•	Metrics & Alarms: We monitor Key Performance Indicators (KPIs) such as CPU Utilization, Memory Usage, and HTTP 5xx Error Rates. CloudWatch Alarms trigger SNS topics to alert the engineering team if thresholds are breached.
•	Distributed Tracing: AWS X-Ray is instrumented in the ECS tasks to visualize latency bottlenecks between the API and the Database.

________________________________________
4. CI/CD Pipeline Design
Tool: GitHub Actions
Environments: Development → Staging → Production
________________________________________
4.1 Trigger Events
Event	Action
Pull Request to main	CI (Lint + Tests)
Push to main	Deploy to Staging
Git Tag (v1.x.x)	Deploy to Production
________________________________________
4.2 CI Pipeline (Build & Test)
1.	Checkout source code
2.	Install dependencies
3.	Run unit tests
4.	Run linting / static analysis
5.	Build UI artifacts
6.	Build Docker image
7.	Security scan (ECR Image Scan)
________________________________________
4.3 Artifact Packaging
•	Tag Docker image with Git SHA
•	Push to Amazon ECR
•	Upload UI build to S3 (versioned files)
________________________________________
4.4 Deployment Pipeline (CD)
Staging Deployment
1.	Run database migrations (transient ECS task)
2.	Update ECS service with the new Docker image
3.	ECS performs rolling deployment:
o	Starts new tasks
o	ALB health checks
o	Shifts traffic
o	Drains old tasks
Production Deployment
•	Triggered via semantic version tag (e.g., v1.0.0)
•	Requires manual approval
•	Same flow as staging but deployed to Production cluster

5. Project Files This repository contains the following files:
•	Architecture Diagram.drawio.pdf: The visual diagram of the AWS architecture.
•	README.md: This document explaining the design.

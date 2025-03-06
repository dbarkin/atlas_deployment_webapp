# MongoDB Atlas Deployment Web Application

**Current Date:** March 05, 2025  
**Document Version:** Updated with IT Infrastructure Architect Feedback

This document outlines the functional, non-functional, and security requirements for a web application designed to deploy MongoDB Atlas clusters via Terraform, integrating user authentication, CI/CD pipelines, and robust infrastructure considerations.

---

## 1. Functional Requirements

### **User Authentication & Authorization**
- The system must authenticate users using **Google OAuth (Google IDP)**.
- Only authorized users should access the application.
- The application must support **Role-Based Access Control (RBAC)** with clear user roles and permissions defined at the FastAPI level.
- Authentication must include token refresh mechanisms and defined session duration policies.
- The system must handle expired tokens gracefully and implement secure token storage on the client (e.g., JWTs or database-based sessions).

### **Terraform Deployment via GitHub CI/CD**
- Users must be able to trigger the deployment of a **MongoDB Atlas cluster**.
- The backend must trigger a **GitHub Actions pipeline** to run Terraform scripts stored in a **corporate GitHub repository**.
- Terraform state must be managed securely using **Google Cloud Storage (GCS)** with **GCP-KMS** encryption and object versioning for recovery. State locking will be implemented via GCS lifecycle policies.
- GitHub Actions workflows must operate under a specific permissions model with manual approval by **one admin** for production Terraform runs.

### **Frontend (Next.js) Responsibilities**
- Implement user authentication via **NextAuth.js**.
- Securely store session tokens (JWT or database-based) using Vercel’s secure environment variables.
- Send API requests to the **FastAPI backend** with authentication tokens over HTTPS.
- Implement proper error handling for API failures (e.g., “Deployment failed—please try again or contact support”) and authentication issues.

### **Backend (FastAPI) Responsibilities**
- Validate OAuth tokens from Google using Google’s public keys.
- Implement **RBAC** to control access to the CI/CD trigger.
- Securely trigger **GitHub Actions** via API using GitHub webhooks for status notifications.
- Implement API rate limiting (e.g., **100 requests/minute per user**, adjustable via config) using Google Cloud API Gateway or FastAPI middleware (e.g., `slowapi`).
- Handle and log authentication failures using **Elasticsearch** for structured logging.

### **Deployment & Database Management**
- The system must support automated MongoDB Atlas deployment with multi-region clusters (e.g., `us-central1`, `us-east1`) across different Availability Zones.
- Terraform will handle cluster configuration and provisioning.
- Deployment logs will be captured in **Elasticsearch** and monitored via **Grafana** for troubleshooting.
- Implement rollback mechanisms for failed deployments and notify admins within **5 minutes** via GitHub webhooks.

---

## 2. Non-Functional Requirements

### **Security**
- OAuth token validation must be performed securely at the FastAPI backend.
- Only authorized users can access deployment functionalities, enforced via RBAC.
- API communication must use **HTTPS** and **CORS policies** configured in Google Cloud API Gateway.
- GitHub Actions must use **GitHub Secrets** for Terraform credentials with least privilege access and manual approval gates for production.
- Terraform state will be stored in **GCS** with **GCP-KMS** encryption and access controls.

### **Scalability & Performance**
- The system must handle multiple users securely, with **Vercel auto-scaling** for Next.js and **Cloud Run auto-scaling** for FastAPI (e.g., max 100 instances, 80 concurrent requests per instance).
- API requests will be optimized via Google Cloud API Gateway to reduce latency.
- Address bottlenecks in FastAPI by scaling based on CPU/memory usage.

### **Reliability & Maintainability**
- Comprehensive logging will use **ELK Stack (Elasticsearch, Logstash, Kibana)**, with monitoring via **Prometheus + Grafana**.
- Key metrics: **API error rate > 5%** triggers alerts; deployment failures notified within **5 minutes**.
- CI/CD pipelines will handle failures gracefully with rollback mechanisms.
- Detailed error handling will include exponential backoff (initial delay: **1s**, max retries: **3**).

### **Infrastructure & Hosting**
- **Frontend Hosting**: **Vercel on GCP** (serverless, auto-scaling).
- **Backend Hosting**: **GCP Cloud Run** (containerized FastAPI, auto-scaling up to 100 instances).
- **API Gateway**: **Google Cloud API Gateway** for rate limiting and authentication.
- **Environments**:
  - **Development**: Local Docker Compose with MongoDB Atlas free tier.
  - **Staging**: Mirror production with smaller instances (e.g., `e2-small`).
  - **Production**: Multi-AZ deployment (e.g., `us-central1-a`, `us-central1-b`) with reserved MongoDB Atlas capacity.
- **Network Boundaries**: VPC with private subnets for FastAPI, public access via API Gateway (port 443 ingress only).

---

## 3. Solution Options Considered

### **Option 1: Google OAuth with NextAuth.js (Frontend Authentication Only)**
- **Pros**: Easy integration, built-in session management.
- **Cons**: Requires backend security mechanisms.

### **Option 2: Firebase Authentication with Google Sign-In**
- **Pros**: Managed authentication.
- **Cons**: Dependency on Firebase.

### **Option 3: Google OAuth with FastAPI Backend**
- **Pros**: Full backend control.
- **Cons**: Increased token management effort.

### **Option 4: NextAuth.js + FastAPI for Secure API Calls**
- **Pros**: Clean separation, secure API communication.
- **Cons**: Requires token validation and API protection.

---

## 4. Selected Solution: Option 4 - NextAuth.js + FastAPI

### **Why Option 4?**
- Leverages **NextAuth.js** for seamless frontend authentication.
- Ensures **FastAPI validates OAuth tokens** and enforces RBAC.
- Integrates with **GitHub Actions** for Terraform automation, secured by **GCS** state storage.

---

## 5. High-Level Architecture Diagram

+----------------------+

| Next.js (Vercel/GCP) |

| - NextAuth.js       |

| - Google OAuth      |

| - HTTPS API Calls   |

+----------------------+

           |

           v

+----------------------+       +----------------------+

| GCP API Gateway      |<----->| FastAPI (Cloud Run)  |

| - Rate Limiting      |       | - Token Validation   |

| - Authentication     |       | - RBAC Enforcement   |

+----------------------+       | - GitHub API Trigger |

           |                   +----------------------+

           v                          |

+----------------------+              v

| GitHub Actions       |<----Terraform State (GCS)----

| - Terraform Scripts  |                               |

| - Manual Approval    |                               |

| - Structured Logs    |                               |

+----------------------+                               |

          |                                            |

          v                                            v

+----------------------+                    +-----------------------+

| MongoDB Atlas        |                    | ELK/Prometheus        |

| - Multi-Region/AZ    |                    | - Logs (Elasticsearch)|

| - 7-Day Backups      |                    | - Metrics (Grafana)   |

+----------------------+                    +-----------------------+

---

## 6. Security Considerations

### **OAuth Token Security**
- Use **Google OAuth JWTs**, validated at FastAPI with Google’s public keys.
- Implement token refresh and session duration policies.

### **RBAC & Access Control**
- Only authorized users trigger deployments, enforced at FastAPI.

### **API & Data Security**
- Secure APIs with **HTTPS** and **CORS** via Google Cloud API Gateway.
- Rate limiting set to **100 requests/minute per user**.

### **GitHub CI/CD Security**
- Use **GitHub Secrets** and manual approval by **one admin** for production.
- Terraform state in **GCS** with **GCP-KMS** encryption.

### **Logging & Monitoring**
- Audit logs in **Elasticsearch**; metrics (e.g., API latency) in **Prometheus + Grafana**.
- Alerts for **API error rate > 5%**.

---

## 7. Deployment Architecture

### **Environment Tiers**
- **Development**: Local Docker Compose + MongoDB Atlas free tier.
- **Staging**: Smaller instances (e.g., `e2-small`) mirroring production.
- **Production**: Multi-AZ (e.g., `us-central1-a`, `us-central1-b`) with reserved capacity.

### **Infrastructure Components**
- **Frontend**: Vercel on GCP.
- **Backend**: GCP Cloud Run.
- **API Gateway**: Google Cloud API Gateway.
- **Monitoring**: ELK Stack + Prometheus/Grafana.

### **Network Boundaries**
- VPC with private subnets for FastAPI, public access via API Gateway.

---

## 8. Error Handling & Resilience

- **API Failures**: Retry with exponential backoff (initial delay: **1s**, max retries: **3**).
- **Terraform Failures**: Rollback and notify via GitHub webhooks.
- **Authentication Failures**: Graceful handling with user guidance (e.g., “Deployment failed—please try again or contact support”).

---

## 9. Disaster Recovery

- **RTO**: 24 hours (consider reducing to 4-8 hours for critical systems).
- **RPO**: 15 minutes via MongoDB Atlas backups.
- **Replication**: Multi-region (e.g., `us-central1`, `us-east1`) across AZs.
- **Backups**: MongoDB Atlas automated with **7-day retention**.

---

## 10. Future Considerations
- **Notifications**: GitHub webhooks integrated with GCP Cloud Functions for status updates.
- **Scalability**: Support for other clouds (e.g., AWS) via Terraform providers.
- **Observability**: Add distributed tracing with **Jaeger** as complexity grows.

---

## Feedback Integration Notes
- Hosting clarified with **Vercel on GCP** and **GCP Cloud Run**.
- Disaster recovery enhanced with specific RTO/RPO and multi-region details.
- Monitoring specified with **ELK Stack** and **Prometheus + Grafana**.
- CI/CD security updated with **GCS** and **GCP-KMS**.
- Scalability detailed with Cloud Run auto-scaling parameters.


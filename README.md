# Highly Available Infra for 3-tier Micro-service Application

<img width="1041" height="1071" alt="ha_diagram drawio (1)" src="https://github.com/user-attachments/assets/882dd273-5edb-495e-9666-9a3eedd61fc2" />

## Explanation

### 1. VPC
> We used a **single /24 CIDR (`10.0.0.0/24`)** for the entire VPC because:
> - It gives exactly 256 IPs, enough to accommodate all required subnets without waste.
> - Keeps the design simple but still allows proper separation for public, frontend, backend, and database tiers.
> - The IP ranges are fully consumed by the planned subnets, leaving no unused space.

---

### 2. Subnets
> The VPC is split into **8 subnets** across **two AZs** for high availability.  
> The CIDR sizes were chosen based on the role and scaling needs of each tier:
>
> - **Public Subnets** → `/28` (14 usable IPs each)  
>   These only host ALB ENIs and NAT Gateway, so a small IP block is enough.
>
> - **Frontend Subnets** → `/27` (30 usable IPs each)  
>   These run the UI/web services and need moderate scaling room for EC2/ECS tasks.
>
> - **Backend Subnets** → `/26` (62 usable IPs each)  
>   This is the largest tier, running multiple microservices that require more IPs for horizontal scaling.
>
> - **Database Subnets** → `/28` (14 usable IPs each)  
>   These host RDS instances only, so IP usage is minimal.
>
> **Subnet-to-AZ mapping:**
> - **us-east-1a:** Public A, Frontend A, Backend A, DB A
> - **us-east-1b:** Public B, Frontend B, Backend B, DB B

---

### 3. Internet Gateway
> Placed at the VPC level to give **public subnets** internet access.  
> The public subnets are only used for inbound traffic termination at ALB and outbound NAT gateway traffic, not for hosting application workloads directly.

---

### 4. NAT Gateway
> Placed in **Public Subnet A** so private subnets can reach the internet without being exposed to it.  
> Chosen to avoid giving frontend/backend/database servers public IPs.

---

### 5. Application Load Balancer (ALB)
> Internet-facing ALB in public subnets distributes requests to **frontend** workloads in private subnets.  
> This design:
> - Keeps application servers private.
> - Allows scaling frontend independently from backend.
> - Supports multi-AZ failover.

---

### 6. Security Groups
> Configured to enforce **tier-to-tier traffic flow**:
> - ALB → Frontend
> - Frontend → Backend
> - Backend → Database  
> No tier is directly exposed to layers it shouldn’t talk to.

---

### 7. Network ACLs
> Subnet-level filtering is aligned with the security group rules for an extra layer of protection.  
> - Public NACL: Allows ALB and NAT traffic.
> - Frontend NACL: Only accepts from public ALB, only sends to backend.
> - Backend NACL: Only accepts from frontend, only sends to DB.
> - DB NACL: Only accepts from backend.

---

### 8. Databases
> Two private `/28` subnets in separate AZs are used for **RDS MySQL Multi-AZ**:
> - DB A: Primary instance
> - DB B: Standby  
> CIDR is small because DB layer only needs 2–3 IPs per AZ.

---

### 9. Three-Tier Design
> - **Frontend tier** in `/27` subnets to balance scaling needs with IP usage efficiency.
> - **Backend tier** in `/26` subnets as it’s the most IP-hungry layer due to multiple microservices and scaling replicas.
> - **Database tier** in `/28` subnets because it’s small, fixed capacity.

---

### 10. Data Flow
1. User hits Route 53 DNS → ALB in public subnets.
2. ALB routes to frontend workloads in `/27` private subnets.
3. Frontend communicates with backend workloads in `/26` subnets.
4. Backend queries RDS in `/28` DB subnets.
5. Outbound internet requests from private subnets go via NAT Gateway in Public Subnet A.

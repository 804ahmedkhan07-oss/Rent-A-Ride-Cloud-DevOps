# Task 16 — AWS Terraform, Private Kubernetes & Application Load Balancer

**Project:** Rent-A-Ride
**Environment:** AWS EC2, us-east-1 (N. Virginia)
**Branch:** feature/task16-aws-terraform

## Objective

Build a production-style AWS architecture using Terraform for the networking and EC2 infrastructure, run the Rent-A-Ride application inside a private EC2 instance using Kind Kubernetes, and expose the application to the Internet through a manually created AWS Application Load Balancer.

The final architecture separates public access, private application infrastructure, and administrative SSH access.

---

## 1. AWS Network Architecture

A dedicated VPC was created:

* VPC: `10.0.0.0/16`
* Availability Zone 1: `us-east-1a`
* Availability Zone 2: `us-east-1b`

### Subnets

| Subnet    | CIDR           | AZ         | Purpose         |
| --------- | -------------- | ---------- | --------------- |
| Public 1  | `10.0.1.0/24`  | us-east-1a | Bastion         |
| Public 2  | `10.0.2.0/24`  | us-east-1b | ALB             |
| Private 1 | `10.0.11.0/24` | us-east-1a | Private EC2     |
| Private 2 | `10.0.12.0/24` | us-east-1b | Private network |

### Internet Gateway

An Internet Gateway was attached to the VPC.

The public route table sends Internet traffic:

```text
Private/Public Resource
        |
        v
Route Table
        |
        v
Internet Gateway
        |
        v
Internet
```

The public subnets therefore have Internet connectivity.

---

## 2. NAT Gateway

A NAT Gateway was created inside the first public subnet.

The private route table sends outbound Internet traffic through the NAT Gateway:

```text
Private EC2
    |
    v
Private Route Table
    |
    v
NAT Gateway
    |
    v
Internet Gateway
    |
    v
Internet
```

### Why NAT?

The private EC2 does not have a public IP, but it still needs outbound Internet access for tasks such as:

* Installing packages
* Downloading Docker/Kubernetes tools
* Pulling container images
* Accessing external repositories

The NAT Gateway provides outbound access without making the private EC2 directly reachable from the Internet.

---

## 3. Security Groups

Two main Terraform-managed security groups were created.

### Bastion Security Group

Allows:

```text
Internet → Bastion :22
```

SSH access is allowed on port `22`.

The Bastion is located in the public subnet and has a public IP.

### Private EC2 Security Group

Allows:

```text
Bastion → Private EC2 :22
ALB → Private EC2 :80
```

The private EC2 does not have a public IP.

This creates the intended management path:

```text
Developer Laptop
      |
      v
Internet
      |
      v
Bastion EC2
      |
     SSH
      |
      v
Private EC2
```

---

## 4. EC2 Infrastructure

Two EC2 instances were used.

### Bastion EC2

Purpose:

* Public SSH entry point
* Jump host for administration
* Located in public subnet
* Has public IP

### Private EC2

Purpose:

* Runs Docker
* Runs Kind Kubernetes
* Hosts the Rent-A-Ride application

The private EC2 is located in:

```text
Private Subnet
10.0.11.0/24
```

It has **no public IP**.

---

## 5. Terraform Infrastructure

Terraform was used to create the AWS infrastructure.

Terraform manages:

* VPC
* Public/private subnets
* Internet Gateway
* Public/private route tables
* NAT Gateway
* Elastic IP for NAT
* Security groups
* Bastion EC2
* Private EC2

Basic workflow:

```bash
terraform init
terraform plan
terraform apply
```

Terraform state files and `.terraform/` were intentionally excluded from Git.

The Terraform configuration is stored in:

```text
task16-terraform/main.tf
```

---

## 6. Private EC2 — Docker, Kind & kubectl

The private EC2 was prepared with:

* Docker
* kubectl
* Kind
* Helm

Kind was used to create a Kubernetes cluster inside Docker.

Cluster:

```text
Kind Cluster: rent-a-ride
Kubernetes Node: kindest/node:v1.33.0
Kind Version: v0.30.0
```

The Kind cluster uses port mapping:

```text
EC2 :80
   |
   v
Kind Ingress :80
```

This allows the AWS ALB to forward HTTP traffic to the private EC2.

---

## 7. Rent-A-Ride Kubernetes Deployment

The existing Helm chart was deployed into:

```text
Namespace: rent-a-ride
```

The application contains:

```text
Frontend
Backend
MongoDB
```

Services:

```text
frontend-service :8080
backend-service  :3000
mongodb-service  :27017
```

The Helm chart was validated with:

```bash
helm lint ./rent-a-ride-chart
```

The application was successfully installed with Helm.

---

## 8. Kubernetes Ingress

The Kind cluster uses `ingress-nginx`.

Ingress:

```text
rent-a-ride-ingress
```

Traffic flow inside Kubernetes:

```text
HTTP Request
     |
     v
Ingress-Nginx
     |
     v
Frontend Service
     |
     v
Frontend Pod
```

The frontend communicates with the backend through the Kubernetes backend service.

Backend communicates with MongoDB through:

```text
mongodb-service :27017
```

---

## 9. Application Load Balancer

The AWS Application Load Balancer was intentionally created **manually**, not through Terraform, according to the Task 16 requirement.

ALB:

```text
task16-manual-alb
```

The ALB is:

* Internet-facing
* IPv4
* Deployed across both public subnets
* Listening on HTTP port `80`

Traffic flow:

```text
Internet
   |
   v
AWS Application Load Balancer :80
   |
   v
Target Group
   |
   v
Private EC2 :80
   |
   v
Kind Ingress
   |
   v
Rent-A-Ride Frontend
```

---

## 10. ALB Target Group

Target group:

```text
task16-manual-tg
```

Configuration:

* Target type: Instance
* Protocol: HTTP
* Port: `80`
* Health check path: `/`
* Health check protocol: HTTP
* Success code: `200`

Target:

```text
Private EC2
```

The target became:

```text
Healthy
```

This confirmed that the ALB could successfully reach the application running on the private EC2.

---

## 11. Final End-to-End Architecture

```text
                         INTERNET
                            |
                            v
                  +-------------------+
                  |   AWS ALB :80     |
                  |  Public Subnets   |
                  +---------+---------+
                            |
                            v
                  +-------------------+
                  |   Private EC2     |
                  |     :80           |
                  +---------+---------+
                            |
                            v
                  +-------------------+
                  |   Kind Kubernetes |
                  |  Ingress-Nginx    |
                  +---------+---------+
                            |
                            v
                    +---------------+
                    |    Frontend   |
                    +-------+-------+
                            |
                            v
                    +---------------+
                    |    Backend    |
                    |     :3000     |
                    +-------+-------+
                            |
                            v
                    +---------------+
                    |    MongoDB    |
                    |    :27017     |
                    +---------------+


        ADMINISTRATION / SSH PATH

        Developer Laptop
               |
               v
            Internet
               |
               v
        Bastion EC2
        Public Subnet
               |
              SSH
               |
               v
        Private EC2
```

---

## 12. Verification

The infrastructure and application were verified at multiple levels.

### Kubernetes

Verified:

* Frontend pods running
* Backend pod running
* MongoDB pod running
* Ingress-Nginx controller running
* Services available

### Private EC2

Application was tested locally:

```bash
curl -I http://localhost
```

Result:

```text
HTTP/1.1 200 OK
```

### AWS ALB

The public ALB DNS was tested:

```bash
curl -I http://task16-manual-alb-508971476.us-east-1.elb.amazonaws.com
```

Result:

```text
HTTP/1.1 200 OK
```

The application was also successfully opened through the browser.

---

## 13. Final Result

The Task 16 architecture successfully provides:

* Terraform-managed AWS networking
* Two Availability Zones
* Public and private subnets
* Internet Gateway
* NAT Gateway
* Public Bastion EC2
* Private EC2 without public IP
* Restricted SSH access
* Docker + Kind Kubernetes
* Helm-based Rent-A-Ride deployment
* Kubernetes Ingress
* Manually created AWS Application Load Balancer
* ALB target group with healthy private EC2 target
* End-to-end Internet → ALB → Private EC2 → Kubernetes → Application connectivity

### Final Traffic Flow

```text
Internet
   ↓
AWS ALB
   ↓
Private EC2
   ↓
Kind Kubernetes
   ↓
Frontend
   ↓
Backend
   ↓
MongoDB
```

### Administration Flow

```text
Developer
   ↓
Bastion
   ↓
SSH
   ↓
Private EC2
```

**Task 16 completed successfully. ✅**


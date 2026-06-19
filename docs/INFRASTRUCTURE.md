# Infrastructure — build log and reasoning

This document records what was actually built in AWS, in order, along with
the reasoning behind each decision. Each entry maps back to a control listed
in THREAT_MODEL.md.

## 1. VPC

- Name: aws-cloud-security-vpc
- CIDR block: 10.0.0.0/16
- Tenancy: Default

A /16 reserves roughly 65,000 addresses for the entire project. Every AWS
account ships with a pre-existing default VPC (172.31.0.0/16); that default
VPC was intentionally left untouched, and all resources in this project live
inside the custom VPC above.

## 2. Subnets

Three subnets were created, corresponding to the three tiers in the
architecture:

| Subnet | CIDR block | Purpose |
|---|---|---|
| public-subnet-1 | 10.0.1.0/24 | Holds the load balancer |
| private-app-subnet-1 | 10.0.2.0/24 | Holds the application server |
| private-db-subnet-1 | 10.0.3.0/24 | Holds the database |

At creation, the subnets are not yet differentiated as public or private in
any enforced sense — that distinction is created by route tables, below.

## 3. Internet Gateway

- Name: aws-cloud-security-igw
- Attached to: aws-cloud-security-vpc

Attaching a gateway to the VPC does nothing on its own. A subnet only gains
internet access if its route table explicitly points traffic at this
gateway.

## 4. Route table — public-route-table

- Routes: 10.0.0.0/16 -> local (default), 0.0.0.0/0 -> aws-cloud-security-igw
- Subnet association: public-subnet-1 only

This is the actual mechanism that makes a subnet public: not a label, but a
route. private-app-subnet-1 and private-db-subnet-1 were left on the VPC's
default main route table, which has no route to the Internet Gateway at all.

Note: while building this, the route table was initially associated with
private-db-subnet-1 by mistake instead of public-subnet-1 — the opposite of
the intended design. It was caught by checking the "Explicit subnet
associations" field before moving forward, and corrected before any resource
was deployed behind it.

## 5. NAT Gateway

- Name: aws-cloud-security-nat
- Subnet: public-subnet-1 (must live in a public subnet, even though it
  serves private ones)
- Connectivity type: Public

The NAT Gateway only permits connections that originate inside the VPC and
travel outward. It does not accept inbound connections initiated from the
internet. This gives the app server outbound access (OS updates, external
API calls) without reopening any inbound attack surface.

## 6. Route table — private-route-table

- Routes: 10.0.0.0/16 -> local (default), 0.0.0.0/0 -> aws-cloud-security-nat
- Subnet associations: private-app-subnet-1, private-db-subnet-1

With this in place, the network layer is functionally complete: the public
subnet has two-way internet access; both private subnets have outbound-only
access with no inbound path from the internet at all.

## 7. Security groups

### lb-sg

- Inbound: HTTPS (443), source 0.0.0.0/0
- This security group fronts the load balancer, the only component meant to
  be internet-facing. It does not directly expose a server or database.

### app-sg

- Inbound: HTTP (80), source: lb-sg (security group reference, not an IP range)
- Outbound: default (all traffic)
- TLS is terminated at the load balancer; only plaintext HTTP, confined to
  the private subnet, ever reaches the app server.

### db-sg

- Inbound: MYSQL/Aurora (3306), source: app-sg
- Outbound: MYSQL/Aurora (3306), destination: app-sg
- Outbound is deliberately restricted here, unlike app-sg: the database has
  no legitimate reason to reach anything except the app server. Combined
  with having no internet route at the network layer, the database is
  protected by two independent layers of defense.

## 8. EC2 application server

- Name: aws-cloud-security-app-server
- AMI: Amazon Linux 2023
- Instance type: t2.micro (free-tier eligible)
- Subnet: private-app-subnet-1
- Public IP: None (disabled at launch)
- Security group: app-sg

Access method: AWS Systems Manager Session Manager, not SSH. This instance
has no public IP and lives in a private subnet with no inbound route from
the internet. An IAM role (app-server-ssm-role, scoped to the
AmazonSSMManagedInstanceCore policy) is attached to the instance, allowing
it to register with Systems Manager and accept shell sessions initiated
through the AWS console. No inbound port is opened to enable this -- the
instance reaches Systems Manager over its existing outbound path through
the NAT Gateway. This is a stronger access pattern than opening port 22 to
the internet and relying on key-pair hygiene alone.

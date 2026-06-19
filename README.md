# aws-3-tier-secure-vpc

A hardened, 3-tier AWS network architecture built from scratch and documented
end-to-end — including the threat model that drove every design decision,
not just the resources that were created.

This project intentionally goes through the design process a security
engineer would actually follow: identify what's being protected and from
whom, then build infrastructure that enforces those decisions, then
document the reasoning so it's auditable by someone who wasn't in the room
when it was built.

## What's actually in here

- docs/THREAT_MODEL.md — what this architecture protects, who it's
  realistically defending against, and a control-by-control mapping of why
  each piece of infrastructure exists.
- docs/INFRASTRUCTURE.md — every AWS resource that was built, in the order it
  was built, with the reasoning behind each configuration choice.

## Architecture overview

A single VPC (10.0.0.0/16) divided into three subnets across two tiers of trust:

- Public subnet (10.0.1.0/24) — internet-facing. Holds the load balancer, the
  only resource in this architecture with a direct two-way route to the internet.
- Private app subnet (10.0.2.0/24) — holds the application server. Reachable
  only from the load balancer; outbound-only internet access via a NAT Gateway.
- Private database subnet (10.0.3.0/24) — holds the database. No route to the
  internet in either direction. Reachable only from the application server.

Isolation is enforced at two independent layers:

1. Network layer — route tables determine which subnets have any path to the
   internet at all. The database subnet has no such path, regardless of any
   firewall rule applied on top of it.
2. Security group layer — host-level firewall rules restrict each resource to
   the minimum source it actually needs.

## A real mistake, left in on purpose

While building this, the public route table was initially associated with the
database subnet instead of the public subnet — the opposite of the intended
design. It was caught by checking the subnet associations before moving
forward, and corrected before any resource was deployed behind it.

## Status

Network and firewall layers: built and verified in a live AWS account.
Application layer (EC2, RDS, load balancer, blog app with authentication) is
in progress.

# Threat model — aws-3-tier-secure-vpc

## What this is

Before writing any infrastructure or application code, this document defines
what this architecture is protecting, who is realistically likely to attack
it, and what each security control is specifically defending against. Every
decision in INFRASTRUCTURE.md traces back to a line in this file.

## What we're protecting (assets)

| Asset | Why it matters |
|---|---|
| Admin credentials | The login that allows creating, editing, or deleting content. If compromised, an attacker can deface the site or post content under the owner's identity. |
| Database contents | Application content and, depending on the auth implementation, a password hash. Hashes are not safe to leak outright — a weakly hashed password can be cracked offline. |
| AWS account credentials | Higher value than the app itself. Compromised AWS keys give an attacker control of the infrastructure, not just the application. |
| Site availability | Defacement or downtime is reputationally damaging even without data theft. |

## Who is realistically likely to attack this

This is a portfolio project, not a high-value target — the threat model is
calibrated to that, not to a hypothetical nation-state actor.

- Automated internet-wide scanners that continuously probe every public IP
  for open ports, exposed databases, and known CVEs.
- Opportunistic exploit attempts: generic SQL injection, common login
  brute-force attempts run by bots against any public site.
- Credential stuffing, if the admin's email/password pair ever appears in an
  unrelated breach dump.

## Control-to-threat mapping

| Control | Defends against |
|---|---|
| Database in a private subnet, no route to the internet | Direct internet exposure of the database, even if the app server is compromised. |
| Database security group: inbound only from the app server's security group | Lateral movement and opportunistic port scanning of the database tier. |
| Load balancer: HTTPS only | Network eavesdropping (man-in-the-middle) on credentials and session data in transit. |
| App server in a private subnet, reachable only via the load balancer | Removes the app server from direct internet exposure and fingerprinting. |
| NAT Gateway: outbound-only internet access for private subnets | Lets private resources reach the internet for updates without exposing any inbound port. |
| Least-privilege security groups | Closes the class of misconfiguration (e.g. 0.0.0.0/0 on a database port) responsible for many real-world cloud breaches. |

## Explicitly out of scope (v1)

- DDoS protection at scale (AWS Shield/WAF rate-based rules)
- Multi-factor authentication on the admin login
- Automated dependency vulnerability scanning

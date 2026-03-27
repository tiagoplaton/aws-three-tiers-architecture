# AWS Three-Tier Architecture with High Availability

![Status](https://img.shields.io/badge/Status-Completed-success)
![AWS](https://img.shields.io/badge/AWS-Solutions%20Architect-orange)
![Security](https://img.shields.io/badge/Focus-Information%20Security-blue)

A three-tier web architecture on AWS built around one core principle: **no layer trusts the layer above it**. The database has no idea the internet exists. The application servers have no public IPs. Traffic only moves forward if it passes the right controls.

The project covers VPC design, function-based subnet segmentation, layered ALBs, Multi-AZ RDS, and a security stack with WAF, GuardDuty, and CloudTrail.

---

## Architecture Diagram

> Add your architecture diagram image here.

---

## Components

### Network (VPC)

| Resource | CIDR / Detail | Purpose |
|---|---|---|
| VPC | 10.0.0.0/16 | Main isolated network |
| Public Subnet A | 10.0.1.0/24 | Web tier — AZ us-east-1a |
| Public Subnet B | 10.0.2.0/24 | Web tier — AZ us-east-1b |
| Private Subnet A | 10.0.3.0/24 | App tier — AZ us-east-1a |
| Private Subnet B | 10.0.4.0/24 | App tier — AZ us-east-1b |
| Isolated Subnet A | 10.0.5.0/24 | Data tier — AZ us-east-1a |
| Isolated Subnet B | 10.0.6.0/24 | Data tier — AZ us-east-1b |
| Internet Gateway | — | Internet access for public subnets |
| NAT Gateway | Public Subnet A | Controlled outbound for private subnets |

### Tier 1 — Web (Public)

The only entry point into the application is the external ALB, protected by WAF. Web servers are never directly exposed — they only accept traffic from the ALB's Security Group.

### Tier 2 — Application (Private)

No public IP, no inbound route from the internet. The only way to reach this tier is through the internal ALB. Outbound traffic for updates and integrations goes through the NAT Gateway, keeping these servers invisible to the outside world.

### Tier 3 — Data (Isolated)

The data subnet has no internet route — inbound or outbound. RDS only accepts connections from the app servers' Security Group, on the exact database port. Any other access attempt is dropped before it arrives.

---

## Security Decisions

### Segmentation by function, not convenience

The three subnet types aren't aesthetic choices — each has a different route table:

- Public: default route `0.0.0.0/0` via Internet Gateway
- Private: default route `0.0.0.0/0` via NAT Gateway (outbound without exposure)
- Isolated: no default route — traffic stays within the VPC

This means compromising a web server doesn't give direct access to the database. An attacker still has to get past the internal ALB and the app tier's Security Group.

### Security Groups as control layers

```
External ALB:
  Inbound:  443 (HTTPS) from 0.0.0.0/0
  Outbound: 80 to Web Servers SG

Web Servers:
  Inbound:  80 from External ALB SG only
  Outbound: 80 to Internal ALB SG

App Servers:
  Inbound:  80 from Internal ALB SG only
  Outbound: 3306 to RDS SG

RDS:
  Inbound:  3306 from App Servers SG only
  Outbound: none
```

No rule uses `0.0.0.0/0` beyond the external ALB. Each resource only talks to what it needs to talk to.

### Monitoring and detection

| Service | What it covers |
|---|---|
| **CloudTrail** | Full API call history — who did what and when |
| **GuardDuty** | Anomalous behavior, network reconnaissance, compromised credentials |
| **AWS Config** | Continuous configuration compliance against defined policies |
| **VPC Flow Logs** | Accepted and rejected network traffic — essential for incident investigation |

---

## High Availability

Every tier is distributed across two Availability Zones. A complete AZ failure won't take down the service — ALBs detect unavailability via health checks and reroute traffic automatically.

RDS Multi-AZ replicates transactions synchronously to the standby instance. Failover is automatic and takes around 1–2 minutes with no manual intervention.

---

## How to Reproduce

1. Create the VPC (`10.0.0.0/16`) with DNS hostnames enabled
2. Create the 6 subnets distributed across two AZs
3. Create and attach the Internet Gateway
4. Create the NAT Gateway in a public subnet with an Elastic IP
5. Configure separate route tables for each subnet type
6. Create the Security Groups following the rules above
7. Launch EC2 instances in their corresponding subnets
8. Create the external and internal ALBs with Target Groups
9. Create the RDS instance with Multi-AZ enabled in the isolated subnet group
10. Enable CloudTrail, GuardDuty, and attach WAF to the external ALB

---

## What I Actually Learned

The biggest surprise was realizing the subnet name means nothing — what actually makes a subnet public or private is its route table. You can label something "public" and forget to add the IGW route, and it stays completely isolated. That shifts how you think about network control entirely.

The other insight: referencing Security Groups by ID instead of CIDR is far more reliable. When an instance gets replaced, the rule still holds — no manual updates needed.

---

## Next Steps

- [ ] Auto Scaling Groups for web and app tiers
- [ ] CloudFront for static content with edge caching
- [ ] AWS Backup with retention policy for RDS
- [ ] Rebuild the entire infrastructure as CloudFormation templates

---

## About

Transitioning into AWS Solutions Architecture with a background in Information Security. My focus is designing environments that are hard to compromise — not just functional ones.

[LinkedIn](https://www.linkedin.com/in/tiagopmadeira/) • [GitHub](https://github.com/tiagoplaton)

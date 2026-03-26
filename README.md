AWS Three-Tier Architecture with High Availability

A three-tier web architecture on AWS built around one core principle: no layer trusts the layer above it. The database has no idea the internet exists. The application servers have no public IPs. Traffic only moves forward if it passes the right controls.
The project covers VPC design, function-based subnet segmentation, layered ALBs, Multi-AZ RDS, and a security stack with WAF, GuardDuty, and CloudTrail.

Architecture Overview
Internet
    │
    ▼
[External ALB] ── WAF
    │
    ├── Public Subnet A (AZ-a)    ── EC2 Web Server
    └── Public Subnet B (AZ-b)    ── EC2 Web Server
                │
           [Internal ALB]
                │
    ├── Private Subnet A (AZ-a)   ── EC2 App Server
    └── Private Subnet B (AZ-b)   ── EC2 App Server
                │
    ├── Isolated Subnet A (AZ-a)  ── RDS Primary
    └── Isolated Subnet B (AZ-b)  ── RDS Standby (Multi-AZ)

Components
Network (VPC)
ResourceCIDR / DetailPurposeVPC10.0.0.0/16Main isolated networkPublic Subnet A10.0.1.0/24Web tier — AZ us-east-1aPublic Subnet B10.0.2.0/24Web tier — AZ us-east-1bPrivate Subnet A10.0.3.0/24App tier — AZ us-east-1aPrivate Subnet B10.0.4.0/24App tier — AZ us-east-1bIsolated Subnet A10.0.5.0/24Data tier — AZ us-east-1aIsolated Subnet B10.0.6.0/24Data tier — AZ us-east-1bInternet Gateway—Internet access for public subnetsNAT GatewayPublic Subnet AControlled outbound for private subnets
Tier 1 — Web (Public)
The only entry point into the application is the external ALB, protected by WAF. Web servers are never directly exposed — they only accept traffic from the ALB's Security Group.
Tier 2 — Application (Private)
No public IP, no inbound route from the internet. The only way to reach this tier is through the internal ALB. Outbound traffic for updates and integrations goes through the NAT Gateway, keeping these servers invisible to the outside world.
Tier 3 — Data (Isolated)
The data subnet has no internet route — inbound or outbound. RDS only accepts connections from the app servers' Security Group, on the exact database port. Any other access attempt is dropped before it arrives.

Security Decisions
Segmentation by function, not convenience
The three subnet types aren't aesthetic choices — each has a different route table:

Public: default route 0.0.0.0/0 via Internet Gateway
Private: default route 0.0.0.0/0 via NAT Gateway (outbound without exposure)
Isolated: no default route — traffic stays within the VPC

This means compromising a web server doesn't give direct access to the database. An attacker still has to get past the internal ALB and the app tier's Security Group.
Security Groups as control layers
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
No rule uses 0.0.0.0/0 beyond the external ALB. Each resource only talks to what it needs to talk to.
Monitoring and detection
ServiceWhat it coversCloudTrailFull API call history — who did what and whenGuardDutyAnomalous behavior, network reconnaissance, compromised credentialsAWS ConfigContinuous configuration compliance against defined policiesVPC Flow LogsAccepted and rejected network traffic — essential for incident investigation

High Availability
Every tier is distributed across two Availability Zones. A complete AZ failure won't take down the service — ALBs detect unavailability via health checks and reroute traffic automatically.
RDS Multi-AZ replicates transactions synchronously to the standby instance. Failover is automatic and takes around 1–2 minutes with no manual intervention.

How to Reproduce

Create the VPC (10.0.0.0/16) with DNS hostnames enabled
Create the 6 subnets distributed across two AZs
Create and attach the Internet Gateway
Create the NAT Gateway in a public subnet with an Elastic IP
Configure separate route tables for each subnet type
Create the Security Groups following the rules above
Launch EC2 instances in their corresponding subnets
Create the external and internal ALBs with Target Groups
Create the RDS instance with Multi-AZ enabled in the isolated subnet group
Enable CloudTrail, GuardDuty, and attach WAF to the external ALB


What I Actually Learned
The biggest surprise was realizing the subnet name means nothing — what actually makes a subnet public or private is its route table. You can label something "public" and forget to add the IGW route, and it stays completely isolated. That shifts how you think about network control entirely.
The other insight: referencing Security Groups by ID instead of CIDR is far more reliable. When an instance gets replaced, the rule still holds — no manual updates needed.

Next Steps

 Auto Scaling Groups for web and app tiers
 CloudFront for static content with edge caching
 AWS Backup with retention policy for RDS
 Rebuild the entire infrastructure as CloudFormation templates


About
Transitioning into AWS Solutions Architecture with a background in Information Security. My focus is designing environments that are hard to compromise — not just functional ones.

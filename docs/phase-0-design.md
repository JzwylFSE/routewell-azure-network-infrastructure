# RouteWell — Phase 0 Infrastructure Design

**Project:** Multi-Tier VNet Architecture with Access Control  
**Client:** RouteWell Logistics  
**Cloud Platform:** Microsoft Azure  
**Team:** Cloud & DevOps Bootcamp — Group 1  
**Status:** Proposed design — pending deployment and validation

## 1. Project Overview

RouteWell Logistics requires a secure, cost-conscious Azure network architecture for its dispatch system.

The company previously operated on a flat network, creating the risk that an unauthorized device could access sensitive customer and driver information.

Our proposed solution separates the infrastructure into three tiers:

- **Web tier:** Hosts the public-facing dispatch interface.
- **Application tier:** Processes business logic and requests from the web tier.
- **Database tier:** Stores sensitive dispatch, driver, and customer records.

The design applies network segmentation, least-privilege access controls, and capacity planning to meet the company's operational and security requirements.

## 2.1 CIDR Planning

### VNet Address Space

`10.10.0.0/16`

A `/16` IPv4 network contains 65,536 addresses. We will divide a portion of this address space into three non-overlapping subnets.

Azure reserves five IPv4 addresses per subnet.

The calculation used is:

`Usable Azure IPs = 2^(32 - CIDR prefix) - 5`

### Host Requirements

| Tier | Current hosts | Hosts in 6 months | Chosen subnet |
|---|---:|---:|---|
| Web | 12 | 12 | /27 |
| Application | 20 | 40 | /26 |
| Database | 6 | 6 | /28 |

### Web Subnet

The web tier requires 12 hosts.

- `/28`: 16 total addresses − 5 reserved = 11 usable (insufficient).
- `/27`: 32 total addresses − 5 reserved = 27 usable (sufficient).

**Decision:** `10.10.0.0/27`

- Subnet mask: `255.255.255.224`
- Full address range: `10.10.0.0–10.10.0.31`
- Azure assignable range: `10.10.0.4–10.10.0.30`
- Usable addresses: 27

A `/27` is the smallest suitable IPv4 subnet for the stated web-tier requirement.

### Application Subnet

The application tier currently requires 20 hosts and is expected to require 40 hosts within six months.

- `/27`: 27 usable addresses (insufficient for future growth).
- `/26`: 59 usable addresses (sufficient).

**Decision:** `10.10.0.64/26`

- Subnet mask: `255.255.255.192`
- Full address range: `10.10.0.64–10.10.0.127`
- Azure assignable range: `10.10.0.68–10.10.0.126`
- Usable addresses: 59

The `/26` supports the projected 40 hosts while providing 19 additional assignable addresses.

### Database Subnet

The database tier requires six hosts.

- `/29`: 8 total addresses − 5 reserved = 3 usable (insufficient).
- `/28`: 16 total addresses − 5 reserved = 11 usable (sufficient).

**Decision:** `10.10.0.32/28`

- Subnet mask: `255.255.255.240`
- Full address range: `10.10.0.32–10.10.0.47`
- Azure assignable range: `10.10.0.36–10.10.0.46`
- Usable addresses: 11

### CIDR Design Justification

These subnet sizes are the smallest that satisfy the stated current and six-month host requirements under Azure's IPv4 reservation rules.

The ranges do not overlap and are fully contained inside `10.10.0.0/16`.

The address range `10.10.0.48–10.10.0.63` remains unallocated, alongside the unused remainder of the VNet address space.

The demo will provision one Linux VM per subnet. The larger host figures represent network capacity requirements, not the number of VMs required for the initial deployment.

## 2.2 Network Security Group Rule Justification

### Security Objective

RouteWell requires:

1. Internet users to access the web tier.
2. The web tier to communicate with the application tier.
3. The application tier to communicate with the database tier.
4. Direct Web-to-Database access to be blocked.
5. Unnecessary internal traffic to be denied.

We propose one NSG per subnet:

- `nsg-web`
- `nsg-app`
- `nsg-db`

### Application Protocol Assumptions

| Service | Protocol | Port |
|---|---|---|
| Public web server | TCP | 80 |
| Application API | TCP | 3000 |
| PostgreSQL database | TCP | 5432 |

These ports are implementation decisions for the capstone demonstration. Port 3000 assumes an application service listening on that port, and port 5432 assumes PostgreSQL.

### Proposed Inbound NSG Rules

| NSG | Priority | Source | Destination | Port | Action | Justification |
|---|---:|---|---|---|---|---|
| nsg-web | 100 | Internet | Web subnet | TCP 80 | Allow | Enables public access to the dispatch interface. Without it, public requests would be denied by the remaining inbound policy. |
| nsg-web | 200 | VirtualNetwork | Web subnet | Any | Deny | Blocks unnecessary internal access to the web tier. Without it, Azure's default intra-VNet allow could permit unwanted connections. |
| nsg-app | 100 | Web subnet | App subnet | TCP 3000 | Allow | Enables web-to-application communication. Without it, API requests would be blocked by the later deny rule. |
| nsg-app | 200 | VirtualNetwork | App subnet | Any | Deny | Prevents other unauthorized intra-VNet traffic from entering the application tier. |
| nsg-db | 100 | App subnet | DB subnet | TCP 5432 | Allow | Enables authorized database connections from the application tier. Without it, database requests would be blocked by the later deny rule. |
| nsg-db | 200 | VirtualNetwork | DB subnet | Any | Deny | Prevents direct and unauthorized intra-VNet database connections, including Web-to-DB traffic. |

### Proposed Outbound NSG Rules

| NSG | Priority | Source | Destination | Port | Action | Justification |
|---|---:|---|---|---|---|---|
| nsg-web | 100 | Web subnet | App subnet | TCP 3000 | Allow | Required for web-initiated backend requests. |
| nsg-web | 200 | Web subnet | VirtualNetwork | Any | Deny | Prevents unnecessary web-tier connections to internal resources. |
| nsg-app | 100 | App subnet | DB subnet | TCP 5432 | Allow | Required for application-initiated database connections. |
| nsg-app | 200 | App subnet | VirtualNetwork | Any | Deny | Prevents application-tier access to other internal resources beyond explicitly permitted services. |
| nsg-db | 200 | DB subnet | VirtualNetwork | Any | Deny | Restricts unnecessary database-initiated connections to other internal resources. |

### Rule Processing

Azure evaluates NSG rules in ascending numerical priority. The first matching rule determines whether traffic is allowed or denied.

NSGs are stateful: response traffic belonging to an established, permitted connection does not require a separate reverse-direction allow rule.

Azure's default `AllowVNetInBound` and `AllowVNetOutBound` rules would otherwise permit broad intra-VNet communication. Our higher-priority rules restrict that behavior.

### Platform and Management Traffic

The rules above define the intended application-tier communication policy.

Before deployment, we must separately validate necessary DNS, VM administration, Azure platform traffic, package installation, and update requirements.

Any additional management exceptions must be narrowly scoped, justified, documented, and tested rather than introduced as unrestricted convenience rules.

## 2.3 Public Access Mechanism

### Decision: Public IP on the Web VM

We will expose RouteWell's web tier using a Standard SKU Azure Public IP associated with the web VM's network interface. The web subnet's NSG will permit inbound HTTP traffic on TCP port 80 while restricting unnecessary connections. The application and database VMs will use private IP addresses only, preventing direct inbound connections from the public internet. We chose this approach instead of a public load balancer because the initial deployment contains only one web VM and RouteWell requires a cost-conscious architecture. Although multiple web instances behind a load balancer would improve availability and scalability, they are not required for the initial network segmentation demonstration.

### Security and Cost Considerations

- Only the web VM will have a public IP.
- The app and database VMs will communicate through private IP addresses.
- Public HTTP access is used for the capstone demonstration; production systems handling sensitive traffic should implement HTTPS.
- Remote VM administration must use a secure, explicitly justified access mechanism.
- VM sizes, regional pricing, public IP charges, disk costs, and private-subnet outbound connectivity will be reviewed before deployment.

## 2.4 Architecture Diagram

The architecture diagram will be stored at:

`docs/architecture.png`

It must show:

- Azure VNet boundary: `10.10.0.0/16`
- Web subnet: `10.10.0.0/27`
- App subnet: `10.10.0.64/26`
- DB subnet: `10.10.0.32/28`
- Public IP attached to the web VM
- Three subnet-associated NSGs
- Allowed Internet-to-Web communication on TCP 80
- Allowed Web-to-App communication on TCP 3000
- Allowed App-to-DB communication on TCP 5432
- Blocked Web-to-DB communication

### Example VM Private IP Plan

| VM | Subnet | Proposed private IP | Public IP |
|---|---|---|---|
| vm-web | snet-web | 10.10.0.4 | Yes |
| vm-app | snet-app | 10.10.0.68 | No |
| vm-db | snet-db | 10.10.0.36 | No |

These addresses are proposed static private IP assignments that must be implemented and verified during deployment.

## 3. Design Tradeoffs

**Security versus simplicity:** Three subnet-associated NSGs create clear tier boundaries without requiring additional firewall appliances.

**Cost versus availability:** A single web VM is a lean demonstration architecture but remains a single point of failure.

**Current demand versus future growth:** The application subnet is sized for the projected 40 hosts, avoiding an immediate need for subnet readdressing.

**Automation versus manual setup:** Azure CLI and Bash will be used to make the infrastructure deployment reproducible.

## 4. Planned Validation

The deployed network must pass these tests:

| Test | Expected result |
|---|---|
| Internet → Web, TCP 80 | Allowed |
| Web → App, TCP 3000 | Allowed |
| App → DB, TCP 5432 | Allowed |
| Web → DB, TCP 5432 | Blocked |
| Internet → DB, TCP 5432 | No direct public access |

Connectivity tests must use services actively listening on the intended ports. A failed TCP test alone does not establish that an NSG caused the failure.

Test outputs and screenshots will be saved in the repository after deployment.

## 5. Pending Implementation Decisions

Before infrastructure deployment, the team must finalize:

- Azure subscription and deployment region
- Linux VM sizes
- Private VM administration method
- Outbound connectivity for private VM setup and updates
- DNS and Azure platform-service exceptions, if required
- Exact application and database service configuration

Changes from this design will be documented with reasons and incorporated into the final architecture documentation.

---

**Document status:** Phase 0 proposal, prepared for group technical review and validation before Azure deployment.
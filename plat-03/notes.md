# AWS VPC Networking — Complete Study Notes

## 1. What is a VPC?

A **Virtual Private Cloud (VPC)** is a logically isolated virtual network in AWS where you deploy resources such as EC2 instances, load balancers, and databases.

Think of a VPC as your own network in the AWS cloud.

### Example network

- VPC CIDR: `10.0.0.0/16`
- Public subnet: `10.0.1.0/24`
- Private application subnet: `10.0.2.0/24`
- Private database subnet: `10.0.3.0/24`

```text
                    AWS CLOUD
                        |
                 +--------------+
                 |     VPC      |
                 | 10.0.0.0/16  |
                 +--------------+
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
    +-----------+ +-----------+ +-----------+
    |  PUBLIC   | |  PRIVATE  | |  PRIVATE  |
    |  SUBNET   | |    APP    | | DATABASE  |
    |10.0.1.0/24| |10.0.2.0/24| |10.0.3.0/24|
    +-----------+ +-----------+ +-----------+
          |             |             |
         ALB            EC2           RDS
```

**Remember:** A VPC is the network; subnets divide its IP address space into smaller networks.

---

## 2. What is a subnet?

A **subnet** is a range of IP addresses within a VPC. In AWS, each subnet belongs to exactly one Availability Zone (AZ), although one AZ can contain multiple subnets.

```text
VPC: 10.0.0.0/16
|
+-- AZ-a
|   +-- Public subnet: 10.0.1.0/24
|   +-- Private subnet: 10.0.2.0/24
|
+-- AZ-b
    +-- Public subnet: 10.0.4.0/24
    +-- Private subnet: 10.0.5.0/24
```

Deploying resources across multiple AZs can improve availability.

### Public vs. private subnet

- **Public subnet:** Its associated route table has a route to an Internet Gateway (IGW).
- **Private subnet:** It typically has no direct route to an IGW for general internet-bound IPv4 traffic.

A private subnet can still communicate with other VPC subnets through the VPC's local route.

**Important:** A subnet being public does not automatically make every instance in it publicly accessible. For direct inbound IPv4 access, the instance also needs appropriate public addressing and network permissions.

---

## 3. Understanding CIDR blocks

CIDR stands for **Classless Inter-Domain Routing**. It represents an IP address range.

| CIDR | Total IPv4 addresses |
|---|---:|
| `10.0.0.0/16` | 65,536 |
| `10.0.1.0/24` | 256 |
| `10.0.1.0/25` | 128 |
| `10.0.1.0/26` | 64 |
| `10.0.1.0/28` | 16 |

AWS IPv4 subnet sizes range from `/16` to `/28`. AWS reserves five IPv4 addresses in every subnet, so a `/24` subnet has 251 available IPv4 addresses for AWS resources.

### Private IPv4 address ranges

| Range | CIDR |
|---|---|
| Private range | `10.0.0.0/8` |
| Private range | `172.16.0.0/12` |
| Private range | `192.168.0.0/16` |

**Correction:** The standard private range is `192.168.0.0/16`, not `192.128.0.0/16`.

---

## 4. What is a VPC router?

Think of the VPC router as the logical routing function that forwards traffic using the applicable route tables. You don't normally deploy or manage a separate router EC2 instance for standard VPC routing.

The route table determines the next hop for traffic based on the destination IP address.

```text
EC2 Instance
     |
     v
Route Table
     |
     +---- Destination belongs to VPC
     |              |
     |              v
     |         Local Route
     |              |
     |              v
     |         Another Subnet
     |
     +---- Internet destination
                    |
                    v
              Internet Gateway
```

The actual path depends on the destination and the matching routes.

**Important:** The route table doesn't decide whether an application connection is allowed. Security Groups and NACLs provide network filtering.

---

## 5. What is a route table?

A **route table** contains rules that determine where network traffic should go. Each route has a destination and a target.

### Example: Public subnet route table

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |
| `0.0.0.0/0` | Internet Gateway |

The `local` route is automatically created for the VPC's IPv4 CIDR. The default route `0.0.0.0/0` sends otherwise-unmatched IPv4 destinations toward the IGW.

### How does AWS choose a route?

AWS uses the most specific matching route.

For example, if a route table contains both `10.0.0.0/16 → local` and a valid `10.0.2.0/24` route to another target, traffic to `10.0.2.10` matches the more specific `/24` route.

**Key point:** A route table tells AWS where to send a packet. It doesn't automatically authorize that packet to reach an application.

---

## 6. How can a public EC2 instance reach a private EC2 instance?

Suppose:

- Public EC2: `10.0.1.10`
- Private EC2: `10.0.2.10`
- VPC CIDR: `10.0.0.0/16`

Both instances are in the same VPC.

```text
PUBLIC SUBNET                     PRIVATE SUBNET

+----------------+               +----------------+
| Public EC2     |               | Private EC2    |
| 10.0.1.10      |               | 10.0.2.10      |
+----------------+               +----------------+
         |                                ^
         |                                |
         +------ VPC local route ---------+
```

The public EC2 instance sends traffic to `10.0.2.10`. Because the destination belongs to the VPC CIDR, the local route provides the path.

The connection can succeed if the relevant Security Groups, NACLs, and other network controls permit it.

### Example: Application on port 8080

If the private EC2 instance hosts an application on TCP port `8080`, its Security Group must allow inbound traffic on that port from the public EC2 instance (or another appropriate source).

**Remember:** Traffic between subnets in the same VPC normally uses the local route. It doesn't need an IGW or NAT Gateway.

---

## 7. What is an Internet Gateway (IGW)?

An **Internet Gateway** provides connectivity between a VPC and the internet. It must be attached to the VPC and referenced by a route in the relevant route table.

For direct internet connectivity over IPv4, a resource generally needs a public IPv4 address or Elastic IP, a valid route to the IGW, and appropriate network permissions.

### Outbound request

Suppose an EC2 instance has:
- Private IP: `10.0.1.10`
- Public IP: `54.10.20.30`

It wants to access an external website.

```text
EC2
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

The route table directs internet-bound traffic to the IGW. The IGW maps the instance's private IPv4 address to its associated public IPv4 address for internet communication.

### Inbound request

A user accesses the instance's public IP address.

```text
User on Internet
       |
       v
Internet Gateway
       |
       v
Public IP mapped to private IP
       |
       v
EC2: 10.0.1.10
```

The IGW maps the public destination IPv4 address to the instance's private IPv4 address. The VPC then delivers the packet to the instance, subject to routing and network controls.

**Important:** An IGW doesn't automatically make every resource in the VPC publicly accessible. The resource needs appropriate public addressing, routing, and network permissions.

---

## 8. Why can't the internet directly reach a private EC2 instance?

Suppose an EC2 instance has:
- Private IP: `10.0.2.10`
- No public IP address
- No direct route from its subnet to the IGW
- A Security Group allowing TCP port `80` from `0.0.0.0/0`

Can anyone on the internet access it directly?

**No, not through the ordinary direct IPv4 internet path.**

The Security Group allows matching HTTP traffic, but it doesn't create a network path from the internet to the instance. The instance has no public IPv4 address through which an internet client can directly address it, and its subnet has no direct route to an IGW.

**Key principle:** A Security Group permits traffic; it doesn't create internet connectivity.

A private instance can still be reached through an explicitly configured path, such as a load balancer, VPN, or another network connection.

---

## 9. What is a NAT Gateway?

A **NAT Gateway** enables resources in private subnets to initiate outbound IPv4 connections without requiring their own public IPv4 addresses.

A public NAT Gateway is normally deployed in a public subnet and associated with an Elastic IP.

### Architecture

```text
                 INTERNET
                    ^
                    |
                   IGW
                    ^
                    |
              NAT Gateway
                    ^
                    |
             Private Route Table
                    ^
                    |
               Private EC2
```

### Route tables

**Private subnet:**

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |
| `0.0.0.0/0` | NAT Gateway |

**Public subnet containing the NAT Gateway:**

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |
| `0.0.0.0/0` | Internet Gateway |

### What happens?

1. The private EC2 instance initiates an internet connection.
2. Its route table sends the traffic to the NAT Gateway.
3. The NAT Gateway translates the private source address and port.
4. Traffic travels through the IGW to the internet.
5. The response returns through the NAT Gateway to the private instance.

### Can the internet initiate a new connection to the private instance through NAT?

Ordinarily, no. The NAT Gateway permits outbound connections initiated by private resources and their associated response traffic. It doesn't provide unsolicited inbound connectivity to those resources.

**Remember:** NAT Gateway provides outbound internet connectivity for private IPv4 resources.

---

## 10. What is a Security Group?

A **Security Group (SG)** is a stateful virtual firewall associated with an EC2 instance's network interface or another supported network resource. It controls permitted inbound and outbound traffic.

Rules can specify:
- Protocol
- Port or port range
- Source or destination IP address
- Another Security Group

Security Groups support allow rules, not explicit deny rules.

### Example

Private EC2:
- IP: `10.0.2.10`
- Application port: `8080`

Inbound Security Group rule:

| Protocol | Port | Source |
|---|---|---|
| TCP | 8080 | `10.0.1.10/32` |

This permits TCP connections on port `8080` from that specific IPv4 address, subject to other applicable network controls.

### Why is a Security Group needed if a route exists?

The route table provides a path to the private instance. The Security Group decides whether matching traffic is permitted to reach the instance.

```text
Public EC2
     |
     v
Route Table
     |
     v
VPC Local Route
     |
     v
Private EC2 Security Group
     |
     +---- Rule allows port 8080 --> Traffic permitted
     |
     +---- No matching allow rule --> Traffic blocked
```

Security Groups are stateful. Return traffic for an allowed connection is automatically permitted by the Security Group.

---

## 11. What is a Network ACL (NACL)?

A **Network ACL (NACL)** is a subnet-level traffic filter. It supports both ALLOW and DENY rules and is stateless. NACLs filter traffic entering and leaving the subnet.

### Example: Web, application, and database subnets

```text
                 WEB TIER
                    |
                    | Port 8080
                    v
              APPLICATION TIER
                    |
                    | Port 3306
                    v
               DATABASE TIER

Web tier --------X--------> Database tier
                 Block
```

Suppose your requirements are:
- Web tier can reach the application tier.
- Application tier can reach the database tier.
- Web tier cannot connect directly to the database on MySQL port `3306`.

A NACL associated with the database subnet can allow MySQL traffic from the application subnet and explicitly deny MySQL traffic from the web subnet.

Example database subnet NACL rules:

| Rule number | Protocol | Source | Port | Action |
|---|---|---|---|---|
| 100 | TCP | `10.0.2.0/24` | 3306 | ALLOW |
| 110 | TCP | `10.0.1.0/24` | 3306 | DENY |
| `*` | All | Any | Any | DENY |

NACL rules are evaluated in ascending numerical order, and the first matching rule determines the action.

Because NACLs are stateless, outbound rules must also permit the relevant response traffic, including the client's ephemeral destination port.

### Security Group vs. NACL

| Feature | Security Group | NACL |
|---|---|---|
| Applied to | Resource network interface | Subnet |
| Stateful? | Yes | No |
| Supports ALLOW | Yes | Yes |
| Supports explicit DENY | No | Yes |
| Return traffic | Automatically permitted for allowed connections | Must be permitted by separate rules |
| Typical use | Restrict access to an application or resource | Filter traffic at the subnet boundary |

**Recommended approach:** Use Security Groups as the primary control for application access. Use NACLs when you need subnet-level filtering or explicit deny rules.

---

## 12. Why use a Load Balancer?

Suppose your application runs on multiple EC2 instances in private subnets. You want users to access the application without exposing those instances directly to the internet.

An internet-facing **Application Load Balancer (ALB)** can provide a public entry point.

### Architecture

```text
                  USERS
                    |
                    v
               INTERNET
                    |
                    v
                   IGW
                    |
                    v
          +-------------------+
          | Public Subnets    |
          | Application LB    |
          +-------------------+
                    |
                    | Private IPs
                    v
          +-------------------+
          | Private Subnets   |
          | EC2 App Servers   |
          +-------------------+
```

The ALB receives requests and forwards them to registered targets, such as EC2 instances in private subnets.

### Why can the ALB reach a private EC2 instance?

Both resources can communicate through the VPC's internal network. The local route provides the path, and the application server's Security Group allows the ALB's traffic.

The application server does not need a public IP address.

### Recommended Security Groups

**ALB Security Group**
- Allow inbound HTTPS from permitted clients.
- Allow outbound application traffic to the application servers.

**Application EC2 Security Group**
- Allow inbound application traffic, such as TCP port `8080`, only from the ALB's Security Group.

This prevents direct internet clients from connecting to the application servers, assuming no other access path permits it.

---

## 13. Does an ALB have a static IP?

An **Application Load Balancer (ALB)** provides a DNS name. Its underlying IP addresses can change, so clients should use its DNS name rather than rely on fixed IP addresses.

If a client requires fixed IP addresses for firewall allowlisting, a **Network Load Balancer (NLB)** is an option.

An NLB can have one Elastic IP address per enabled Availability Zone when configured with Elastic IPs in its subnet mappings.

For example, a two-AZ NLB using Elastic IPs has two static IP addresses, one for each enabled AZ. The NLB DNS name resolves to the NLB's addresses.

### When would you put an NLB in front of an ALB?

When a client requires static IP addresses but you also want ALB features such as host-based and path-based routing. The NLB is the external entry point and forwards traffic to the ALB, which handles application-layer routing.

**Remember:**
- **Subnet mappings** determine where the NLB is deployed.
- **Target groups** determine where it sends traffic.

---

# PART 7 — THE THREE INTERVIEW QUESTIONS

## Q1. Does a load balancer give users one static IP to reach our app?

An internet-facing ALB provides a DNS name rather than one static IP, and its underlying IP addresses can change. If clients require fixed IPs for firewall allowlisting, an NLB can use one Elastic IP per enabled Availability Zone. An NLB can sit in front of an ALB when fixed IPs and ALB application-layer routing are both required.

## Q2. How does traffic get from a public subnet instance to a private subnet instance, and why doesn't internet traffic go straight into the private subnet?

Instances in public and private subnets of the same VPC communicate through the VPC's local route, provided the network controls permit it. Internet clients cannot ordinarily reach a private instance directly because it lacks a public IP and its subnet has no direct route to the IGW. A NAT Gateway enables outbound connections from private resources but does not allow unsolicited inbound connections.

## Q3. (a) What decides where traffic from a subnet goes? (b) What stops the internet reaching a private subnet?

**(a)** The subnet's associated route table determines where traffic goes based on the destination IP address. The VPC's local route provides paths between subnets within the VPC.

**(b)** A private instance typically has no public IP and no direct route from its subnet to an IGW. Security Groups and NACLs provide additional traffic filtering, but a Security Group rule alone cannot create internet connectivity.

---

## 14. Final mental map: troubleshoot a connection

Whenever you troubleshoot an AWS network connection, ask these questions in order:

```text
1. ADDRESSING
   Does the destination have the addressing needed?
                |
                v
2. ROUTING
   Is there a valid path to the destination?
                |
                v
3. SECURITY
   Do the Security Groups and NACLs permit traffic?
                |
                v
4. APPLICATION
   Is the service running and listening on the port?
                |
                v
5. RESULT
   The connection succeeds if all required conditions are met.
```

### The five concepts to remember

| Component | Main responsibility |
|---|---|
| Route table | Determines where traffic goes |
| Internet Gateway | Provides VPC-to-internet connectivity |
| NAT Gateway | Enables outbound IPv4 internet connections from private resources |
| Security Group | Controls permitted traffic at the resource level |
| NACL | Filters traffic at the subnet level |

**The most important distinction:** Routing establishes a path, addressing makes the destination reachable, and security controls determine whether traffic is permitted.

## 15. Three traffic flows to practise from memory

### Flow A: Internet → Public EC2

```text
Internet client
    → EC2 public IP
    → Internet Gateway
    → VPC delivery to private IP
    → Security Group / NACL checks
    → Application
```

Assumptions: The EC2 instance has appropriate public IPv4 addressing, its subnet has a route to the IGW, network rules permit the connection, and the application is listening.

### Flow B: Public EC2 → Private EC2

```text
Public EC2
    → VPC local route
    → Private EC2 Security Group / NACL checks
    → Application on the destination port
```

No IGW or NAT Gateway is needed for this same-VPC path.

### Flow C: Private EC2 → Internet

```text
Private EC2
    → Private subnet route table
    → NAT Gateway in a public subnet
    → Internet Gateway
    → Internet
```

The response returns through the NAT Gateway. The private instance does not need its own public IPv4 address for this outbound IPv4 pattern.

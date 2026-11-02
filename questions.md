# AWS VPC Networking — Three Questions

## Q1. Does a load balancer give users one static IP to reach our app? If not, what does the user actually hit?

An internet-facing Application Load Balancer (ALB) gives users a DNS name, not one fixed IP address. The DNS name resolves to ALB IP addresses that can change. If a client requires fixed IPs for firewall allowlisting, a Network Load Balancer (NLB) can use one Elastic IP per enabled Availability Zone. An NLB can sit in front of an ALB when fixed IPs and ALB application-layer routing are both required.

## Q2. How does traffic get from a public subnet instance to a private subnet instance, and why doesn’t internet traffic go straight into the private subnet?

Instances in public and private subnets of the same VPC can communicate through the VPC's built-in `local` route, provided the relevant Security Groups and Network ACLs allow the traffic. Internet clients cannot ordinarily reach a private instance directly because it has no public IP address and its subnet has no direct route to an Internet Gateway (IGW). A NAT Gateway lets private instances initiate outbound internet connections; it does not allow unsolicited inbound connections from the internet.

## Q3. (a) What decides where traffic from a subnet goes? (b) What stops the internet reaching a private subnet?

**(a)** The subnet's associated route table determines where traffic goes by matching the destination IP address against its routes. The VPC's built-in `local` route provides paths between subnets in the same VPC.

**(b)** A private instance typically has no public IP address, and its subnet has no direct route to an IGW. Security Groups and Network ACLs provide additional traffic filtering, but allowing a port in a Security Group does not create internet connectivity.

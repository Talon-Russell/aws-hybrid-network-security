# AWS Hybrid Network Security Project

## 1. Overview

For this project, I used GNS3 and AWS to build and test a hybrid network.

The main goal was to connect an on-premises network to AWS through an IPsec VPN while controlling which departments could access local and cloud resources.

I configured VLANs, DHCP, routing, NAT, AWS security groups, and a VPN between a VyOS firewall and an EC2 instance running Libreswan.

After completing the setup, I documented eight test cases to verify the network configuration, connectivity, and access controls.

## 2. Business Scenario

For this project, I used a fictional company called Copper Creek Legal.

The company has employees in IT, HR, and Finance, along with a local server network. It also uses AWS to host an application server and a database server.

I wanted to create a network that allowed approved employees to access resources across both environments.

For example, IT needed access to the local server and the AWS application server, while HR did not need the same access.

This gave me a way to practice connecting two environments while keeping access controlled by department.

## 3. Objectives

The goals for this project were to:

- Create separate VLANs for HR, Finance, IT, and servers.
- Configure VLAN trunks between the switches and firewall.
- Configure DHCP for employee devices.
- Assign a static IP address to the local server.
- Configure routing and NAT for external connectivity.
- Create an AWS VPC with application and database subnets.
- Configure AWS security groups.
- Connect the local network to AWS through an IPsec VPN.
- Verify both allowed and blocked connections.
- Troubleshoot addressing, routing, NAT, and VPN issues.

## 4. Technologies Used

- GNS3
- VyOS
- Amazon VPC
- Amazon EC2
- Amazon Linux 2023
- Libreswan
- AWS Security Groups
- AWS Route Tables
- AWS Internet Gateway
- DHCP
- NAT
- 802.1Q VLAN Trunking
- IPsec Site-to-Site VPN

## 5. Implementation

### Step 1 - Creating the Local Network

I started by building the on-premises network in GNS3.

The network included employee devices, a local server, switches, a VyOS firewall, an ISP router, and a NAT connection.

**Screenshot - Initial GNS3 Setup**

![Initial GNS3 setup](<screenshots/GNS3 Intial setup.png>)

I separated the devices into four VLANs:

| Department | VLAN | Subnet |
| --- | --- | --- |
| HR | 10 | 192.168.10.0/24 |
| Finance | 20 | 192.168.20.0/24 |
| IT | 30 | 192.168.30.0/24 |
| Servers | 40 | 192.168.40.0/24 |

I assigned the devices to their VLANs and configured 802.1Q trunks between the switches and toward FW-ONPREM.

**Screenshot - Access Switch VLAN Configuration**

![Access switch VLAN configuration](<screenshots/SW-ACC VLAN Config.png>)

**Screenshot - Core Switch VLAN Configuration**

![Core switch VLAN configuration](<screenshots/SW-CORE VLAN.png>)

I also configured VLAN subinterfaces on FW-ONPREM so each subnet had a gateway.

This separated the departments at Layer 2 while allowing the firewall to control routed communication between them.

**Screenshot - Firewall VLAN Interfaces**

![Firewall VLAN interfaces](<screenshots/Firewall VLAN segmentation.png>)

### Step 2 - Configuring DHCP and Static Addressing

Next, I configured DHCP on FW-ONPREM for the HR, Finance, and IT networks.

This allowed employee devices to receive their IP addresses, gateway information, and DNS settings automatically.

I checked the client devices to make sure they received addresses from the correct department subnets.

**Screenshot - HR DHCP Addressing**

![HR DHCP addressing](<screenshots/HR-PC DHCP.png>)

**Screenshot - Finance DHCP Addressing**

![Finance DHCP addressing](<screenshots/FIN-PC DHCP - Copy.png>)

For SRV-01, I used the static address:

```text
192.168.40.10/24
```

I checked the server's network configuration and verified its local connectivity.

**Screenshot - Server Static Address**

![Server static IP configuration](<screenshots/SRV-01 Static IP.png>)

### Step 3 - Configuring Local External Connectivity

After configuring the local addresses, I worked on external connectivity.

Traffic from the local network passed through FW-ONPREM, ISP-RTR, and the NAT connection.

I tested external connectivity from IT-PC using:

```text
ping 8.8.8.8
```

The ping received replies after I corrected the NAT configuration.

This verified that the local client could reach an external IP address through the routing and NAT setup.

The NAT issue and its resolution are explained in the Lessons Learned section.

### Step 4 - Testing Local Access Controls

I configured access controls so IT-PC could reach SRV-01, while HR-PC could not reach the same server.

From IT-PC, I tested:

```text
ping 192.168.40.10
```

The ping received replies.

**Screenshot - IT Access to the Local Server**

![IT successfully reaching the local server](<screenshots/Firewall Authorized traffic - Copy.png>)

I then tested the same destination from HR-PC.

After confirming that HR-PC received a valid DHCP address, I repeated the ping. The requests timed out.

**Screenshot - HR Access to the Local Server Blocked**

![HR receiving a DHCP address and failing to reach the local server](<screenshots/TEST CASE 3 FINALLY.png>)

The results were:

| Source | Destination | Result |
| --- | --- | --- |
| IT-PC | SRV-01 at 192.168.40.10 | Received replies |
| HR-PC | SRV-01 at 192.168.40.10 | Requests timed out |

This helped me verify that communication between VLANs was being controlled.

### Step 5 - Creating the AWS Network

Next, I created the AWS portion of the project.

I created a VPC named `Copper-Creek-Legal-VPC` using:

```text
172.16.0.0/16
```

**Screenshot - AWS VPC**

![AWS VPC configuration](<screenshots/AWS VPC setup.png>)

Inside the VPC, I created two subnets:

| Subnet | Address Range | Purpose |
| --- | --- | --- |
| Application | 172.16.10.0/24 | Application server and VPN instance |
| Database | 172.16.20.0/24 | Database server |

**Screenshot - AWS Subnets**

![AWS application and database subnets](<screenshots/AWS Subnet created.png>)

I created APP-01 and DB-01 in their assigned subnets.

**Screenshot - Created EC2 Instances**

![Created AWS instances](<screenshots/AWS Instances created.png>)

The instances used these private addresses:

| Instance | Private IP Address |
| --- | --- |
| APP-01 | 172.16.10.191 |
| DB-01 | 172.16.20.10 |
| Copper-Creek-VPN | 172.16.10.157 |

APP-01 used an AWS-assigned private address. DB-01 used the fixed private address `172.16.20.10`.

**Screenshot - APP-01 Network Settings**

![APP-01 network settings](<screenshots/APP 1 network settings Test 5.png>)

**Screenshot - DB-01 Network Settings**

![DB-01 network settings](<screenshots/DB Network settings Test 5.png>)

I reviewed the route tables to confirm that the VPC local route was present.

**Screenshot - AWS Route Table**

![AWS route table](<screenshots/AWS Route table.png>)

I also checked the subnet associations and configured the application subnet's default route through the internet gateway.

**Screenshot - Subnet Association**

![AWS route table subnet association](<screenshots/AWS Subnet Assosication.png>)

**Screenshot - Application Route Table**

![Application subnet route table](<screenshots/APP Route .png>)

### Step 6 - Testing AWS Connectivity and Security Groups

I tested external connectivity from APP-01 using:

```bash
ping 8.8.8.8
```

APP-01 received replies.

**Screenshot - APP-01 External Connectivity**

![APP-01 reaching an external IP address](<screenshots/APP 1 Connection to External Resources.png>)

I then tested communication from APP-01 to DB-01.

In `Copper-Creek-DB-SG`, I allowed ICMP traffic from the application subnet:

```text
172.16.10.0/24
```

From APP-01, I ran:

```bash
ping 172.16.20.10
```

The ping succeeded with the rule enabled.

**Screenshot - Successful Application-to-Database Ping**

![Successful ping from APP-01 to DB-01](<screenshots/successful ping test 6.png>)

I then temporarily removed the ICMP rule and repeated the test.

**Screenshot - Database Security Group Change**

![Database security group after the rule change](<screenshots/DB rule change Test 6.png>)

This time, the ping failed.

**Screenshot - Failed Application-to-Database Ping**

![Failed ping from APP-01 to DB-01](<screenshots/Failed ping Test 6 - Copy.png>)

After completing the test, I restored the ICMP rule.

| Security Group Configuration | Result |
| --- | --- |
| ICMP allowed from the application subnet | Ping succeeded |
| ICMP rule removed | Ping failed |

This showed that the security group controlled the tested connection even though the network route was still available.

### Step 7 - Configuring the IPsec VPN

After verifying the local and AWS networks, I connected them through an IPsec VPN.

I used VyOS on FW-ONPREM for the local endpoint and Libreswan on Copper-Creek-VPN for the AWS endpoint.

| Setting | Configuration |
| --- | --- |
| On-premises endpoint | FW-ONPREM |
| AWS endpoint | Copper-Creek-VPN |
| VyOS peer name | AWS-VPN |
| On-premises protected network | 192.168.0.0/16 |
| AWS protected network | 172.16.0.0/16 |

The tunnel connected through the AWS VPN instance's public IP address.

I enabled IP forwarding on the AWS instance and configured matching IKE and ESP settings on both endpoints.

I also added an AWS route for `192.168.0.0/16` that pointed toward the VPN instance's network interface.

**Screenshot - AWS Route Back to the Local Network**

![AWS route pointing to the VPN instance network interface](<screenshots/test case 7 agian.png>)

Once the configuration was complete, I checked the VyOS IPsec status and confirmed that the tunnel was up.

**Screenshot - VPN Connection Status**

![VPN connection status](<screenshots/VPN Connected test 7.png>)

From IT-PC, I tested APP-01 using its private address:

```text
ping 172.16.10.191
```

The ping succeeded.

I also checked the IPsec counters and saw packets and bytes moving in both directions.

**Screenshot - IPsec Status and Traffic Counters**

![IPsec status and traffic counters](<screenshots/firewall VPN proof test 7.png>)

### Step 8 - Testing Hybrid Access Controls

For the final test, I wanted to verify that the VPN did not give every local department access to APP-01.

In `Copper-Creek-APP-SG`, I allowed ICMP from the IT subnet:

```text
192.168.30.0/24
```

I did not add an equivalent rule for the HR subnet.

**Screenshot - Application Security Group Rules**

![Application security group inbound rules](<screenshots/inbound rules test 8 - Copy.png>)

I then tested the same destination from IT-PC and HR-PC:

```text
ping 172.16.10.191
```

IT-PC received replies.

**Screenshot - IT Access to AWS**

![IT-PC successfully reaching APP-01 through the VPN](<screenshots/VPN success test 8.png>)

HR-PC did not receive replies.

**Screenshot - HR Access to AWS Blocked**

![HR-PC failing to reach APP-01](<screenshots/HR PC cannot reach test 8.png>)

The results were:

| Source | Destination | Result |
| --- | --- | --- |
| IT-PC | APP-01 at 172.16.10.191 | Received replies |
| HR-PC | APP-01 at 172.16.10.191 | Did not receive replies |

Both departments had a route toward AWS, but only the IT connection was permitted by the tested security-group configuration.

## 6. Validation

After completing the configuration, I reviewed the local network, AWS resources, and VPN connection.

I used eight test cases to document the results.

### Test Results

| Test | Result |
| --- | --- |
| Verified local VLAN assignments and trunk configuration | Pass |
| Verified DHCP, static addressing, and local external connectivity | Pass |
| Verified IT access and blocked HR access to SRV-01 | Pass |
| Verified AWS subnet configuration and permitted communication | Pass |
| Verified AWS addressing and APP-01 external connectivity | Pass |
| Verified database security-group allow and block behavior | Pass |
| Verified VPN status, private connectivity, and traffic counters | Pass |
| Verified IT access and blocked HR access to APP-01 | Pass |

The configurations I tested worked as expected.

IT-PC could reach the approved local and cloud destinations, while the equivalent HR tests failed.

The AWS security-group test also showed that removing the ICMP rule blocked the connection from APP-01 to DB-01.

Most of the connectivity tests used ping. These results verify ICMP reachability and do not prove that application services or every other protocol would behave the same way.

## 7. Security Considerations

### Least Privilege

I limited the tested access based on department.

IT was allowed to reach the local server and AWS application server. HR was not given the same ICMP access.

This helped me practice allowing specific connections instead of giving every subnet unrestricted access.

### Network Segmentation

I used VLANs locally and separate subnets in AWS to organize the network.

The firewall and security groups provided access controls between those networks.

### VPN Connectivity

The IPsec tunnel provided an encrypted connection between the local and AWS environments.

I still needed access controls because the VPN itself did not decide which departments should reach each resource.

### Credential Security

VPN secrets, private keys, and passwords should not be included in a public repository.

Before publishing screenshots, I would also review them for account details, public endpoint addresses, and unrelated personal information.

## 8. Lessons Learned

This project helped me understand how local networking and AWS networking work together.

I gained practice configuring VLANs, DHCP, routing, NAT, security groups, and an IPsec VPN.

One of the main things I learned was that a working route or VPN does not automatically mean a connection should be allowed.

I also learned why it is useful to test both permitted and blocked traffic. A successful connection showed that the path worked, while the blocked tests helped me check the access controls.

### Challenges I Encountered

#### DHCP Configuration

The DHCP settings on VyOS did not save correctly at first.

I corrected the configuration and had the client request a new lease.

After that, the client received the correct address, gateway, and DNS settings.

**Screenshot - DHCP Configuration Troubleshooting**

![DHCP configuration issue and resolution](<screenshots/Firewall DHCP ISSUE and Resolution - Copy.png>)

#### NAT Configuration

FW-ONPREM could reach ISP-RTR but could not reach `8.8.8.8`.

I found that the existing masquerade rule only covered `192.168.0.0/16`.

The firewall's traffic used the `198.51.100.0/30` transit network, so it was not covered by that rule.

I added another masquerade rule for the transit network and verified that external connectivity worked afterward.

#### VPN Software

I initially tried to use StrongSwan, but it was not available in the Amazon Linux 2023 lab environment.

I switched to Libreswan and used it for the AWS VPN endpoint.

#### VyOS Peer Configuration

I received an error when I used the AWS IP address as the VPN peer name.

I changed the name to `AWS-VPN` and entered the remote address separately.

After matching the IKE and ESP settings and adding the routes, the tunnel came up.

#### Checking Client Addressing Before Testing

During the local HR test, the first failed connection happened before HR-PC had a valid IP address.

That result alone did not verify the firewall restriction.

I requested a DHCP lease, confirmed the client received `192.168.10.100/24`, and repeated the test. The requests still timed out.

This helped me understand why I needed to verify client addressing before using a failed ping as evidence of an access restriction.

### Final Thoughts

This project gave me hands-on experience building and troubleshooting a hybrid network.

I was able to connect the local environment to AWS, verify private connectivity, and test access restrictions across both environments.

For future improvements, I would like to add sanitized configuration files, automate the AWS setup with Terraform, and test application ports in addition to ping.

I would also like to explore VPC Flow Logs and VPN redundancy.

## References

- [AWS Documentation - Subnets for Your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)
- [AWS Documentation - Subnet Route Tables](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-route-tables.html)
- [AWS Documentation - Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [AWS Documentation - Internet Gateways](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
- [AWS Documentation - Routing Options and Middlebox Appliances](https://docs.aws.amazon.com/vpc/latest/userguide/route-table-options.html)



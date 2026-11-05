# Project – Implement Virtual Networking in Microsoft Azure (AZ-104 Lab 04)

![Azure](https://img.shields.io/badge/Microsoft%20Azure-Virtual%20Networking-0078D4?style=flat)
![Status](https://img.shields.io/badge/Status-Completed-success)
![Domain](https://img.shields.io/badge/Domain-VNets%20%7C%20NSG%20%7C%20ASG%20%7C%20DNS%20%7C%20ARM-blueviolet)

---

## Overview

This project implements a two-network Azure environment for a fictional organization with a **core services** network and a **manufacturing** network. It covers virtual network and subnet design, Infrastructure as Code with an edited ARM template, traffic filtering with **network security groups (NSGs)** and **application security groups (ASGs)**, Windows Server VM deployment, and both **public and private Azure DNS zones**, all documented with screenshots from the Azure portal.

Based on the AZ-104 *Implement Virtual Networking* lab, following the walkthrough by Gbenga Adekunle: [YouTube video](https://youtu.be/qKKVV3Ym-yQ).

---

## Environment

| Tool | Purpose |
|------|---------|
| Azure Portal | Resource creation and administration |
| ARM templates (`template.json`, `parameters.json`) | Infrastructure as Code deployment of the second virtual network |
| Visual Studio Code | Editing the exported template and parameters file |
| Windows Server 2025 Datacenter VMs (Standard_D2s_v3) | Workloads placed in each virtual network |
| Azure DNS (public and private zones) | Name resolution |
| Command Prompt (`nslookup`) | Verifying public DNS resolution |
| draw.io | Architecture diagram |
| GitHub | Documentation and version control |

**Resource group:** `RG4AZ104` | **Region:** East US

---

## Architecture

![Architecture Diagram](images/01-architecture-diagram.png)
*Architecture: two virtual networks, an NSG with an ASG rule, and two VMs*

### Network Layout

| Virtual Network | Address Space | Subnet | Subnet Range |
|-----------------|---------------|--------|--------------|
| CoreServicesVnet | 10.20.0.0/16 | SharedServicesSubnet | 10.20.10.0/24 |
| CoreServicesVnet | 10.20.0.0/16 | DatabaseSubnet | 10.20.20.0/24 |
| ManufacturingVnet | 10.30.0.0/16 | SensorSubnet1 | 10.30.20.0/24 |
| ManufacturingVnet | 10.30.0.0/16 | SensorSubnet2 | 10.30.21.0/24 |

### Resources Deployed

| Resource | Details |
|----------|---------|
| asg-web | Application security group |
| myNSGSecure | Network security group with custom inbound and outbound rules |
| VM1 | Private IP 10.20.10.4, CoreServicesVnet / SharedServicesSubnet, member of asg-web |
| VM2 | Private IP 10.30.20.4, ManufacturingVnet / SensorSubnet1 |
| Luiscorp.xyz | Public DNS zone |
| privateluiscorp.xyz | Private DNS zone linked to ManufacturingVnet |

---

## Lab Tasks

---

### Task 1 – Create CoreServicesVnet

**Actions Taken:**
1. Created the virtual network **CoreServicesVnet** in resource group `RG4AZ104`, region East US
2. Set the address space to **10.20.0.0/16**
3. Created **SharedServicesSubnet** (10.20.10.0/24) and **DatabaseSubnet** (10.20.20.0/24)
4. Confirmed the deployment succeeded

**Principle Applied:** Planning address space up front so subnets are sized for growth and do not overlap with the second network.

![Create CoreServicesVnet](images/02-corevnet-create-basics.png)
*Basics tab: CoreServicesVnet in RG4AZ104, East US*

![Address Space and Subnets](images/03-corevnet-address-space-subnets.png)
*Address space 10.20.0.0/16 with SharedServicesSubnet and DatabaseSubnet*

![CoreServicesVnet Deployment](images/04-corevnet-deployment-complete.png)
*Deployment succeeded*

---

### Task 2 – Create ManufacturingVnet from an Edited ARM Template

**Scenario:** Instead of rebuilding the second network by hand, I reused the first network's exported template and edited it.

**Actions Taken:**
1. Downloaded `template.json` and `parameters.json` from the CoreServicesVnet deployment
2. In Visual Studio Code, used Find and Replace on `template.json` to rename and re-address the network:

| Find | Replace With |
|------|--------------|
| `CoreServicesVnet` | `ManufacturingVnet` |
| `10.20.0.0` | `10.30.0.0` |
| `SharedServicesSubnet` | `SensorSubnet1` |
| `10.20.10.0` | `10.30.20.0` |
| `DatabaseSubnet` | `SensorSubnet2` |
| `10.20.20.0` | `10.30.21.0` |

3. Saved the template and updated `parameters.json` so the parameter name matches the renamed network
4. Loaded the edited template through **Deploy a custom template > Edit template** in the portal (1 parameter, 0 variables, 3 resources)
5. Deployed to `RG4AZ104` and confirmed success

**Principle Applied:** Infrastructure as Code. One template produced a consistent second network with different names and address ranges.

![Rename the Virtual Network](images/05-template-rename-vnet.png)
*Replacing CoreServicesVnet with ManufacturingVnet in template.json*

![Replace the Address Space](images/06-template-address-space.png)
*Replacing 10.20.0.0 with 10.30.0.0*

![Rename Subnet 1](images/07-template-subnet1-name.png)
*Replacing SharedServicesSubnet with SensorSubnet1*

![Subnet 1 Prefix](images/08-template-subnet1-prefix.png)
*Replacing 10.20.10.0 with 10.30.20.0*

![Rename Subnet 2](images/09-template-subnet2-name.png)
*Replacing DatabaseSubnet with SensorSubnet2*

![Subnet 2 Prefix](images/10-template-subnet2-prefix.png)
*Replacing 10.20.20.0 with 10.30.21.0*

![Save the Template](images/11-template-save.png)
*Saving the edited template*

![Parameters File](images/12-parameters-file-edit.png)
*Updating parameters.json for the renamed network*

![Edit Template in the Portal](images/13-custom-deployment-edit-template.png)
*Edit template blade showing the ManufacturingVnet parameter and 3 resources*

![ManufacturingVnet Deployment](images/14-manufacturingvnet-deployment-complete.png)
*ManufacturingVnet deployment succeeded*

---

### Task 3 – Application Security Group and Network Security Group

**Actions Taken:**
1. Created the application security group **asg-web** in `RG4AZ104` (East US)
2. Created the network security group **myNSGSecure** in `RG4AZ104` (East US)
3. Associated myNSGSecure with **SharedServicesSubnet** in CoreServicesVnet (see architecture diagram)
4. Added an **inbound** rule: priority 100, name **AllowASG**, TCP ports 80 and 443, source **asg-web**, destination Any, action Allow
5. Added an **outbound** rule: priority 300, name **DenyAnyCustom8080Outbound**, port 8080, destination Internet, action Deny

**Principle Applied:** Least privilege. Web traffic is allowed only from members of the application security group, and a specific outbound port is blocked. The default rules (`AllowVnetInBound`, `DenyAllInBound`, `AllowInternetOutBound`, `DenyAllOutBound` and others) remain below the custom rules.

![Create asg-web](images/15-asg-web-create.png)
*Application security group asg-web: validation passed*

![Create myNSGSecure](images/16-nsg-create.png)
*Network security group myNSGSecure: validation passed*

![NSG Inbound Rule](images/17-nsg-inbound-rule-allowasg.png)
*Inbound rule AllowASG (priority 100, TCP 80 and 443, source asg-web)*

![NSG Outbound Rule](images/18-nsg-outbound-rule-deny8080.png)
*Outbound rule DenyAnyCustom8080Outbound (priority 300, port 8080, destination Internet, Deny)*

---

### Task 4 – Deploy Virtual Machines

#### VM1: CoreServicesVnet

**Actions Taken:**
1. Created **VM1** in East US with image **Windows Server 2025 Datacenter – x64 Gen2**, size **Standard_D2s_v3**, security type Trusted launch
2. Set **Public inbound ports** to **None**
3. Placed VM1 in **CoreServicesVnet / SharedServicesSubnet** (10.20.10.0/24)
4. Confirmed deployment succeeded
5. Added VM1's network interface (`vm1534`) to the application security group **asg-web**
6. Verified VM1's private IP address: **10.20.10.4**

**Principle Applied:** Reducing attack surface. VM1 has no public inbound ports, and the ASG ties it to the web-traffic rule.

![VM1 Basics](images/19-vm1-create-basics.png)
*VM1 basics: East US, Windows Server 2025 Datacenter Gen2, Standard_D2s_v3*

![VM1 Inbound Ports](images/20-vm1-inbound-ports-none.png)
*Public inbound ports set to None*

![VM1 Networking](images/21-vm1-networking-coreservicesvnet.png)
*VM1 placed in CoreServicesVnet / SharedServicesSubnet*

![VM1 Deployment](images/22-vm1-deployment-complete.png)
*VM1 deployment succeeded*

![VM1 ASG Membership](images/23-vm1-asg-membership.png)
*VM1's network interface is a member of asg-web*

![VM1 Overview](images/24-vm1-overview-ip-addresses.png)
*VM1 overview showing private IP 10.20.10.4 and its virtual network and subnet*

#### VM2: ManufacturingVnet

**Actions Taken:**
1. Created **VM2** with the same image and size
2. Placed VM2 in **ManufacturingVnet / SensorSubnet1** (10.30.20.0/24)
3. Allowed inbound **RDP (3389)** for remote access
4. Confirmed deployment succeeded
5. Verified VM2's private IP address: **10.30.20.4**

![VM2 Basics](images/25-vm2-create-basics.png)
*VM2 basics*

![VM2 Inbound Ports](images/26-vm2-inbound-ports-rdp.png)
*Inbound port rule: RDP (3389)*

![VM2 Networking](images/27-vm2-networking-manufacturingvnet.png)
*VM2 placed in ManufacturingVnet / SensorSubnet1*

![VM2 Deployment](images/28-vm2-deployment-complete.png)
*VM2 deployment succeeded*

![VM2 Overview](images/29-vm2-overview-ip-addresses.png)
*VM2 overview showing private IP 10.30.20.4 and its virtual network and subnet*

---

### Task 5 – Public Azure DNS Zone

**Actions Taken:**
1. Created the public DNS zone **Luiscorp.xyz** in `RG4AZ104`
2. Confirmed deployment and reviewed the four Azure name servers assigned to the zone
3. Added an **A record** named **www** (TTL 1 hour) pointing to **10.20.10.4**
4. Verified the record set list shows the NS, SOA, and www records
5. Tested resolution with `nslookup www.Luiscorp.xyz ns1-01.azure-dns.com.`, which returned **10.20.10.4**

**Principle Applied:** Understanding how a hosted DNS zone resolves names and verifying records directly against the authoritative Azure name server.

![Create DNS Zone](images/30-public-dns-zone-create.png)
*Create a DNS zone: Luiscorp.xyz in RG4AZ104*

![DNS Zone Deployment](images/31-public-dns-zone-deployment-complete.png)
*DNS zone deployment succeeded*

![DNS Zone Overview](images/32-public-dns-zone-overview-nameservers.png)
*Zone overview showing the four Azure name servers*

![Add www Record](images/33-public-dns-www-record-name-type.png)
*Add record set: name www, type A, TTL 1 hour*

![www Record IP Address](images/34-public-dns-www-record-ip.png)
*A record IP address 10.20.10.4*

![Public DNS Record Sets](images/35-public-dns-recordsets.png)
*Record sets: NS, SOA, and www A record (10.20.10.4)*

![nslookup Test](images/36-public-dns-nslookup-test.png)
*nslookup against ns1-01.azure-dns.com resolves www.Luiscorp.xyz to 10.20.10.4*

---

### Task 6 – Private Azure DNS Zone

**Actions Taken:**
1. Created the private DNS zone **privateluiscorp.xyz** in `RG4AZ104` (East US)
2. Opened **Virtual Network Links** and confirmed there were no links yet
3. Added a virtual network link named **manufacturing-link** to **ManufacturingVnet**
4. Confirmed the link status is **Completed** with auto-registration disabled
5. Added an **A record** named **vm2** (TTL 1 hour) pointing to **10.30.20.4**
6. Verified the record sets: the vm2 A record and the SOA record

**Principle Applied:** Private name resolution. A private DNS zone only resolves names for virtual networks that are explicitly linked to it.

![Create Private DNS Zone](images/37-private-dns-zone-review-create.png)
*Create Private DNS Zone: privateluiscorp.xyz, validation passed*

![Virtual Network Links Empty](images/38-private-dns-vnet-links-empty.png)
*Virtual Network Links before linking: 0 links*

![Add Virtual Network Link](images/39-private-dns-add-vnet-link.png)
*Adding manufacturing-link to ManufacturingVnet*

![Virtual Network Link Completed](images/40-private-dns-vnet-link-complete.png)
*manufacturing-link status Completed, auto-registration disabled*

![Add vm2 Record](images/41-private-dns-vm2-record.png)
*Adding an A record for vm2 pointing to 10.30.20.4*

![Private DNS Record Sets](images/42-private-dns-recordsets.png)
*Record sets: vm2 A record and SOA record*

---

## Skills Demonstrated

| Skill | How It Was Applied |
|-------|--------------------|
| Virtual Network Design | Planned non-overlapping address spaces and subnets for two networks |
| Infrastructure as Code | Exported, edited, and redeployed an ARM template and parameters file |
| Network Security Groups | Created custom inbound and outbound rules with explicit priorities |
| Application Security Groups | Grouped VM network interfaces so rules reference workloads instead of IP addresses |
| Least Privilege Networking | Allowed only the required web ports, denied a specific outbound port, and gave VM1 no public inbound ports |
| Virtual Machine Deployment | Deployed Windows Server 2025 VMs into specific subnets |
| Azure DNS (Public) | Created a zone, added an A record, and verified it with `nslookup` |
| Azure DNS (Private) | Created a private zone, linked it to a virtual network, and added a record |
| Documentation | Screenshot evidence for each step of the build |

---

## Lessons Learned

**Exported templates make networks repeatable.** Editing an exported ARM template was faster and less error-prone than rebuilding the second network by hand. The parameters file has to match the template's renamed parameter, or the deployment will not line up.

**ASGs keep rules readable.** Referencing `asg-web` in the inbound rule means new web servers only need to join the group. The rule does not need to change when IP addresses do.

**NSG priority decides the outcome.** Custom rules (100 and 300) are evaluated before the default rules, which start at 65000. That is why the outbound rule for port 8080 takes effect ahead of the default allow-internet-outbound rule.

**Private DNS depends on VNet links.** The private zone resolves names only for networks linked to it. In this lab, only ManufacturingVnet is linked to `privateluiscorp.xyz`.

---

## References

- [Lab 04 – Implement Virtual Networking (Microsoft Learn AZ-104 labs)](https://microsoftlearning.github.io/AZ-104-MicrosoftAzureAdministrator/Instructions/Labs/LAB_04-Implement_Virtual_Networking.html)
- [Walkthrough video by Gbenga Adekunle](https://youtu.be/qKKVV3Ym-yQ)
- [Network security groups overview](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
- [Application security groups](https://learn.microsoft.com/en-us/azure/virtual-network/application-security-groups)
- [Azure DNS overview](https://learn.microsoft.com/en-us/azure/dns/dns-overview)
- [Azure Private DNS overview](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview)

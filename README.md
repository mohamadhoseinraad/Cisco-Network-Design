# Cisco Network Project: Headquarters & Branch Connectivity

![Cisco Packet Tracer](https://img.shields.io/badge/Cisco-Packet_Tracer-1BA0D7?style=flat&logo=cisco)
![University Project](https://img.shields.io/badge/University-Project-blue)
![Status](https://img.shields.io/badge/Status-Completed-success)

## 📋 Project Overview
This repository contains my implementation of a university network project for **Computer Networks 2** course under Dr. Fatemeh Rezaei. The project simulates a real-world enterprise network with headquarters and branch office connected securely over the internet.

**Student ID:** 40119783

## 🏗 Network Architecture

### Network Components
| Device | Role | Location |
|--------|------|----------|
| R1-HQ | Main Router (Router-on-a-Stick) | Headquarters |
| R3-Branch | Branch Router | Branch Office |
| ISP Router | Internet Simulation | Cloud |
| HQ-Switch | Layer 2 Switch with VLANs | Headquarters |
| Branch-Switch | Branch Switch | Branch Office |
| DHCP Server | Centralized IP Allocation | HQ (VLAN 30) |
| Web Server | HTTPS Service | Branch (172.17.17.100) |

### IP Addressing Scheme
| Network | Subnet | VLAN | Gateway |
|---------|--------|------|---------|
| Users VLAN 10 | 172.16.10.0/24 | 10 | 172.16.10.1 |
| Users VLAN 20 | 172.16.20.0/24 | 20 | 172.16.20.1 |
| DHCP Server | 172.16.30.0/24 | 30 | 172.16.30.1 |
| Branch LAN | 172.17.17.0/24 | - | 172.17.17.1 |
| GRE Tunnel | 10.10.10.0/30 | - | - |
| WAN Links | 100.100.100.0/30, 200.200.200.0/30 | - | - |

## 🔧 Implemented Features

### ✅ Core Requirements
1. **VLAN Implementation** - Separated HQ network into VLAN 10, 20, and 30
2. **Router-on-a-Stick** - R1 configured with subinterfaces for inter-VLAN routing
3. **DHCP Service** - Central DHCP server assigns IPs to VLAN 10 & 20 clients
4. **NAT/PAT** - Enabled on R1 and R3 for internet access (ping 8.8.8.8)
5. **GRE Tunnel** - Site-to-site tunnel between R1 and R3
6. **OSPF Routing** - Dynamic routing over GRE tunnel

### ⭐ Bonus Features
7. **ACL Security** - Web server (172.17.17.100) accepts HTTPS only, blocks ICMP

## 📁 Repository Contents

### Configuration Files
- [R1-HQ Complete Configuration](Configs/R1-HQ_config.txt)
- [R3-Branch Complete Configuration](Configs/R3-Branch_config.txt)
- [HQ-Switch Configuration](Configs/HQ-Switch_config.txt)
- [DHCP Server Setup](Configs/DHCP_Server_Config.md)

### Step-by-Step Documentation
- [01 - VLAN Design and Trunking](Documentation/01-VLAN_Design.md)
- [02 - Router-on-a-Stick Implementation](Documentation/02-Router_on_Stick.md)
- [03 - NAT/PAT Configuration](Documentation/03-NAT_PAT_Configuration.md)
- [04 - DHCP Server and Relay](Documentation/04-DHCP_Configuration.md)
- [05 - GRE Tunnel Setup](Documentation/05-GRE_Tunnel_Setup.md)
- [06 - OSPF Dynamic Routing](Documentation/06-OSPF_Routing.md)
- [07 - ACL Security Configuration](Documentation/07-ACL_Security.md)
- [08 - Verification and Testing](Documentation/08-Verification_Tests.md)

## 📝 Key Configuration Highlights

### 1. VLAN Configuration on HQ-Switch
```cisco
vlan 10
 name Users_VLAN10
vlan 20
 name Users_VLAN20
vlan 30
 name DHCP_Server
!
interface f0/1
 switchport mode access
 switchport access vlan 10
!
interface f0/2
 switchport mode access
 switchport access vlan 20
!
interface g0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30

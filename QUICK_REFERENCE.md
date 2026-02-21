# Quick Command Reference

## Essential Verification Commands

### Router Status
```cisco
show ip interface brief          # Interface status and IPs
show ip route                    # Routing table
show running-config              # Current configuration
show startup-config              # Saved configuration
```

### VLAN Commands (Switch)
```cisco
show vlan brief                  # VLAN assignments
show interfaces trunk            # Trunk status
show interfaces status           # Port status
```

### OSPF Commands
```cisco
show ip ospf neighbor            # OSPF neighbors
show ip ospf interface brief     # OSPF-enabled interfaces
show ip route ospf              # OSPF routes
show ip ospf database           # OSPF link-state database
```

### NAT Commands
```cisco
show ip nat translations        # Active NAT translations
show ip nat statistics          # NAT statistics
clear ip nat translation *      # Clear all NAT entries
```

### Tunnel Commands
```cisco
show interfaces tunnel 0        # GRE tunnel status
show ip interface brief | include Tunnel
ping 10.10.10.2                # Test tunnel connectivity
```

### ACL Commands
```cisco
show access-lists              # Display all ACLs
show ip interface Gi0/1        # ACL applied to interface
show access-lists WEB-SERVER-ACL  # Specific ACL with counters
```

### DHCP Commands
```cisco
show running-config | section dhcp
show ip dhcp binding           # DHCP leases (if router is DHCP server)
```

## Common Configuration Snippets

### Basic Router Setup
```cisco
enable
configure terminal
hostname R1-HQ
no ip domain-lookup
enable secret class
line con 0
 password cisco
 login
 logging synchronous
line vty 0 4
 password cisco
 login
service password-encryption
banner motd # Unauthorized Access Prohibited #
end
copy running-config startup-config
```

### Create VLAN on Switch
```cisco
enable
configure terminal
vlan 10
 name Users_VLAN10
exit
interface fastEthernet 0/1
 switchport mode access
 switchport access vlan 10
exit
```

### Configure Trunk Port
```cisco
interface gigabitEthernet 0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
exit
```

### Router Subinterface (Router-on-Stick)
```cisco
interface gigabitEthernet 0/1.10
 encapsulation dot1Q 10
 ip address 172.16.10.1 255.255.255.0
 ip helper-address 172.16.30.10
exit
```

### NAT Configuration
```cisco
! Inside interfaces
interface gigabitEthernet 0/1.10
 ip nat inside
exit

! Outside interface
interface gigabitEthernet 0/0
 ip nat outside
exit

! ACL and NAT overload
access-list 1 permit 172.16.0.0 0.0.255.255
ip nat inside source list 1 interface gigabitEthernet 0/0 overload
```

### GRE Tunnel
```cisco
interface Tunnel 0
 ip address 10.10.10.1 255.255.255.252
 tunnel source gigabitEthernet 0/0
 tunnel destination 200.200.200.2
exit
```

### OSPF Configuration
```cisco
router ospf 1
 router-id 1.1.1.1
 network 172.16.10.0 0.0.0.255 area 0
 network 10.10.10.0 0.0.0.3 area 0
 passive-interface gigabitEthernet 0/1.10
exit
```

### Extended ACL
```cisco
ip access-list extended WEB-SERVER-ACL
 deny icmp any host 172.17.17.100 echo
 permit tcp any host 172.17.17.100 eq 443
 permit ip any any
exit

interface gigabitEthernet 0/1
 ip access-group WEB-SERVER-ACL in
exit
```

### Static Route
```cisco
ip route 0.0.0.0 0.0.0.0 100.100.100.1
```

## Troubleshooting Commands

### Connectivity Issues
```cisco
ping 8.8.8.8                    # Test internet
traceroute 172.17.17.10         # Trace path
show ip route 172.17.17.0       # Check specific route
show arp                        # ARP table
```

### Interface Issues
```cisco
show ip interface brief         # All interfaces status
show interfaces Gi0/0           # Detailed interface info
show controllers                # Physical layer info
```

### Debug Commands (use sparingly)
```cisco
debug ip ospf events           # OSPF events
debug ip nat                   # NAT translations
debug ip packet                # IP packet flow

! Always turn off debug when done
undebug all    or    no debug all
```

### Diagnostic Commands
```cisco
show cdp neighbors             # Directly connected Cisco devices
show version                   # IOS version and uptime
show processes cpu             # CPU utilization
show memory                    # Memory usage
```

## Configuration Management

### Save Configuration
```cisco
copy running-config startup-config
! or shorthand:
write memory
! or even shorter:
wr
```

### Backup Configuration
```cisco
show running-config            # Copy output to text file
```

### Reset Configuration
```cisco
erase startup-config
reload
! When prompted, choose "no" to save
```

### Load Configuration
```cisco
configure terminal
! Paste configuration here
end
copy running-config startup-config
```

## Password Recovery (Packet Tracer)

Packet Tracer doesn't usually lock you out, but on real routers:
```cisco
! During boot, press Ctrl+Break
rommon> confreg 0x2142
rommon> reset
! After boot:
enable
copy startup-config running-config
configure terminal
enable secret newpassword
config-register 0x2102
end
copy running-config startup-config
reload
```

## Common Port Numbers

| Port | Service | Protocol |
|------|---------|----------|
| 20/21 | FTP | TCP |
| 22 | SSH | TCP |
| 23 | Telnet | TCP |
| 25 | SMTP | TCP |
| 53 | DNS | UDP/TCP |
| 67/68 | DHCP | UDP |
| 69 | TFTP | UDP |
| 80 | HTTP | TCP |
| 110 | POP3 | TCP |
| 143 | IMAP | TCP |
| 161/162 | SNMP | UDP |
| 443 | HTTPS | TCP |
| 3389 | RDP | TCP |

## Wildcard Mask Calculator

| Subnet Mask | Wildcard Mask |
|-------------|---------------|
| 255.255.255.255 (/32) | 0.0.0.0 |
| 255.255.255.252 (/30) | 0.0.0.3 |
| 255.255.255.248 (/29) | 0.0.0.7 |
| 255.255.255.240 (/28) | 0.0.0.15 |
| 255.255.255.224 (/27) | 0.0.0.31 |
| 255.255.255.192 (/26) | 0.0.0.63 |
| 255.255.255.128 (/25) | 0.0.0.127 |
| 255.255.255.0 (/24) | 0.0.0.255 |
| 255.255.254.0 (/23) | 0.0.1.255 |
| 255.255.252.0 (/22) | 0.0.3.255 |
| 255.255.0.0 (/16) | 0.0.255.255 |
| 255.0.0.0 (/8) | 0.255.255.255 |

## Keyboard Shortcuts

| Shortcut | Function |
|----------|----------|
| Ctrl+C | Cancel current command |
| Ctrl+Z | Exit to privileged mode |
| Ctrl+Shift+6 | Interrupt ping/traceroute |
| Tab | Complete command |
| ? | Context-sensitive help |
| Up Arrow | Previous command |
| Down Arrow | Next command |

## Configuration Modes

```
User EXEC Mode:              Router>
Privileged EXEC Mode:        Router#
Global Config Mode:          Router(config)#
Interface Config Mode:       Router(config-if)#
Router Config Mode:          Router(config-router)#
Line Config Mode:            Router(config-line)#
```

### Entering Modes
```cisco
Router> enable                        # User → Privileged
Router# configure terminal            # Privileged → Global Config
Router(config)# interface Gi0/0       # Global → Interface Config
Router(config)# router ospf 1         # Global → Router Config
Router(config)# line console 0        # Global → Line Config
```

### Exiting Modes
```cisco
Router(config-if)# exit              # Back one level
Router(config)# end                  # Back to privileged mode
Router# disable                      # Back to user mode
```

## IP Addressing Scheme (This Project)

| Network | IP Range | Purpose |
|---------|----------|---------|
| 172.16.10.0/24 | .1-.254 | HQ VLAN 10 (.1=gateway, .10-.59=DHCP) |
| 172.16.20.0/24 | .1-.254 | HQ VLAN 20 (.1=gateway, .10-.59=DHCP) |
| 172.16.30.0/24 | .1-.254 | HQ VLAN 30 (.1=gateway, .10=DHCP server) |
| 172.17.17.0/24 | .1-.254 | Branch LAN (.1=gateway, .100=web server) |
| 10.10.10.0/30 | .1-.2 | GRE Tunnel (.1=R1-HQ, .2=R3-Branch) |
| 100.100.100.0/30 | .1-.2 | WAN HQ-ISP (.1=ISP, .2=R1-HQ) |
| 200.200.200.0/30 | .1-.2 | WAN Branch-ISP (.1=ISP, .2=R3-Branch) |

## Quick Formulas

### Subnet Calculation
```
Number of hosts = 2^(host bits) - 2
Number of subnets = 2^(borrowed bits)
```

### OSPF Cost
```
Cost = Reference Bandwidth / Interface Bandwidth
Default Reference Bandwidth = 100 Mbps
```

### Wildcard Mask
```
Wildcard = 255.255.255.255 - Subnet Mask
Example: 255.255.255.255 - 255.255.255.0 = 0.0.0.255
```

---

**Print this page for quick reference during your networking labs!**

# Step 01 - VLAN Design and Trunking

## Overview
This document explains how to design and implement VLANs for network segmentation at the headquarters.

## Why VLANs?
- **Broadcast Domain Separation:** Reduces broadcast traffic and improves performance
- **Security:** Isolates different user groups (Users in VLAN 10 and 20, Servers in VLAN 30)
- **Organization:** Logical grouping of devices regardless of physical location
- **Scalability:** Easy to add new devices to existing VLANs

## VLAN Design for HQ

| VLAN ID | Name | Network | Purpose |
|---------|------|---------|---------|
| 10 | Users_VLAN10 | 172.16.10.0/24 | First group of users |
| 20 | Users_VLAN20 | 172.16.20.0/24 | Second group of users |
| 30 | Server_DHCP | 172.16.30.0/24 | DHCP Server |

## Implementation Steps

### Step 1: Create VLANs on HQ-Switch

```cisco
enable
configure terminal
hostname HQ-Switch

! Create VLAN 10
vlan 10
 name Users_VLAN10
 exit

! Create VLAN 20
vlan 20
 name Users_VLAN20
 exit

! Create VLAN 30
vlan 30
 name Server_DHCP
 exit
```

### Step 2: Assign Access Ports to VLANs

**For VLAN 10 Users:**
```cisco
! PC1 on FastEthernet0/1
interface FastEthernet0/1
 description PC1 - VLAN 10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 exit

! PC2 on FastEthernet0/2
interface FastEthernet0/2
 description PC2 - VLAN 10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 exit
```

**For VLAN 20 Users:**
```cisco
! PC3 on FastEthernet0/3
interface FastEthernet0/3
 description PC3 - VLAN 20
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 exit

! PC4 on FastEthernet0/4
interface FastEthernet0/4
 description PC4 - VLAN 20
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 exit
```

**For VLAN 30 Server:**
```cisco
! DHCP Server on FastEthernet0/5
interface FastEthernet0/5
 description DHCP Server
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 exit
```

### Step 3: Configure Trunk Port to Router

```cisco
! Trunk to R1-HQ on GigabitEthernet0/1
interface GigabitEthernet0/1
 description Trunk to R1-HQ
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 exit
```

**Important Notes about Trunking:**
- Trunk carries traffic for multiple VLANs
- Uses 802.1Q tagging to identify VLAN membership
- Native VLAN (default VLAN 1) carries untagged traffic
- Only specified VLANs (10, 20, 30) are allowed on this trunk

### Step 4: Save Configuration

```cisco
end
copy running-config startup-config
```

### Step 5: Verify VLAN Configuration

**Show VLANs:**
```cisco
show vlan brief
```

**Expected Output:**
```
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    
10   Users_VLAN10                     active    Fa0/1, Fa0/2
20   Users_VLAN20                     active    Fa0/3, Fa0/4
30   Server_DHCP                      active    Fa0/5
```

**Show Trunk:**
```cisco
show interfaces trunk
```

**Expected Output:**
```
Port        Mode         Encapsulation  Status        Native vlan
Gig0/1      on           802.1q         trunking      1

Port        Vlans allowed on trunk
Gig0/1      10,20,30
```

## Key Concepts

### Access Port vs Trunk Port
- **Access Port:** Belongs to one VLAN, connected to end devices (PCs, servers)
- **Trunk Port:** Carries multiple VLANs, connects switches to routers/switches

### 802.1Q Tagging
- Industry standard VLAN tagging protocol
- Adds 4-byte tag to Ethernet frames
- Tag includes VLAN ID (12 bits = 4094 VLANs possible)

### PortFast
- Immediately transitions to forwarding state
- Only use on access ports connected to end devices
- Reduces convergence time during device boot

## Common Issues and Troubleshooting

### Issue: Devices in same VLAN can't communicate
**Solutions:**
1. Check VLAN assignment: `show vlan brief`
2. Verify cable connections
3. Check if VLAN exists on the switch

### Issue: Trunk not passing VLAN traffic
**Solutions:**
1. Verify trunk mode: `show interfaces trunk`
2. Check allowed VLANs: `switchport trunk allowed vlan 10,20,30`
3. Ensure both sides are configured as trunk

### Issue: PC shows wrong VLAN
**Solutions:**
1. Check port assignment: `show running-config interface fa0/X`
2. Reassign to correct VLAN
3. Save configuration

## Testing
1. **Physical Layer:** Check link lights on switch ports
2. **VLAN Assignment:** Use `show vlan brief` to verify port membership
3. **Trunk Status:** Use `show interfaces trunk` to confirm trunking

## Next Steps
- Proceed to **02-Router_on_Stick.md** for inter-VLAN routing configuration

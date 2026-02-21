# Step 02 - Router-on-a-Stick Implementation

## Overview
Router-on-a-Stick is a technique to enable inter-VLAN routing using a single physical router interface with multiple logical subinterfaces.

## Why Router-on-a-Stick?
- **Cost-Effective:** Only one physical link needed between router and switch
- **Simple Design:** Fewer cables and interfaces to manage
- **Scalable:** Easy to add new VLANs by creating subinterfaces
- **Standard Practice:** Common in small to medium networks

## Concept Diagram
```
[HQ-Switch]---Trunk (802.1Q)---[R1-HQ Gi0/1]
                                    |
                    +---------------+---------------+
                    |               |               |
              Gi0/1.10        Gi0/1.20        Gi0/1.30
           (172.16.10.1)   (172.16.20.1)   (172.16.30.1)
            Gateway for     Gateway for     Gateway for
              VLAN 10        VLAN 20         VLAN 30
```

## Implementation Steps on R1-HQ

### Step 1: Configure Physical Interface

The physical interface should be **UP** but have **NO IP address**.

```cisco
enable
configure terminal
hostname R1-HQ

! Physical interface (no IP)
interface GigabitEthernet0/1
 description Trunk to HQ-Switch
 no ip address
 no shutdown
 exit
```

**Why no IP address?**
- IP addresses are assigned to subinterfaces
- Physical interface just carries tagged frames

### Step 2: Create Subinterface for VLAN 10

```cisco
! Subinterface for VLAN 10
interface GigabitEthernet0/1.10
 description VLAN 10 - Users Network
 encapsulation dot1Q 10
 ip address 172.16.10.1 255.255.255.0
 ip helper-address 172.16.30.10
 exit
```

**Explanation:**
- `.10` is the subinterface number (can be any number, typically matches VLAN ID)
- `encapsulation dot1Q 10` associates this subinterface with VLAN 10
- `172.16.10.1` is the default gateway for VLAN 10 devices
- `ip helper-address` forwards DHCP broadcasts to the DHCP server

### Step 3: Create Subinterface for VLAN 20

```cisco
! Subinterface for VLAN 20
interface GigabitEthernet0/1.20
 description VLAN 20 - Users Network
 encapsulation dot1Q 20
 ip address 172.16.20.1 255.255.255.0
 ip helper-address 172.16.30.10
 exit
```

### Step 4: Create Subinterface for VLAN 30

```cisco
! Subinterface for VLAN 30 (DHCP Server)
interface GigabitEthernet0/1.30
 description VLAN 30 - Server Network
 encapsulation dot1Q 30
 ip address 172.16.30.1 255.255.255.0
 exit
```

**Note:** VLAN 30 doesn't need `ip helper-address` because the DHCP server is IN this VLAN.

### Step 5: Save Configuration

```cisco
end
copy running-config startup-config
```

## Verification Commands

### Show IP Interface Brief
```cisco
show ip interface brief
```

**Expected Output:**
```
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     100.100.100.2   YES manual up                    up
GigabitEthernet0/1     unassigned      YES unset  up                    up
Gig0/1.10              172.16.10.1     YES manual up                    up
Gig0/1.20              172.16.20.1     YES manual up                    up
Gig0/1.30              172.16.30.1     YES manual up                    up
```

### Show Running Configuration for Interface
```cisco
show running-config interface GigabitEthernet0/1
```

### Show Specific Subinterface
```cisco
show running-config interface GigabitEthernet0/1.10
```

### Show All Subinterfaces
```cisco
show ip interface brief | include Gig0/1
```

## How Inter-VLAN Routing Works

### Example: PC1 (VLAN 10) pings PC3 (VLAN 20)

1. **PC1 (172.16.10.10)** wants to reach **PC3 (172.16.20.10)**
2. PC1 checks: Is 172.16.20.10 in my subnet? **No** (different subnet)
3. PC1 sends packet to **default gateway (172.16.10.1)** on R1-HQ
4. Packet arrives at R1-HQ subinterface **Gi0/1.10** with VLAN 10 tag
5. Router checks routing table for 172.16.20.0/24
6. Router finds **Gi0/1.20** is the exit interface
7. Router forwards packet out **Gi0/1.20** with VLAN 20 tag
8. HQ-Switch receives VLAN 20 tagged frame and forwards to PC3 on VLAN 20

### Packet Flow Diagram
```
PC1 (VLAN 10)
    |
    | ARP for gateway 172.16.10.1
    v
HQ-Switch Port Fa0/1 (VLAN 10)
    |
    | 802.1Q tagged with VLAN 10
    v
R1-HQ Gi0/1.10 (receives)
    |
    | Routing decision
    v
R1-HQ Gi0/1.20 (sends)
    |
    | 802.1Q tagged with VLAN 20
    v
HQ-Switch Port Fa0/3 (VLAN 20)
    |
    v
PC3 (VLAN 20)
```

## IP Helper-Address Explained

### What is it?
- Converts broadcast packets to unicast
- Forwards DHCP requests from clients to DHCP server
- Necessary because broadcasts don't cross router boundaries

### How it works:
1. PC in VLAN 10 sends DHCP DISCOVER (broadcast 255.255.255.255)
2. Broadcast reaches R1-HQ subinterface Gi0/1.10
3. Router sees `ip helper-address 172.16.30.10`
4. Router converts broadcast to unicast and forwards to 172.16.30.10 (DHCP server)
5. DHCP server responds with OFFER
6. Router relays OFFER back to client

### Protocols Forwarded by IP Helper
- DHCP (ports 67, 68)
- DNS (port 53)
- TFTP (port 69)
- NetBIOS (ports 137, 138)
- TACACS (port 49)

## Testing Inter-VLAN Routing

### Test 1: Ping Gateway from PC
**From PC1 (VLAN 10):**
```
ping 172.16.10.1
```
**Expected:** Success ✓

### Test 2: Ping Different VLAN Gateway
**From PC1 (VLAN 10):**
```
ping 172.16.20.1
```
**Expected:** Success ✓

### Test 3: Ping PC in Different VLAN
**From PC1 (VLAN 10):**
```
ping 172.16.20.10
```
**Expected:** Success ✓ (PC3's IP)

### Test 4: Ping DHCP Server
**From PC1 (VLAN 10):**
```
ping 172.16.30.10
```
**Expected:** Success ✓

## Common Issues and Troubleshooting

### Issue 1: Can't ping gateway
**Symptoms:** PC can't ping its gateway (e.g., 172.16.10.1)

**Solutions:**
1. Check subinterface is UP: `show ip interface brief`
2. Verify encapsulation matches VLAN: `show running-config interface Gi0/1.10`
3. Ensure physical interface is UP: `no shutdown` on Gi0/1
4. Check switch trunk configuration

### Issue 2: Can ping gateway but not other VLANs
**Symptoms:** PC1 can ping 172.16.10.1 but not 172.16.20.1

**Solutions:**
1. Check routing table: `show ip route`
2. All VLAN subnets should show as "directly connected"
3. Verify all subinterfaces are UP
4. Check if there are any ACLs blocking traffic

### Issue 3: DHCP not working
**Symptoms:** PCs don't get IP addresses via DHCP

**Solutions:**
1. Verify `ip helper-address 172.16.30.10` on subinterfaces
2. Check DHCP server is reachable: `ping 172.16.30.10` from router
3. Verify DHCP service is running on server
4. Check DHCP pools are configured correctly

### Issue 4: Subinterface shows down/down
**Symptoms:** Subinterface status is "down/down"

**Solutions:**
1. Ensure physical interface is UP: `no shutdown` on Gi0/1
2. Check cable connection to switch
3. Verify trunk is established on switch side
4. Check VLAN exists on switch

## Key Takeaways

✅ **Physical interface must be UP but has NO IP address**  
✅ **Subinterfaces get IP addresses and serve as gateways**  
✅ **Subinterface number doesn't have to match VLAN ID (but it's best practice)**  
✅ **Encapsulation dot1Q MUST match VLAN ID**  
✅ **IP helper-address forwards DHCP broadcasts**  
✅ **Router performs routing between VLANs**  

## Next Steps
- Proceed to **03-NAT_PAT_Configuration.md** for internet access configuration

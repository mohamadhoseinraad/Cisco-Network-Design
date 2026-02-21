# Step 05 - GRE Tunnel Setup

## Overview
Generic Routing Encapsulation (GRE) is a tunneling protocol that creates a virtual point-to-point link over the Internet. It encapsulates packets inside IP packets, allowing different protocols to travel over an IP network.

## Why GRE Tunnel?
- **Site-to-Site Connectivity:** Connect HQ and Branch over the Internet
- **Private Communication:** Logical direct connection between sites
- **Routing Protocol Support:** Can run OSPF/EIGRP through tunnel
- **Cost-Effective:** Use existing Internet connection instead of dedicated leases
- **Flexibility:** Traffic appears to be on same network

## Tunnel Characteristics

| Feature | Description |
|---------|-------------|
| **Tunnel Network** | 10.10.10.0/30 |
| **HQ Tunnel IP** | 10.10.10.1 |
| **Branch Tunnel IP** | 10.10.10.2 |
| **Source (HQ)** | 100.100.100.2 (R1-HQ WAN IP) |
| **Source (Branch)** | 200.200.200.2 (R3-Branch WAN IP) |
| **Encapsulation** | GRE (IP Protocol 47) |

## GRE Tunnel Concept

### Physical Topology:
```
[R1-HQ] --100.100.100.2-- [ISP] --200.200.200.2-- [R3-Branch]
```

### Logical Topology (After Tunnel):
```
[R1-HQ]--------Tunnel0--------[R3-Branch]
      10.10.10.1        10.10.10.2
      (Appears as direct connection)
```

### Packet Encapsulation:
```
Original Packet:
[IP Header: SRC=172.16.10.10, DST=172.17.17.10] [Data]

After GRE Encapsulation:
[New IP: SRC=100.100.100.2, DST=200.200.200.2] [GRE] [Original IP] [Data]
                    Outer Header                  |-- Inner Packet --|
```

## Implementation on R1-HQ

```cisco
enable
configure terminal

! Create Tunnel Interface
interface Tunnel0
 description GRE Tunnel to R3-Branch
 
 ! Set tunnel IP address (HQ side)
 ip address 10.10.10.1 255.255.255.252
 
 ! Set tunnel source (physical WAN interface)
 tunnel source GigabitEthernet0/0
 
 ! Set tunnel destination (Branch WAN IP)
 tunnel destination 200.200.200.2
 
 ! Enable NAT inside (for Branch-to-Internet via HQ)
 ip nat inside
 
 ! Bring interface up
 no shutdown
 exit

! Save configuration
end
copy running-config startup-config
```

## Implementation on R3-Branch

```cisco
enable
configure terminal

! Create Tunnel Interface
interface Tunnel0
 description GRE Tunnel to R1-HQ
 
 ! Set tunnel IP address (Branch side)
 ip address 10.10.10.2 255.255.255.252
 
 ! Set tunnel source (physical WAN interface)
 tunnel source GigabitEthernet0/0
 
 ! Set tunnel destination (HQ WAN IP)
 tunnel destination 100.100.100.2
 
 ! Enable NAT inside
 ip nat inside
 
 ! Bring interface up
 no shutdown
 exit

! Save configuration
end
copy running-config startup-config
```

## Understanding the Configuration

### tunnel source
- **Purpose:** Specifies the local endpoint of the tunnel
- **Can be:** Interface name or IP address
- **Best Practice:** Use interface name (automatically uses its IP)
- **Example:** `tunnel source GigabitEthernet0/0`

### tunnel destination
- **Purpose:** Specifies the remote endpoint of the tunnel
- **Must be:** IP address (reachable via routing)
- **Example:** `tunnel destination 200.200.200.2`

### ip address
- **Purpose:** Creates logical point-to-point network
- **Subnet:** /30 provides exactly 2 usable IPs (efficient)
- **Network:** 10.10.10.0/30
  - Network: 10.10.10.0
  - R1-HQ: 10.10.10.1
  - R3-Branch: 10.10.10.2
  - Broadcast: 10.10.10.3

## Verification Commands

### Check Tunnel Status
```cisco
show ip interface brief
```

**Expected Output:**
```
Interface              IP-Address      OK? Method Status                Protocol
Tunnel0                10.10.10.1      YES manual up                    up
```

**Status Meanings:**
- **up/up:** Tunnel is working ✓
- **up/down:** Tunnel configured but destination unreachable
- **down/down:** Interface shutdown or source interface down

### Show Interface Details
```cisco
show interfaces Tunnel0
```

**Look for:**
```
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 10.10.10.1/30
  Tunnel source 100.100.100.2 (GigabitEthernet0/0)
  Tunnel destination 200.200.200.2
  Tunnel protocol/transport GRE/IP
```

### Show Running Configuration
```cisco
show running-config interface Tunnel0
```

### Test Tunnel Connectivity
```cisco
ping 10.10.10.2
```

**From R1-HQ to R3-Branch tunnel IP**
- **Success:** Tunnel is working ✓
- **Failure:** Check tunnel configuration

## How GRE Tunnel Works

### Example: PC1 (HQ) pings Branch-PC (172.17.17.10)

#### Step 1: PC sends packet
```
Source: 172.16.10.10
Destination: 172.17.17.10
```

#### Step 2: R1-HQ receives packet
- Checks routing table
- Route to 172.17.17.0/24 via Tunnel0 (learned via OSPF)
- Decision: Send via Tunnel0

#### Step 3: GRE Encapsulation
```
Outer IP Header:
  Source: 100.100.100.2 (R1-HQ WAN)
  Destination: 200.200.200.2 (R3-Branch WAN)
  Protocol: 47 (GRE)

GRE Header:
  [GRE protocol information]

Inner IP Header (Original):
  Source: 172.16.10.10
  Destination: 172.17.17.10

Payload:
  [ICMP Echo Request Data]
```

#### Step 4: Packet travels over Internet
- ISP sees: 100.100.100.2 → 200.200.200.2
- ISP routes based on outer header
- Inner packet is hidden

#### Step 5: R3-Branch receives packet
- Recognizes GRE packet (Protocol 47)
- Decapsulates (removes outer header and GRE header)
- Forwards inner packet to 172.17.17.10

#### Step 6: Reply returns same way
```
Branch-PC → R3-Branch → [GRE Tunnel] → R1-HQ → PC1
```

## Tunnel MTU Considerations

### What is MTU?
- **Maximum Transmission Unit:** Largest packet size
- **Standard Ethernet MTU:** 1500 bytes
- **GRE Overhead:** 24 bytes (20 IP + 4 GRE)
- **Effective MTU over GRE:** 1476 bytes

### MTU Issues:
If sending 1500-byte packet through tunnel:
```
1500 bytes + 24 bytes GRE = 1524 bytes > 1500 MTU = FRAGMENTATION
```

### Solution (Optional):
```cisco
interface Tunnel0
 ip mtu 1476
 exit
```

This tells devices to use smaller packets, avoiding fragmentation.

## Troubleshooting GRE Tunnels

### Issue 1: Tunnel shows up/down
**Symptoms:** `show ip interface brief` shows Tunnel0 up/down

**Causes & Solutions:**

1. **Destination unreachable**
   ```cisco
   ! Check connectivity to destination
   ping 200.200.200.2
   ```
   If ping fails, check:
   - ISP routing
   - WAN interface status
   - Default route

2. **Source interface down**
   ```cisco
   show ip interface brief
   ```
   Ensure Gi0/0 is up/up

3. **Incorrect tunnel destination**
   ```cisco
   show running-config interface Tunnel0
   ```
   Verify: `tunnel destination 200.200.200.2`

### Issue 2: Tunnel up/up but can't ping tunnel IP
**Symptoms:** Tunnel status is up/up but `ping 10.10.10.2` fails

**Solutions:**

1. **Check both sides configured**
   - R1-HQ: 10.10.10.1 → destination 200.200.200.2
   - R3-Branch: 10.10.10.2 → destination 100.100.100.2

2. **Verify tunnel subnet match**
   - Both should use 10.10.10.0/30

3. **Check for ACLs blocking GRE**
   - Protocol 47 must be allowed

4. **Verify NAT doesn't interfere**
   - Tunnel interface should be `ip nat inside`

### Issue 3: Can ping tunnel but not remote networks
**Symptoms:** Can ping 10.10.10.2 but not 172.17.17.0/24

**Solutions:**
1. **Routing problem** - Configure OSPF (next step)
2. **Check routing table:**
   ```cisco
   show ip route
   ```
   Should see route to 172.17.17.0/24

### Issue 4: Tunnel flapping (up/down repeatedly)
**Symptoms:** Tunnel status changes frequently

**Causes:**
1. **Recursive routing** - Tunnel tries to route through itself
2. **Unstable WAN link**
3. **ISP routing issues**

**Solution:**
```cisco
! Ensure default route points to ISP, not tunnel
show ip route 0.0.0.0
```

## Security Considerations

### GRE Limitations:
- **No Encryption:** Data is visible if intercepted
- **No Authentication:** Anyone can create tunnel to your IP
- **Best for:** Trusted networks or as base for IPsec

### GRE + IPsec (Not in this project):
For production environments, combine GRE with IPsec:
- GRE provides routing protocol support
- IPsec provides encryption and authentication

## Testing GRE Tunnel

### Test 1: Tunnel Interface Status
**On R1-HQ:**
```cisco
show ip interface brief | include Tunnel
```
**Expected:** Tunnel0 up/up ✓

### Test 2: Ping Remote Tunnel IP
**From R1-HQ:**
```cisco
ping 10.10.10.2
```
**Expected:** Success ✓

**From R3-Branch:**
```cisco
ping 10.10.10.1
```
**Expected:** Success ✓

### Test 3: Verify Tunnel Details
**On R1-HQ:**
```cisco
show interfaces Tunnel0
```
**Check:**
- Tunnel source: 100.100.100.2 (Gi0/0)
- Tunnel destination: 200.200.200.2
- Protocol: GRE/IP

### Test 4: Check Tunnel Traffic
```cisco
show interfaces Tunnel0 | include packets
```
**Should show:**
- Input packets increasing
- Output packets increasing

## Tunnel Routing Table Entry

After OSPF is configured (next step), routing table will show:

**On R1-HQ:**
```cisco
show ip route
```
```
O    172.17.17.0/24 [110/1001] via 10.10.10.2, Tunnel0
```
This means: "To reach Branch network, send via Tunnel0"

**On R3-Branch:**
```
O    172.16.10.0/24 [110/1001] via 10.10.10.1, Tunnel0
O    172.16.20.0/24 [110/1001] via 10.10.10.1, Tunnel0
O    172.16.30.0/24 [110/1001] via 10.10.10.1, Tunnel0
```

## Key Takeaways

✅ **GRE creates logical point-to-point link over Internet**  
✅ **Tunnel source = local WAN interface**  
✅ **Tunnel destination = remote WAN IP**  
✅ **Tunnel IPs form separate /30 network (10.10.10.0/30)**  
✅ **GRE adds 24 bytes overhead**  
✅ **GRE is not encrypted (use IPsec if needed)**  
✅ **Tunnel must be up/up before OSPF will work**  

## Testing Checklist

- [ ] R1-HQ Tunnel0 configured
- [ ] R3-Branch Tunnel0 configured
- [ ] Tunnel addresses: 10.10.10.1 and 10.10.10.2
- [ ] Source interfaces correct (Gi0/0 on both)
- [ ] Destinations correct (200.200.200.2 and 100.100.100.2)
- [ ] Both tunnels show up/up
- [ ] Can ping from R1-HQ to 10.10.10.2
- [ ] Can ping from R3-Branch to 10.10.10.1

## Next Steps
- Proceed to **06-OSPF_Routing.md** to enable dynamic routing through the tunnel

# Step 04 - DHCP Configuration

## Overview
Dynamic Host Configuration Protocol (DHCP) automatically assigns IP addresses, subnet masks, default gateways, and DNS servers to network devices. This eliminates manual configuration and reduces errors.

## Why DHCP?
- **Automation:** No manual IP configuration needed
- **Centralized Management:** All IP assignments in one place
- **Reduced Errors:** Eliminates typos and duplicate IPs
- **Scalability:** Easy to add new devices
- **Efficient Use:** IP addresses recycled when not in use

## Project Requirements
- **DHCP Server:** 172.16.30.10 (in VLAN 30)
- **DHCP Clients:** PCs in VLAN 10 and VLAN 20
- **Challenge:** DHCP uses broadcasts, but broadcasts don't cross VLANs
- **Solution:** DHCP Relay (IP Helper-Address)

## Architecture

```
VLAN 10 (172.16.10.0/24)            VLAN 30 (172.16.30.0/24)
  PC1 (DHCP Client)                   DHCP Server (172.16.30.10)
       |                                     |
       |                                     |
    [HQ-Switch]              [HQ-Switch]
       |                                     |
       +--------- R1-HQ Gi0/1.10 ----+      |
                       |                    |
                       +-- Gi0/1.30 --------+
                       
DHCP Request (broadcast) --> Router converts to unicast --> DHCP Server
```

## Implementation Steps

### Part 1: Configure DHCP Server (172.16.30.10)

#### Step 1: Set Static IP on Server
1. Click on **DHCP Server** in Packet Tracer
2. Go to **Desktop** → **IP Configuration**
3. Select **Static**
4. Enter:
   - **IP Address:** 172.16.30.10
   - **Subnet Mask:** 255.255.255.0
   - **Default Gateway:** 172.16.30.1
   - **DNS Server:** 8.8.8.8

#### Step 2: Configure DHCP Pool for VLAN 10
1. Go to **Services** → **DHCP**
2. Click **Add** button
3. Configure:
   - **Pool Name:** VLAN10_Pool
   - **Default Gateway:** 172.16.10.1
   - **DNS Server:** 8.8.8.8
   - **Start IP Address:** 172.16.10.10
   - **Subnet Mask:** 255.255.255.0
   - **Maximum Number of Users:** 50
4. Ensure **Service** is **On**
5. Click **Save**

#### Step 3: Configure DHCP Pool for VLAN 20
1. Click **Add** button again
2. Configure:
   - **Pool Name:** VLAN20_Pool
   - **Default Gateway:** 172.16.20.1
   - **DNS Server:** 8.8.8.8
   - **Start IP Address:** 172.16.20.10
   - **Subnet Mask:** 255.255.255.0
   - **Maximum Number of Users:** 50
3. Ensure **Service** is **On**
4. Click **Save**

### Part 2: Configure DHCP Relay on R1-HQ

DHCP Relay (IP Helper-Address) forwards DHCP broadcasts from clients to the DHCP server.

```cisco
enable
configure terminal

! VLAN 10 subinterface
interface GigabitEthernet0/1.10
 ip helper-address 172.16.30.10
 exit

! VLAN 20 subinterface
interface GigabitEthernet0/1.20
 ip helper-address 172.16.30.10
 exit

! Save configuration
end
copy running-config startup-config
```

**Note:** VLAN 30 doesn't need `ip helper-address` because the DHCP server is already in that VLAN.

## How DHCP Works (DORA Process)

### Normal DHCP (Same Subnet):
```
1. DISCOVER: Client broadcasts "I need an IP!"
2. OFFER:    Server broadcasts "Here's an IP for you"
3. REQUEST:  Client broadcasts "Yes, I accept this IP"
4. ACK:      Server broadcasts "Confirmed, IP is yours"
```

### DHCP with Relay (Different Subnets):
```
PC1 (VLAN 10)          R1-HQ (Relay)          DHCP Server (VLAN 30)
     |                      |                        |
     |--DISCOVER(broadcast)->|                        |
     |                      |--DISCOVER(unicast)---->|
     |                      |                        |
     |                      |<---OFFER(unicast)------|
     |<--OFFER(broadcast)---|                        |
     |                      |                        |
     |--REQUEST(broadcast)->|                        |
     |                      |--REQUEST(unicast)----->|
     |                      |                        |
     |                      |<---ACK(unicast)--------|
     |<--ACK(broadcast)----|                        |
     |                      |                        |
   Configured           Forwarded              Assigned IP
```

### Detailed DORA with Relay:

#### 1. DISCOVER
```
PC1 → Broadcast (255.255.255.255)
↓
HQ-Switch (VLAN 10) → Forwards to R1-HQ Gi0/1.10
↓
R1-HQ sees ip helper-address 172.16.30.10
↓
R1-HQ converts to unicast: SRC=172.16.10.1, DST=172.16.30.10
↓
DHCP Server receives unicast DISCOVER
```

#### 2. OFFER
```
DHCP Server → Unicast to 172.16.10.1 (relay)
↓
R1-HQ receives OFFER
↓
R1-HQ broadcasts in VLAN 10
↓
PC1 receives OFFER
```

#### 3. REQUEST
```
PC1 → Broadcast REQUEST
↓
R1-HQ → Unicast to 172.16.30.10
↓
DHCP Server receives REQUEST
```

#### 4. ACK
```
DHCP Server → Unicast to relay
↓
R1-HQ → Broadcast ACK in VLAN 10
↓
PC1 receives ACK and configures IP
```

## Configuring Clients to Use DHCP

### On PCs in VLAN 10 and VLAN 20:
1. Click on the PC
2. Go to **Desktop** → **IP Configuration**
3. Select **DHCP**
4. Click **Request** (or wait a few seconds)

**Expected Results:**
- **VLAN 10 PCs:** Receive IPs from 172.16.10.10-172.16.10.59
- **VLAN 20 PCs:** Receive IPs from 172.16.20.10-172.16.20.59
- **Subnet Mask:** 255.255.255.0
- **Default Gateway:** 172.16.10.1 (VLAN 10) or 172.16.20.1 (VLAN 20)
- **DNS Server:** 8.8.8.8

## Verification Commands

### On DHCP Server (Packet Tracer):
1. Go to **Services** → **DHCP**
2. Scroll down to **Current Leases** section
3. You should see assigned IPs, MAC addresses, and lease times

### On R1-HQ Router:
```cisco
! Show IP helper-address configuration
show running-config interface GigabitEthernet0/1.10
show running-config interface GigabitEthernet0/1.20
```

### On Client PCs:
```cmd
ipconfig

or

ipconfig /all
```

**Expected Output:**
```
IP Address........: 172.16.10.10
Subnet Mask.......: 255.255.255.0
Default Gateway...: 172.16.10.1
DNS Server........: 8.8.8.8
```

## DHCP Lease Information

### Lease Time
- Default lease: 24 hours
- Client must renew before lease expires
- At 50% of lease time, client tries to renew
- At 87.5% of lease time, client tries to discover new DHCP server

### Lease States:
- **Leased:** IP assigned and in use
- **Expired:** Lease time ended, IP available for reassignment
- **Reserved:** (If configured) IP permanently assigned to specific MAC

## Testing DHCP

### Test 1: Initial DHCP Request
1. Set PC to DHCP mode
2. Click **Request**
3. Verify IP assigned from correct pool
4. Check Default Gateway is correct
5. Verify DNS Server is 8.8.8.8

### Test 2: Release and Renew
**On PC:**
```cmd
ipconfig /release
ipconfig /renew
```

### Test 3: Connectivity After DHCP
**From DHCP client PC:**
```cmd
ping 172.16.30.10    (DHCP Server)
ping 172.16.10.1     (Gateway)
ping 8.8.8.8         (Internet via NAT)
```

All should succeed ✓

### Test 4: Multiple Clients
1. Configure 2-3 PCs in VLAN 10 for DHCP
2. Verify each gets a different IP
3. Check DHCP server shows all leases

## Common Issues and Troubleshooting

### Issue 1: PC doesn't get IP address
**Symptoms:** PC shows 169.254.x.x (APIPA) or 0.0.0.0

**Solutions:**
1. **Check DHCP Server:**
   - Is service **On** for the pool?
   - Is pool configured correctly?
   - Ping server from router: `ping 172.16.30.10`

2. **Check IP Helper-Address:**
   ```cisco
   show running-config interface Gi0/1.10
   ```
   Should show: `ip helper-address 172.16.30.10`

3. **Check VLAN Configuration:**
   - Is PC in correct VLAN?
   - Is trunk carrying the VLAN?
   - `show vlan brief` on switch

4. **Check Router Subinterface:**
   ```cisco
   show ip interface brief
   ```
   Subinterface should be UP/UP

5. **Try Manual Release/Renew:**
   - On PC: Desktop → Command Prompt
   - `ipconfig /release`
   - `ipconfig /renew`

### Issue 2: Gets IP but wrong gateway
**Symptoms:** IP from correct range, but wrong default gateway

**Solutions:**
1. Check DHCP pool configuration
2. Verify **Default Gateway** matches VLAN gateway:
   - VLAN 10: 172.16.10.1
   - VLAN 20: 172.16.20.1
3. Recreate the pool if needed

### Issue 3: Gets IP but can't reach internet
**Symptoms:** Has IP, can ping gateway, but can't ping 8.8.8.8

**Solutions:**
1. Check NAT is configured on router
2. Verify default route: `show ip route`
3. Test from router: `ping 8.8.8.8`
4. Check DNS setting in DHCP pool

### Issue 4: IP conflict
**Symptoms:** "IP address conflict detected"

**Solutions:**
1. Ensure no static IPs overlap with DHCP range
2. DHCP starts at .10, so use .2-.9 for static if needed
3. Clear DHCP bindings on server
4. Release/renew on client

### Issue 5: DHCP works in VLAN 10 but not VLAN 20
**Symptoms:** VLAN 10 PCs get IPs, VLAN 20 don't

**Solutions:**
1. Check `ip helper-address` on Gi0/1.20
2. Verify VLAN 20 pool exists and service is **On**
3. Check VLAN 20 subnet is correct (172.16.20.0/24)
4. Verify VLAN 20 subinterface is UP

## IP Addressing Plan

### Reserved for Static:
- **172.16.10.1:** R1-HQ gateway (VLAN 10)
- **172.16.20.1:** R1-HQ gateway (VLAN 20)
- **172.16.30.1:** R1-HQ gateway (VLAN 30)
- **172.16.30.10:** DHCP Server

### DHCP Pools:
- **VLAN 10:** 172.16.10.10 - 172.16.10.59 (50 addresses)
- **VLAN 20:** 172.16.20.10 - 172.16.20.59 (50 addresses)

### Available for Future Static:
- **VLAN 10:** 172.16.10.2-9, 172.16.10.60-254
- **VLAN 20:** 172.16.20.2-9, 172.16.20.60-254
- **VLAN 30:** 172.16.30.2-9, 172.16.30.11-254

## Security Considerations

### DHCP Snooping (Advanced - not in this project)
- Prevents rogue DHCP servers
- Only authorized ports can offer DHCP
- Builds binding database

### Best Practices:
1. **Separate Server VLAN:** DHCP server in VLAN 30 (done ✓)
2. **Limited Scope:** Only assign what's needed (50 addresses each)
3. **Monitor Leases:** Check for unusual DHCP activity
4. **Backup Configuration:** Document DHCP pools

## Testing Checklist

- [ ] DHCP Server has static IP (172.16.30.10)
- [ ] VLAN 10 pool configured correctly
- [ ] VLAN 20 pool configured correctly
- [ ] Both pools have Service **On**
- [ ] IP helper-address on Gi0/1.10
- [ ] IP helper-address on Gi0/1.20
- [ ] PC in VLAN 10 gets IP from correct range
- [ ] PC in VLAN 20 gets IP from correct range
- [ ] Default gateway is correct for each VLAN
- [ ] DNS server is 8.8.8.8
- [ ] Can ping gateway from DHCP client
- [ ] Can ping internet (8.8.8.8) from DHCP client
- [ ] DHCP leases visible on server

## Next Steps
- Proceed to **05-GRE_Tunnel_Setup.md** for site-to-site connectivity

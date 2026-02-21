# Step 08 - Verification and Testing

## Overview
This document provides comprehensive testing procedures to verify that all network components are functioning correctly. Follow these tests sequentially to validate the entire network implementation.

## Testing Strategy

### Test Categories:
1. **Physical Layer** - Cable connections and link status
2. **Data Link Layer** - VLAN configuration and trunking
3. **Network Layer** - IP addressing and routing
4. **Transport Layer** - TCP/UDP services
5. **Application Layer** - DHCP, HTTP/HTTPS services
6. **Security** - ACL functionality
7. **End-to-End** - Complete HQ-to-Branch connectivity

---

## 1. Physical Layer Testing

### Test 1.1: Interface Status on R1-HQ
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
Tunnel0                10.10.10.1      YES manual up                    up
```

✅ **Pass Criteria:** All interfaces show `up up`

### Test 1.2: Interface Status on R3-Branch
```cisco
show ip interface brief
```

**Expected Output:**
```
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     200.200.200.2   YES manual up                    up
GigabitEthernet0/1     172.17.17.1     YES manual up                    up
Tunnel0                10.10.10.2      YES manual up                    up
```

✅ **Pass Criteria:** All interfaces show `up up`

### Test 1.3: Switch Port Status
**On HQ-Switch:**
```cisco
show interfaces status
```

**Check:**
- Fa0/1-4: Connected (PCs)
- Fa0/5: Connected (DHCP Server)
- Gi0/1: Connected (Trunk to R1-HQ)

---

## 2. Data Link Layer Testing

### Test 2.1: VLAN Configuration
**On HQ-Switch:**
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

✅ **Pass Criteria:** VLANs 10, 20, 30 exist with correct port assignments

### Test 2.2: Trunk Configuration
**On HQ-Switch:**
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

✅ **Pass Criteria:** Trunk active, VLANs 10,20,30 allowed

### Test 2.3: Router Subinterfaces
**On R1-HQ:**
```cisco
show running-config | section interface GigabitEthernet0/1
```

**Verify:**
- Subinterfaces .10, .20, .30 configured
- Correct encapsulation (dot1Q 10, 20, 30)
- Correct IP addresses

---

## 3. Network Layer Testing

### Test 3.1: DHCP Server Reachability
**From R1-HQ:**
```cisco
ping 172.16.30.10
```

**Expected:** 
```
!!!!!
Success rate is 100 percent (5/5)
```

✅ **Pass Criteria:** 5/5 packets successful

### Test 3.2: Inter-VLAN Routing
**From R1-HQ:**
```cisco
ping 172.16.10.1
ping 172.16.20.1
ping 172.16.30.1
```

**Expected:** All pings succeed

✅ **Pass Criteria:** Router can reach all its own VLAN gateways

### Test 3.3: GRE Tunnel Connectivity
**From R1-HQ:**
```cisco
ping 10.10.10.2
```

**From R3-Branch:**
```cisco
ping 10.10.10.1
```

**Expected:** Both pings succeed

✅ **Pass Criteria:** Bidirectional tunnel connectivity verified

### Test 3.4: NAT Functionality
**From R1-HQ:**
```cisco
ping 8.8.8.8
```

**From R3-Branch:**
```cisco
ping 8.8.8.8
```

**Expected:** Both pings succeed

✅ **Pass Criteria:** Routers can reach Internet through NAT

**Verify NAT translations:**
```cisco
show ip nat translations
```

Should show active translations.

---

## 4. OSPF Routing Testing

### Test 4.1: OSPF Neighbor Adjacency
**On R1-HQ:**
```cisco
show ip ospf neighbor
```

**Expected Output:**
```
Neighbor ID     Pri   State           Dead Time   Address         Interface
3.3.3.3         0     FULL/  -        00:00:35    10.10.10.2      Tunnel0
```

✅ **Pass Criteria:** Neighbor in FULL state

### Test 4.2: OSPF Routes on R1-HQ
```cisco
show ip route ospf
```

**Expected Output:**
```
O    172.17.17.0/24 [110/1001] via 10.10.10.2, Tunnel0
```

✅ **Pass Criteria:** Route to Branch network learned via OSPF

### Test 4.3: OSPF Routes on R3-Branch
```cisco
show ip route ospf
```

**Expected Output:**
```
O    172.16.10.0/24 [110/1001] via 10.10.10.1, Tunnel0
O    172.16.20.0/24 [110/1001] via 10.10.10.1, Tunnel0
O    172.16.30.0/24 [110/1001] via 10.10.10.1, Tunnel0
```

✅ **Pass Criteria:** Routes to all HQ networks learned via OSPF

### Test 4.4: R1-HQ Complete Routing Table
```cisco
show ip route
```

**Should contain:**
- Directly connected: 172.16.10.0/24, 172.16.20.0/24, 172.16.30.0/24
- Directly connected: 10.10.10.0/30 (tunnel)
- Directly connected: 100.100.100.0/30 (WAN)
- OSPF: 172.17.17.0/24 via Tunnel0
- Static: 0.0.0.0/0 via 100.100.100.1 (default route to ISP)

---

## 5. DHCP Testing

### Test 5.1: DHCP Server Configuration
**On DHCP Server (Packet Tracer):**
1. Click **DHCP Server**
2. Go to **Services** → **DHCP**
3. Verify both pools exist:
   - VLAN10_Pool: 172.16.10.10-59
   - VLAN20_Pool: 172.16.20.10-59
4. Ensure **Service** is **On** for both

### Test 5.2: IP Helper-Address Configuration
**On R1-HQ:**
```cisco
show running-config interface GigabitEthernet0/1.10
show running-config interface GigabitEthernet0/1.20
```

**Verify both have:**
```
ip helper-address 172.16.30.10
```

### Test 5.3: DHCP Client - VLAN 10
**On PC1 (VLAN 10):**
1. Desktop → IP Configuration
2. Select **DHCP**
3. Click **Request** (or wait)

**Expected:**
- IP Address: 172.16.10.10-59
- Subnet Mask: 255.255.255.0
- Default Gateway: 172.16.10.1
- DNS Server: 8.8.8.8

✅ **Pass Criteria:** PC receives correct IP configuration

### Test 5.4: DHCP Client - VLAN 20
**On PC3 (VLAN 20):**
1. Desktop → IP Configuration
2. Select **DHCP**
3. Click **Request**

**Expected:**
- IP Address: 172.16.20.10-59
- Subnet Mask: 255.255.255.0
- Default Gateway: 172.16.20.1
- DNS Server: 8.8.8.8

✅ **Pass Criteria:** PC receives correct IP configuration

### Test 5.5: Verify DHCP Leases
**On DHCP Server:**
1. Services → DHCP
2. Scroll to **Current Leases**
3. Should see assigned IPs with MAC addresses

---

## 6. NAT/PAT Testing

### Test 6.1: Internet Access from VLAN 10
**From PC1 (VLAN 10):**
```cmd
ping 8.8.8.8
```

**Expected:**
```
Reply from 8.8.8.8: bytes=32 time<1ms TTL=128
```

✅ **Pass Criteria:** Successful ping to Internet

### Test 6.2: Internet Access from VLAN 20
**From PC3 (VLAN 20):**
```cmd
ping 8.8.8.8
```

**Expected:** Success

### Test 6.3: Internet Access from Branch
**From Branch PC:**
```cmd
ping 8.8.8.8
```

**Expected:** Success

### Test 6.4: NAT Translation Table
**On R1-HQ after PC pings:**
```cisco
show ip nat translations
```

**Expected:** 
- Should see translations from 172.16.x.x to 100.100.100.2

**On R3-Branch:**
```cisco
show ip nat translations
```

**Expected:**
- Should see translations from 172.17.17.x to 200.200.200.2

### Test 6.5: NAT Statistics
**On R1-HQ:**
```cisco
show ip nat statistics
```

**Check:**
- Inside interfaces: Gi0/1.10, Gi0/1.20, Gi0/1.30, Tunnel0
- Outside interface: Gi0/0
- Hits counter should be > 0

---

## 7. ACL Security Testing

### Test 7.1: ACL Configuration
**On R3-Branch:**
```cisco
show access-lists
```

**Expected Output:**
```
Extended IP access list WEB-SERVER-ACL
    10 deny icmp any host 172.17.17.100 echo
    20 permit tcp any host 172.17.17.100 eq 443
    30 permit ip any any
```

### Test 7.2: ACL Applied to Interface
```cisco
show ip interface GigabitEthernet0/1
```

**Verify:**
```
Inbound access list is WEB-SERVER-ACL
```

### Test 7.3: ICMP Blocked to Web Server
**From PC1 (HQ VLAN 10):**
```cmd
ping 172.17.17.100
```

**Expected:**
```
Request timed out.
```

✅ **Pass Criteria:** Ping FAILS (blocked by ACL)

### Test 7.4: ICMP Allowed to Other Branch Devices
**From PC1 (HQ):**
```cmd
ping 172.17.17.10
```

**Expected:** Success (only web server ICMP is blocked)

### Test 7.5: HTTPS Access to Web Server
**Configure Web Server first:**
1. Click Web Server
2. Desktop → IP Configuration
3. IP: 172.17.17.100, Mask: 255.255.255.0, Gateway: 172.17.17.1
4. Services → HTTP → Enable **HTTPS** (port 443)

**From PC1 (HQ):**
1. Desktop → Web Browser
2. Enter: `https://172.17.17.100`
3. Should display web page

✅ **Pass Criteria:** HTTPS connection succeeds

### Test 7.6: ACL Hit Counters
**On R3-Branch:**
```cisco
show access-lists
```

**Expected:**
- Line 10 (deny icmp): Matches should increase after ping attempts
- Line 20 (permit tcp 443): Matches should increase after HTTPS access
- Line 30 (permit any): Matches for other traffic

---

## 8. End-to-End Connectivity Testing

### Test 8.1: HQ VLAN 10 to Branch LAN
**From PC1 (172.16.10.x):**
```cmd
ping 172.17.17.10
```

**Expected:** Success

### Test 8.2: HQ VLAN 20 to Branch LAN
**From PC3 (172.16.20.x):**
```cmd
ping 172.17.17.10
```

**Expected:** Success

### Test 8.3: Branch to HQ VLAN 10
**From Branch PC:**
```cmd
ping 172.16.10.10
```

**Expected:** Success

### Test 8.4: Branch to HQ VLAN 20
**From Branch PC:**
```cmd
ping 172.16.20.10
```

**Expected:** Success

### Test 8.5: Branch to DHCP Server
**From Branch PC:**
```cmd
ping 172.16.30.10
```

**Expected:** Success

### Test 8.6: Traceroute HQ to Branch
**From PC1 (HQ):**
```cmd
tracert 172.17.17.10
```

**Expected Path:**
```
1   172.16.10.1 (R1-HQ gateway)
2   10.10.10.2 (R3-Branch tunnel IP)
3   172.17.17.10 (Branch PC)
```

---

## 9. Service-Level Testing

### Test 9.1: DNS Resolution (Simulated)
**From any PC:**
```cmd
ping 8.8.8.8
```

Since we configured DNS server as 8.8.8.8, connectivity should work.

### Test 9.2: Web Access from HQ to Branch Server
**From PC1 (HQ VLAN 10):**
1. Desktop → Web Browser
2. Enter: `https://172.17.17.100`
3. Verify web page loads

### Test 9.3: Inter-VLAN Communication
**From PC1 (VLAN 10):**
```cmd
ping 172.16.20.10
```
(PC in VLAN 20)

**Expected:** Success (inter-VLAN routing works)

---

## 10. Comprehensive Test Matrix

| Source | Destination | Test | Expected Result | Status |
|--------|-------------|------|-----------------|--------|
| PC1 (VLAN 10) | 172.16.10.1 | Ping gateway | Success | ☐ |
| PC1 (VLAN 10) | 172.16.20.10 | Ping other VLAN | Success | ☐ |
| PC1 (VLAN 10) | 172.16.30.10 | Ping DHCP server | Success | ☐ |
| PC1 (VLAN 10) | 172.17.17.10 | Ping Branch PC | Success | ☐ |
| PC1 (VLAN 10) | 172.17.17.100 | Ping Web Server | **FAIL** | ☐ |
| PC1 (VLAN 10) | 172.17.17.100:443 | HTTPS Web Server | Success | ☐ |
| PC1 (VLAN 10) | 8.8.8.8 | Ping Internet | Success | ☐ |
| PC3 (VLAN 20) | 172.16.20.1 | Ping gateway | Success | ☐ |
| PC3 (VLAN 20) | 172.17.17.10 | Ping Branch PC | Success | ☐ |
| PC3 (VLAN 20) | 8.8.8.8 | Ping Internet | Success | ☐ |
| Branch PC | 172.16.10.10 | Ping HQ VLAN 10 | Success | ☐ |
| Branch PC | 172.16.20.10 | Ping HQ VLAN 20 | Success | ☐ |
| Branch PC | 8.8.8.8 | Ping Internet | Success | ☐ |
| R1-HQ | 10.10.10.2 | Ping tunnel | Success | ☐ |
| R3-Branch | 10.10.10.1 | Ping tunnel | Success | ☐ |

---

## 11. Performance Testing

### Test 11.1: DHCP Lease Time
1. Release IP on PC: `ipconfig /release`
2. Time the renewal: `ipconfig /renew`
3. Should be < 5 seconds

### Test 11.2: OSPF Convergence
1. Shut down Tunnel0 on R1-HQ
   ```cisco
   interface Tunnel0
   shutdown
   ```
2. Check OSPF neighbor: `show ip ospf neighbor` (should disappear)
3. Bring up Tunnel0:
   ```cisco
   interface Tunnel0
   no shutdown
   ```
4. Monitor convergence:
   ```cisco
   show ip ospf neighbor
   ```
   Should return to FULL state within 40 seconds

### Test 11.3: NAT Session Handling
1. From multiple PCs, ping 8.8.8.8 simultaneously
2. Check NAT translations:
   ```cisco
   show ip nat translations
   ```
3. Verify different port numbers for each PC

---

## 12. Troubleshooting Quick Reference

### Issue: PC has 169.254.x.x address
**Cause:** DHCP not working
**Fix:**
1. Check DHCP server is running
2. Verify `ip helper-address` on router subinterface
3. Check VLAN and trunk configuration

### Issue: Can ping gateway but not other VLANs
**Cause:** Inter-VLAN routing problem
**Fix:**
1. Verify router subinterfaces are up: `show ip interface brief`
2. Check routing table: `show ip route`

### Issue: Can't ping across tunnel
**Cause:** GRE tunnel or OSPF problem
**Fix:**
1. Check tunnel status: `show ip interface brief | include Tunnel`
2. Verify OSPF neighbor: `show ip ospf neighbor`
3. Check routing table: `show ip route ospf`

### Issue: Can't reach Internet
**Cause:** NAT or routing problem
**Fix:**
1. Verify NAT configuration: `show ip nat statistics`
2. Check default route: `show ip route 0.0.0.0`
3. Test from router: `ping 8.8.8.8`

### Issue: Web server accessible but shouldn't be
**Cause:** ACL misconfiguration
**Fix:**
1. Check ACL: `show access-lists`
2. Verify applied: `show ip interface Gi0/1`
3. Check rule order (specific before general)

---

## 13. Configuration Backup Checklist

Before marking the project complete, save all configurations:

### On R1-HQ:
```cisco
copy running-config startup-config
show running-config
! (Copy output to text file)
```

### On R3-Branch:
```cisco
copy running-config startup-config
show running-config
! (Copy output to text file)
```

### On ISP Router:
```cisco
copy running-config startup-config
```

### On HQ-Switch:
```cisco
copy running-config startup-config
show running-config
```

### On Branch-Switch:
```cisco
copy running-config startup-config
```

### Save Packet Tracer File:
1. File → Save As
2. Name: `40119783-NetProject.pkt`
3. Save in project directory

---

## 14. Final Validation Checklist

- [ ] All router interfaces up/up
- [ ] All switch ports operational
- [ ] VLANs 10, 20, 30 configured correctly
- [ ] Trunk port carrying all VLANs
- [ ] Router-on-a-stick subinterfaces working
- [ ] DHCP server assigning IPs correctly
- [ ] IP helper-address configured on subinterfaces
- [ ] All PCs receive DHCP addresses
- [ ] NAT/PAT translating HQ networks
- [ ] NAT/PAT translating Branch network
- [ ] Can ping 8.8.8.8 from all PCs
- [ ] GRE tunnel up/up on both routers
- [ ] Can ping tunnel IPs bidirectionally
- [ ] OSPF neighbors in FULL state
- [ ] OSPF routes learned correctly
- [ ] Inter-VLAN routing working
- [ ] HQ to Branch connectivity working
- [ ] ACL blocking ping to web server
- [ ] ACL allowing HTTPS to web server
- [ ] Web server accessible via HTTPS
- [ ] All configurations saved

---

## Conclusion

If all tests pass, congratulations! Your network implementation is complete and functional.

**Project Requirements Met:**
✅ VLAN design and implementation  
✅ Router-on-a-Stick configuration  
✅ DHCP server with relay  
✅ NAT/PAT for internet access  
✅ GRE tunnel between sites  
✅ OSPF dynamic routing  
✅ ACL security for web server (Bonus)  

**Student ID:** 40119783  
**Course:** Computer Networks 2  
**Instructor:** Dr. Fatemeh Rezaei  

---

## Submit for Grading

Your submission should include:
1. ✅ Packet Tracer file (.pkt)
2. ✅ Configuration files (text format)
3. ✅ Network diagram
4. ✅ Documentation (this file)
5. ✅ Test results and screenshots

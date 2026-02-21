# Step 07 - ACL Security Configuration

## Overview
Access Control Lists (ACLs) are packet filtering mechanisms that permit or deny traffic based on various criteria. In this project, we implement ACLs to protect the web server at the Branch office.

## Project Requirement (Bonus Feature)
**Goal:** Web server (172.17.17.100) should:
- ✅ Accept HTTPS connections (TCP port 443)
- ❌ Reject ICMP ping requests
- ✅ Allow all other traffic (for testing purposes)

## Why ACLs?
- **Security:** Control what traffic reaches sensitive devices
- **Resource Protection:** Prevent unauthorized access
- **Network Performance:** Block unwanted traffic early
- **Policy Enforcement:** Implement security policies

## ACL Types

### Standard ACLs (1-99, 1300-1999)
- Filter based on **source IP only**
- Simple but limited
- Example: Block entire subnet

### Extended ACLs (100-199, 2000-2699)
- Filter based on:
  - Source IP
  - Destination IP
  - Protocol (TCP, UDP, ICMP, etc.)
  - Port numbers
  - More granular control ✓ (Used in this project)

## Implementation on R3-Branch

### Step 1: Create Extended ACL

```cisco
enable
configure terminal

! Create named extended ACL
ip access-list extended WEB-SERVER-ACL
 
 ! Rule 1: Deny ICMP echo (ping) to web server
 deny icmp any host 172.17.17.100 echo
 
 ! Rule 2: Permit HTTPS (TCP 443) to web server
 permit tcp any host 172.17.17.100 eq 443
 
 ! Rule 3: Permit all other traffic (for testing)
 permit ip any any
 
 exit
```

### Step 2: Apply ACL to Interface

```cisco
! Apply ACL to LAN interface (inbound direction)
interface GigabitEthernet0/1
 ip access-group WEB-SERVER-ACL in
 exit

! Save configuration
end
copy running-config startup-config
```

## Understanding the ACL

### Rule-by-Rule Breakdown:

#### Rule 1: Deny ICMP Ping
```cisco
deny icmp any host 172.17.17.100 echo
```
**Components:**
- `deny` - Drop matching packets
- `icmp` - Internet Control Message Protocol
- `any` - From any source IP
- `host 172.17.17.100` - To this specific server
- `echo` - Echo request (ping)

**What it does:** Blocks ping requests to the web server

**Why:** Prevents reconnaissance; attackers often ping first to find active hosts

#### Rule 2: Permit HTTPS
```cisco
permit tcp any host 172.17.17.100 eq 443
```
**Components:**
- `permit` - Allow matching packets
- `tcp` - Transmission Control Protocol
- `any` - From any source IP
- `host 172.17.17.100` - To the web server
- `eq 443` - Port equals 443 (HTTPS)

**What it does:** Allows secure web traffic to the server

**Why:** This is the server's primary function

#### Rule 3: Permit All (Default)
```cisco
permit ip any any
```
**Components:**
- `permit` - Allow matching packets
- `ip` - Any IP protocol (TCP, UDP, ICMP, etc.)
- `any any` - From any source to any destination

**What it does:** Allows all other traffic

**Why:** For flexibility and testing (in production, you might want `deny ip any any`)

### ACL Processing Logic

ACLs are processed **top-to-bottom**, **first match wins**.

**Example 1: Ping to Web Server**
```
Packet: ICMP echo from 172.16.10.10 to 172.17.17.100
↓
Rule 1: deny icmp any host 172.17.17.100 echo → MATCH! → DENIED ✓
↓
(Processing stops, packet dropped)
```

**Example 2: HTTPS to Web Server**
```
Packet: TCP SYN from 172.16.10.10:52000 to 172.17.17.100:443
↓
Rule 1: deny icmp... → No match (not ICMP)
↓
Rule 2: permit tcp any host 172.17.17.100 eq 443 → MATCH! → PERMITTED ✓
↓
(Processing stops, packet forwarded)
```

**Example 3: SSH to Web Server**
```
Packet: TCP SYN from 172.16.10.10:52001 to 172.17.17.100:22
↓
Rule 1: deny icmp... → No match (not ICMP)
↓
Rule 2: permit tcp ... eq 443 → No match (port 22, not 443)
↓
Rule 3: permit ip any any → MATCH! → PERMITTED ✓
```

**Example 4: Ping to Branch PC**
```
Packet: ICMP echo from 172.16.10.10 to 172.17.17.10 (other PC)
↓
Rule 1: deny icmp any host 172.17.17.100 → No match (different destination)
↓
Rule 2: permit tcp ... → No match (not TCP)
↓
Rule 3: permit ip any any → MATCH! → PERMITTED ✓
```

## ACL Direction: IN vs OUT

### Inbound ACL (`in`):
```cisco
interface GigabitEthernet0/1
 ip access-group WEB-SERVER-ACL in
```
- Filters packets **entering** the interface
- Checked **before** routing decision
- More efficient (drops unwanted traffic earlier)
- **Used in this project** ✓

### Outbound ACL (`out`):
```cisco
interface GigabitEthernet0/1
 ip access-group WEB-SERVER-ACL out
```
- Filters packets **leaving** the interface
- Checked **after** routing decision

### Visualization:
```
         Inbound ACL              Outbound ACL
              ↓                         ↓
[Source] → Interface → Routing → Interface → [Destination]
           (filter)    (decision)  (filter)
```

**For our use case:**
- Protect web server from external pings
- Use **inbound** on Gi0/1 (LAN interface)
- Filters traffic entering the Branch LAN

## Verification Commands

### Show ACL Configuration
```cisco
show access-lists
```

**Expected Output:**
```
Extended IP access list WEB-SERVER-ACL
    10 deny icmp any host 172.17.17.100 echo (5 matches)
    20 permit tcp any host 172.17.17.100 eq 443 (10 matches)
    30 permit ip any any (100 matches)
```

**Match counters** show how many packets hit each rule.

### Show ACL Applied to Interfaces
```cisco
show ip interface GigabitEthernet0/1
```

**Look for:**
```
Outgoing access list is not set
Inbound access list is WEB-SERVER-ACL
```

### Show Running Configuration
```cisco
show running-config | section access-list
```

## Testing ACLs

### Test 1: Ping Web Server (Should Fail)
**From HQ PC (172.16.10.10):**
```cmd
ping 172.17.17.100
```

**Expected:** Request timed out ❌ (ICMP denied)

### Test 2: Ping Other Branch Device (Should Work)
**From HQ PC:**
```cmd
ping 172.17.17.10
```

**Expected:** Reply from 172.17.17.10 ✓ (Only web server ICMP is blocked)

### Test 3: HTTPS to Web Server (Should Work)
**From HQ PC:**
1. Open **Desktop** → **Web Browser**
2. Enter: `https://172.17.17.100`
3. Should see web page ✓

**Note:** In Packet Tracer, the server must have HTTPS service enabled.

### Test 4: Check ACL Matches
**On R3-Branch after tests:**
```cisco
show access-lists
```

**Expected:**
```
Extended IP access list WEB-SERVER-ACL
    10 deny icmp any host 172.17.17.100 echo (5 matches)  ← Ping attempts
    20 permit tcp any host 172.17.17.100 eq 443 (10 matches) ← HTTPS traffic
    30 permit ip any any (100 matches) ← Other traffic
```

Counters should increase as you test.

## Configuring Web Server in Packet Tracer

For HTTPS testing, configure the web server:

### Step 1: Set Server IP
1. Click on **Web Server**
2. **Desktop** → **IP Configuration**
3. Set:
   - IP Address: `172.17.17.100`
   - Subnet Mask: `255.255.255.0`
   - Default Gateway: `172.17.17.1`

### Step 2: Enable HTTPS Service
1. Go to **Services** → **HTTP**
2. Check **HTTPS** checkbox (port 443)
3. Ensure service is **On**

### Step 3: Verify Connectivity
From Branch router:
```cisco
ping 172.17.17.100
```
Should succeed (router is not blocked by ACL from inside)

## Common ICMP Types

| Type | Code | Description | Use Case |
|------|------|-------------|----------|
| 8 | 0 | Echo Request | Ping request |
| 0 | 0 | Echo Reply | Ping reply |
| 3 | - | Destination Unreachable | Network/host down |
| 5 | - | Redirect | Route change notification |
| 11 | - | Time Exceeded | TTL expired (traceroute) |

**In our ACL:**
```cisco
deny icmp any host 172.17.17.100 echo
```
Only blocks Type 8 (echo request), not all ICMP.

## Common TCP Ports

| Port | Service | Description |
|------|---------|-------------|
| 20/21 | FTP | File Transfer |
| 22 | SSH | Secure Shell |
| 23 | Telnet | Remote Access (insecure) |
| 25 | SMTP | Email (sending) |
| 53 | DNS | Domain Name Resolution |
| 80 | HTTP | Web (insecure) |
| **443** | **HTTPS** | **Web (secure)** ✓ |
| 3389 | RDP | Remote Desktop |

## ACL Best Practices

### 1. Placement
- **Standard ACLs:** Close to destination
- **Extended ACLs:** Close to source (filters earlier)

### 2. Order Matters
```cisco
! WRONG ORDER:
permit ip any any           ← Matches everything first!
deny icmp any host 172.17.17.100 echo  ← Never reached!

! CORRECT ORDER:
deny icmp any host 172.17.17.100 echo  ← Specific first
permit ip any any           ← General last
```

### 3. Implicit Deny
Every ACL ends with hidden `deny ip any any`.
If no rules match, packet is **denied by default**.

**Our ACL has explicit permit all:**
```cisco
permit ip any any  ← Overrides implicit deny
```

### 4. Documentation
```cisco
ip access-list extended WEB-SERVER-ACL
 remark Block ping to web server
 deny icmp any host 172.17.17.100 echo
 remark Allow HTTPS to web server
 permit tcp any host 172.17.17.100 eq 443
 remark Allow all other traffic
 permit ip any any
```

### 5. Test Before Deployment
- Test with `show access-lists` counters
- Verify expected traffic is permitted
- Confirm unwanted traffic is denied

## Modifying ACLs

### Remove Specific Rule (by sequence number):
```cisco
ip access-list extended WEB-SERVER-ACL
 no 10  ← Removes rule number 10
 exit
```

### Add Rule at Specific Position:
```cisco
ip access-list extended WEB-SERVER-ACL
 15 permit tcp any host 172.17.17.100 eq 80  ← Insert at sequence 15
 exit
```

### Remove Entire ACL:
```cisco
no ip access-list extended WEB-SERVER-ACL
```

**Warning:** If ACL is applied to interface and you delete it, interface has no filtering!

## Troubleshooting ACLs

### Issue 1: Web server not accessible via HTTPS
**Symptoms:** Can't access https://172.17.17.100

**Solutions:**
1. **Check ACL allows HTTPS:**
   ```cisco
   show access-lists
   ```
   Ensure: `permit tcp any host 172.17.17.100 eq 443`

2. **Verify ACL applied correctly:**
   ```cisco
   show ip interface Gi0/1
   ```
   Should show: `Inbound access list is WEB-SERVER-ACL`

3. **Check web server has HTTPS enabled**
   - Services → HTTP → HTTPS checkbox

4. **Verify routing:**
   ```cisco
   show ip route
   ```
   Ensure route to web server exists

### Issue 2: Ping still works to web server
**Symptoms:** Ping succeeds but should be blocked

**Solutions:**
1. **Check ACL order:**
   ```cisco
   show access-lists
   ```
   Deny rule should be **before** permit all

2. **Verify ACL applied:**
   ```cisco
   show ip interface Gi0/1
   ```

3. **Check direction:**
   - Should be `in` on Gi0/1

4. **Verify rule syntax:**
   ```cisco
   deny icmp any host 172.17.17.100 echo
   ```

### Issue 3: ACL counters not increasing
**Symptoms:** `show access-lists` shows 0 matches

**Solutions:**
1. **Generate test traffic:**
   - Ping from HQ PC to web server
   - Access web server via browser

2. **Check ACL is applied:**
   ```cisco
   show running-config interface Gi0/1
   ```
   Should have: `ip access-group WEB-SERVER-ACL in`

3. **Verify traffic is reaching the interface**

### Issue 4: Blocked all traffic accidentally
**Symptoms:** Nothing works after applying ACL

**Solution:**
```cisco
interface GigabitEthernet0/1
 no ip access-group WEB-SERVER-ACL in
 exit
```
This removes the ACL from the interface (ACL still exists, just not applied).

Then fix the ACL and reapply.

## Production-Ready ACL (Alternative)

For a real production environment, you might want:

```cisco
ip access-list extended WEB-SERVER-ACL
 ! Deny ICMP echo (ping) to web server
 deny icmp any host 172.17.17.100 echo
 
 ! Allow HTTPS from anywhere
 permit tcp any host 172.17.17.100 eq 443
 
 ! Allow HTTP (optional)
 permit tcp any host 172.17.17.100 eq 80
 
 ! Allow SSH from management network only
 permit tcp 172.16.30.0 0.0.0.255 host 172.17.17.100 eq 22
 
 ! Allow established connections (return traffic)
 permit tcp any any established
 
 ! Allow OSPF
 permit ospf any any
 
 ! Deny all other traffic (explicit)
 deny ip any any log
 exit
```

## Key Takeaways

✅ **ACLs filter traffic based on rules**  
✅ **Extended ACLs filter by source, destination, protocol, and port**  
✅ **Rules processed top-to-bottom, first match wins**  
✅ **Place extended ACLs close to source**  
✅ **Apply ACL to interface with direction (in/out)**  
✅ **Implicit deny at end of every ACL**  
✅ **Test with counters: show access-lists**  
✅ **Specific rules before general rules**  

## Testing Checklist

- [ ] ACL created: WEB-SERVER-ACL
- [ ] Rule 1: Deny ICMP to 172.17.17.100
- [ ] Rule 2: Permit TCP 443 to 172.17.17.100
- [ ] Rule 3: Permit all other traffic
- [ ] ACL applied to Gi0/1 inbound
- [ ] Web server at 172.17.17.100 configured
- [ ] HTTPS service enabled on server
- [ ] Ping to web server fails from HQ PCs
- [ ] Ping to other Branch devices succeeds
- [ ] HTTPS to web server succeeds
- [ ] ACL match counters increasing
- [ ] `show ip interface` confirms ACL applied

## Next Steps
- Proceed to **08-Verification_Tests.md** for comprehensive system testing

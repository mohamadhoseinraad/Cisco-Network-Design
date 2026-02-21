# Step 03 - NAT/PAT Configuration

## Overview
Network Address Translation (NAT) allows private IP addresses to access the internet using public IP addresses. Port Address Translation (PAT), also called NAT Overload, allows multiple devices to share a single public IP address.

## Why NAT/PAT?
- **IPv4 Conservation:** Allows thousands of devices to share one public IP
- **Security:** Hides internal network structure
- **Flexibility:** Change internal addressing without affecting external connectivity
- **Cost-Effective:** ISPs charge per public IP address

## NAT Types Comparison

| Type | Description | IP Ratio | Use Case |
|------|-------------|----------|----------|
| Static NAT | One-to-one mapping | 1:1 | Servers that need consistent public IPs |
| Dynamic NAT | Pool of public IPs | Many:Many | Multiple public IPs available |
| **PAT (NAT Overload)** | **Port-based** | **Many:1** | **Most common - used in this project** |

## Implementation on R1-HQ

### Step 1: Define Inside Interfaces

"Inside" interfaces are connected to your private network.

```cisco
enable
configure terminal

! VLAN 10 subinterface
interface GigabitEthernet0/1.10
 ip nat inside
 exit

! VLAN 20 subinterface
interface GigabitEthernet0/1.20
 ip nat inside
 exit

! VLAN 30 subinterface
interface GigabitEthernet0/1.30
 ip nat inside
 exit

! GRE Tunnel to Branch
interface Tunnel0
 ip nat inside
 exit
```

### Step 2: Define Outside Interface

"Outside" interface is connected to the internet (ISP).

```cisco
! WAN interface to ISP
interface GigabitEthernet0/0
 ip nat outside
 exit
```

### Step 3: Create Access List for NAT

Define which traffic is allowed to be NATted.

```cisco
! Allow HQ networks (172.16.0.0/16)
access-list 1 permit 172.16.0.0 0.0.255.255

! Allow GRE tunnel network (for branch traffic)
access-list 1 permit 10.10.10.0 0.0.0.3
```

**Wildcard Mask Explanation:**
- `0.0.255.255` matches 172.16.0.0 through 172.16.255.255
- `0.0.0.3` matches 10.10.10.0 through 10.10.10.3

### Step 4: Configure PAT (NAT Overload)

```cisco
! NAT Overload using outside interface IP
ip nat inside source list 1 interface GigabitEthernet0/0 overload
```

**Breakdown:**
- `ip nat inside source list 1` - Traffic matching ACL 1
- `interface GigabitEthernet0/0` - Use this interface's public IP
- `overload` - Enable PAT (multiple devices share one IP)

### Step 5: Save Configuration

```cisco
end
copy running-config startup-config
```

## Implementation on R3-Branch

### Step 1: Define Inside Interfaces

```cisco
enable
configure terminal

! Branch LAN
interface GigabitEthernet0/1
 ip nat inside
 exit

! GRE Tunnel to HQ
interface Tunnel0
 ip nat inside
 exit
```

### Step 2: Define Outside Interface

```cisco
! WAN to ISP
interface GigabitEthernet0/0
 ip nat outside
 exit
```

### Step 3: Create Access List

```cisco
! Allow branch network (172.17.17.0/24)
access-list 1 permit 172.17.17.0 0.0.0.255

! Allow GRE tunnel
access-list 1 permit 10.10.10.0 0.0.0.3
```

### Step 4: Configure PAT

```cisco
ip nat inside source list 1 interface GigabitEthernet0/0 overload
```

### Step 5: Save Configuration

```cisco
end
copy running-config startup-config
```

## How PAT Works

### Example: PC1 (172.16.10.10) accessing Internet (8.8.8.8)

#### Without NAT (Doesn't work):
```
Source: 172.16.10.10 (private)  -X->  Internet
Problem: Internet routers won't route private IPs back
```

#### With PAT (Works):
```
Inside Local          Inside Global           Outside
172.16.10.10:1025  -> 100.100.100.2:1025  ->  8.8.8.8:80
                   <-                      <-
```

### NAT Translation Table Entry:
```
Inside Local          Inside Global           Protocol
172.16.10.10:1025     100.100.100.2:1025      tcp
172.16.10.11:1026     100.100.100.2:1026      tcp
172.16.20.10:1027     100.100.100.2:1027      tcp
```

**Key Point:** Different devices use different port numbers, allowing R1-HQ to track which response goes to which internal device.

## Verification Commands

### Show NAT Translations
```cisco
show ip nat translations
```

**Sample Output:**
```
Pro Inside global      Inside local       Outside local    Outside global
tcp 100.100.100.2:1025 172.16.10.10:1025  8.8.8.8:80      8.8.8.8:80
tcp 100.100.100.2:1026 172.16.20.10:1026  8.8.8.8:80      8.8.8.8:80
```

### Show NAT Statistics
```cisco
show ip nat statistics
```

**Sample Output:**
```
Total active translations: 2 (0 static, 2 dynamic; 2 extended)
Outside interfaces:
  GigabitEthernet0/0
Inside interfaces:
  GigabitEthernet0/1.10
  GigabitEthernet0/1.20
  GigabitEthernet0/1.30
  Tunnel0
Hits: 50  Misses: 0
```

### Clear NAT Translations (if needed)
```cisco
clear ip nat translation *
```

### Debug NAT (for troubleshooting)
```cisco
debug ip nat
```
**Don't forget to turn off:**
```cisco
no debug ip nat
```

## Testing NAT/PAT

### Test 1: Ping Internet from PC
**From PC1 (VLAN 10):**
```
ping 8.8.8.8
```

**Expected:** Success ✓

**What happens:**
1. PC1 sends ICMP to 8.8.8.8 with source 172.16.10.10
2. Packet reaches R1-HQ Gi0/1.10
3. R1-HQ translates source to 100.100.100.2
4. ISP routes packet to 8.8.8.8
5. Reply comes back to 100.100.100.2
6. R1-HQ translates back to 172.16.10.10
7. PC1 receives reply

### Test 2: Check NAT Translation Table
**On R1-HQ after ping:**
```cisco
show ip nat translations
```

You should see the ICMP translation entry.

### Test 3: Verify Statistics
```cisco
show ip nat statistics
```

Check "Hits" counter increases with each NAT translation.

### Test 4: Web Access from PC
**From PC (using Web Browser):**
1. Go to Desktop > Web Browser
2. Enter: http://8.8.8.8 (or any external IP)
3. Should receive response

## NAT Terminology

| Term | Description | Example |
|------|-------------|---------|
| **Inside Local** | Private IP before translation | 172.16.10.10 |
| **Inside Global** | Public IP after translation | 100.100.100.2 |
| **Outside Local** | Destination as seen from inside | 8.8.8.8 |
| **Outside Global** | Actual destination IP | 8.8.8.8 |

## Common Issues and Troubleshooting

### Issue 1: Can't ping 8.8.8.8
**Symptoms:** Ping fails, "Request timed out"

**Solutions:**
1. Check NAT configuration:
   ```cisco
   show running-config | section nat
   ```
2. Verify inside/outside interfaces:
   ```cisco
   show ip nat statistics
   ```
3. Check ACL permits your network:
   ```cisco
   show access-lists
   ```
4. Verify default route exists:
   ```cisco
   show ip route
   ```

### Issue 2: No NAT translations appearing
**Symptoms:** `show ip nat translations` is empty

**Solutions:**
1. Check inside/outside interface designation
2. Verify ACL matches source traffic:
   ```cisco
   show access-lists
   ```
3. Debug NAT to see what's happening:
   ```cisco
   debug ip nat
   ```

### Issue 3: Some devices NAT, others don't
**Symptoms:** PC1 can reach internet, PC2 cannot

**Solutions:**
1. Check ACL covers both subnets (172.16.0.0/16 covers all)
2. Verify both subinterfaces have `ip nat inside`
3. Test from working PC: `ping 8.8.8.8`
4. Test from non-working PC: `ping 8.8.8.8`
5. Check NAT statistics for misses

### Issue 4: NAT working but very slow
**Symptoms:** Internet access is extremely slow

**Solutions:**
1. Clear old translations:
   ```cisco
   clear ip nat translation *
   ```
2. Check for NAT table overflow:
   ```cisco
   show ip nat statistics
   ```
3. Increase NAT timeout (if needed)

## Security Benefits of NAT

1. **IP Hiding:** External devices see only 100.100.100.2, not internal IPs
2. **Reconnaissance Prevention:** Attackers can't easily map internal network
3. **Additional Layer:** Not a firewall, but provides basic protection
4. **Stateful:** Return traffic is only allowed for established connections

## PAT Port Range

### How many devices can share one public IP?
- PAT uses source ports (1024-65535)
- Theoretical maximum: ~64,000 simultaneous connections
- Practical limit: Usually sufficient for small to medium networks

### Port Allocation:
```
Device 1: 100.100.100.2:1024
Device 2: 100.100.100.2:1025
Device 3: 100.100.100.2:1026
...
Device N: 100.100.100.2:65535
```

## Testing Checklist

- [ ] R1-HQ configured with NAT
- [ ] R3-Branch configured with NAT  
- [ ] Inside interfaces designated (`ip nat inside`)
- [ ] Outside interfaces designated (`ip nat outside`)
- [ ] ACLs permit private networks
- [ ] PAT overload configured
- [ ] PC can ping 8.8.8.8
- [ ] NAT translations appear in table
- [ ] NAT statistics show hits

## Next Steps
- Proceed to **04-DHCP_Configuration.md** for automatic IP assignment

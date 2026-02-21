# Step 06 - OSPF Dynamic Routing

## Overview
Open Shortest Path First (OSPF) is a link-state routing protocol that dynamically learns and advertises network routes. It allows R1-HQ and R3-Branch to automatically know how to reach each other's networks through the GRE tunnel.

## Why OSPF?
- **Dynamic:** Automatically updates routes when topology changes
- **Fast Convergence:** Quickly adapts to network changes
- **Scalability:** Works well in large networks
- **Load Balancing:** Can use multiple paths
- **No Hop Count Limit:** Unlike RIP (15 hop limit)

## Without OSPF vs With OSPF

### Without OSPF (Static Routes):
```
R1-HQ:
ip route 172.17.17.0 255.255.255.0 Tunnel0

R3-Branch:
ip route 172.16.10.0 255.255.255.0 Tunnel0
ip route 172.16.20.0 255.255.255.0 Tunnel0
ip route 172.16.30.0 255.255.255.0 Tunnel0
```
**Problem:** Manual configuration, doesn't adapt to changes

### With OSPF (Dynamic):
```
router ospf 1
 network 172.16.0.0 0.0.255.255 area 0
 network 10.10.10.0 0.0.0.3 area 0
```
**Benefit:** Automatic route learning, adaptive to changes

## OSPF Concepts

### Key Terminology:

| Term | Description |
|------|-------------|
| **Router ID** | Unique identifier for each OSPF router |
| **Area** | Logical grouping of networks (Area 0 = backbone) |
| **Network Statement** | Tells OSPF which interfaces to activate |
| **Wildcard Mask** | Inverse of subnet mask (used in network statement) |
| **Adjacency** | Neighbor relationship between two routers |
| **LSA** | Link-State Advertisement (routing information) |
| **Cost** | OSPF metric (based on bandwidth) |
| **Passive Interface** | OSPF enabled but doesn't send Hello packets |

### OSPF Areas:
```
        Area 0 (Backbone)
       /                 \
[R1-HQ]---Tunnel0---[R3-Branch]
   |                      |
HQ Networks         Branch Networks
```

All networks in this project are in **Area 0** (single area design).

## Implementation on R1-HQ

```cisco
enable
configure terminal

! Configure OSPF Process 1
router ospf 1
 
 ! Set Router ID (identifies this router)
 router-id 1.1.1.1
 
 ! Advertise VLAN 10 network
 network 172.16.10.0 0.0.0.255 area 0
 
 ! Advertise VLAN 20 network
 network 172.16.20.0 0.0.0.255 area 0
 
 ! Advertise VLAN 30 network (DHCP Server)
 network 172.16.30.0 0.0.0.255 area 0
 
 ! Advertise GRE Tunnel network
 network 10.10.10.0 0.0.0.3 area 0
 
 ! Prevent OSPF on LAN interfaces (security/efficiency)
 passive-interface GigabitEthernet0/1.10
 passive-interface GigabitEthernet0/1.20
 passive-interface GigabitEthernet0/1.30
 
 exit

! Save configuration
end
copy running-config startup-config
```

## Implementation on R3-Branch

```cisco
enable
configure terminal

! Configure OSPF Process 1
router ospf 1
 
 ! Set Router ID (identifies this router)
 router-id 3.3.3.3
 
 ! Advertise Branch LAN network
 network 172.17.17.0 0.0.0.255 area 0
 
 ! Advertise GRE Tunnel network
 network 10.10.10.0 0.0.0.3 area 0
 
 ! Prevent OSPF on Branch LAN (no other OSPF routers there)
 passive-interface GigabitEthernet0/1
 
 exit

! Save configuration
end
copy running-config startup-config
```

## Understanding the Configuration

### Router ID
- **Format:** IPv4 address (doesn't have to be real IP)
- **Purpose:** Uniquely identifies router in OSPF domain
- **Selection (if not manually set):**
  1. Highest IP on loopback interface
  2. Highest IP on physical interface
- **Best Practice:** Manually set for consistency

### Network Statement
```cisco
network 172.16.10.0 0.0.0.255 area 0
```
**Breakdown:**
- `172.16.10.0` - Network address
- `0.0.0.255` - Wildcard mask
- `area 0` - OSPF area

**What it does:**
1. Finds interfaces whose IP matches this network
2. Enables OSPF on those interfaces
3. Advertises the network to OSPF neighbors

### Wildcard Mask Calculation

**Wildcard = Inverse of Subnet Mask**

| Subnet Mask | Wildcard Mask | Network Example |
|-------------|---------------|-----------------|
| 255.255.255.0 (/24) | 0.0.0.255 | 172.16.10.0 |
| 255.255.255.252 (/30) | 0.0.0.3 | 10.10.10.0 |
| 255.255.0.0 (/16) | 0.0.255.255 | 172.16.0.0 |

**Calculation Example:**
```
Subnet Mask:    255.255.255.0
Binary:         11111111.11111111.11111111.00000000

Wildcard Mask:  0.0.0.255
Binary:         00000000.00000000.00000000.11111111

Rule: 0 = must match, 1 = don't care
```

### Passive Interface
```cisco
passive-interface GigabitEthernet0/1.10
```

**Why use passive-interface?**
1. **Security:** Don't send OSPF Hello packets to LAN (users don't need it)
2. **Efficiency:** Reduces unnecessary OSPF traffic
3. **Still Advertises:** Network is still advertised, just no neighbor relationships

**When to use:**
- Interfaces connected to end users
- Interfaces where no OSPF routers exist

**When NOT to use:**
- Tunnel0 (needs to form adjacency with remote router)

## How OSPF Works Over GRE Tunnel

### Step 1: Hello Packets
```
R1-HQ (Tunnel0) ---Hello---> R3-Branch (Tunnel0)
R1-HQ (Tunnel0) <--Hello---- R3-Branch (Tunnel0)
```

**Hello Packet contains:**
- Router ID
- Area ID
- Network mask
- Hello/Dead intervals
- Authentication (if configured)

### Step 2: Neighbor Adjacency
```
Init → 2-Way → ExStart → Exchange → Loading → Full
```

**Full State:** Routers have complete topology information

### Step 3: LSA Exchange
```
R1-HQ: "I have routes to 172.16.10.0, 172.16.20.0, 172.16.30.0"
R3-Branch: "I have a route to 172.17.17.0"
```

### Step 4: SPF Calculation
Each router runs Dijkstra's algorithm to build routing table.

### Step 5: Routing Table Update
**R1-HQ learns:**
```
O    172.17.17.0/24 [110/1001] via 10.10.10.2, Tunnel0
```

**R3-Branch learns:**
```
O    172.16.10.0/24 [110/1001] via 10.10.10.1, Tunnel0
O    172.16.20.0/24 [110/1001] via 10.10.10.1, Tunnel0
O    172.16.30.0/24 [110/1001] via 10.10.10.1, Tunnel0
```

## Verification Commands

### Show OSPF Neighbors
```cisco
show ip ospf neighbor
```

**Expected Output on R1-HQ:**
```
Neighbor ID     Pri   State           Dead Time   Address         Interface
3.3.3.3         0     FULL/  -        00:00:35    10.10.10.2      Tunnel0
```

**State Meanings:**
- **FULL:** Complete adjacency ✓
- **2WAY:** Neighbors see each other
- **INIT:** Hello received but not acknowledged
- **DOWN:** No communication

### Show OSPF Interfaces
```cisco
show ip ospf interface brief
```

**Expected Output:**
```
Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Tu0          1     0               10.10.10.1/30      1000  P2P   1/1
Gi0/1.10     1     0               172.16.10.1/24     1     DR    0/0
Gi0/1.20     1     0               172.16.20.1/24     1     DR    0/0
Gi0/1.30     1     0               172.16.30.1/24     1     DR    0/0
```

### Show OSPF Routes in Routing Table
```cisco
show ip route ospf
```

**Expected Output on R1-HQ:**
```
O    172.17.17.0/24 [110/1001] via 10.10.10.2, Tunnel0
```

**Expected Output on R3-Branch:**
```
O    172.16.10.0/24 [110/1001] via 10.10.10.1, Tunnel0
O    172.16.20.0/24 [110/1001] via 10.10.10.1, Tunnel0
O    172.16.30.0/24 [110/1001] via 10.10.10.1, Tunnel0
```

**Route Code Meanings:**
- `O` = OSPF route
- `[110/1001]` = [Administrative Distance / Metric]
  - 110 = OSPF default AD
  - 1001 = Cost (lower is better)
- `via 10.10.10.2` = Next hop IP
- `Tunnel0` = Exit interface

### Show OSPF Database
```cisco
show ip ospf database
```

Shows LSAs (Link-State Advertisements) - routing information.

### Show OSPF Configuration
```cisco
show running-config | section router ospf
```

### Show OSPF Process Information
```cisco
show ip ospf
```

**Shows:**
- Router ID
- Area information
- SPF calculations
- Timers

## Testing OSPF

### Test 1: Check OSPF Neighbor
**On R1-HQ:**
```cisco
show ip ospf neighbor
```
**Expected:** Neighbor 3.3.3.3 in FULL state ✓

### Test 2: Verify OSPF Routes
**On R1-HQ:**
```cisco
show ip route ospf
```
**Expected:** Route to 172.17.17.0/24 via Tunnel0 ✓

**On R3-Branch:**
```cisco
show ip route ospf
```
**Expected:** Routes to 172.16.10.0, 172.16.20.0, 172.16.30.0 ✓

### Test 3: Ping from HQ to Branch
**From R1-HQ:**
```cisco
ping 172.17.17.1
```
**Expected:** Success ✓ (R3-Branch LAN interface)

### Test 4: End-to-End Connectivity
**From PC1 (HQ VLAN 10):**
```cmd
ping 172.17.17.10
```
**Expected:** Success ✓ (Branch PC)

**From Branch PC:**
```cmd
ping 172.16.10.10
ping 172.16.20.10
```
**Expected:** Success ✓

## Common Issues and Troubleshooting

### Issue 1: No OSPF neighbor
**Symptoms:** `show ip ospf neighbor` is empty

**Solutions:**

1. **Check tunnel is up:**
   ```cisco
   show ip interface brief | include Tunnel
   ```
   Must be up/up

2. **Verify can ping tunnel IP:**
   ```cisco
   ping 10.10.10.2
   ```

3. **Check OSPF network statements:**
   ```cisco
   show running-config | section ospf
   ```
   Must include: `network 10.10.10.0 0.0.0.3 area 0`

4. **Check tunnel is not passive:**
   ```cisco
   show ip ospf interface
   ```
   Tunnel0 should NOT be passive

5. **Verify area matches:**
   - Both routers must use same area (Area 0)

### Issue 2: Neighbor in INIT state
**Symptoms:** Neighbor stuck in INIT, not reaching FULL

**Solutions:**
1. Check bidirectional communication
2. Verify Hello/Dead intervals match
3. Check for ACLs blocking OSPF (protocol 89)
4. Verify MTU matches on both ends

### Issue 3: Routes not appearing
**Symptoms:** Neighbor is FULL but routes missing

**Solutions:**

1. **Check routing table:**
   ```cisco
   show ip route
   ```

2. **Verify networks are advertised:**
   ```cisco
   show ip ospf database
   ```

3. **Check passive interfaces:**
   - LAN interfaces should be passive
   - Tunnel should NOT be passive

4. **Verify connected networks:**
   ```cisco
   show ip route connected
   ```
   Networks must be connected before OSPF can advertise

### Issue 4: Adjacency flapping
**Symptoms:** Neighbor keeps going up/down

**Causes:**
1. Unstable tunnel (GRE issue)
2. Unstable WAN connection
3. MTU mismatch
4. Hello/Dead timer mismatch

## OSPF Metrics

### Cost Calculation:
```
Cost = Reference Bandwidth / Interface Bandwidth
Default Reference Bandwidth = 100 Mbps
```

**Examples:**
- FastEthernet (100 Mbps): Cost = 100/100 = **1**
- GigabitEthernet (1000 Mbps): Cost = 100/1000 = **1**
- Tunnel (default 9 Kbps): Cost = 100/0.009 = **11111** (shown as 1000)

### Path Selection:
OSPF chooses path with **lowest total cost**.

```
R1-HQ → (Cost 1000) → R3-Branch
Total Cost to 172.17.17.0: 1000
```

## Advanced OSPF Configuration (Optional)

### Adjust Interface Cost:
```cisco
interface Tunnel0
 ip ospf cost 100
 exit
```

### Change Reference Bandwidth:
```cisco
router ospf 1
 auto-cost reference-bandwidth 10000
 exit
```

### Adjust Hello/Dead Timers:
```cisco
interface Tunnel0
 ip ospf hello-interval 10
 ip ospf dead-interval 40
 exit
```

## Key Takeaways

✅ **OSPF provides dynamic routing between HQ and Branch**  
✅ **Tunnel network (10.10.10.0/30) must be advertised in OSPF**  
✅ **Router IDs must be unique**  
✅ **Passive-interface on LAN interfaces (not Tunnel0)**  
✅ **Neighbor must reach FULL state**  
✅ **OSPF routes appear with "O" code**  
✅ **Lower cost = preferred path**  

## Testing Checklist

- [ ] OSPF configured on R1-HQ
- [ ] OSPF configured on R3-Branch
- [ ] Router IDs set (1.1.1.1 and 3.3.3.3)
- [ ] All networks advertised correctly
- [ ] Tunnel0 network (10.10.10.0/30) included
- [ ] LAN interfaces set as passive
- [ ] Tunnel0 NOT passive
- [ ] OSPF neighbor shows FULL state
- [ ] R1-HQ has route to 172.17.17.0/24
- [ ] R3-Branch has routes to 172.16.x.x networks
- [ ] Can ping from HQ PC to Branch PC
- [ ] Can ping from Branch PC to HQ PCs

## Next Steps
- Proceed to **07-ACL_Security.md** to implement access control for the web server

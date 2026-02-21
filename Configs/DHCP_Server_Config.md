# DHCP Server Configuration

## Server Details
- **IP Address:** 172.16.30.10
- **Subnet Mask:** 255.255.255.0
- **Default Gateway:** 172.16.30.1 (R1-HQ)
- **DNS Server:** 8.8.8.8
- **VLAN:** 30 (Server VLAN)

## Configuration Steps in Packet Tracer

### Step 1: Basic Server Setup
1. Click on the DHCP Server
2. Go to **Desktop** > **IP Configuration**
3. Select **Static** IP configuration
4. Enter the following:
   - IP Address: `172.16.30.10`
   - Subnet Mask: `255.255.255.0`
   - Default Gateway: `172.16.30.1`
   - DNS Server: `8.8.8.8`

### Step 2: Configure DHCP Service

#### Pool for VLAN 10 (172.16.10.0/24)
1. Go to **Services** > **DHCP**
2. Click **Add** to create a new pool
3. Configure Pool Name: `VLAN10_Pool`
4. Set the following parameters:

```
Pool Name: VLAN10_Pool
Default Gateway: 172.16.10.1
DNS Server: 8.8.8.8
Start IP Address: 172.16.10.10
Subnet Mask: 255.255.255.0
Maximum Number of Users: 50
```

5. Click **Save**

#### Pool for VLAN 20 (172.16.20.0/24)
1. Click **Add** to create another pool
2. Configure Pool Name: `VLAN20_Pool`
3. Set the following parameters:

```
Pool Name: VLAN20_Pool
Default Gateway: 172.16.20.1
DNS Server: 8.8.8.8
Start IP Address: 172.16.20.10
Subnet Mask: 255.255.255.0
Maximum Number of Users: 50
```

4. Click **Save**

### Step 3: Enable DHCP Service
1. Ensure **Service** is **On** for both pools
2. Close the server configuration window

### Step 4: Verify DHCP Relay
Make sure R1-HQ has the following commands on VLAN subinterfaces:

```cisco
interface GigabitEthernet0/1.10
 ip helper-address 172.16.30.10

interface GigabitEthernet0/1.20
 ip helper-address 172.16.30.10
```

## Testing DHCP

### On Client PCs (VLAN 10 and VLAN 20)
1. Click on a PC
2. Go to **Desktop** > **IP Configuration**
3. Select **DHCP**
4. Click **Request**
5. Verify that IP address is assigned from the correct range

### Expected Results
- **VLAN 10 PCs:** Should receive IPs from 172.16.10.10 - 172.16.10.59
- **VLAN 20 PCs:** Should receive IPs from 172.16.20.10 - 172.16.20.59
- **Default Gateway:** Should match the VLAN gateway (172.16.10.1 or 172.16.20.1)
- **DNS Server:** Should be 8.8.8.8

## Troubleshooting

### DHCP Not Working?
1. **Check DHCP Service:**
   - Ensure DHCP service is **On** for both pools
   
2. **Verify IP Helper Address:**
   - On R1-HQ, verify `ip helper-address 172.16.30.10` on both VLAN subinterfaces
   
3. **Check Connectivity:**
   - From R1-HQ, ping `172.16.30.10` (DHCP server)
   - Should receive replies
   
4. **VLAN Configuration:**
   - Ensure HQ-Switch has correct VLAN assignments
   - Verify trunk port allows VLANs 10, 20, and 30
   
5. **Release/Renew:**
   - On client PC: `ipconfig /release`
   - Then: `ipconfig /renew`

## DHCP Lease Information
View active leases on the DHCP server:
1. Go to **Services** > **DHCP**
2. Check the **Current Leases** section at the bottom
3. You should see assigned IPs, MAC addresses, and lease times

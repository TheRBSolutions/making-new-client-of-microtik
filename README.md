# Complete MikroTik OpenVPN Setup Guide

A comprehensive guide for creating and configuring OpenVPN clients on MikroTik RouterOS.

---

## Table of Contents

1. [Understanding PPP Profiles](#understanding-ppp-profiles)
2. [Prerequisites](#prerequisites)
3. [Step 1: Configure OpenVPN Server](#step-1-configure-openvpn-server)
4. [Step 2: Create VPN Profile](#step-2-create-vpn-profile)
5. [Step 3: Add VPN User Account](#step-3-add-vpn-user-account)
6. [Step 4: Export Certificates](#step-4-export-certificates)
7. [Step 5: Create Client Config File](#step-5-create-client-config-file)
8. [Step 6: Verify and Test](#step-6-verify-and-test)
9. [Troubleshooting](#troubleshooting)

---

## Understanding PPP Profiles

**What is a PPP Profile?**

A PPP (Point-to-Point Protocol) profile in MikroTik defines the **connection settings and rules** for VPN users. Think of it as a **template** that controls:

- IP address assignment
- Encryption settings
- Firewall rules
- Routing behavior
- Session limits
- Compression options

**Why do we need profiles?**

Instead of configuring each VPN user individually, profiles let you:
- Apply consistent settings to multiple users
- Manage security rules centrally
- Simplify user management
- Control network access per group

**Common Profile Parameters:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `local-address` | Router's IP in VPN tunnel | 192.168.33.1 |
| `remote-address` | Pool of IPs for clients | 192.168.33.100-192.168.33.200 |
| `use-encryption` | Require encryption | yes/no |
| `only-one` | Allow only one session per user | yes/no |
| `change-tcp-mss` | Adjust MSS for better performance | yes/default |

---

## Prerequisites

Before starting, ensure you have:

- ✅ MikroTik router with RouterOS 6.x or 7.x
- ✅ Admin access via Winbox, WebFig, or SSH
- ✅ Valid SSL certificate (or generate one)
- ✅ Public IP or DDNS hostname
- ✅ Port forwarding configured (if behind NAT)

---

## Step 1: Configure OpenVPN Server

### 1.1 Check if OpenVPN Server Exists

```bash
/interface ovpn-server server print
```

### 1.2 Enable and Configure OpenVPN Server

```bash
/interface ovpn-server server set enabled=yes port=1194 mode=ip netmask=24 certificate=SERVER-HUSNARA require-client-certificate=no auth=sha1 cipher=aes256 default-profile=VPN-PROFILE
```

**Parameter Explanation:**

- `enabled=yes` - Turns on OpenVPN server
- `port=1194` - Default OpenVPN port (change to 80 if behind NAT)
- `mode=ip` - Use IP mode (not ethernet/bridged)
- `netmask=24` - Subnet mask for VPN clients
- `certificate=SERVER-HUSNARA` - SSL certificate name
- `require-client-certificate=no` - Username/password auth (not cert-based)
- `auth=sha1` - Authentication algorithm
- `cipher=aes256` - Encryption cipher
- `default-profile=VPN-PROFILE` - Profile to use for connections

### 1.3 Verify Server is Running

```bash
/interface ovpn-server server print
```

**Expected output:**
```
enabled: yes
port: 1194
...
```

---

## Step 2: Create VPN Profile

### 2.1 Check Existing Profiles

```bash
/ppp profile print
```

### 2.2 Create New VPN Profile

```bash
/ppp profile add name=VPN-PROFILE local-address=192.168.33.1 remote-address=192.168.33.100-192.168.33.200 use-encryption=yes only-one=yes change-tcp-mss=yes
```

**Parameter Explanation:**

- `name=VPN-PROFILE` - Profile name (reference this when adding users)
- `local-address=192.168.33.1` - **MikroTik's VPN gateway IP**
  - This is YOUR router's address in the VPN tunnel
  - All clients will see this as their gateway
- `remote-address=192.168.33.100-192.168.33.200` - **IP pool for clients**
  - Range of IPs that will be assigned to VPN clients
  - Each connecting user gets one IP from this range
- `use-encryption=yes` - Enforce encrypted connections
- `only-one=yes` - **Restrict multiple logins per user**
  - Prevents same username from connecting twice
  - Good for security
- `change-tcp-mss=yes` - **Optimize packet size**
  - Prevents fragmentation issues
  - Improves VPN performance

### 2.3 Verify Profile Created

```bash
/ppp profile print detail
```

---

## Step 3: Add VPN User Account

### 3.1 Add New VPN User

```bash
/ppp secret add name=test-linux password=TestLinux123 service=ovpn profile=VPN-PROFILE local-address=192.168.33.1 remote-address=192.168.33.200
```

**Parameter Explanation:**

| Parameter | What It Does | Example Value |
|-----------|--------------|---------------|
| `name` | **VPN username** - credentials for login | test-linux |
| `password` | **VPN password** - must match on client | TestLinux123 |
| `service=ovpn` | **Restrict to OpenVPN only** - won't work for PPTP/L2TP | ovpn |
| `profile` | **Use specific profile** - applies all profile settings | VPN-PROFILE |
| `local-address` | **MikroTik's IP for this user** | 192.168.33.1 |
| `remote-address` | **Assign specific IP to this user** | 192.168.33.200 |

**Why specify remote-address?**

- If you want this user to **always get the same IP** (static assignment)
- Useful for firewall rules, port forwarding, or tracking
- If not specified, router assigns from profile's IP pool

### 3.2 Check All VPN Users

```bash
/ppp secret print
```

**Example output:**
```
 #   NAME          SERVICE PASSWORD      PROFILE     LOCAL-ADDRESS  REMOTE-ADDRESS
 0   halinkroad    ovpn    ********      VPN-PROFILE 192.168.33.1   192.168.33.151
 1   umer1         ovpn    ********      VPN-PROFILE 192.168.33.1   172.99.0.6
 2   test-linux    ovpn    TestLinux123  VPN-PROFILE 192.168.33.1   192.168.33.200
```

---

## Step 4: Export Certificates

### 4.1 List Available Certificates

```bash
/certificate print
```

Look for your CA certificate (e.g., `CA-HUSNARA`)

### 4.2 Export CA Certificate

```bash
/certificate export-certificate CA-HUSNARA type=pem
```

**What this does:**
- Creates a `.crt` file containing the CA certificate
- Needed by OpenVPN clients to verify server identity
- File will be: `cert_export_CA-HUSNARA.crt`

### 4.3 Verify Export

```bash
/file print where name~"cert"
```

**Expected output:**
```
 #   NAME                              TYPE        SIZE
 0   cert_export_CA-HUSNARA.crt        .crt file   1234
```

### 4.4 Download Certificate

**Method 1: Via WebFig**
1. Open browser: `https://172.99.0.100`
2. Login with admin credentials
3. Go to: **Files**
4. Click download icon next to `cert_export_CA-HUSNARA.crt`

**Method 2: Via FTP**
```bash
ftp 172.99.0.100
# Login and download cert file
```

**Method 3: Via SCP (if SSH enabled)**
```bash
scp admin@172.99.0.100:/cert_export_CA-HUSNARA.crt ~/
```

---

## Step 5: Create Client Config File

### 5.1 Create .ovpn File on Linux

On your Linux machine:

```bash
nano ~/test-vpn.ovpn
```

### 5.2 Paste Complete Config

```
client
dev tun
proto tcp
remote hgb09mjagx2.sn.mynetname.net 80
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
auth-user-pass
data-ciphers AES-256-GCM:AES-128-GCM:AES-256-CBC
cipher AES-256-CBC
auth SHA1
verb 4

<ca>
-----BEGIN CERTIFICATE-----
[PASTE CONTENT OF cert_export_CA-HUSNARA.crt HERE]
-----END CERTIFICATE-----
</ca>
```

**Configuration Explanation:**

| Line | Purpose |
|------|---------|
| `client` | This is a client config (not server) |
| `dev tun` | Use TUN device (routed mode) |
| `proto tcp` | Use TCP protocol (more reliable through NAT) |
| `remote [hostname] 80` | Server address and port |
| `resolv-retry infinite` | Keep trying if DNS fails |
| `nobind` | Don't bind to local port |
| `persist-key` | Don't re-read keys on restart |
| `persist-tun` | Don't close/reopen TUN on restart |
| `remote-cert-tls server` | Verify server certificate |
| `auth-user-pass` | Use username/password authentication |
| `data-ciphers` | Allowed encryption ciphers |
| `cipher AES-256-CBC` | Primary encryption |
| `auth SHA1` | HMAC authentication |
| `verb 4` | Verbosity level (4 = detailed logs) |

### 5.3 Add CA Certificate Content

```bash
cat cert_export_CA-HUSNARA.crt
```

Copy **everything** including:
- `-----BEGIN CERTIFICATE-----`
- The certificate data
- `-----END CERTIFICATE-----`

Paste between `<ca>` and `</ca>` tags in the .ovpn file.

### 5.4 Save Config File

Press `Ctrl+X`, then `Y`, then `Enter`

---

## Step 6: Verify and Test

### 6.1 Test Port Accessibility (From External Network)

```bash
nmap -p 80 103.73.101.14
```

**Expected result:**
```
PORT   STATE SERVICE
80/tcp open  http
```

If closed, check:
- Port forwarding on ISP router
- Firewall rules on MikroTik
- OpenVPN server is running

### 6.2 Check Firewall Rules

```bash
/ip firewall filter print where chain=input
```

Ensure there's a rule allowing port 80:
```bash
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=80 comment="Allow OpenVPN"
```

### 6.3 Connect from Linux Client

```bash
sudo openvpn --config ~/test-vpn.ovpn
```

**Enter credentials when prompted:**
- Username: `test-linux`
- Password: `TestLinux123`

### 6.4 Verify Connection on MikroTik

```bash
/ppp active print
```

**Expected output:**
```
 #   NAME        SERVICE CALLER-ID          ADDRESS          UPTIME
 0   test-linux  ovpn    203.45.67.89       192.168.33.200   00:01:23
```

### 6.5 Test Connectivity from Client

From the connected VPN client:

```bash
ping 192.168.33.1    # Ping MikroTik gateway
ping 172.99.0.100    # Ping office network
```

---

## What Happens When Client Connects

### Network Topology

```
┌─────────────────┐         VPN Tunnel        ┌──────────────────┐
│  Client         │◄─────────────────────────►│  MikroTik Router │
│  192.168.33.200 │                            │  192.168.33.1    │
└─────────────────┘                            └──────────────────┘
                                                         │
                                                         │
                                               ┌─────────▼─────────┐
                                               │  Office Network   │
                                               │  172.99.0.0/16    │
                                               └───────────────────┘
```

### Before VPN Connection

```
Client IP: 203.45.67.89 (public internet IP)
Can access: Only public internet
Cannot access: 172.99.0.x network ❌
Cannot access: 192.168.33.x network ❌
```

### After VPN Connection

```
Client IP: 192.168.33.200 (VPN tunnel IP)
Can access: Office network (172.99.0.x) ✅
Can access: Server network (192.168.33.x) ✅
Routes internet: Through MikroTik router ✅
Gateway: 192.168.33.1 (MikroTik) ✅
```

---

## Troubleshooting

### Issue 1: Port 80 Closed from Outside

**Symptoms:**
```bash
nmap -p 80 103.73.101.14
PORT   STATE  SERVICE
80/tcp closed http
```

**Solutions:**

1. **Check if behind NAT:**
   ```bash
   /ip address print where interface=CABLENET-ether2
   ```
   If shows private IP (192.168.x.x), you need port forwarding on ISP router.

2. **Add firewall rule:**
   ```bash
   /ip firewall filter add chain=input action=accept protocol=tcp dst-port=80 place-before=0 comment="Allow OpenVPN"
   ```

3. **Verify OpenVPN server is on port 80:**
   ```bash
   /interface ovpn-server server print
   ```

### Issue 2: Connection Timeout

**Check logs:**
```bash
/log print where message~"ovpn"
```

**Common causes:**
- Wrong port in client config
- Firewall blocking connections
- Certificate mismatch
- Server not running

### Issue 3: Authentication Failed

**Verify credentials:**
```bash
/ppp secret print where name="test-linux"
```

Make sure username/password match exactly.

### Issue 4: Connected but Can't Access Office Network

**Check routing on MikroTik:**
```bash
/ip route print
```

**Add route if needed:**
```bash
/ip route add dst-address=172.99.0.0/16 gateway=192.168.33.200
```

**Check firewall forward rules:**
```bash
/ip firewall filter print where chain=forward
```

---

## Advanced Configuration

### Allow Multiple Connections Per User

```bash
/ppp profile set VPN-PROFILE only-one=no
```

### Assign DNS Servers to VPN Clients

```bash
/ppp profile set VPN-PROFILE dns-server=8.8.8.8,8.8.4.4
```

### Limit Bandwidth for VPN Users

```bash
/ppp profile set VPN-PROFILE rate-limit=10M/10M
```

### Enable Compression

```bash
/ppp profile set VPN-PROFILE use-compression=yes
```

---

## Security Best Practices

1. **Use strong passwords:**
   ```bash
   /ppp secret set test-linux password="MyStr0ng!P@ssw0rd2025"
   ```

2. **Restrict to OpenVPN only:**
   - Always use `service=ovpn` to prevent PPTP/L2TP access

3. **Enable firewall rules:**
   - Only allow VPN from specific countries/IPs if possible

4. **Monitor active sessions:**
   ```bash
   /ppp active print
   ```

5. **Check logs regularly:**
   ```bash
   /log print where topics~"ovpn"
   ```

6. **Use certificate-based auth (advanced):**
   ```bash
   /interface ovpn-server server set require-client-certificate=yes
   ```

---

## Quick Reference Commands

### Check Status
```bash
/interface ovpn-server server print       # Server status
/ppp profile print                        # List profiles
/ppp secret print                         # List VPN users
/ppp active print                         # Active connections
/log print where topics~"ovpn"            # OpenVPN logs
```

### Manage Users
```bash
/ppp secret add name=user1 password=pass123 service=ovpn profile=VPN-PROFILE
/ppp secret remove [find name="user1"]
/ppp secret set [find name="user1"] password=newpass456
/ppp secret disable [find name="user1"]
/ppp secret enable [find name="user1"]
```

### Disconnect User
```bash
/ppp active remove [find name="test-linux"]
```

---

## Project Billing Information

**Complexity:** Medium

**Why Medium Complexity:**
- Requires understanding of VPN concepts
- Multiple steps (server setup, profile config, user creation, certificate export)
- Troubleshooting NAT/port forwarding issues
- Testing from external network

**Timeline:** 3-4 hours
- Server configuration: 1 hour
- User setup and testing: 1 hour
- Troubleshooting port forwarding: 1-2 hours

**Your Rate:** $3/hour

**Budget:**
- **Minimum:** $9 USD (₹757 INR) - 3 hours
- **Maximum:** $12 USD (₹1,010 INR) - 4 hours

**Recommended:** $12 USD (₹1,010 INR) to account for potential troubleshooting.

---

## License

MIT License - Feel free to use and modify.

---

## Contributing

Pull requests welcome! Please test all commands before submitting.

---

## Author

**RB Solutions**  
Network & System Administration  
[GitHub](https://github.com/TheRBSolutions)

---

**Last Updated:** October 29, 2025

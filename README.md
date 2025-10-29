# making-new-client-of-microtik

/ppp secret add name=test-linux password=TestLinux123 service=ovpn profile=VPN-PROFILE local-address=192.168.33.1 remote-address=192.168.33.200
```

---

### What Each Part Does:

**`/ppp secret add`**
- Creates a new VPN user account
- PPP = Point-to-Point Protocol (used by VPN)

---

**`name=test-linux`**
- **Username** for VPN login
- Client will use this to connect
- Like: "test-linux" logs in

---

**`password=TestLinux123`**
- **Password** for this user
- Client needs this to authenticate

---

**`service=ovpn`**
- Specifies this is for **OpenVPN** only
- Won't work for PPTP, L2TP, etc.
- Only OpenVPN connections can use this account

---

**`profile=VPN-PROFILE`**
- Uses existing profile settings
- Profile contains: firewall rules, routing, encryption settings
- Check your profiles: `/ppp profile print`

---

**`local-address=192.168.33.1`**
- **MikroTik's VPN IP** (server side)
- This is YOUR router's address in the VPN tunnel
- Client will see this as the gateway

---

**`remote-address=192.168.33.200`**
- **Client's VPN IP** (assigned to test-linux user)
- When test-linux connects, they get IP: 192.168.33.200
- This IP is used to identify this client in VPN tunnel

---

## What Happens When Client Connects:
```
[Client Computer] ← VPN Tunnel → [MikroTik Router]
   192.168.33.200                    192.168.33.1
```

**Client can then access:**
- Your office network (172.99.0.x)
- Your server (192.168.33.x)
- Internet through your router

---

## Visual Example:

**Before VPN:**
```
Client: 39.61.51.154 (random internet IP)
Cannot access: 172.99.0.x network
```

**After VPN connected:**
```
Client: 192.168.33.200 (VPN IP)
Can access: 172.99.0.x network
Can access: 192.168.33.x server
Routes through: MikroTik (192.168.33.1)
```

---

## Check Existing VPN Users:
```
/ppp secret print

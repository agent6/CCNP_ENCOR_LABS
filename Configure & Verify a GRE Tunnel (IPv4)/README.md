## Student Lab Guide  
### Configure & Verify a GRE Tunnel (IPv4)

Load CML lab https://github.com/agent6/CCNP_ENCOR_LABS/blob/main/Configure%20%26%20Verify%20a%20GRE%20Tunnel%20(IPv4)/Configure_Verify_a_GRE_Tunnel_(IPv4).yaml
All devices already have their *baseline* configuration (WAN /30 links, LAN /24 links, and BGP between each site and the ISP).  
At the moment:

* **SiteA-R ↔ SiteB-R** can ping across the ISP (10.0.0.1 ↔ 10.0.0.5).  
* **SiteA-PC (192.168.1.10)** **cannot** reach **SiteB-PC (192.168.2.10)**.

Your tasks:

> 1. Build **GRE Tunnel0** between the two site routers  
> 2. Add static routes so that the LANs use the tunnel  
> 3. Verify the tunnel and end-to-end reachability  

---

### 1 Build Tunnel0 on both routers

|  | **SiteA-R (AS 65100)** | **SiteB-R (AS 65200)** |
|---|---|---|
| 1 | `conf t` | `conf t` |
| 2 | `interface Tunnel0` | `interface Tunnel0` |
| 3 | `ip address 172.16.0.1 255.255.255.252` | `ip address 172.16.0.2 255.255.255.252` |
| 4 | `tunnel source GigabitEthernet0/0` | `tunnel source GigabitEthernet0/0` |
| 5 | `tunnel destination 10.0.0.5` | `tunnel destination 10.0.0.1` |
| 6 | `no shutdown` | `no shutdown` |
| 7 | `end` | `end` |

*Why?* `source` and `destination` point to the public WAN IPs so GRE packets traverse the ISP.  
No `tunnel mode` is necessary—IOSv defaults to GRE /IP.

---

### 2 Add static LAN routes

| Router | Command (global-config mode) |
|--------|-----------------------------|
| **SiteA-R** | `ip route 192.168.2.0 255.255.255.0 172.16.0.2` |
| **SiteB-R** | `ip route 192.168.1.0 255.255.255.0 172.16.0.1` |

These direct all LAN traffic into the new GRE tunnel.

---

### 3 Verify everything works

| Check | Command | What you should see |
|-------|---------|---------------------|
| Tunnel state | `show ip interface brief` | `Tunnel0 … up up` |
| GRE details | `show interfaces Tunnel0` | Protocol up; Src 10.0.0.x, Dst 10.0.0.x |
| Static routes | `show ip route static` | S 192.168.1.0/24 or /24 via Tunnel0 |
| Ping peer tunnel | `SiteA-R# ping 172.16.0.2` | 5 × `!` |
| End-to-end ping | `SiteA-PC$ ping 192.168.2.10` | Replies received |
| Trace (optional) | `SiteA-PC$ traceroute 192.168.2.10` | Hop 1 = 192.168.1.1, Hop 2 = 172.16.0.2 |

Troubleshooting tips if something is down:

* Re-check `tunnel destination` IPs—both ends must be correct.  
* Make sure each router can still ping its peer’s **WAN** address.  
* Confirm you used `/30` masks on Tunnel0 and `/24` masks on LANs.

---

### 4 Save (optional)

```plaintext
SiteA-R# copy run start
SiteB-R# copy run start
```

You’ve completed the three tasks—**both PCs can now communicate through the GRE tunnel.**

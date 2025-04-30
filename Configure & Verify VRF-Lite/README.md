## Student Lab Guide  
### Configure & Verify VRF-Lite 

![image](https://github.com/agent6/CCNP_ENCOR_LABS/blob/main/Configure%20%26%20Verify%20VRF-Lite/VRF-Lite-Diagram.png)

[Load CML Lab](https://github.com/agent6/CCNP_ENCOR_LABS/blob/main/Configure%20%26%20Verify%20VRF-Lite/VRF-Lite_Lab.yaml)

All devices already have their **baseline** configuration:

* Four PCs are up with the IPs shown in the diagram.
* Both routers have every interface administratively **up** but **no IP addresses**.
* No VRFs exist yet, so every ping across sites presently fails.

Your tasks:

> 1. Create **RED** and **GREEN** VRFs on both routers.  
> 2. Bring up the WAN link, then build two dot1Q sub-interfaces (VLAN 10 for RED, VLAN 20 for GREEN).  
> 3. Bind each LAN interface to the correct VRF and give it an IP.  
> 4. Add static routes so each VRF knows the remote /24.  
> 5. Verify that hosts can reach only their own colour-mates—even though both colours reuse the same subnets.

---

### 1 Create the VRF instances (both routers)

```plaintext
conf t
 ip vrf RED
 exit
 ip vrf GREEN
 exit
end
```

*Context* – A VRF is just a second (or third) routing table.  In VRF-Lite the `rd` (route-distinguisher) is optional, so we skip it for cleaner configs.

---

### 2 Transport link: one trunk, two logical paths

#### 2-A  Turn the physical cable on

```plaintext
conf t
 interface GigabitEthernet0/0
  no shutdown
 exit
end
```

*Why* – Sub-interfaces inherit the state of the parent; if Gi0/0 is down, nothing will pass.

#### 2-B  Add a sub-interface per colour

|  | **SiteA-R** | **SiteB-R** |
|---|---|---|
| RED path | `int Gi0/0.10`<br>`encap dot1Q 10`<br>`ip vrf forwarding RED`<br>`ip addr 172.16.10.1 255.255.255.252` | same commands but IP **172.16.10.2 /30** |
| GREEN path | `int Gi0/0.20`<br>`encap dot1Q 20`<br>`ip vrf forwarding GREEN`<br>`ip addr 172.16.20.1 255.255.255.252` | same commands but IP **172.16.20.2 /30** |

*Context* – VLAN tags 10 and 20 keep the two VRFs logically separate while sharing the copper.  
No `tunnel` tricks: plain Ethernet trunk.

---

### 3 Hook the LAN cables into their VRFs

|  | **SiteA-R** | **SiteB-R** |
|---|---|---|
| **Gig 0/1** (PC-1) | `int Gi0/1`<br>`no shut`<br>`ip vrf forwarding RED`<br>`ip addr 10.1.1.1 255.255.255.0` | `int Gi0/1`<br>`no shut`<br>`ip vrf forwarding RED`<br>`ip addr 10.2.2.1 255.255.255.0` |
| **Gig 0/2** (PC-2) | `int Gi0/2`<br>`no shut`<br>`ip vrf forwarding GREEN`<br>`ip addr 10.1.1.1 255.255.255.0` | `int Gi0/2`<br>`no shut`<br>`ip vrf forwarding GREEN`<br>`ip addr 10.2.2.1 255.255.255.0` |

*Context* – The **same IP** appears twice on each router, but in different VRFs, so IOS-XE is happy.

---

### 4 Static routes (per VRF, both directions)

```plaintext
! On SiteA-R
ip route vrf RED   10.2.2.0 255.255.255.0 172.16.10.2
ip route vrf GREEN 10.2.2.0 255.255.255.0 172.16.20.2
!
! On SiteB-R
ip route vrf RED   10.1.1.0 255.255.255.0 172.16.10.1
ip route vrf GREEN 10.1.1.0 255.255.255.0 172.16.20.1
```

*Context* – Only four routes total.  Later you could replace these with OSPF-per-VRF or MP-BGP.

---

### 5 Validate isolation and reachability

| Test | Expect |
|------|--------|
| `PC-A1$ ping 10.2.2.10` | **Success** (same RED VRF) |
| `PC-A1$ ping 10.2.2.20` | **Fail** (GREEN VRF) |
| `PC-A2$ ping 10.2.2.20` | **Success** |
| `PC-A2$ ping 10.2.2.10` | **Fail** |

Router deep-dives:

```plaintext
show ip vrf                         ! list VRFs & members
show ip route vrf RED               ! only /30 + local /24
show ip cef vrf GREEN summary       ! separate FIB
traceroute vrf RED 10.2.2.10        ! hop 1 10.1.1.1, hop 2 172.16.10.2
```

If pings fail:

* Check the sub-interface VLAN tags match on both sides.  
* Confirm you typed **`ip vrf forwarding …` before** the `ip address`; doing it afterwards wipes the address.  
* Verify the static routes with `show run | inc ip route vrf`.

---

### 6 Muscle-memory drill

1. **Lab ▸ Stop Lab**  
2. **Lab ▸ Wipe Lab** (restores blank configs)  
3. **Lab ▸ Start Lab**  
4. Re-enter steps 1–4.  

Repeat until the sequence is effortless—VRF-Lite mastered!

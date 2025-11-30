# GNS3 Network Simulation Project (Part B)

## Group Project – Multi-Site Enterprise Network  
This project simulates a realistic enterprise network consisting of multiple geographic locations (HQ, Branch, and DMZ), a central firewall, intrusion detection, and honeypots. All routing, switching, and security appliances are emulated using **GNS3**, **Cisco Routers**, **Wireshark**, **FortiGate FIrewall**, **Snort IDS**, **Ubuntu Honeypot**, and **VPCS**.

---

## 1. Project Overview

This GNS3 topology models a multi-location corporate infrastructure with the following components:

- **HQ Site** with two VLAN subnets  
- **Branch Site** with two VLAN subnets  
- **DMZ Site** hosting servers and honeypots  
- **Main Core Router (R1)** connecting all other sites  
- **FortiGate firewall** performing NAT, routing, segmentation, and security policies  
- **Snort IDS** deployed inline at HQ for detection  
- **Ubuntu-based Honeypot** in DMZ for adversary interaction  
- **Kali Linux attacker node** for testing  
- **Wireshark capture node** for monitoring  
- **Layer-2 and Layer-3 switches**

The final topology demonstrates real-world enterprise routing, segmentation, traffic inspection, and network defense mechanisms.

---

## 2. Final Network Diagram


<img width="1067" height="702" alt="image" src="https://github.com/user-attachments/assets/b2e2bb5f-48a7-4e10-87a7-e8058a28b9fb" />


Network architecture reference:

```
R1 ↔ HQ (R2)
R1 ↔ Branch (R3)
R1 ↔ FortiGate (port2)
FortiGate ↔ NAT Cloud (port1 – WAN)
FortiGate ↔ DMZ Router R4 (port3)
```

Switches connect all PCs, IDS, Honeypots, and Kali Linux nodes as shown in the topology diagram.

---

## 3. IP Addressing Summary

### HQ Network (R2)
| Interface | Subnet | IP |
|----------|--------|-----|
| f0/0 | 10.0.1.0/30 | 10.0.1.1 |
| f1/0 | 192.168.10.0/24 | 192.168.10.1 |
| f2/0 | 192.168.20.0/24 | 192.168.20.1 |

### Main Router (R1)
| Interface | Subnet | IP |
|----------|--------|-----|
| f1/0 | 10.0.1.2 | HQ link |
| f2/0 | 10.0.2.2 | Branch link |
| f0/0 | 10.0.0.1 | To FortiGate port2 |

### Branch Router (R3)
| Interface | Subnet | IP |
|----------|--------|-----|
| f0/0 | 10.0.2.1 |
| f1/0 | 192.168.30.1 |
| f2/0 | 192.168.40.1 |

### DMZ Router (R4)
| Interface | Subnet | IP |
|----------|--------|-----|
| f0/1 | 172.168.10.1 |
| f0/0 | 172.168.100.1 | To FortiGate |

### Firewall (FortiGate)
| Port | Subnet | IP |
|------|---------|------|
| port1 | 192.168.122.0/24 | DHCP from NAT |
| port2 | 10.0.0.0/24 | 10.0.0.254 |
| port3 | 172.168.100.0/24 | 172.168.100.2 |

---

## 4. PC Address Table

| Location | PC | IP |
|---------|----|----------|
| HQ | PC1 | 192.168.10.10 |
| HQ | PC2 | 192.168.10.11 |
| HQ | Kali Linux | 192.168.10.12 |
| HQ | PC3 | 192.168.20.10 |
| HQ | PC4 | 192.168.20.11 |
| HQ | PC5 | 192.168.20.12 |
| Branch | PC6 | 192.168.30.10 |
| Branch | Wireshark | 192.168.30.11 |
| Branch | PC7 | 192.168.40.10 |
| Branch | PC8 | 192.168.40.11 |
| DMZ | Honeypot | 172.168.10.10 |
| DMZ | PC9 | 172.168.10.11 |


---

## 5. Router Configuration Summary

All routers use static routing.

### R1 (Core Router)
```bash
interface f0/0
 ip address 10.0.0.1 255.255.255.0
interface f1/0
 ip address 10.0.1.2 255.255.255.252
interface f2/0
 ip address 10.0.2.2 255.255.255.252

ip route 0.0.0.0 0.0.0.0 10.0.0.254
ip route 192.168.10.0 255.255.255.0 10.0.1.1
ip route 192.168.20.0 255.255.255.0 10.0.1.1
ip route 192.168.30.0 255.255.255.0 10.0.2.1
ip route 192.168.40.0 255.255.255.0 10.0.2.1
ip route 172.168.10.0 255.255.255.0 172.168.100.1
end
wr mem
```

(Similar relevant configurations are applied to R2, R3, and R4.)

---

## 6. FortiGate Firewall Configuration

### 6.1 Interface Setup
```bash
config system interface
edit port1
 set mode dhcp
 set allowaccess ping http https ssh
next
edit port2
 set ip 10.0.0.254/24
next
edit port3
 set ip 172.168.100.2/24
next
end
```

### 6.2 Static Routes
```bash
config router static
edit 1
 set gateway 192.168.122.1
 set device port1
next
edit 2
 set dst 10.0.0.0/24
 set device port2
next
edit 3
 set dst 172.168.10.0/24
 set gateway 172.168.100.1
 set device port3
next
end
```

### 6.3 Firewall Policies

#### LAN/DMZ → Internet
```bash
config firewall policy
edit 1
 set srcintf "port2" "port3"
 set dstintf "port1"
 set action accept
 set service "ALL"
 set nat enable
next
end
```

#### HQ ↔ DMZ
```bash
edit 2
 set srcintf "port2"
 set dstintf "port3"
 set action accept
next
```
#### Firewall address rule for the next policy
```bash

config firewall address
    edit "GoogleDNS"
        set subnet 8.8.8.8 255.255.255.255
    next
end
```
#### DMZ Restricted Rule
```bash
edit 3
 set srcintf "port3"
 set dstaddr "GoogleDNS"
 set action deny
next
end
```

---

## 7. Security Tools

### Snort IDS
- Connected behind Switch-HQ2  
- Mirrors HQ LAN2 traffic  
- Detects attacks from Kali Linux (scans, brute-force, exploits)

### Honeypot (Ubuntu Cowrie) 
- Located in DMZ (172.168.10.0/24)  
- Captures attacker activity  
- Used for threat analysis
- [Current Limitation: Works on CLI & supports SSH only]
  
### Kali Linux
- Used to perform attacks and scanning  
- Static config:
```bash
ip addr add 192.168.10.12/24 dev eth0
ip route add default via 192.168.10.1
```

### Wireshark Node
- Packet capture node in the Branch network



---

## Conclusion

This project demonstrates key enterprise networking & this GNS3 simulation successfully models a multi-site corporate network with segmentation, routing, firewall rules, IDS monitoring, and honeypot security.



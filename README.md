# Cyber Labs — Enterprise Network Design & Incident Response

An enterprise-scale network — HQ core with redundant gateways, a loop-safe access layer, dual-path WAN to a branch site, controlled internet egress, and full traffic segmentation — designed, deployed, broken, and documented the way a real NOC/network engineering team would. 19 devices, 8+ integrated technologies, 12 real build-time issues diagnosed and resolved, 4 staged incident-response tests with full tickets.

Built in Cisco Packet Tracer.

Scenario

Cyber Labs is a mid-sized company with a Head Office and one branch office, redesigning its network after two real incidents: a single core switch failure took down HQ internet access with no gateway redundancy in place, and a rogue DHCP server on an access port broke connectivity for a department. The brief: eliminate single points of failure, harden against misconfiguration, and prove every protection actually works — not just that it's configured.

Requirements:

Three HQ departments (Sales, IT, Finance), each its own VLAN/subnet, with Sales↔Finance isolation and IT able to reach both
No single point of failure at the gateway (→ HSRP)
No single point of failure in the access-layer switching path, without causing a loop (→ STP)
Branch office stays connected even if the primary WAN link fails (→ OSPF, triangulated WAN mesh)
Single controlled internet exit point, internal addressing hidden (→ NAT/PAT)
Only the legitimate DHCP server may hand out leases (→ DHCP Snooping)
Only authorized devices may use an access port (→ Port Security)
Secure, remote device management only — no cleartext protocols (→ SSH)
Segmentation enforced at the routing layer, not by convention (→ ACLs, duplicated across both cores)
Centralized visibility into every critical device (→ Syslog + SNMP)
Every fault — real or staged — documented with full incident tickets
Topology

topology.png

Layer	Devices	Role
Core	2× L3 switch (CoreA, CoreB)	HSRP-paired gateways, inter-VLAN routing, split active roles
Access	3× L2 switch, triangle mesh	STP loop prevention with a live redundant path
Edge	1× router	NAT/PAT, OSPF hub to WAN mesh
WAN mesh	Edge + Backup WAN + Branch router, full triangle	OSPF with a genuine second path, not just a second interface
Branch	1× router, 1× switch	Site B gateway and LAN
Internet sim	1× router (loopback as test destination)	Stand-in ISP, since Packet Tracer has no routable internet cloud
Monitoring	1× server	Syslog receiver, SNMP poll target (IT VLAN)
End users	8× PC	6 HQ (2 per VLAN), 2 Branch
IP Addressing
Segment	Subnet
Sales (VLAN 10)	10.10.10.0/24
IT (VLAN 20)	10.10.20.0/24
Finance (VLAN 30)	10.10.30.0/24
Core A ↔ Core B	10.10.99.0/30
Edge ↔ Core A / Core B / Backup WAN / Branch	10.255.255.0/30, .4/30, .8/30, .12/30
Backup WAN ↔ Branch	10.255.255.16/30
Branch LAN	10.20.10.0/24
Edge ↔ ISP (simulated public range)	203.0.113.0/30
Design Decisions
HSRP active roles split across VLANs (Core A active for Sales/IT, Core B active for Finance) — both cores do real work day-to-day rather than one sitting fully idle as a pure spare.
Full WAN mesh (Edge/Backup/Branch all directly connected), not a simple primary+backup chain — gives OSPF a genuinely independent second path, not a single-router dependency disguised as redundancy.
ACLs duplicated on both cores, on both the Sales and Finance SVIs — segmentation must hold regardless of which core is Active at any given moment; see Deployment Log, Issue #4 for the real gap this closed.
Deployment Log — Build-Time Issues

Real issues hit and resolved while standing up the network — not staged faults. Documented separately from the incident tickets below since these are normal infrastructure deployment troubleshooting, not simulated live incidents.

#	Issue	Root Cause	Fix
1	Edge Router interfaces rejected ip address	Slots populated with L2 EtherSwitch modules, not routed Ethernet	Swapped to NIM-2T serial modules
2	Serial links wouldn't come up	Needed a distinct "Serial DCE" cable type	Used correct cable; set clock rate on DCE end
3	ACL drops looked like failures	Assumed drops were silent	ip unreachables sends an explicit ICMP admin-prohibited reply by default — this is the correct signature of a working ACL
4	Isolation silently disappeared after HSRP failover	ACLs only existed on whichever core was Active for a VLAN at config time	Duplicated both ACLs onto both VLAN10 and VLAN30 SVIs, on both cores
5	Core-to-core traffic broken during failover testing	Core A↔Core B link had a correct OSPF network statement but the interface itself was never given an IP (no switchport + address missing)	Assigned real IPs on both ends — adjacency formed immediately
6	Confusing "unreachable" during a simulated SVI failure	shutdown on an SVI disables L3 routing but NOT L2 switching — traffic was still physically forwarded by the "failed" core	Simulate real failures by powering off the device, not shutting a single SVI
7	Misunderstood DHCP snooping "trust"	Assumed trust gated whether a binding occurs	Trust only gates server-side offers/acks; client requests always pass
8	DHCP offers dropped intermittently in the access triangle	Only one inter-switch link was trusted; snooping doesn't adapt to current STP forwarding state	Trust every infrastructure-facing port unconditionally
9	DHCP DISCOVER dropped with "Option 82; not configured to trust relay information"	DHCP snooping inserts Option 82; Core A (the DHCP server) wasn't configured to accept it	ip dhcp relay information trust-all on Core A
10	snmp-server location/contact/enable traps rejected	This IOS image only implements the community sub-command	Documented as a platform limitation, not a config error
11	logging trap informational rejected	Image only accepts debugging as an explicit keyword	Unneeded — informational was already the default trap level
12	Syslog and SNMP both silently failed	Monitoring server's switchport was in the wrong VLAN relative to its configured IP — unreachable at Layer 2 despite a "correct" IP/gateway	show vlan brief confirmed the mismatch; corrected port VLAN assignment

The throughline: nearly every serious issue shared the same shape — a setting that looked correct in isolation but was invisible or inconsistent from a different vantage point in the topology. None were wrong commands; they were correct commands applied to an incomplete picture of the topology's actual redundant paths.

Incident Response — Staged Fault Injection
Ticket	Title	Status
INC-001	HQ core switch failure — HSRP failover validation	✅ Resolved
INC-002	Branch WAN link failure — OSPF automatic reroute	✅ Resolved
INC-003	Rogue DHCP server on an access port	✅ Resolved
INC-004	Unauthorized device on a port-secured link	✅ Resolved
Monitoring — Zabbix (Real Deployment)

Packet Tracer has no functional SNMP poller or alerting engine, so proactive monitoring is demonstrated separately on a real Linux VM: Zabbix installed and configured to monitor live hosts, with real triggers (host down, high CPU, interface down) and a genuinely fired alert, documented with setup commands and screenshots. See /zabbix.

Known Platform Limitations
This IOS image's SNMP support is limited to the community sub-command — location, contact, and enable traps are unavailable on this platform.
Packet Tracer has no built-in SNMP manager/poller. SNMP was verified via the PC-PT MIB Browser (manual query only); automated polling and alerting is demonstrated separately via a real Zabbix installation on a VM — see /zabbix.
Packet Tracer's HSRP simulation produced one logically inconsistent log line during INC-001 (an HSRP state change on an interface that had just gone down) — documented in that ticket as a likely simulation quirk rather than a real design issue.
Verification Commands
show standby brief              — HSRP active/standby split
show ip ospf neighbor           — full adjacency on every device
show ip route ospf              — default route learned everywhere
show ip access-lists            — ACL match counters
show ip nat translations        — PAT actively translating
show port-security interface    — sticky MAC, violation state
show ip dhcp snooping binding   — real client leases tracked
show logging                    — syslog destination + message count
show vlan brief                 — port-to-VLAN assignment
Tools Used
Cisco Packet Tracer
Zabbix (separate real deployment, monitoring/alerting proof)
Repo Structure
/screenshots     — topology, verification evidence, fault + recovery captures
/configs         — final working configs per device
/tickets         — INC-001 through INC-004, staged incident-response tests
/project-file    — the .pkt lab file itself, open and explore directly
/license
README.md

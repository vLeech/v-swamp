# FS Router configuration
**Note**: setup extremely quickly if starting from factory defaults via eth1 and web ui.
## Setting up the firewall:
### 1. Build `WAN_IN`: this protects networks behind the router.  
WAN_IN handles packets that arrive through eth0 and are being forwarded through the router—for example, an office computer attempting to reach a switch on 172.30.2.0/24.  
#### Creating the policy does not affect traffic yet. It only becomes active after we attach it to eth0.
```
set firewall name WAN_IN description 'PROTECT_LAB_FROM_OFFICE'
set firewall name WAN_IN default-action drop
```
This creates the policy and says: if traffic does not match an explicit permit rule, discard it.  
#### Now permit return traffic:  
```
set firewall name WAN_IN rule 10 description 'ALLOW_ESTABLISHED_RELATED'
set firewall name WAN_IN rule 10 action accept
set firewall name WAN_IN rule 10 state established enable
set firewall name WAN_IN rule 10 state related enable
```
**Why this is necessary**:  
`established` permits packets belonging to a connection that a lab device already initiated.  
`related` permits supporting traffic associated with an existing connection.  
It does not permit an office device to initiate an arbitrary new connection into the lab.  
#### Now explicitly discard invalid connection-state traffic:  
```
set firewall name WAN_IN rule 20 description 'DROP_INVALID'
set firewall name WAN_IN rule 20 action drop
set firewall name WAN_IN rule 20 state invalid enable
```
`invalid` packets cannot be associated properly with a recognized connection. They are normally malformed, stale, or otherwise unsafe to forward.
**Note**: check pending changes with `compare`
  
### 2. Build `WAN_LOCAL`: protects the router itself.
#### Create `WAN_LOCAL`: This protects services running on the router, including:
- SSH
- Web Administration
- DNS or DHCP services
- Routing Protocols
- ICMP Directed at the router
```
set firewall name WAN_LOCAL description 'PROTECT_ROUTER_FROM_OFFICE'
set firewall name WAN_LOCAL default-action drop
```
The default action means unsolicited office-side connections directed at the router will be discarded unless we create a specific exception.  
#### Permit replies associated with connections initiated by the router:
```
set firewall name WAN_LOCAL rule 10 description 'ALLOW_ESTABLISHED_RELATED'
set firewall name WAN_LOCAL rule 10 action accept
set firewall name WAN_LOCAL rule 10 state established enable
set firewall name WAN_LOCAL rule 10 state related enable
```
**Example**: When the router sends an NTP or DNS request, the corresponding response can return through this rule.  
#### Drop Invalid State:
```
set firewall name WAN_LOCAL rule 20 description 'DROP_INVALID'
set firewall name WAN_LOCAL rule 20 action drop
set firewall name WAN_LOCAL rule 20 state invalid enable
```

### 3. Attach both policies to eth0
This establishes the following packet paths: 
`WAN_IN`: Destination is a network behind the router
`WAN_LOCAL`: Destination is the EdgeRouter itself
```
set interfaces ethernet eth0 firewall in name WAN_IN
set interfaces ethernet eth0 firewall local name WAN_LOCAL
```
`firewall in`: Examines traffic entering eth0 that the router would forward to VLAN 1 or 2.  
`firewall local`: Examines traffic entering eth0 addressed to the router itself.  
#### What they DO NOT DO:
- Filter traffic entering eth8
- Modify the office switch or office router
- Cannot affect the office while eth0 is physically disconnected.
#### Commit the WAN firewall Checkpoint:
`commit`: activates the candidate configuration in the running system. **It does not yet make it persistent across a reboot.**
**Reason**: Currently connected via Console, eth0 is disconnected at this point and polocies apply only to eth0.  
#### Verify:
```
show interfaces ethernet eth0 firewall
```
#### Save: 
`save`: saves configuration for next boot cycle.
### 4. Isolate VLAN 1 from management VLAN 2
We want VLAN 1 clients (172.30.1.0/24) to reach the internet but not the switch-management network (172.30.2.0/24).
#### Create the policy:
```
set firewall name VLAN1_IN description 'BLOCK_CLIENTS_FROM_MANAGEMENT'
set firewall name VLAN1_IN default-action accept
set firewall name VLAN1_IN rule 10 description 'BLOCK_VLAN2_MANAGEMENT'
set firewall name VLAN1_IN rule 10 action drop
set firewall name VLAN1_IN rule 10 destination address 172.30.2.0/24
```
**Why default is** `accept`:
- Only management VLAN 2 must be blocked.
- Other routed traffic, including future internet traffic through eth0, remains permitted.
- Rule 10 takes precedence whenever the destination is `172.30.2.0/24` (Our labs local admin subnet)

### 5. Add NAT for VLAN 1 internet access.  
#### Attach it to inbound traffic on physcial interface eth8:
`set interfaces ethernet eth8 firewall in name VLAN1_IN`
**Why eth8**:
- Untagged VLAN1 traffic enters the router through physical eth8
- Tagged management VLAN2 traffic enters through eth8.2
- Therefore: attaching this policy to eth8 filters VLAN1 clients without applying it to VLAN2 administrators
#### Compare + Commit:
**AFTER TESTING**: plug into an access port and ping vlan1 gateway and vlan 2 management ips
`compare`: shows pending edits waiting for commit
`commit`: commits the pending editions
`save`: saves commits to reboot configuration.

### 6. Protect the router from other VLAN 1 Clients
#### Create seperate VLAN1_LOCAL Policy
It will: 
- Allow DHCP
- Allow clients to ping their proper gateway, 172.30.1.1
- Allow replies to router-local connections
- Block SSH, Web Administration and access to 172.30.2.1 from VLAN 1
#### Create policy:
```
set firewall name VLAN1_LOCAL description 'PROTECT_ROUTER_FROM_VLAN1_CLIENTS'
set firewall name VLAN1_LOCAL default-action drop

set firewall name VLAN1_LOCAL rule 10 description 'ALLOW_ESTABLISHED_RELATED'
set firewall name VLAN1_LOCAL rule 10 action accept
set firewall name VLAN1_LOCAL rule 10 state established enable
set firewall name VLAN1_LOCAL rule 10 state related enable

set firewall name VLAN1_LOCAL rule 20 description 'DROP_INVALID'
set firewall name VLAN1_LOCAL rule 20 action drop
set firewall name VLAN1_LOCAL rule 20 state invalid enable
```
#### Allow DHCP requests from clients:
```
set firewall name VLAN1_LOCAL rule 30 description 'ALLOW_DHCP'
set firewall name VLAN1_LOCAL rule 30 action accept
set firewall name VLAN1_LOCAL rule 30 protocol udp
set firewall name VLAN1_LOCAL rule 30 source port 68
set firewall name VLAN1_LOCAL rule 30 destination port 67
```
#### Allow diagnostic pings only to the correct VLAN 1 gateway:
```
set firewall name VLAN1_LOCAL rule 40 description 'ALLOW_PING_TO_VLAN1_GATEWAY'
set firewall name VLAN1_LOCAL rule 40 action accept
set firewall name VLAN1_LOCAL rule 40 protocol icmp
set firewall name VLAN1_LOCAL rule 40 destination address 172.30.1.1
```

### 7. Attach the local policy to VLAN 1
Enter:
```
set interfaces ethernet eth8 firewall local name VLAN1_LOCAL
```
#### Commit & Test
Check the firewall counters on the router after pings:
```
run show firewall name VLAN1_LOCAL statistics
run show firewall name VLAN1_IN statistics
```

### 8. Configure source NAT for VLAN 1
VLAN 1 uses private addresses (172.30.1.0/24). The office network does not have a route back to those addresses, so the EdgeRouter must translate them when clients access the office network or internet.
#### Create the NAT rule:
```
set service nat rule 5000 description 'VLAN1_TO_OFFICE_WAN'
set service nat rule 5000 type masquerade
set service nat rule 5000 outbound-interface eth0
set service nat rule 5000 source address 172.30.1.0/24
```
The translation will work like this:
`172.30.1.100 > EdgeRouter > 216.252.192.57 > Office gateway`
**Boundaries**:
- Only traffic sourced from `172.30.1.0/24` matches
- Only traffic leaving through `eth0` matches
- VLAN 2 management traffic is not included
- NAT does not expose VLAN 1 to inbound office connections
- NAT does not forward DHCP into the office.
- It has no effect while eth0 is disconnected


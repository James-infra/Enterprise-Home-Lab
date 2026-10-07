# 02 – pfSense Connection Check

## Purpose

Configure FW01 as the firewall and router for the Enterprise Home Lab, establish management access, and validate WAN connectivity and DNS resolution before deploying server and client workloads.

This is a guided home-lab project. It documents lab work, not production administration experience.

## Environment and topology

- Hypervisor: VMware Workstation Pro on Ubuntu.
- Firewall: FW01, pfSense 2.9 (version reported during installation).
- Hostname: `fw01`.
- Configured domain: `home.lab.morpheus`.
- Separate VMware virtual networks are used; VLANs are not configured.

| Role | VMware network | Subnet | pfSense IPv4 configuration |
|---|---|---|---|
| WAN | VMnet8 — NAT | `192.168.171.0/24` | DHCP; exact lease not recorded here |
| LAN — Management | VMnet2 | `10.10.10.0/24` | `10.10.10.1/24` |
| OPT1 — SERVERS | VMnet3 | `10.10.20.0/24` | `10.10.20.1/24` |
| OPT2 — CLIENTS | VMnet4 | `10.10.30.0/24` | `10.10.30.1/24` |

Traffic path: lab VM → pfSense → VMware NAT → upstream network.

The Ubuntu management adapter uses `10.10.10.2/24`. VMware DHCP is disabled on the internal lab networks. Server and client interface addresses are configuration targets from this setup; their dashboard values were not separately captured in this conversation.

## Interface assignment and management access

1. Disconnect the installer ISO and boot FW01 from its virtual disk.
2. Select console option **1 — Assign Interfaces**.
3. Match each pfSense interface to its VMware adapter using MAC addresses, rather than assuming interface names or adapter order.
4. Answer **No** to VLAN configuration.
5. Assign WAN, LAN, OPT1, and OPT2 using the mapping above.
6. Select console option **2 — Set interface(s) IP address** and configure LAN as `10.10.10.1/24`.
7. Leave the LAN upstream gateway unset, disable LAN DHCP, and retain HTTPS management.

Open `https://10.10.10.1` from the Ubuntu host. A certificate warning is expected with the initial self-signed certificate; verify the destination is the lab firewall before proceeding.

The setup wizard includes changing the initial administrator password. Do not publish passwords, recovery secrets, or unredacted configuration backups in the repository.

## Troubleshooting: management IP conflict

### Symptom and evidence

Ubuntu's VMnet2 adapter initially owned `10.10.10.1/24`, the address intended for pfSense LAN. The route lookup returned:

```text
local 10.10.10.1 dev lo src 10.10.10.1
```

Ping succeeded, but it reached Ubuntu itself. This result did not prove pfSense connectivity.

### Smallest corrective change

Retain pfSense at `10.10.10.1` and move the Ubuntu adapter to the unused address `10.10.10.2`:

```bash
sudo ip addr del 10.10.10.1/24 dev vmnet2
sudo ip addr add 10.10.10.2/24 dev vmnet2
```

These commands temporarily interrupt management traffic through VMnet2. They are a runtime fix and may be lost when VMware networking restarts. Persistent host configuration remains a separate verification item.

### Verification

```bash
ip -4 addr show dev vmnet2
ip -4 route get 10.10.10.1
ping -c 4 10.10.10.1
```

Expected route:

```text
10.10.10.1 dev vmnet2 src 10.10.10.2
```

Management access subsequently progressed to the pfSense setup wizard.

**Root cause:** the host adapter and intended firewall LAN configuration used the same IP address. A successful ping to a local address was initially mistaken for a remote connectivity test.

## Setup wizard configuration

| Setting | Configuration |
|---|---|
| Hostname | `fw01` |
| Domain | `home.lab.morpheus` |
| Primary and secondary DNS | Blank; use the default recursive DNS Resolver |
| WAN DNS override | Disabled |
| WAN address type | DHCP |
| WAN MAC override, MTU, MSS | Unset; retain defaults |
| WAN private-network blocking | Disabled because VMnet8 uses private IPv4 space |
| WAN bogon blocking | Enabled |
| LAN address | `10.10.10.1/24` |
| Internal upstream gateways | None |
| Internal DHCP | Disabled during initial setup |
| IPv6 | Not configured for this initial lab stage |

The default recursive DNS Resolver does not require manually configured forwarding DNS servers. The domain field sets the firewall's DNS naming context; it does not join pfSense to Active Directory.

## Confirmed connection checks

After completing the wizard, both requested pfSense tests were reported as passing:

| Check                    | Location and input                             | Result  |
| ------------------------ | ---------------------------------------------- | ------- |
| Public IPv4 reachability | Diagnostics → Ping; host `1.1.1.1`, source WAN | Passed  |
| External DNS resolution  | Diagnostics → DNS Lookup; `www.microsoft.com`  | Passed  |

These checks validate traffic originating from pfSense. They do not prove that a VM on SERVERS or CLIENTS can reach the internet through firewall rules and outbound NAT.

## Initial firewall policy

### Aliases

| Alias | Type | Entries |
|---|---|---|
| `PRIVATE_NETS` | Network(s) | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |
| `WEB_PORTS` | Port(s) | `80`, `443` |

Use the alias's exact spelling. Address aliases are selected through **Address or Alias**; the web port alias belongs in the destination port fields.

### Rule order

Apply this policy independently on SERVERS and CLIENTS. Replace `<interface>` with the relevant interface name.

| Order | Action | Protocol | Source | Destination | Destination port |
|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | `<interface> net` | `<interface> address` | 53 |
| 2 | Pass | ICMP — Echo request | `<interface> net` | `<interface> address` | N/A |
| 3 | Block; log enabled | Any | `<interface> net` | `PRIVATE_NETS` | Any |
| 4 | Pass | TCP | `<interface> net` | Any | `WEB_PORTS` |
| 5 | Pass | ICMP — Echo request | `<interface> net` | Any | N/A |

Use IPv4, source port Any, and the default gateway setting for these rules. Leave destination inversion unchecked. Save and apply changes.

Rules are evaluated on the ingress interface, from top to bottom. DNS and gateway ping exceptions must precede the private-network block. The block must precede internet-access rules.

CLIENTS rule creation was reported complete. The actual ruleset was not supplied for visual review; SERVERS rule completion and end-to-end tests were not confirmed in this conversation.

### Policy boundaries

- Preserve LAN management access and the anti-lockout rule during initial setup.
- Do not add inbound WAN access for these tests.
- Unmatched traffic on SERVERS and CLIENTS is blocked by default.
- The initial policy blocks new connections between private subnets, including access to management from SERVERS and CLIENTS.
- Same-subnet traffic does not pass through pfSense and needs endpoint controls where isolation is required.
- This is an initial lab policy, not a complete production egress policy. Additional services, such as NTP and AD, require explicit rules.
- Domain clients must eventually use AD DNS. Add required client-to-domain-controller exceptions above the private-network block before attempting domain join.

## End-to-end validation procedure

The following procedure is a test plan, not a claim that these tests passed during this phase.

Connect a Windows test VM to VMnet4 and assign an unused address:

| Setting | Initial test value |
|---|---|
| IP address | `10.10.30.10` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `10.10.30.1` |
| DNS | `10.10.30.1` |

Use these temporary DNS settings only before AD integration. For an existing domain client, retain its AD DNS configuration and current working IP settings.

```powershell
ipconfig /all
ping 10.10.30.1
ping 1.1.1.1
nslookup www.microsoft.com
Test-NetConnection www.microsoft.com -Port 443
Test-NetConnection 10.10.10.1 -Port 443
```

| Test | Expected result |
|---|---|
| Ping client gateway | Replies |
| Ping public IP | Replies, if the remote endpoint permits ICMP |
| DNS lookup | External address records returned |
| Public HTTPS connection | `TcpTestSucceeded: True` |
| Management HTTPS connection from CLIENTS | `TcpTestSucceeded: False` and a matching firewall block log |

A failed connection alone does not prove firewall isolation; correlate it with the rule's log entry under **Status → System Logs → Firewall**.

If gateway access fails, check the VMnet4 attachment, client IP settings, interface status, and rule source. If public connectivity fails, check the private-network block destination and outbound NAT coverage for `10.10.30.0/24`. If only DNS fails, check rule 1 and DNS Resolver listening settings.

## Completion record

- [x] pfSense installation completed.
- [x] Four VMware adapter mappings visible.
- [x] Management wizard accessed after identifying the host IP conflict.
- [x] pfSense WAN ping passed.
- [x] pfSense external DNS lookup passed.
- [x] CLIENTS firewall rule creation  completed.
- [x] Capture and review final interface addresses and firewall rule order.
- [ ] Record end-to-end SERVERS and CLIENTS test evidence.
- [ ] Verify the Ubuntu management IP survives a networking restart.
- [ ] Export and securely store a pfSense configuration backup.

Later AD, DHCP, and GPO work belongs in subsequent project chapters; it is not evidence that this initial ruleset remained unchanged.

## References

- [Netgate — Assign Interfaces](https://docs.netgate.com/pfsense/en/latest/install/assign-interfaces.html)
- [Netgate — Setup Wizard](https://docs.netgate.com/pfsense/en/latest/config/setup-wizard.html)
- [Netgate — Configuring Firewall Rules](https://docs.netgate.com/pfsense/en/latest/firewall/configure.html)
- [Netgate — Firewall Rule Basics](https://docs.netgate.com/pfsense/en/latest/firewall/firewall-rule-basics.html)


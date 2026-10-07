# 03 - Windows Server 2025 Setup

## Overview

This guided home-lab project deployed Windows Server 2025 as **DC01**, created an Active Directory forest, configured DNS, and joined a Windows client across separate pfSense network segments. It demonstrates lab administration and troubleshooting, not production deployment experience.

**Status:** AD DS, DNS, domain membership, secure-channel validation, standard-user sign-in, and default computer Group Policy processing are verified. The custom inactivity-lock GPO remains pending verification in this document.

## Environment

| Component | Configuration |
|---|---|
| Hypervisor | VMware Workstation on Ubuntu |
| Firewall/router | pfSense FW01 |
| Domain controller | DC01 — Windows Server 2025 |
| AD forest and domain | `home.lab.morpheus` |
| NetBIOS domain name | `MORPHEUS` |
| Server segment | VMnet3 — `10.10.20.0/24` |
| Client segment | VMnet4 — `10.10.30.0/24` |
| Client | CL01 — Windows workstation |

The private domain name is used for this lab. Future cloud identity integration will require a suitable verified domain for cloud sign-in. Exact Windows Server edition and final VM CPU, memory, and disk allocations were not captured in the supplied validation output.

## 1. Install and name the server

Windows Server 2025 was installed in a VMware VM with one network adapter connected to the server segment. The installed server was renamed to DC01.

Run in elevated PowerShell when reproducing the initial configuration:

```powershell
Rename-Computer -NewName "DC01" -Restart
```

The rename restarts the VM. Complete updates and retain a suitable backup or pre-promotion recovery point before promoting a new server. Updates, VMware Tools installation, and the suggested pre-promotion snapshot were recommended during the guided setup but were not independently confirmed.

## 2. Configure static networking

| Setting | DC01 |
|---|---|
| IPv4 address | `10.10.20.10` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `10.10.20.1` |
| Initial DNS before promotion | `10.10.20.1` |
| Final DNS after promotion | `10.10.20.10` |

A static address gives clients a predictable DNS and domain-controller endpoint. The pfSense SERVERS interface provides routing between segments and outbound connectivity.

```powershell
hostname
ipconfig /all
ping 10.10.20.1
nslookup microsoft.com 10.10.20.1
Test-NetConnection www.microsoft.com -Port 443
```

**Observed:** hostname DC01, static IPv4 `10.10.20.10`, gateway replies with no packet loss, successful public DNS resolution, and `TcpTestSucceeded : True` for HTTPS.

The `nslookup` text “Non-authoritative answer” was displayed as a PowerShell error-stream message, but the query returned addresses successfully. It was not a resolution failure.

## 3. Deploy AD DS and DNS

The guided deployment used the following commands in elevated PowerShell:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSForest `
    -DomainName "home.lab.morpheus" `
    -DomainNetbiosName "MORPHEUS" `
    -InstallDns
```

Forest creation prompts for a Directory Services Restore Mode password and restarts the server. Keep that recovery password private and separate from documentation. The first domain controller supplies directory services and AD-integrated DNS.

Validation:

```powershell
Get-ADDomain | Select-Object DNSRoot,NetBIOSName
dcdiag /test:Advertising /test:SysVolCheck
```

**Observed:**

```text
DNSRoot           NetBIOSName
home.lab.morpheus MORPHEUS

DC01 passed test Connectivity
DC01 passed test Advertising
DC01 passed test SysVolCheck
```

## 4. Configure DNS forwarding

DC01 uses its own DNS service for AD resolution and forwards external queries to pfSense.

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
    -ServerAddresses "10.10.20.10"

Set-DnsServerForwarder -IPAddress "10.10.20.1" -PassThru
```

`Set-DnsServerForwarder` replaces the existing server-level forwarder list. Avoid configuring pfSense to forward all external queries back to DC01, which would create a forwarding loop.

```powershell
Resolve-DnsName DC01.home.lab.morpheus -Type A -Server 10.10.20.10
Resolve-DnsName _ldap._tcp.dc._msdcs.home.lab.morpheus `
    -Type SRV -Server 10.10.20.10
Resolve-DnsName www.microsoft.com -Type A -Server 10.10.20.10
```

**Observed:** DC01 resolves to `10.10.20.10`; the LDAP SRV record targets `dc01.home.lab.morpheus` on port 389; external names return public DNS records.

Later project validation also confirmed the AD-integrated server reverse zone `20.10.10.in-addr.arpa` uses secure dynamic updates and the PTR for `10.10.20.10` returns `DC01.home.lab.morpheus`.

## 5. Connect the client across pfSense

The client was configured with:

| Setting | CL01 during initial validation |
|---|---|
| IPv4 address | `10.10.30.10` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `10.10.30.1` |
| DNS server | `10.10.20.10` |

These are the static settings observed during this setup stage; later DHCP work is outside this chapter. Domain clients use AD DNS to locate domain services.

### Firewall exceptions

On the pfSense **CLIENTS** interface, allow traffic from CLIENTS subnets to **DC01 only**, before the PRIVATE_NETS block:

| Rule | Protocol | Destination ports |
|---|---|---|
| DNS to DC01 | TCP/UDP | `53` |
| AD TCP to DC01 | TCP | `AD_TCP_PORTS` |
| AD UDP to DC01 | UDP | `AD_UDP_PORTS` |

Port aliases:

```text
AD_TCP_PORTS: 88, 135, 389, 445, 464, 3268, 49152:65535
AD_UDP_PORTS: 88, 123, 389, 464
```

Each alias value is a separate entry; `49152:65535` is one range entry. Source ports remain Any. The high TCP range supports dynamic RPC. These rules are scoped to this lab's domain controller and do not grant general access to the server network.

## 6. Join CL01 to the domain

The client was renamed to CL01 and joined using prompted credentials:

```powershell
Rename-Computer -NewName "CL01" -Restart

# After restarting, run in elevated PowerShell:
Add-Computer -DomainName "home.lab.morpheus" `
    -Credential (Get-Credential "MORPHEUS\Administrator") `
    -Restart
```

Both operations restart the client. Using the domain Administrator for this guided join is a lab convenience; a production process should use appropriately delegated joining permissions.

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name,Domain,PartOfDomain

Test-ComputerSecureChannel -Verbose
```

**Observed:**

```text
Name Domain            PartOfDomain
CL01 home.lab.morpheus True

True
The secure channel ... is in good condition.
```

## 7. Troubleshooting evidence

### DNS timeout presented as “no internet”

**Symptom:** CL01 could not resolve public names through DC01.

**Evidence:** the gateway responded; TCP 443 to `1.1.1.1` succeeded; a DNS query to `10.10.20.10` timed out. The first pfSense DNS rule targeted CLIENTS address (`10.10.30.1`), leaving queries to DC01 subject to PRIVATE_NETS blocking.

**Smallest change:** changed the DNS rule destination to `10.10.20.10`, keeping TCP/UDP 53 and placement above the block.

**Verification:** CL01 resolved both DC01's A record and Microsoft's public A record. The user confirmed the DNS issue was resolved.

### Computer Group Policy failed with event 1055

**Symptom:** `gpupdate /force` failed for computer policy; initial `gpresult` contained stale local policy data.

**Evidence gathered:**

- CL01 and DC01 IPv4 name resolution succeeded.
- `nltest /dsgetdc:home.lab.morpheus` found DC01 successfully.
- TCP 88, 135, 389, and 445 passed after AD exceptions were added.
- CL01's secure channel remained healthy.
- DC01 had SYSVOL and NETLOGON shares; `dcdiag` NetLogons and SysVolCheck passed.
- Event 1055 XML reported **1722: The RPC server is unavailable**.
- The user added `49152:65535` as one TCP alias entry, then reported successful policy processing.

**Supported conclusion:** dynamic RPC access was the likely cause of the policy failure. The successful policy result after the range update supports this conclusion; a packet capture identifying the exact blocked RPC endpoint was not collected.

A blank `DNSHostName` attribute was also observed on CL01's AD object. A targeted correction was suggested, but its execution and final attribute value were not confirmed. Do not present that correction as a verified cause or completed fix.

Manual SYSVOL access from the local user session returned a path-not-found error. That result alone did not establish a missing share: server-side share checks passed, and subsequent computer policy processing succeeded.

## 8. Verify Group Policy and domain sign-in

```powershell
gpupdate /target:computer /force
gpresult /scope computer /r
```

**Observed:**

```text
Computer Policy update has completed successfully.
Group Policy was applied from: DC01.home.lab.morpheus
Domain Name: MORPHEUS
Applied Group Policy Objects:
    Default Domain Policy
```

A standard lab account was created in the **Lab Users** OU. On CL01:

```powershell
whoami
gpresult /scope user /r
```

**Observed:** `morpheus\lab.user01`, user distinguished name under `OU=Lab Users`, Domain Users membership, and policy processing from DC01. No user GPO was listed as applied in that run. This does not itself indicate a processing failure.

## Verified outcomes and limitations

| Item | Evidence status |
|---|---|
| Windows Server 2025 installed and named DC01 | Confirmed |
| Static server networking and outbound HTTPS | Confirmed |
| AD forest `home.lab.morpheus` / `MORPHEUS` | Confirmed |
| Internal, external, and server reverse DNS | Confirmed |
| CL01 domain membership and secure channel | Confirmed |
| Default computer GPO processing | Confirmed |
| Standard domain-user sign-in | Confirmed |
| Lab Users OU | Confirmed by user policy output |
| CL01 moved to Workstations OU | Instructed; final location not supplied |
| Custom 600-second inactivity-lock GPO | Instructed; application and behavior not supplied |
| GPO backup, AD recovery test, second DC | Not demonstrated in this chapter |

This single-DC environment is a learning lab, not a production high-availability design. Snapshot recommendations do not replace tested AD-aware backups.

## Next validation

1. Confirm CL01's Workstations OU placement and final AD DNSHostName attribute.
2. Validate the custom inactivity-lock GPO through `gpresult`, its effective setting, and an idle-session test.
3. Back up the GPO and retain validation evidence.
4. Continue DHCP and further GPO work in their own chapter.

## References

- [Windows Server 2025 Evaluation Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025)
- [Install-ADDSForest](https://learn.microsoft.com/en-us/powershell/module/addsdeployment/install-addsforest?view=windowsserver2025-ps)
- [Set-DnsServerForwarder](https://learn.microsoft.com/en-us/powershell/module/dnsserver/set-dnsserverforwarder)
- [Configure firewalls for AD domains and trusts](https://learn.microsoft.com/en-us/troubleshoot/windows-server/identity/config-firewall-for-ad-domains-and-trusts)
- [Set-ADComputer](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-adcomputer)

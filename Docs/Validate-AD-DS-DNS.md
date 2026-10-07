# 04 – Validate AD DS & DNS

## Overview

Validated the existing Windows Server 2025 domain controller and DNS configuration in my Enterprise Home Lab. Checked domain identity, essential services, domain controller diagnostics, DNS zones, external name resolution, and reverse resolution. Identified a missing reverse lookup zone and verified the fix.

**Experience type:** Guided home-lab work, not a production deployment.

**Status:** The AD DS and DNS checks documented below passed. This stage does not establish full production readiness or redundant infrastructure.

## Lab configuration

| Component | Verified value |
|---|---|
| Domain controller | DC01.home.lab.morpheus |
| Operating system | Windows Server 2025 |
| IPv4 address | 10.10.20.10 |
| Servers subnet | 10.10.20.0/24 |
| AD DNS domain | home.lab.morpheus |
| NetBIOS domain | MORPHEUS |
| Domain functional level | Windows2025Domain |
| AD site | Default-First-Site-Name |
| Global Catalog | Enabled |
| Ethernet0 DNS server | 10.10.20.10 |
| DNS forwarder | 10.10.20.1 (pfSense) |
| Root-hint fallback setting | UseRootHint = True |

## 1. Validate domain identity and services

Ran the following commands in elevated PowerShell on DC01:

```powershell
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DomainMode

Get-ADDomainController |
    Format-List HostName, IPv4Address, Site, IsGlobalCatalog

Get-Service NTDS, DNS, Netlogon, KDC |
    Format-Table Name, Status -AutoSize
```

Confirmed DC01's hostname, address, site, and Global Catalog status. All four services were running:

| Service | Purpose | Observed status |
|---|---|---|
| NTDS | Active Directory Domain Services | Running |
| DNS | DNS Server | Running |
| Netlogon | Domain logon support and DC record registration | Running |
| KDC | Kerberos authentication | Running |

## 2. Run domain controller diagnostics

```powershell
dcdiag /test:Advertising /test:SysVolCheck /test:NetLogons
dcdiag /test:DNS /DnsBasic
dcdiag /test:DNS /DnsRecordRegistration
```

| Check | Observed result |
|---|---|
| Connectivity | Passed |
| Advertising | Passed |
| SysVolCheck | Passed |
| NetLogons | Passed |
| DNS basic test | Passed for DC01 and the domain |
| DNS record registration test | Passed for DC01 and the domain |

These results established a baseline for DC discovery, SYSVOL readiness, Netlogon shares, and essential DNS registration. They do not replace client-side testing or a full operational review.

## 3. Inspect DNS configuration

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 |
    Format-Table InterfaceAlias, ServerAddresses -AutoSize

Get-DnsServerZone |
    Format-Table ZoneName, ZoneType, IsDsIntegrated, DynamicUpdate -AutoSize

Get-DnsServerForwarder |
    Format-List IPAddress, UseRootHint
```

Confirmed Ethernet0 used `10.10.20.10` for DNS. The following AD zones were primary, AD-integrated, and configured for secure dynamic updates:

- `home.lab.morpheus`
- `_msdcs.home.lab.morpheus`

AD-integrated zones store DNS data in Active Directory; secure dynamic updates restrict dynamic changes to authenticated, authorized identities.

The configured forwarder was `10.10.20.1`, with root-hint fallback enabled. The external lookup below confirmed resolution through DC01, but did not independently prove which upstream path answered the query.

## 4. Verify DNS resolution

```powershell
# Domain controller address
Resolve-DnsName DC01.home.lab.morpheus -Server 10.10.20.10 -Type A

# LDAP domain controller discovery
Resolve-DnsName _ldap._tcp.dc._msdcs.home.lab.morpheus `
    -Server 10.10.20.10 -Type SRV

# External name resolution
Resolve-DnsName www.microsoft.com -Server 10.10.20.10 -Type A

# Reverse lookup
Resolve-DnsName 10.10.20.10 -Server 10.10.20.10 -Type PTR
```

Evidence from this stage and the preceding server setup:

| Query | Observed result |
|---|---|
| DC01 A record | 10.10.20.10 |
| LDAP SRV record | dc01.home.lab.morpheus, port 389 |
| www.microsoft.com | CNAME chain followed by an IPv4 answer |
| DC01 PTR lookup before correction | DNS_ERROR_RCODE_NAME_ERROR |

The external IPv4 answer was `23.221.242.102` at the time of testing. CDN addresses and TTLs can change; this address is not a fixed acceptance criterion.

## 5. Troubleshoot the reverse lookup failure

### Symptom

Forward resolution worked, but the PTR query returned:

```text
Resolve-DnsName : 10.20.10.10.in-addr.arpa : DNS name does not exist
FullyQualifiedErrorId : DNS_ERROR_RCODE_NAME_ERROR
```

### Evidence and root cause

The zone inventory contained the AD forward zones and default reverse zones, but no reverse lookup zone for `10.10.20.0/24`. DC01 therefore had no local reverse zone containing its PTR mapping.

For this /24 subnet, the reverse zone is `20.10.10.in-addr.arpa`. DC01's final address octet, `10`, becomes the PTR record name. The full reverse query is `10.20.10.10.in-addr.arpa`.

### Smallest appropriate change

Added an AD-integrated reverse zone with domain replication scope and secure dynamic updates, then added DC01's PTR record:

```powershell
# Creation commands: run only when the zone and record do not exist.
Add-DnsServerPrimaryZone -NetworkId "10.10.20.0/24" `
    -ReplicationScope Domain -DynamicUpdate Secure

Add-DnsServerResourceRecordPtr -Name "10" `
    -ZoneName "20.10.10.in-addr.arpa" `
    -PtrDomainName "dc01.home.lab.morpheus."
```

This added reverse lookup support without rebuilding AD DS or changing the working forward zones. Reverse DNS supports diagnostics and IP-to-name lookups; its absence alone did not demonstrate an AD authentication failure.

### Verify the result

```powershell
Get-DnsServerZone -Name "20.10.10.in-addr.arpa" |
    Format-Table ZoneName, IsDsIntegrated, DynamicUpdate

Resolve-DnsName 10.10.20.10 -Server 10.10.20.10 -Type PTR
```

Actual verification output:

```text
ZoneName              IsDsIntegrated DynamicUpdate
--------              -------------- -------------
20.10.10.in-addr.arpa           True Secure

Name                     Type TTL  Section NameHost
----                     ---- ---  ------- --------
10.20.10.10.in-addr.arpa PTR  1200 Answer  DC01.home.lab.morpheus
```

**Result:** Reverse resolution succeeded and the new zone used secure dynamic updates.

## Skills demonstrated

- Validating AD domain identity and domain controller services.
- Using DCDiag to check AD DS and DNS health.
- Inspecting AD-integrated zones and dynamic update settings.
- Testing A, SRV, external, and PTR resolution.
- Diagnosing a missing reverse lookup zone from command output.
- Applying and verifying a focused DNS configuration change with PowerShell.

## Limitations and next steps

- This validation focused on one domain controller; multi-DC replication and failover were not tested.
- External DNS resolution succeeded, but forwarder-only behavior and root-hint fallback were not separately tested.
- Backup recovery, DNS scavenging, and broader security hardening were not validated in this stage.
- Continue DHCP and Group Policy validation using the existing client. Client-side DNS and domain discovery should be checked from the CLIENTS subnet.

## Microsoft references

- [DCDiag](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/dcdiag)
- [Verify DNS functionality to support directory replication](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/troubleshoot/verify-dns-functionality-to-support-directory-replication)
- [Add-DnsServerPrimaryZone](https://learn.microsoft.com/en-us/powershell/module/dnsserver/add-dnsserverprimaryzone?view=windowsserver2025-ps)
- [Add-DnsServerResourceRecordPtr](https://learn.microsoft.com/en-us/powershell/module/dnsserver/add-dnsserverresourcerecordptr?view=windowsserver2025-ps)

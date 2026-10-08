# 05 - DHCP & Group Policy

## Overview

This guided home-lab project implements Windows DHCP across segmented VMware networks using pfSense relay, then validates computer and user Group Policy on a domain-joined client. This is lab experience, separate from production administration.

**Status:** DHCP lease assignment, both GPO behaviours, and a disabled-link recovery exercise verified on October 7–8, 2026.

## Environment

| Component | Configuration |
|---|---|
| Domain | `home.lab.morpheus` |
| NetBIOS domain | `MORPHEUS` |
| DC01 | Windows Server 2025; AD DS, DNS and DHCP; static `10.10.20.10/24` |
| Server gateway | pfSense `10.10.20.1` |
| Server network | VMnet3 — `10.10.20.0/24` |
| Client network | VMnet4 — `10.10.30.0/24` |
| Client gateway | pfSense `10.10.30.1` |
| CL01 | Domain-joined Windows client; DHCP lease `10.10.30.100` |
| Computer OU | `OU=Workstations,DC=home,DC=lab,DC=morpheus` |
| Test user | `MORPHEUS\lab.user01` in `OU=Lab Users` |

## DHCP installation and authorization

Run on DC01 in elevated Windows PowerShell with the required domain permissions:

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
Get-WindowsFeature -Name DHCP
Get-Service -Name DHCPServer

Add-DhcpServerInDC -DnsName "DC01.home.lab.morpheus" -IPAddress 10.10.20.10
Get-DhcpServerInDC
```

Verified installation returned `Success=True`, `Restart Needed=No`, the feature state was `Installed`, and the service was `Running`. The authorized server list contained `dc01.home.lab.morpheus` at `10.10.20.10`.

Initial checks on DC01 returned command-not-found errors and no `DHCPServer` service. Installing the DHCP role and management tools resolved these symptoms. The earlier reported installation had not been verified on DC01.

## Client scope and options

The scope was created inactive while its options and relay were configured:

```powershell
Add-DhcpServerv4Scope `
    -Name "CLIENTS" `
    -StartRange 10.10.30.100 `
    -EndRange 10.10.30.199 `
    -SubnetMask 255.255.255.0 `
    -State InActive

Set-DhcpServerv4OptionValue `
    -ScopeId 10.10.30.0 `
    -Router 10.10.30.1 `
    -DnsServer 10.10.20.10 `
    -DnsDomain "home.lab.morpheus"
```

| Setting | Verified value |
|---|---|
| Scope | `CLIENTS`, `10.10.30.0/24` |
| Pool | `10.10.30.100–10.10.30.199` |
| Lease duration | 8 days |
| Option 003 — Router | `10.10.30.1` |
| Option 006 — DNS Servers | `10.10.20.10` |
| Option 015 — DNS Domain Name | `home.lab.morpheus` |

The pool leaves lower addresses outside dynamic allocation. Domain clients use DC01 for DNS to locate AD services; pfSense provides their default route.

## pfSense DHCP relay

DC01 and CL01 occupy different subnets. The relay forwards client DHCP broadcasts to DC01.

Under **Services → DHCP Relay**, the configured values were:

| Field | Value |
|---|---|
| Enable DHCP Relay | Checked |
| Downstream Interfaces | CLIENTS only |
| Upstream Servers | `10.10.20.10` |
| CARP Status VIP | None |
| Append circuit ID and agent ID | Unchecked |

pfSense DHCP server must be disabled on all interfaces before enabling its relay. VMware DHCP must remain disabled on VMnet4 to avoid competing leases. VMware VMnet8 NAT DHCP and the pfSense WAN DHCP client are separate services and remain part of the WAN design.

After saving relay configuration, the scope was activated:

```powershell
Set-DhcpServerv4Scope -ScopeId 10.10.30.0 -State Active
Get-DhcpServerv4Scope
Get-DhcpServerv4OptionValue -ScopeId 10.10.30.0
```

## DHCP and domain validation

CL01 was configured to obtain IPv4 and DNS settings automatically. DC01 retained its static address. Client changes were made through the VMware console because changing addressing can interrupt connectivity.

On CL01:

```powershell
ipconfig /renew
ipconfig /all
Resolve-DnsName DC01.home.lab.morpheus
nltest /dsgetdc:home.lab.morpheus
```

Observed client results:

| Check | Result |
|---|---|
| DHCP enabled | Yes |
| IPv4 address | `10.10.30.100` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `10.10.30.1` |
| DHCP server | `10.10.20.10` |
| DNS server | `10.10.20.10` |
| Connection DNS suffix | `home.lab.morpheus` |
| DC01 DNS lookup | A record returned `10.10.20.10` |
| Domain-controller discovery | Successful; DC01 returned |

Server-side verification on DC01:

```powershell
Get-DhcpServerv4Lease -ScopeId 10.10.30.0 |
    Format-Table IPAddress, HostName, ClientId, AddressState, LeaseExpiryTime
```

The server listed `CL01.home.lab.morpheus` at `10.10.30.100` with `AddressState=Active`. Its client ID matched CL01's network adapter MAC address. The displayed expiry was truncated, so an exact expiry timestamp is not recorded here.

## Computer GPO: inactivity lock

**GPO name:** `Workstation - Inactivity Lock`  
**Link target:** Workstations OU  
**Setting:** Interactive logon: Machine inactivity limit = `600` seconds

The GPO was created and linked using Group Policy Management on DC01. Its setting path is:

**Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options**

Validation commands:

```powershell
# DC01: confirm the computer's OU
Get-ADComputer CL01 | Select-Object Name, DistinguishedName

# CL01: run policy checks in an administrator terminal
gpupdate /target:computer /force
gpresult /scope computer /r

Get-ItemPropertyValue `
    -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
    -Name InactivityTimeoutSecs
```

Verified results:

- CL01 was in the Workstations OU.
- `Workstation - Inactivity Lock` and `Default Domain Policy` appeared in applied computer policies.
- The effective registry value was `600`.
- The user confirmed the session locked during the inactivity test.

## User GPO: remove Run

Initial user `gpresult` showed no applied user GPOs. DC01 had only the default GPOs, and Lab Users had no directly linked GPO. Policy processing itself succeeded; `N/A` was not evidence of a connectivity failure.

The following commands created and linked the test policy on DC01:

```powershell
New-GPO -Name "LAB - Users - Remove Run"

Set-GPRegistryValue `
    -Name "LAB - Users - Remove Run" `
    -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
    -ValueName "NoRun" -Type DWord -Value 1

New-GPLink `
    -Name "LAB - Users - Remove Run" `
    -Target "OU=Lab Users,DC=home,DC=lab,DC=morpheus" `
    -LinkEnabled Yes
```

On CL01, as `MORPHEUS\lab.user01`:

```powershell
gpupdate /target:user /force
gpresult /scope user /r
```

The provided screenshots verified:

- `LAB - Users - Remove Run` appeared in applied user policies from DC01.
- Attempting to open Run displayed a restrictions message.
- Run remained visible in search, but launching it was blocked.

This policy demonstrates user-scoped enforcement. Blocking Run is not a complete application-control or security boundary.

## Troubleshooting exercise: disabled GPO link

| Stage | Action and evidence |
|---|---|
| Symptom | Inactivity-lock GPO absent from applied computer policies |
| Controlled cause | Disabled only its link on the Workstations OU |
| Evidence | After `gpupdate`, `gpresult` listed only Default Domain Policy |
| Smallest correction | Re-enabled the same link without recreating the GPO |
| Verification | After refresh, `Workstation - Inactivity Lock` returned |
| Final state | Link enabled; policy restored |

The GPO continued to exist while its link was disabled. Successful policy processing did not mean that this particular GPO applied. Security settings can persist after a GPO stops applying; the exercise verified application with `gpresult`, not disappearance of the registry value.

## Final server-side checks

```powershell
Get-GPInheritance -Target "OU=Workstations,DC=home,DC=lab,DC=morpheus" |
    Select-Object -ExpandProperty GpoLinks |
    Format-Table DisplayName, Enabled

Get-GPInheritance -Target "OU=Lab Users,DC=home,DC=lab,DC=morpheus" |
    Select-Object -ExpandProperty GpoLinks |
    Format-Table DisplayName, Enabled
```

Both links returned `Enabled=True`:

- `Workstation - Inactivity Lock`
- `LAB - Users - Remove Run`

## Scope and limitations

- Evidence covers one Windows client and one test user in a guided lab.
- DHCP shares DC01 with AD DS and DNS to fit the small lab; no DHCP failover or additional domain controller was tested.
- The lock setting is a single control, not a complete workstation security baseline.
- The disabled-link exercise was intentional and was restored.
- No backup/restore validation, scale testing, or production deployment is claimed.
- Creation commands are a record of the completed setup, not an idempotent script: do not rerun them against existing scopes or GPOs without checking first.

## Skills demonstrated

Windows DHCP installation and AD authorization; scope and option configuration; DHCP relay across subnets; client/server lease validation; OU-based computer and user policy targeting; effective-setting verification; and evidence-led troubleshooting with the smallest corrective change.

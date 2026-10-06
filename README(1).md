# Enterprise Home Lab

An enterprise-style home lab for developing practical skills in networking, Windows Server, identity, endpoint administration, and structured troubleshooting.

**Status:** In progress  
**Experience category:** Personal home lab and guided learning  
**Documentation baseline:** October 6, 2026

This repository documents work I personally perform in a lab. It does not represent a production deployment or professional administration experience with every technology listed. Planned components are identified separately from reported progress. Guidance and documentation assistance do not imply independently designed implementations.

## Objectives

- Build a segmented virtual environment for administration and troubleshooting exercises.
- Understand dependencies between routing, firewall rules, DNS, DHCP, and Active Directory.
- Practice Windows Server and endpoint administration with repeatable verification.
- Record symptoms, evidence, corrective actions, and lessons learned.
- Extend the environment to Microsoft cloud services after the local foundation is validated.

## Project documentation

Start with [00 – Design Network Architecture](docs/00-design-network-architecture.md) for the topology, addressing plan, design decisions, and validation requirements.

| Workstream | Scope | Current documentation status |
|---|---|---|
| VMware networking | Virtual networks and VM adapter mappings | In progress; VMnet3/4 selection issue reported |
| pfSense | Routing, network interfaces, DNS settings, firewall policy | Installation reported; firewall configuration in progress |
| Windows Server 2025 | AD DS, DNS, DHCP, and domain client integration | Planned for this project; deployment not established by current evidence |
| Group Policy | Windows configuration and policy validation | Planned |
| PowerShell | Lab inventory, diagnostics, and repeatable administration | Planned; no production scripting claim |
| Entra ID and Intune | Identity and Windows endpoint management exercises | Future phase |
| Azure and Microsoft 365 | Cloud administration and integration exercises | Future phase; separate Azure exercises do not establish integration here |

## Current progress and evidence boundary

The following details were reported during the current build:

- VMware configuration identifies VMnet3 as `10.10.20.0/24` and VMnet4 as `10.10.30.0/24`, with VMware DHCP and host virtual adapters disabled on those networks.
- The Ubuntu host has a VMnet2 adapter at `10.10.10.1/24`.
- VMware NAT and DHCP services were reported running on VMnet8.
- pfSense installation was reported as version 2.9; the precise edition and build remain to be recorded.
- Two pfSense-related connectivity tests were reported passing. Their source, destination, and outputs must be recorded before documenting a specific end-to-end capability.
- Firewall rule configuration is underway; completion and rule behaviour are not yet verified in this repository.

These are reported observations, not a claim that the entire architecture is operational. In particular, a ping from the Ubuntu host to its own `10.10.10.1` address does not prove connectivity to pfSense.

## Documentation structure

The initial deliverable contains this README and the architecture document. Add the following folders as evidence is produced:

| Path | Purpose |
|---|---|
| `docs/00-design-network-architecture.md` | Architecture, addressing, dependencies, and validation plan |
| `docs/implementation/` | Configuration steps and reasons for each choice |
| `docs/troubleshooting/` | Incident-style troubleshooting records |
| `evidence/` | Sanitized screenshots and command output |
| `scripts/` | Personally tested lab scripts and usage instructions |

## Verification standard

Mark a milestone **completed and verified** only when its evidence includes:

1. What was configured, including the relevant VM, interface, or service.
2. The test performed and expected result.
3. The actual result, date, and supporting screenshot or command output.
4. Any limitations, exceptions, or unresolved issues.

Use **planned**, **in progress**, and **completed and verified** consistently. For guided exercises, identify the guidance used and explain any changes I personally made.

## Troubleshooting approach

For each issue, document the symptom, likely causes, evidence collected, safe tests, smallest appropriate change, verification, and root cause. If the cause remains uncertain, record that uncertainty rather than assigning an unsupported explanation.

Current candidates include the VMnet3/4 adapter-selection issue and the distinction between host self-connectivity and actual gateway connectivity. Record a resolution only after it has been tested.

## Security and recovery

- Keep administrative access limited to approved lab management paths.
- Use separate administrative and standard test accounts where practical.
- Publish sanitized evidence; exclude passwords, tokens, private keys, sensitive exports, and employer information.
- Before disruptive changes, identify dependencies and preserve an appropriate backup or recovery point.
- Treat VM snapshots as short-term recovery aids, not as a replacement for tested backups.
- Distinguish implemented controls from intended controls.

## Lab limitations

This environment may consolidate roles and run on a single physical host. It does not demonstrate production availability, redundancy, disaster recovery, or operational scale. Cloud integration, domain services, and endpoint management remain planned unless their implementation and validation are documented.

## Next milestone

Confirm virtual network availability and pfSense adapter mappings. Record each interface address, verify that `10.10.10.1` is not assigned to both the Ubuntu host and pfSense, and capture named connectivity tests before deploying dependent services.


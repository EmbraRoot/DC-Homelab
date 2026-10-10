# Active Directory Homelab

A self-built Active Directory environment used to practice domain administration, misconfiguration troubleshooting, and (in later stages) domain-focused attacks like Kerberoasting and password spraying.

## Overview

This project builds a small corporate-style Windows domain from scratch: a domain controller, organizational units, user accounts, and a domain-joined client, all running in an isolated VirtualBox lab network. Built as a learning project while pursuing entry-level help desk and cybersecurity roles — domain administration and troubleshooting are core day-to-day skills in both.

**Tools used:** VirtualBox, Windows Server (Evaluation, Desktop Experience), Windows 11 Pro, Active Directory Domain Services (AD DS), DNS Server role

## Architecture

| VM | Role | IP (host-only) | Specs |
|---|---|---|---|
| AD-DC01 | Domain controller (AD DS + DNS) | 192.168.56.106 | 2 vCPU, 4 GB RAM |
| Windows 11 client | Domain-joined workstation | 192.168.56.100 | 2 vCPU, 4 GB RAM |
| Kali Linux | Attacker (used in later stages) | 192.168.56.104 | 2 vCPU, 2-3 GB RAM |

All VMs run on an isolated VirtualBox host-only network. Given 16 GB of host RAM, VMs are run selectively rather than all simultaneously, except the DC and client, which must run together for domain features to work.

**Domain:** `lab.local`
**Structure:** `Employees` OU with several domain user accounts and a `Help Desk` security group.

## Build Process

1. Installed Windows Server (Desktop Experience) as a standalone VM, then added the **Active Directory Domain Services** role.
2. Promoted the server to a new forest root domain controller (`lab.local`).
3. Created an `Employees` OU, several domain user accounts, and a `Help Desk` group to populate the directory with realistic objects.
4. Joined a Windows 11 client to the domain and confirmed login with a domain account (`LAB\<user>`).

## Troubleshooting Notes

Real-world-style misconfigurations came up during the build, each is documented here because diagnosing them was as valuable as the build itself.

**Domain controller promotion stalled repeatedly.** The AD DS role install and the "waiting for DNS flush" step during promotion hung for over an hour on two separate attempts. Root causes: multiple VMs competing for the host's 16 GB RAM, and the DC's network adapter was on DHCP rather than a static IP. Fix: run only the DC VM during promotion, and set a static IP with DNS pointed at `127.0.0.1` before promoting.

**Client couldn't resolve the domain (`nslookup` returned non-existent domain).** The DC's DNS server and zone were healthy (confirmed by running `nslookup` locally on the DC), so the issue was isolated to the client. Diagnosis: the client had two network adapters (NAT and host-only), and Windows was querying the NAT adapter's DNS server (the home router) instead of the DC, even though the host-only adapter's DNS setting was correct.

**Fix:** adapter metric priority. Using `Get-NetIPInterface` to find each adapter's interface index, then:
```powershell
Set-NetIPInterface -InterfaceIndex <host-only index> -InterfaceMetric 10
Set-NetIPInterface -InterfaceIndex <NAT index> -InterfaceMetric 20
```
Lower metric wins routing/DNS priority, so the host-only adapter (and therefore the DC's DNS) is preferred, while NAT still provides internet access. This let both adapters coexist without disabling either one permanently.

**DC's IP address drifted after a snapshot restore.** Reverting to a clean snapshot reset the DC's network adapter back to DHCP, changing its IP and breaking the client's DNS pointer, which had been hardcoded to the originally planned address. Lesson: set static IPs *after* confirming a stable build, and re-verify them after any snapshot restore, since a restore reverts network config along with everything else.

## Lessons Learned

- A domain controller's clients depend on it for both authentication and DNS; neither recovers gracefully from a wrong DNS pointer, even when basic network connectivity (ping) is fine. Ping failing rules out networking; ping succeeding does not rule out DNS.
- Multiple active network adapters on one machine can silently compete for DNS/routing priority. Windows doesn't always pick the "obviously correct" one by default, interface metrics decide this explicitly.
- Snapshots preserve VM state exactly as taken, including stale network configuration. A static IP set before a snapshot is restored along with it; one set after is lost on the next revert.
- Resource contention on a 16 GB lab host can look identical to a genuine software hang (install progress stalling for 10+ minutes). Checking Task Manager for actual CPU/disk activity before assuming something is broken saved a lot of wasted restarts.

## Next Steps

- Domain attacks: Kerberoasting, password spraying against the created user accounts
- Group Policy hardening in response to those attacks
- Optionally reconnect Wazuh to monitor and alert on domain-based attacks in real time

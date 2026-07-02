# Active Directory Home Lab — AD-01
### Domain Setup, Promotion & Custom Forest Deployment

**Domain:** `kawther-dhifaoui.local` | **Environment:** VirtualBox | **Author:** Kawther Dhifaoui

---

## 1. Objectives

This lab documents the deployment of a fully functional single-forest Active Directory environment from the ground up, using VirtualBox as the hypervisor. The goal was to go beyond a copy-paste tutorial result by customizing the forest identity, encountering real infrastructure issues, and resolving them using standard sysadmin diagnostic methodology.

- Install and configure Windows Server 2022 as a Domain Controller (DC1)
- Promote a custom Active Directory forest under a personal domain name: `kawther-dhifaoui.local`
- Configure static IP addressing and DNS resolution across the lab network
- Join a Windows 11 workstation (CLIENT1) to the domain
- Diagnose and resolve real VirtualBox networking and hypervisor display issues
- Validate the domain join through Active Directory Users and Computers (ADUC) and PowerShell

---

## 2. Final Network Architecture

| Parameter | DC1 (Windows Server 2022) | CLIENT1 (Windows 11) |
|---|---|---|
| Role | Domain Controller / DNS Server | Domain-joined Workstation |
| Static IPv4 Address | `192.168.1.10` | `192.168.1.20` |
| Subnet Mask | `255.255.255.0` | `255.255.255.0` |
| Default Gateway | `192.168.1.1` | `192.168.1.1` |
| Preferred DNS Server | `127.0.0.1` (loopback — self-hosted DNS) | `192.168.1.10` (points to DC1) |
| Hostname / FQDN | `DC1.kawther-dhifaoui.local` | `CLIENT1.kawther-dhifaoui.local` |

Both adapters were bound to a reconfigured **Host-Only Ethernet Adapter #2** with its internal DHCP server disabled, ensuring IP assignment was fully controlled through static configuration on each guest OS — a deliberate best practice, since a Domain Controller must never rely on DHCP-assigned addressing.

```
[Windows Server 2022]                [Windows 11 Client]
 DC1.kawther-dhifaoui.local   <-->    joined to kawther-dhifaoui.local
 IP: 192.168.1.10                    IP: 192.168.1.20
 Roles: AD DS, DNS                   DNS points to DC1
```

---

## 3. Key Deployment Steps

### 3.1 Base Server Build
- Deployed Windows Server 2022 Standard (Desktop Experience) inside VirtualBox — 4096 MB RAM, 60 GB dynamically allocated VDI disk
- Assigned static IPv4 addressing (`192.168.1.10 /24`, gateway `192.168.1.1`, DNS `127.0.0.1`) before installing any role
- Renamed the host to `DC1` and restarted to apply the change

![Static IP](../../screenshots/03-dc1-static-ip.png)


### 3.2 AD DS Installation & Initial Promotion
- Installed the Active Directory Domain Services role via Server Manager
- Performed an initial promotion using the tutorial's default forest name, `corp.local`, to validate the base procedure end-to-end

### 3.3 Custom Forest Rebuild — Personal Branding
To move beyond a copy-paste tutorial result, I demoted DC1 out of the `corp.local` forest (destroying it), then re-ran the "Add a new forest" wizard and rebuilt a clean forest under my own name: **`kawther-dhifaoui.local`**.

![AD DS Configuration Wizard — new forest kawther-dhifaoui.local](./screenshots/05-new-forest-wizard.png)

Windows automatically generated a NetBIOS (short) name for the domain. Because Windows enforces a hard **15-character limit** on NetBIOS names, the 20-character label `kawther-dhifaoui` was truncated at the logon screen to `KAWTHER-DHIFAOU`. This is documented in the Troubleshooting Log (Issue #1) below.

![Logon screen showing truncated NetBIOS label KAWTHER-DHIFAOU](./screenshots/06-netbios-truncated-logon.png)

### 3.4 Client Preparation & Domain Join
- Built CLIENT1 as a Windows 11 VM on the same reconfigured Host-Only network as DC1
- Applied static IPv4 addressing (`192.168.1.20 /24`, gateway `192.168.1.1`, DNS `192.168.1.10`)
- Verified reachability with `ping 192.168.1.10` and `ping DC1.kawther-dhifaoui.local` before attempting the join

![CLIENT1 IPv4 config, DNS pointed to 192.168.1.10](./screenshots/08-client1-static-ip.png)

- Joined CLIENT1 to `kawther-dhifaoui.local` via System Properties → Change → Domain, authenticating with the full UPN `Administrator@kawther-dhifaoui.local`

![Welcome to the kawther-dhifaoui.local domain dialog](./screenshots/09-domain-join-success.png)

---

## 4. Troubleshooting Log

| # | Issue Encountered | Root Cause | Resolution Applied |
|---|---|---|---|
| 1 | NetBIOS name at logon screen truncated to `KAWTHER-DHIFAOU` instead of the full custom domain name | Windows enforces a strict 15-character limit on the NetBIOS (short) domain name, independent of the FQDN length | Authenticated using the full User Principal Name (UPN) format: `Administrator@kawther-dhifaoui.local` |
| 2 | *"An Active Directory Domain Controller for the domain kawther-dhifaoui.local could not be contacted"* during CLIENT1's domain-join attempt | VirtualBox's default Host-Only network (Adapter #1) is bound to `192.168.56.0/24`, which conflicted with the `192.168.1.0/24` static addressing configured in Windows | Reconfigured `Host-Only Ethernet Adapter #2` with gateway `192.168.1.1` in VirtualBox's Host Network Manager, disabled its built-in DHCP server, and reattached both DC1 and CLIENT1 to Adapter #2 |
| 3 | Windows 11 installation repeatedly crashed / hung inside the VM during setup | Insufficient virtual CPU allocation and video memory for the Windows 11 installer under VirtualBox's default VM profile | Increased the VM to 3 CPU cores and raised video memory to 128 MB before restarting the installation |
| 4 | Mouse cursor became unresponsive / frozen inside the VM display after installing the OS | Default VBoxVGA graphics controller has known pointer-integration issues with modern guest OS builds | Changed the graphics controller to VBoxSVGA and installed official Oracle VirtualBox Guest Additions on Windows Server 2022 |

![Failed ping / "Domain Controller could not be contacted" error](./screenshots/10-ping-fail-before-fix.png)

![VirtualBox Host Network Manager — Adapter #2 reconfigured](./screenshots/02-hostonly-adapter-fix.png)

![VM Display settings — VBoxSVGA controller + Guest Additions installed](./screenshots/11-vboxsvga-guest-additions.png)

---

## 5. Final Validation

- CLIENT1 successfully authenticated to the domain and displayed the welcome confirmation dialog
- On DC1, the Active Directory Users and Computers (ADUC) console was opened and the `kawther-dhifaoui.local` tree was expanded to the built-in **Computers** OU
- CLIENT1 appeared registered as a secure computer object inside the Computers OU, confirming a fully successful, end-to-end domain join

![ADUC console — kawther-dhifaoui.local → Computers OU showing CLIENT1](./screenshots/12-aduc-client1-registered.png)

Additional health verification was performed with `Get-ADDomain`, `Get-ADComputer -Filter *`, and `dcdiag /test:dns` on DC1, confirming both machines were correctly registered and DNS resolution was fully functional.

![PowerShell — Get-ADComputer -Filter * listing DC1 and CLIENT1](./screenshots/13-get-adcomputer-output.png)

---

## 6. Conclusion

This lab confirms hands-on competency across the full lifecycle of an Active Directory deployment: server build, static network configuration, forest promotion, domain identity customization, DNS-dependent workstation onboarding, and diagnosis of real virtualization-layer networking faults. The environment was deliberately rebuilt under a personal domain identity and pushed through authentic failure conditions — a NetBIOS truncation edge case, a hypervisor networking misconfiguration, and guest-OS display/performance issues — each resolved using standard, reproducible sysadmin methodology.

The domain `kawther-dhifaoui.local` is now fully operational, with DC1 acting as the forest root Domain Controller and DNS authority, and CLIENT1 successfully joined and validated.

**Next lab:** [AD-02 — User & Group Management](../AD-02/README.md) — Organizational Units, users, groups, and simulated help-desk tickets (account lockouts, password resets, disabled accounts).

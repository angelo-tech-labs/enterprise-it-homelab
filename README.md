# Enterprise IT Support Homelab

A practical Windows enterprise homelab built in Hyper-V to develop hands-on experience with Active Directory, Windows Server, networking, permissions and IT troubleshooting.

## Lab Environment

| Device | Role | IP |
|---|---|---|
| DC01 | Windows Server 2025 / Domain Controller / DNS | 10.10.10.10 |
| CLIENT01 | Windows 11 domain workstation | 10.10.10.x |
| Hyper-V Host | Lab gateway / NAT | 10.10.10.1 |

**Domain:** `corp.example.com`  
**Network:** `10.10.10.0/24`

---

## What I Built

- Windows Server 2025 Domain Controller
- Active Directory Domain Services
- DNS
- Windows 11 domain-joined workstation
- Organisational Units for IT, Finance, HR and Sales
- Test user accounts
- Departmental security groups
- SMB network file shares
- NTFS and Share permissions
- Group-based access control

Department groups:

```text
GG-IT
GG-Finance
GG-HR
GG-Sales
```

Users are assigned access through security groups rather than permissions being applied directly to individual accounts.

---

## Active Directory

I created a small fictional company structure with separate OUs for users, computers and departmental groups.

### AD Structure

<!-- PUT SCREENSHOT HERE:
Screenshot showing Company expanded with:
Computers, Finance, Groups, HR, IT, Sales
and GG-Finance, GG-HR, GG-IT, GG-Sales
-->

![Active Directory Structure](screenshots/ad-structure.png)

### Group Membership

Example IT users were added to the `GG-IT` security group.

<!-- PUT SCREENSHOT HERE:
GG-IT Properties → Members
showing Alex Morgan and Sam Wilson
-->

![GG-IT Membership](screenshots/gg-it-members.png)

---

## Department File Shares

I created departmental network shares on DC01:

```text
Finance
HR
IT
Sales
```

Access is controlled using the corresponding Active Directory security group.

| Share | Group |
|---|---|
| Finance | GG-Finance |
| HR | GG-HR |
| IT | GG-IT |
| Sales | GG-Sales |

Department users receive **Modify** access, allowing them to create, edit and delete files without being able to change folder security permissions.

### NTFS Permissions

<!-- PUT SCREENSHOT HERE:
One Security tab screenshot showing:
SYSTEM
Administrators
GG-Finance
with GG-Finance = Modify
-->

![NTFS Permissions](screenshots/finance-ntfs-permissions.png)

---

## Access Control Testing

I tested the permissions from CLIENT01.

A Finance user was able to access the Finance share and create, edit and delete files.

<!-- PUT SCREENSHOT HERE:
Finance network share open on CLIENT01
with Finance-Test.txt visible
-->

![Finance Share Access](screenshots/finance-share-success.png)

I then tested the same Finance share while logged in as **Alex Morgan**, a member of `GG-IT`.

Windows correctly denied access because the user was not a member of `GG-Finance`.

<!-- PUT SCREENSHOT HERE:
"You do not have permission to access..."
for the Finance share
-->

![Access Denied](screenshots/finance-access-denied.png)

This confirmed that departmental access control was working correctly.

---

## Troubleshooting

During file-share testing I used basic network and service troubleshooting.

```powershell
ping 10.10.10.10
```

Confirmed IP connectivity between CLIENT01 and DC01.

```powershell
Test-NetConnection 10.10.10.10 -Port 445
```

Confirmed connectivity to the SMB file-sharing service on TCP port 445.

This demonstrated the difference between a device being reachable at the network layer and a specific service being reachable.

---

## Skills Demonstrated

- Windows Server 2025
- Active Directory
- DNS
- Windows domain administration
- Users and security groups
- SMB file sharing
- NTFS permissions
- Role-based access control
- Windows 11 client administration
- TCP/IP troubleshooting
- PowerShell

---

## Progress

- [x] Hyper-V lab network
- [x] Windows Server 2025
- [x] Active Directory and DNS
- [x] Users, OUs and security groups
- [x] Windows 11 domain client
- [x] SMB departmental file shares
- [x] NTFS and Share permissions
- [x] Authorised / unauthorised access testing
- [ ] Group Policy
- [ ] Mapped drives
- [ ] DHCP
- [ ] PowerShell administration
- [ ] Wireshark packet analysis
- [ ] IT support troubleshooting scenarios

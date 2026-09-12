# Enterprise IT Support Homelab

A practical Windows enterprise homelab built in Hyper-V to develop hands-on experience with Windows Server, Active Directory, DNS, domain administration, file permissions and troubleshooting.

The environment simulates a small business with separate IT, Finance, HR and Sales departments.

---

## Lab Environment

| Device | Role | IP Address |
|---|---|---|
| Hyper-V Host | Lab gateway / NAT | 10.10.10.1 |
| DC01 | Windows Server 2025 / Domain Controller / DNS | 10.10.10.10 |
| CLIENT01 | Windows 11 domain workstation | 10.10.10.20 |

**Domain:** `corp.example.com`  
**Network:** `10.10.10.0/24`

The lab uses an internal Hyper-V network with NAT through the Windows host.

---

## DC01 Network Configuration

DC01 was configured with a static IP address and acts as the DNS server for the Active Directory domain.

CLIENT01 uses DC01 for DNS so it can locate domain services and authenticate against `corp.example.com`.

![DC01 Network Configuration](screenshots/01-dc01-network.png)

---

## Active Directory

I installed Active Directory Domain Services and created the domain:

```text
corp.example.com
```

I then created a small fictional company structure using Organisational Units for:

- IT
- Finance
- HR
- Sales
- Computers
- Groups

Departmental Global Security Groups were created:

```text
GG-IT
GG-Finance
GG-HR
GG-Sales
```

Users were created inside their relevant departmental OUs and added to the corresponding security groups.

Permissions are assigned to groups rather than directly to individual user accounts.

![Active Directory Structure](screenshots/02-ad-structure.png)

---

## Windows 11 Domain Client

CLIENT01 was configured as a Windows 11 workstation on the lab network.

It was configured to use DC01 for DNS and successfully joined to:

```text
corp.example.com
```

Domain users can then sign into CLIENT01 using their Active Directory accounts.

This allows the client workstation to receive domain authentication, permissions and future Group Policy settings from the server.

---

## Departmental File Shares

I created departmental folders on DC01:

```text
C:\Company Shares\
├── Finance
├── HR
├── IT
└── Sales
```

Each folder was published as an SMB network share.

| Network Share | Active Directory Group |
|---|---|
| Finance | GG-Finance |
| HR | GG-HR |
| IT | GG-IT |
| Sales | GG-Sales |

For example:

```text
\\DC01\Finance
```

or by IP:

```text
\\10.10.10.10\Finance
```

---

## Share and NTFS Permissions

I configured two permission layers for each departmental share.

### Share Permissions

The relevant departmental group receives:

```text
Change
Read
```

Standard users are not given Full Control.

### NTFS Permissions

The corresponding security group receives:

```text
Modify
Read & execute
List folder contents
Read
Write
```

`SYSTEM` and `Administrators` retain Full Control.

Generic domain user access was removed so users must belong to the correct departmental group to access the folder.

This means a user can create, edit, rename and delete departmental files without being able to change folder security permissions.

---

## Access Control Testing

I tested the permissions from CLIENT01 using different domain users.

### Authorised User

A Finance user successfully accessed:

```text
\\10.10.10.10\Finance
```

The user was able to:

- open the share
- create a file
- edit and save it
- rename it
- delete it

![Authorised Finance Access](screenshots/03-finance-share-success.png)

### Unauthorised User

I then tested the same Finance share while logged in as an IT user.

The IT user belongs to:

```text
GG-IT
```

but not:

```text
GG-Finance
```

Windows correctly denied access to the Finance share.

![Unauthorised Access Denied](screenshots/04-finance-access-denied.png)

This confirmed that access was being controlled through Active Directory group membership.

The permission model is effectively:

```text
User
  ↓
Active Directory Security Group
  ↓
Share + NTFS Permissions
  ↓
Department Network Share
```

For example:

```text
Alex Morgan
     ↓
GG-IT
     ↓
IT Share
```

If a user changes department, access can be changed by updating their security group membership instead of manually assigning folder permissions to that individual user.

---

## Networking and Troubleshooting

I also used the lab to practise troubleshooting connectivity between CLIENT01 and DC01.

Basic IP connectivity was tested with:

```powershell
ping 10.10.10.10
```

This confirmed that CLIENT01 could reach DC01.

Because Windows file sharing uses SMB over TCP port 445, I also tested the service directly:

```powershell
Test-NetConnection 10.10.10.10 -Port 445
```

This confirmed connectivity to the SMB service.

This demonstrated an important troubleshooting difference:

```text
Ping succeeds
     ↓
The device is reachable

TCP 445 succeeds
     ↓
The SMB service is reachable
```

A host can respond to ping while a specific service or port is still unavailable.

---

## Skills Demonstrated

- Hyper-V
- Windows Server 2025
- Active Directory Domain Services
- DNS
- Windows domain administration
- Windows 11 domain joining
- Organisational Units
- User account administration
- Active Directory security groups
- Group-based access control
- SMB network shares
- Share permissions
- NTFS permissions
- TCP/IP troubleshooting
- PowerShell network testing

---

## Current Progress

- [x] Hyper-V internal lab network
- [x] NAT gateway
- [x] Windows Server 2025
- [x] DC01 static networking
- [x] Active Directory Domain Services
- [x] DNS
- [x] `corp.example.com` domain
- [x] Organisational Units
- [x] Test users
- [x] Departmental security groups
- [x] User group membership
- [x] Windows 11 CLIENT01
- [x] CLIENT01 domain join
- [x] Departmental SMB shares
- [x] Share permissions
- [x] NTFS permissions
- [x] Authorised access testing
- [x] Unauthorised access testing
- [x] SMB / TCP 445 troubleshooting
- [ ] Group Policy
- [ ] Mapped network drives
- [ ] DHCP
- [ ] PowerShell administration tasks
- [ ] Wireshark packet analysis
- [ ] IT support troubleshooting scenarios

---

## Next Steps

Next I will expand the environment with:

- Group Policy
- mapped network drives
- DHCP
- password resets and account lockouts
- new starter and leaver scenarios
- PowerShell administration
- Wireshark analysis
- realistic IT support tickets

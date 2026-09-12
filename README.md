# Enterprise IT Support Homelab

I built this lab in Hyper-V to get practical experience with Windows Server, Active Directory, DNS, domain-joined clients, file sharing and troubleshooting.

The idea was to create a small company environment with IT, Finance, HR and Sales departments rather than just install Windows Server and stop there.

---

## Lab Environment

| Device | Role | IP Address |
|---|---|---|
| Hyper-V Host | Lab gateway / NAT | 10.10.10.1 |
| DC01 | Windows Server 2025 / Domain Controller / DNS | 10.10.10.10 |
| CLIENT01 | Windows 11 domain workstation | 10.10.10.20 |

**Domain:** `corp.example.com`  
**Network:** `10.10.10.0/24`

The lab runs on an internal Hyper-V virtual switch, with the host providing connectivity for the VMs.

---

## DC01 Networking

I gave DC01 a static IP so the domain controller and DNS server always stay at the same address.

CLIENT01 uses DC01 as its DNS server. This is important because the client needs to find the Active Directory domain and its services through DNS.

![DC01 Network Configuration](screenshots/01-dc01-network.png)

---

## Active Directory

I installed Active Directory Domain Services and created the domain:

```text
corp.example.com
```

I then built a basic company structure with separate OUs for:

- IT
- Finance
- HR
- Sales
- Computers
- Groups

I also created departmental Global Security Groups:

```text
GG-IT
GG-Finance
GG-HR
GG-Sales
```

Users are added to the group for their department instead of giving permissions directly to individual accounts.

That means access can be managed through group membership, which is much easier to maintain.

![Active Directory Structure](screenshots/02-ad-structure.png)

---

## CLIENT01

I created a Windows 11 VM called CLIENT01, connected it to the same lab network and configured it to use DC01 for DNS.

CLIENT01 was then joined to:

```text
corp.example.com
```

I used this machine to test domain logins, network connectivity and permissions from the user's side rather than only configuring everything on the server.

---

## Department File Shares

On DC01 I created separate folders for each department:

```text
C:\Company Shares\
├── Finance
├── HR
├── IT
└── Sales
```

I shared each folder over SMB and matched it to the correct Active Directory group.

| Share | Security Group |
|---|---|
| Finance | GG-Finance |
| HR | GG-HR |
| IT | GG-IT |
| Sales | GG-Sales |

For example:

```text
\\DC01\Finance
```

---

## Permissions

I configured both Share permissions and NTFS permissions.

For the department groups, Share permissions allow:

```text
Change
Read
```

NTFS permissions give the relevant department group:

```text
Modify
Read & execute
List folder contents
Read
Write
```

`SYSTEM` and `Administrators` keep Full Control.

I removed the generic domain user access so a user has to be a member of the correct department group to access that department's files.

---

## Testing the Permissions

I tested everything from CLIENT01 rather than assuming the permissions were correct.

A Finance user could open the Finance share and create, edit and delete files.

![Finance Share Access](screenshots/03-finance-share-success.png)

I then logged in with an IT user and tried to access the same Finance share.

Windows correctly denied access because the IT user was not a member of `GG-Finance`.

![Finance Access Denied](screenshots/04-finance-access-denied.png)

That confirmed that access was actually being controlled by Active Directory group membership.

The basic idea is:

```text
User
 ↓
Department Security Group
 ↓
Share + NTFS Permissions
 ↓
Department Folder
```

So if someone moves department, I can change their group membership rather than manually changing permissions on individual folders.

---

## Troubleshooting I Ran Into

Not everything worked first time, which ended up being one of the more useful parts of the lab.

### SMB connectivity

CLIENT01 could ping DC01, but at one point I thought SMB was failing.

I used:

```powershell
Test-NetConnection 10.10.10.10 -Port 445
```

to test the SMB service directly.

I initially typed port **45 instead of 445**, which made it look like the connection was failing. After checking the command and testing the correct SMB port, the connection succeeded.

It was a useful reminder to verify the exact protocol and port before changing configuration.

### Network path testing

I also initially typed the network path into the File Explorer search box instead of the address bar.

The correct path was:

```text
\\10.10.10.10\Finance
```

Once I entered it in the correct place, I could properly test whether the logged-in user had access.

### Enhanced Session / RDP

When switching between domain users, Hyper-V Enhanced Session also gave me a Remote Desktop sign-in permissions error.

That helped me understand that Enhanced Session uses RDP functionality, and a normal domain user does not automatically have Remote Desktop sign-in rights.

For the file-share testing I used the normal VM console instead.

---

## What I Practised

- Hyper-V
- Windows Server 2025
- Active Directory Domain Services
- DNS
- Windows domain joining
- Organisational Units
- User accounts
- Security groups
- Group-based access control
- SMB file sharing
- Share permissions
- NTFS permissions
- Windows 11 client administration
- TCP/IP troubleshooting
- PowerShell network testing

---

## Progress

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
- [ ] PowerShell administration
- [ ] Wireshark packet analysis
- [ ] IT support troubleshooting scenarios

---

## Next Steps

Next I plan to add:

- Group Policy
- mapped network drives
- DHCP
- password reset and account lockout scenarios
- new starter and leaver tasks
- PowerShell administration
- Wireshark captures
- realistic IT support tickets

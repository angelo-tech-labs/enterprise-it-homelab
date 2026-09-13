# Enterprise IT Support Homelab

I built this lab in Hyper-V to get practical experience with Windows Server, Active Directory, DNS, domain-joined clients, file sharing and troubleshooting.

Rather than just installing Windows Server, I wanted to build something closer to a small company environment with IT, Finance, HR and Sales departments and then test it from a normal Windows client.

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

CLIENT01 uses DC01 as its DNS server so it can locate the Active Directory domain and its services.

![DC01 Network Configuration](https://github.com/user-attachments/assets/592c591c-c309-4cb7-8960-dbba763fb1d2)

---

## Active Directory

I installed Active Directory Domain Services and created:

```text
corp.example.com
```

I then created a basic company structure with separate OUs for:

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

Users are added to the group for their department rather than having permissions assigned directly to each account.

This makes access easier to manage because changing someone's group membership can change what resources they can use.

![Active Directory Structure](https://github.com/user-attachments/assets/d57e91a4-b670-4fa0-86aa-a22ab384731d)

---

## CLIENT01

I created a Windows 11 VM called CLIENT01, connected it to the same lab network and configured it to use DC01 for DNS.

CLIENT01 was then joined to:

```text
corp.example.com
```

I used CLIENT01 to test domain logins, connectivity and permissions from the user's side instead of only configuring everything directly on the server.

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

Each folder was shared over SMB and matched to the correct Active Directory group.

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

## Share and NTFS Permissions

I configured both Share permissions and NTFS permissions for each department.

The departmental groups have these Share permissions:

```text
Change
Read
```

The matching department group has these NTFS permissions:

```text
Modify
Read & execute
List folder contents
Read
Write
```

`SYSTEM` and `Administrators` keep Full Control.

I removed the generic domain user access so users need to belong to the correct departmental group to access that department's files.

This means normal users can create, edit and delete their department's files without being able to change the folder's security permissions.

---

## Testing the Permissions

I tested the configuration from CLIENT01 rather than assuming the permissions were correct.

### Authorised access

A Finance user was able to open the Finance share and create, edit and delete files.

![Finance Share Access](https://github.com/user-attachments/assets/78728055-0635-4f91-8ec4-1a5762503d62)

### Unauthorised access

I then logged in with an IT user and tried to access the same Finance share.

Windows correctly denied access because the IT user was not a member of `GG-Finance`.

![Finance Access Denied](https://github.com/user-attachments/assets/b99a6d93-61ad-4ed6-87bb-6e57b03c810c)

That confirmed that access was actually being controlled through Active Directory group membership.

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

Not everything worked first time, and this ended up being one of the most useful parts of the lab.

### Checking SMB on DC01

CLIENT01 could ping DC01, but when I first tested the file share I thought SMB was failing.

Before changing more settings, I checked the server side.

I confirmed:

- the Windows Server service was running
- TCP port 445 was listening
- DC01 was using the `DomainAuthenticated` network profile

<!-- PASTE Screenshot 2026-09-12 110803.png HERE -->

This helped confirm that the SMB service itself was running correctly on DC01.

### Testing TCP 445 from CLIENT01

I then tested the actual SMB port from CLIENT01:

```powershell
Test-NetConnection 10.10.10.10 -Port 445
```

The successful test returned:

```text
TcpTestSucceeded : True
```

<!-- PASTE NEW CLIENT01 SCREENSHOT SHOWING PORT 445 = TRUE HERE -->

This was useful because ping and a TCP port test answer two different questions:

```text
Ping works
    ↓
CLIENT01 can reach DC01

TCP 445 works
    ↓
CLIENT01 can reach the SMB service
```

### My port mistake

While troubleshooting, I initially typed:

```powershell
-Port 45
```

instead of:

```powershell
-Port 445
```

That made it look like SMB was unavailable and sent me in the wrong direction for a while.

Once I checked the command properly and tested the correct port, the connection succeeded.

It was a good reminder to check the exact protocol, port and command before changing configuration.

### Network path mistake

I also initially entered:

```text
\\10.10.10.10\Finance
```

into the File Explorer search box instead of the address bar.

Once I realised what I had done and entered the UNC path in the address bar, Windows actually attempted to connect to the network share.

### Hyper-V Enhanced Session

When switching between domain users I also received a Remote Desktop sign-in permissions error.

I found that Hyper-V Enhanced Session relies on Remote Desktop functionality, and a normal domain user does not automatically have permission to sign in through RDP.

For the file-share tests I switched back to the normal VM console instead of giving users extra RDP permissions just for the test.

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
- Basic service and port troubleshooting

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

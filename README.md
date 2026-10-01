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

- the Windows Server service (`LanmanServer`) was running
- TCP port 445 was listening
- DC01 was using the `DomainAuthenticated` network profile

![DC01 SMB service, TCP 445 listener and DomainAuthenticated profile](https://github.com/user-attachments/assets/33432dbc-9b91-443a-b107-ea9f2dcb99d7)

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

![PowerShell TCP 445 test returning TcpTestSucceeded True](https://github.com/user-attachments/assets/fe1065a0-e916-4834-9434-61d5ed04e1c0)

The screenshot above shows `SourceAddress: 10.10.10.10`, which is DC01 in this lab. It shows a successful TCP 445 test, but it does not document the connection from CLIENT01 at `10.10.10.20`.

Ping only proves IP reachability. Testing TCP 445 checks whether the SMB port can be reached; the file-share tests check access to the files.

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
## DHCP and Client Networking

After completing the initial Active Directory, file-sharing and SMB work, I expanded the lab by installing and authorising the DHCP Server role on DC01.

I created an IPv4 scope for the lab network:

```text
Network:       10.10.10.0/24
Address pool:  10.10.10.100 - 10.10.10.200
Lease:         8 days
```

The following DHCP options were configured:

```text
003 Router:          10.10.10.1
006 DNS Servers:     10.10.10.10
015 DNS Domain Name: corp.example.com
```

CLIENT01 was changed from a static configuration to DHCP and successfully received:

```text
IPv4:        10.10.10.100
Gateway:     10.10.10.1
DHCP Server: 10.10.10.10
DNS Server:  10.10.10.10
```

<!--
SCREENSHOT 5
RECOMMENDED NAME: 05-dhcp-lease.png
ORIGINAL FILE: Screenshot 2026-09-21 194649.png
SHOWS: DHCP Manager > Address Leases > CLIENT01 at 10.10.10.100
-->

### DHCP Lease

<!-- DRAG SCREENSHOT 5 HERE -->

---

## DNS Troubleshooting

I deliberately introduced a DNS fault on CLIENT01 by manually changing its DNS server to:

```text
8.8.8.8
```

Basic IP connectivity still worked, but the public DNS server did not correctly resolve the private Active Directory hostname:

```text
dc01.corp.example.com
```

`nslookup` showed that CLIENT01 was using:

```text
dns.google
8.8.8.8
```

rather than the internal domain DNS server.

I restored CLIENT01 to receive its DNS configuration automatically from DHCP.

The client then used:

```text
10.10.10.10
```

and successfully resolved:

```text
dc01.corp.example.com -> 10.10.10.10
```

<!--
SCREENSHOT 6
RECOMMENDED NAME: 06-dns-before-after.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 23 September 2026

IDENTIFY IT BY:
The same PowerShell window shows BOTH tests.

First:
Server: dns.google
Address: 8.8.8.8

Then:
Server: Unknown
Address: 10.10.10.10
Name: dc01.corp.example.com
Address: 10.10.10.10
-->

### DNS Fault and Resolution

<!-- DRAG SCREENSHOT 6 HERE -->

This demonstrated an important troubleshooting principle: successful IP connectivity does not necessarily mean name resolution is working.

---

## PowerShell Administration

I also started using PowerShell alongside the GUI for Active Directory administration and troubleshooting.

Commands I practised included:

```powershell
Get-ADUser -Filter *
Get-ADGroup -Filter *
Get-ADGroupMember "GG-IT"
Get-ADGroupMember "GG-Finance"
Get-ADGroupMember "GG-Sales"
Get-ADUser "alex.morgan"
```

One of the first commands I used to inspect departmental membership was:

```powershell
Get-ADGroupMember "GG-IT"
```

which returned Alex Morgan and Sam Wilson.

<!--
SCREENSHOT 7
RECOMMENDED NAME: 07-powershell-groups.png
ORIGINAL FILE: Screenshot 2026-09-21 195248.png
SHOWS:
Get-ADGroupMember "GG-IT"
alex morgan
sam wilson
-->

### PowerShell Group Query

<!-- DRAG SCREENSHOT 7 HERE -->

At this stage I am not trying to memorise every PowerShell command. My focus is understanding what the command is querying or changing and why I would use it during administration or troubleshooting.

---

## New Starter Administration

I completed a new-starter workflow for a Sales user called Mia Turner.

Using PowerShell and Active Directory, I:

```text
Created the account
        ↓
Moved the user into the Sales OU
        ↓
Set a temporary password
        ↓
Enabled the account
        ↓
Added the user to GG-Sales
        ↓
Verified group membership
        ↓
Signed into CLIENT01
        ↓
Verified departmental access
```

After signing into CLIENT01 as Mia, the Sales drive mapped automatically through Group Policy:

```text
Sales (\\DC01) (S:)
```

<!--
SCREENSHOT 8
RECOMMENDED NAME: 08-mia-sales-drive.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 24 September 2026

IDENTIFY IT BY:
File Explorer > This PC
Network Locations
Sales (\\DC01) (S:)
-->

### New Starter Verification

<!-- DRAG SCREENSHOT 8 HERE -->

This gave me practical experience with the type of user onboarding work commonly handled by an IT support or service-desk team.

---

## Packet Analysis with Wireshark

To connect the Windows lab with the networking concepts I am studying for the CCNA, I installed Wireshark on CLIENT01 and captured real traffic generated by the lab.

### ICMP

I captured a ping between:

```text
CLIENT01: 10.10.10.100
DC01:     10.10.10.10
```

The capture showed alternating:

```text
ICMP Echo Request
ICMP Echo Reply
```

<!--
SCREENSHOT 9
RECOMMENDED NAME: 09-wireshark-icmp.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 26 September 2026

IDENTIFY IT BY:
Wireshark filter = icmp
Rows alternate:
10.10.10.100 -> 10.10.10.10 Echo request
10.10.10.10 -> 10.10.10.100 Echo reply
-->

<!-- DRAG SCREENSHOT 9 HERE -->

### ARP

I cleared the client's ARP cache and generated new traffic so I could observe address resolution.

The capture showed the ARP request and response used to associate an IPv4 address with a MAC address on the local network.

<!--
SCREENSHOT 10
RECOMMENDED NAME: 10-wireshark-arp.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 26 September 2026

IDENTIFY IT BY:
Wireshark filter = arp

Visible lines include:
Who has 10.10.10.100? Tell 10.10.10.10
10.10.10.100 is at 00:15:5d:01:68:02
-->

<!-- DRAG SCREENSHOT 10 HERE -->

This helped reinforce the relationship:

```text
IPv4 address
      ↓
ARP resolution
      ↓
MAC address
      ↓
Ethernet frame delivery
```

### DNS

I generated a DNS lookup for:

```text
dc01.corp.example.com
```

and captured the query and response.

The response contained the A record:

```text
10.10.10.10
```

<!--
SCREENSHOT 11
RECOMMENDED NAME: 11-wireshark-dns.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 27 September 2026

IDENTIFY IT BY:
Wireshark filter:
dns.qry.name contains "dc01"

Visible traffic includes:
Standard query A dc01.corp.example.com
Standard query response ... A 10.10.10.10
-->

<!-- DRAG SCREENSHOT 11 HERE -->

### TCP Three-Way Handshake

I captured a TCP connection from CLIENT01 to DC01 on SMB port:

```text
445
```

The beginning of the connection showed:

```text
SYN
SYN, ACK
ACK
```

<!--
SCREENSHOT 12
RECOMMENDED NAME: 12-wireshark-tcp445.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 27 September 2026

IDENTIFY IT BY:
Wireshark filter:
tcp.port == 445

Look for:
52581 -> 445 [SYN]
445 -> 52581 [SYN, ACK]
52581 -> 445 [ACK]
-->

<!-- DRAG SCREENSHOT 12 HERE -->

I also inspected SMB2 traffic including Session Setup, Tree Connect, Create and Close operations, but kept the TCP handshake as the clearer screenshot for the main README.

### DHCP DORA

I released and renewed CLIENT01's DHCP lease while capturing traffic.

The complete process was visible:

```text
Discover
Offer
Request
ACK
```

The capture also showed the client using:

```text
0.0.0.0
```

before it had received an address and communicating using broadcast:

```text
255.255.255.255
```

<!--
SCREENSHOT 13
RECOMMENDED NAME: 13-wireshark-dhcp-dora.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 27 September 2026

IDENTIFY IT BY:
Wireshark filter = dhcp

Rows:
DHCP Release
DHCP Discover
DHCP Offer
DHCP Request
DHCP ACK
-->

<!-- DRAG SCREENSHOT 13 HERE -->

These captures helped connect the Windows services I configured with the packet-level behaviour I am studying for the CCNA.

---

## Jira Service Management

I created an `Enterprise IT Service Desk` in Jira Service Management to practise documenting faults using a realistic first-line support workflow.

The process I use is:

```text
User reports issue
        ↓
Ticket created / assigned
        ↓
Reproduce the problem
        ↓
Investigate
        ↓
Identify root cause
        ↓
Apply fix
        ↓
Verify service
        ↓
Document resolution
        ↓
Close ticket
```

The issues are actually reproduced inside the Hyper-V lab rather than simply writing fictional solutions.

---

## Support Scenario - Account Lockout

Alex Morgan reported that he was unable to sign into CLIENT01.

I reproduced the issue and Windows displayed:

```text
The referenced account is currently locked out
and may not be logged on to.
```

<!--
SCREENSHOT 14
RECOMMENDED NAME: 14-alex-lockout-client.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 30 September 2026

IDENTIFY:
Blue Windows login screen
User: alex morgan
Message says account is currently locked out
-->

### User Error

<!-- DRAG SCREENSHOT 14 HERE -->

I then checked Alex's account in Active Directory Users and Computers.

The Account tab confirmed:

```text
Unlock account.
This account is currently locked out on this
Active Directory Domain Controller.
```

<!--
SCREENSHOT 15
RECOMMENDED NAME: 15-alex-lockout-aduc.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 30 September 2026

IDENTIFY:
alex morgan Properties > Account
"Unlock account. This account is currently locked out..."
-->

### Active Directory Confirmation

<!-- DRAG SCREENSHOT 15 HERE -->

I unlocked the account and confirmed that Alex could sign into CLIENT01 again.

I then recorded the resolution in Jira Service Management and closed the ticket.

<!--
SCREENSHOT 16
RECOMMENDED NAME: 16-jira-alex-completed.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 30 September 2026

IDENTIFY:
Enterprise IT Service Desk > All work
Alex Morgan cannot sign in to CLIENT01
Status: Completed
Resolution: Done

DO NOT USE:
Screenshot 2026-09-30 192243.png
That is only the Resolve window.
-->

### Completed Ticket

<!-- DRAG SCREENSHOT 16 HERE -->

The process was:

```text
Unable to sign in
       ↓
Reproduce error
       ↓
Check AD account
       ↓
Confirm lockout
       ↓
Unlock account
       ↓
Verify login
       ↓
Document and close ticket
```

---

## Support Scenario - Missing Sales Drive

Sophie Brown reported that her normal Sales `S:` drive was missing.

I reproduced the fault on CLIENT01.

File Explorer showed the local disk and DVD drive but no Sales network drive.

<!--
SCREENSHOT 17
RECOMMENDED NAME: 17-sophie-drive-missing.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 1 October 2026

IDENTIFY:
File Explorer > This PC
Only C: and DVD
NO Network Locations
NO Sales S:
-->

### Symptom

<!-- DRAG SCREENSHOT 17 HERE -->

I checked Sophie's Active Directory group membership.

Her account showed only:

```text
Domain Users
```

and the required group:

```text
GG-Sales
```

was missing.

<!--
SCREENSHOT 18
RECOMMENDED NAME: 18-sophie-group-missing.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 1 October 2026

IDENTIFY:
sophie brown Properties > Member Of
Only Domain Users appears
-->

### Root Cause

<!-- DRAG SCREENSHOT 18 HERE -->

The Sales drive mapping uses Group Policy Preferences with Item-Level Targeting based on membership of:

```text
GG-Sales
```

Because Sophie was not a member of that group, the drive mapping did not apply.

I added Sophie back to `GG-Sales`.

The drive did not immediately return because her existing Windows session still contained the old logon security token.

Running:

```cmd
gpupdate /force
```

refreshes Group Policy but does not rebuild an existing user's security token.

I therefore completely signed Sophie out and logged her back in.

Windows created a new token containing `GG-Sales`, allowing the drive-map policy to apply.

The Sales drive was restored:

```text
Sales (\\DC01) (S:)
```

<!--
SCREENSHOT 19
RECOMMENDED NAME: 19-sophie-drive-restored.png
ORIGINAL CHAT FILE: image.png
CAPTURED: 1 October 2026

IDENTIFY:
File Explorer > This PC
Network Locations
Sales (\\DC01) (S:)
-->

### Verification

<!-- DRAG SCREENSHOT 19 HERE -->

The troubleshooting path was:

```text
S: drive missing
       ↓
Check group membership
       ↓
GG-Sales missing
       ↓
Add user to GG-Sales
       ↓
Full sign-out / sign-in
       ↓
New Windows security token
       ↓
S: drive restored
```

This helped demonstrate the difference between the membership currently stored in Active Directory and the group information contained inside an already logged-in user's security token.

---

## Unexpected Hyper-V Network Fault

During the account troubleshooting work, CLIENT01 unexpectedly stopped communicating properly with DC01.

Symptoms included:

```text
ICMP failed
DNS queries timed out
TCP 445 became unreachable
Netlogon could not locate a logon server
```

Both systems still had their expected IPv4 configuration:

```text
DC01:     10.10.10.10
CLIENT01: 10.10.10.100
```

and both VMs were attached to:

```text
LabSwitch
```

ARP information was still present, which made the problem less obvious.

I worked through the issue by checking:

```text
IP configuration
ARP
DNS
TCP 445
Windows Firewall
network profile
DNS service
Netlogon
Kerberos KDC
Hyper-V switch
VLAN configuration
virtual NIC isolation
MAC addresses
```

The important clue was an inconsistency between the MAC address configured for DC01's Hyper-V virtual NIC and the address being used by the guest.

After configuring a consistent static MAC address for the DC01 virtual NIC, communication was restored.

DNS, SMB and live domain authentication then worked again.

This was one of the most useful unexpected problems in the lab because the IPv4 configuration looked correct and ARP information was still present.

It forced me to troubleshoot the network from the lower layers upwards rather than immediately assuming that DNS or Active Directory was the root cause.

---

## What I Practised

- Hyper-V
- Windows Server 2025
- Windows 11 domain administration
- Active Directory Domain Services
- Organisational Units
- user and security-group administration
- SMB file sharing
- share and NTFS permissions
- authorised and unauthorised access testing
- DNS
- DHCP
- Group Policy
- Group Policy Preferences
- Item-Level Targeting
- account lockout policies
- mapped network drives
- PowerShell fundamentals
- new-starter administration
- Wireshark
- ARP
- ICMP
- DNS packet analysis
- TCP three-way handshake
- SMB2
- DHCP DORA
- Jira Service Management
- support ticket troubleshooting
- Hyper-V virtual network troubleshooting

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
- [x] test users
- [x] departmental security groups
- [x] Windows 11 CLIENT01
- [x] CLIENT01 domain join
- [x] departmental SMB shares
- [x] share permissions
- [x] NTFS permissions
- [x] authorised / unauthorised access testing
- [x] SMB / TCP 445 troubleshooting
- [x] Group Policy
- [x] mapped network drives
- [x] DHCP
- [x] DNS fault troubleshooting
- [x] PowerShell administration
- [x] new-starter workflow
- [x] Wireshark packet analysis
- [x] account lockout scenario
- [x] mapped-drive troubleshooting scenario
- [x] Jira Service Management workflow
- [x] Hyper-V network fault investigation
- [ ] remaining selected Jira support tickets
- [ ] final README cleanup

---

## Next Steps

Before marking the Windows homelab as **v1.0 complete**, I plan to:

- complete the remaining selected DNS support scenario
- complete the final selected service-desk scenario
- finish the screenshot documentation
- carry out a final README cleanup

After v1.0, my main lab focus will move to **CCNA networking and Cisco Packet Tracer**.

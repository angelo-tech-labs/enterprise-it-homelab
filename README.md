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
## Group Policy

After completing the initial Active Directory and file-server configuration, I started using Group Policy to manage user resources and restrictions centrally.

### Department Drive Mapping

I created:

```text
GPO - Department Drive Maps
```

The GPO uses Group Policy Preferences to map departmental SMB shares as drive letters.

For example, the Finance mapping was configured as:

```text
Path:  \\DC01\Finance
Drive: F:
Action: Update
```

<!--
FILE NAME: Screenshot 2026-09-15 201531.png
EXACT CHAT FILE NAME.

SHOWS:
Group Policy Management Editor
Finance drive mapping
\\DC01\Finance
Drive F:
Action = Update
-->

<!-- PASTE FILE: Screenshot 2026-09-15 201531.png -->

I used Item-Level Targeting so that a drive is only mapped when the logged-in user belongs to the relevant Active Directory security group.

<!--
FILE NAME: Screenshot 2026-09-15 202209.png
EXACT CHAT FILE NAME.

SHOWS:
Targeting Editor
"The user is a member of the security group"
Used for Item-Level Targeting.
-->

<!-- PASTE FILE: Screenshot 2026-09-15 202209.png -->

I eventually configured all four departmental mappings:

```text
Finance -> F: -> \\DC01\Finance
HR      -> H: -> \\DC01\HR
IT      -> I: -> \\DC01\IT
Sales   -> S: -> \\DC01\Sales
```

<!--
FILE NAME: Screenshot 2026-09-16 195658.png
EXACT CHAT FILE NAME.

SHOWS:
GPO - Department Drive Maps
All four mappings:
F: Finance
H: HR
I: IT
S: Sales
-->

<!-- PASTE FILE: Screenshot 2026-09-16 195658.png -->

The drive-map GPO was linked into the company OU structure so that it could apply to departmental users.

<!--
FILE NAME: Screenshot 2026-09-13 121117.png
EXACT CHAT FILE NAME.

SHOWS:
Group Policy Management
Company OU
GPO - Department Drive Maps linked and enabled.
-->

<!-- PASTE FILE: Screenshot 2026-09-13 121117.png -->

After refreshing Group Policy and signing in with an authorised Finance user, Windows automatically mapped:

```text
Finance (F:)
```

<!--
FILE NAME: Screenshot 2026-09-15 202947.png
EXACT CHAT FILE NAME.

SHOWS:
File Explorer > This PC
Network Locations
Finance (F:)
-->

<!-- PASTE FILE: Screenshot 2026-09-15 202947.png -->

I also used:

```cmd
gpupdate /force
```

while testing policy changes.

<!--
FILE NAME: Screenshot 2026-09-16 193016.png
EXACT CHAT FILE NAME.

SHOWS:
gpupdate /force
Computer Policy update completed successfully
User Policy update completed successfully
-->

<!-- PASTE FILE: Screenshot 2026-09-16 193016.png -->

---

## User Restriction GPO

I created another policy:

```text
GPO - User Restriction
```

This GPO was linked to the HR OU.

<!--
FILE NAME: Screenshot 2026-09-17 195806.png
EXACT CHAT FILE NAME.

SHOWS:
Group Policy Management tree
HR OU
GPO - User Restriction
-->

<!-- PASTE FILE: Screenshot 2026-09-17 195806.png -->

The policy:

```text
Prohibit access to Control Panel and PC settings
```

was enabled.

<!--
FILE NAME: Screenshot 2026-09-17 190443.png
EXACT CHAT FILE NAME.

SHOWS:
"Prohibit access to Control Panel and PC settings"
Enabled.
-->

<!-- PASTE FILE: Screenshot 2026-09-17 190443.png -->

I then tested it from the client.

Windows prevented the restricted user from opening the setting and displayed:

```text
This operation has been cancelled due to restrictions
in effect on this computer.
```

<!--
FILE NAME: Screenshot 2026-09-17 193730.png
EXACT CHAT FILE NAME.

SHOWS:
Windows Restrictions message confirming that the policy is working.
-->

<!-- PASTE FILE: Screenshot 2026-09-17 193730.png -->

---

## Domain Account Lockout Policy

I also created a domain account-lockout policy.

The final configuration was:

```text
Account lockout threshold:          5 invalid logon attempts
Account lockout duration:           15 minutes
Reset account lockout counter:      15 minutes
```

<!--
FILE NAME: Screenshot 2026-09-17 195307.png
EXACT CHAT FILE NAME.

SHOWS:
Group Policy Management Editor
Account Lockout Policy
5 invalid logon attempts
15 minute lockout
15 minute reset counter
-->

<!-- PASTE FILE: Screenshot 2026-09-17 195307.png -->

During testing I found that policy precedence affected which settings were actually active.

After correcting the GPO link order, I verified the effective domain settings using:

```cmd
net accounts /domain
```

The final output showed:

```text
Lockout threshold:                   5
Lockout duration:                   15
Lockout observation window:         15
```

<!--
FILE NAME: Screenshot 2026-09-17 200232.png
EXACT CHAT FILE NAME.

SHOWS:
net accounts /domain
Lockout threshold = 5
Lockout duration = 15
Observation window = 15
-->

<!-- PASTE FILE: Screenshot 2026-09-17 200232.png -->

This lockout policy was later used in one of my Jira support scenarios.

---

## DHCP and Client Networking

I installed and authorised the DHCP Server role on DC01.

I created a DHCP scope for:

```text
10.10.10.0/24
```

with the address pool:

```text
10.10.10.100 - 10.10.10.200
```

<!--
FILE NAME: Screenshot 2026-09-19 120055.png
EXACT CHAT FILE NAME.

SHOWS:
DHCP Manager
Address Pool
Start IP = 10.10.10.100
End IP = 10.10.10.200
-->

<!-- PASTE FILE: Screenshot 2026-09-19 120055.png -->

The following DHCP scope options were configured:

```text
003 Router:          10.10.10.1
006 DNS Servers:     10.10.10.10
015 DNS Domain Name: corp.example.com
```

<!--
FILE NAME: Screenshot 2026-09-19 120033.png
EXACT CHAT FILE NAME.

SHOWS:
DHCP Scope Options
003 Router = 10.10.10.1
006 DNS Servers = 10.10.10.10
015 DNS Domain Name = corp.example.com
-->

<!-- PASTE FILE: Screenshot 2026-09-19 120033.png -->

CLIENT01 was then changed from its original static configuration to DHCP.

The DHCP server successfully issued:

```text
10.10.10.100
```

to CLIENT01.

<!--
FILE NAME: Screenshot 2026-09-21 194649.png
EXACT CHAT FILE NAME.

SHOWS:
DHCP Manager
Address Leases
CLIENT01 = 10.10.10.100
-->

<!-- PASTE FILE: Screenshot 2026-09-21 194649.png -->

I verified the client configuration with:

```cmd
ipconfig /all
```

The output confirmed:

```text
IPv4:        10.10.10.100
Subnet:      255.255.255.0
Gateway:     10.10.10.1
DHCP Server: 10.10.10.10
DNS Server:  10.10.10.10
```

<!--
FILE NAME: Screenshot 2026-09-21 195852.png
EXACT CHAT FILE NAME.

SHOWS:
CLIENT01 ipconfig /all
IPv4 = 10.10.10.100
Gateway = 10.10.10.1
DHCP Server = 10.10.10.10
DNS Server = 10.10.10.10
-->

<!-- PASTE FILE: Screenshot 2026-09-21 195852.png -->

---

## PowerShell Administration

I started using PowerShell alongside the GUI to inspect and administer Active Directory.

For example:

```powershell
Get-ADUser -Filter *
```

allowed me to query domain users.

<!--
FILE NAME: Screenshot 2026-09-21 195017.png
EXACT CHAT FILE NAME.

SHOWS:
Get-ADUser -Filter *
including domain accounts such as Alex Morgan and Sam Wilson.
-->

<!-- PASTE FILE: Screenshot 2026-09-21 195017.png -->

I also used:

```powershell
Get-ADGroup -Filter *
```

to inspect Active Directory groups.

<!--
FILE NAME: Screenshot 2026-09-21 195144.png
EXACT CHAT FILE NAME.

SHOWS:
Get-ADGroup -Filter *
Active Directory group output.
-->

<!-- PASTE FILE: Screenshot 2026-09-21 195144.png -->

To inspect individual departmental groups I used commands such as:

```powershell
Get-ADGroupMember "GG-IT"
```

<!--
FILE NAME: Screenshot 2026-09-21 195248.png
EXACT CHAT FILE NAME.

SHOWS:
Get-ADGroupMember "GG-IT"
Alex Morgan
Sam Wilson
-->

<!-- PASTE FILE: Screenshot 2026-09-21 195248.png -->

I repeated the same process with Finance.

<!--
FILE NAME: Screenshot 2026-09-21 195355.png
EXACT CHAT FILE NAME.

SHOWS:
Get-ADGroupMember "GG-Finance"
Emily Clarke
James Hall
-->

<!-- PASTE FILE: Screenshot 2026-09-21 195355.png -->

I also queried individual users:

```powershell
Get-ADUser "alex.morgan"
```

<!--
FILE NAME: Screenshot 2026-09-21 195654.png
EXACT CHAT FILE NAME.

SHOWS:
Get-ADUser "alex.morgan"
DistinguishedName
Enabled
SamAccountName
UserPrincipalName
-->

<!-- PASTE FILE: Screenshot 2026-09-21 195654.png -->

At this stage I am not trying to memorise every PowerShell command.

My main focus is learning what each command queries or changes and when it is useful during administration or troubleshooting.

---

## DNS Troubleshooting

I deliberately introduced a DNS fault on CLIENT01.

Instead of using the internal domain DNS server:

```text
10.10.10.10
```

I manually configured:

```text
8.8.8.8
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
21 September 2026 around 20:01 UK time.

SHOWS:
IPv4 Properties
Obtain IP address automatically
Manual Preferred DNS server = 8.8.8.8
-->

<!-- PASTE FILE: image.png - IPv4 Properties showing 8.8.8.8 -->

The client still had working IPv4 connectivity.

A ping to DC01 by IP succeeded, but DNS queries were being sent to Google's public resolver.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
23 September 2026 around 18:21 UK time.

SHOWS:
Successful ping to 10.10.10.10
followed by nslookup
Server = dns.google
Address = 8.8.8.8
-->

<!-- PASTE FILE: image.png - ping works but DNS points to Google -->

`ipconfig /all` confirmed the underlying problem:

```text
DNS Servers: 8.8.8.8
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
23 September 2026 around 18:33 UK time.

SHOWS:
CLIENT01 ipconfig /all
IPv4 = 10.10.10.100
DHCP Server = 10.10.10.10
DNS Servers = 8.8.8.8
-->

<!-- PASTE FILE: image.png - ipconfig showing DNS 8.8.8.8 -->

I restored the adapter to obtain its DNS configuration automatically from DHCP.

A final `nslookup` then used DC01 and correctly resolved the private hostname.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
23 September 2026 around 18:58 UK time.

SHOWS BOTH TESTS IN THE SAME POWERSHELL WINDOW:

Before:
Server = dns.google
Address = 8.8.8.8

After:
Server = Unknown
Address = 10.10.10.10
Name = dc01.corp.example.com
Address = 10.10.10.10
-->

<!-- PASTE FILE: image.png - DNS before and after -->

This was a useful demonstration that:

```text
IP connectivity working
```

does not automatically mean:

```text
DNS is configured correctly
```

---

## New Starter Administration - Mia Turner

I completed a new-starter workflow for a fictional Sales user called Mia Turner.

I initially created the account using PowerShell:

```powershell
New-ADUser
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
24 September 2026 around 12:31 UK time.

SHOWS:
New-ADUser
Name = Mia Turner
SamAccountName = mia.turner
UserPrincipalName = mia.turner@corp.example.com
-->

<!-- PASTE FILE: image.png - Mia New-ADUser -->

Immediately after creation I queried the account.

The account existed but was initially:

```text
Enabled : False
```

and was still in the default Users container.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
24 September 2026 around 12:34 UK time.

SHOWS:
Get-ADUser "mia.turner"
Enabled = False
CN=Mia Turner,CN=Users...
-->

<!-- PASTE FILE: image.png - Mia initially disabled -->

I then moved the account into the Sales OU, set a temporary password and enabled it.

After the changes, the account showed:

```text
CN=Mia Turner,OU=Sales,OU=Company,...
Enabled : True
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
24 September 2026 around 13:03 UK time.

SHOWS:
Mia Turner
OU=Sales
Enabled = True
SamAccountName = mia.turner
UserPrincipalName = mia.turner@corp.example.com
-->

<!-- PASTE FILE: image.png - Mia enabled in Sales OU -->

I added Mia to:

```text
GG-Sales
```

and verified the group membership.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
24 September 2026 around 13:10 UK time.

SHOWS:
Get-ADGroupMember "gg-sales"

Members include:
Sophie Brown
Daniel King
Mia Turner
-->

<!-- PASTE FILE: image.png - GG-Sales containing Mia -->

After logging into CLIENT01 as Mia I checked the Group Policy context using:

```cmd
gpresult /r
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
24 September 2026 around 14:01 UK time.

SHOWS:
RSOP data for CORP\mia.turner on CLIENT-01
CN=Mia Turner,OU=Sales,OU=Company...
Group Policy applied from DC01.corp.example.com
-->

<!-- PASTE FILE: image.png - gpresult for Mia -->

The security-group output also showed:

```text
GG-Sales
```

in Mia's user context.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
24 September 2026 around 14:05 UK time.

SHOWS:
gpresult /r
Security groups
GG-Sales
-->

<!-- PASTE FILE: image.png - Mia GG-Sales in logon context -->

The Sales drive then mapped automatically:

```text
Sales (\\DC01) (S:)
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
24 September 2026 around 14:13 UK time.

SHOWS:
File Explorer > This PC
Network Locations
Sales (\\DC01) (S:)
-->

<!-- PASTE FILE: image.png - Mia Sales S drive -->

This exercise combined:

```text
Account creation
OU placement
Password management
Account enablement
Security-group membership
Domain login
Group Policy
Resource access
```

---

## Packet Analysis with Wireshark

I installed Wireshark on CLIENT01 to observe the actual network traffic generated by the lab.

This helped connect the Windows Server work with the networking concepts I am studying for the CCNA.

### ICMP

I captured traffic between:

```text
CLIENT01 10.10.10.100
DC01     10.10.10.10
```

The capture clearly showed:

```text
Echo Request
Echo Reply
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
26 September 2026 around 09:14 UK time.

SHOWS:
Wireshark filter = icmp
10.10.10.100 -> 10.10.10.10 Echo request
10.10.10.10 -> 10.10.10.100 Echo reply
-->

<!-- PASTE FILE: image.png - Wireshark ICMP -->

### ARP

I cleared the ARP cache and generated new traffic.

The capture showed IPv4-to-MAC address resolution.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
26 September 2026 around 09:24 UK time.

SHOWS:
Wireshark filter = arp

Who has 10.10.10.100? Tell 10.10.10.10

10.10.10.100 is at 00:15:5d:01:68:02
-->

<!-- PASTE FILE: image.png - Wireshark ARP -->

This reinforced the relationship:

```text
IPv4 address
      ↓
ARP
      ↓
MAC address
      ↓
Ethernet delivery
```

### DNS

I generated a DNS lookup for:

```text
dc01.corp.example.com
```

and filtered the traffic in Wireshark.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
27 September 2026 around 10:26 UK time.

SHOWS:
Wireshark filter:
dns.qry.name contains "dc01"

Includes:
query for dc01.corp.example.com
response A 10.10.10.10
-->

<!-- PASTE FILE: image.png - Wireshark DNS -->

The capture showed the client sending the query to DC01 and receiving:

```text
A 10.10.10.10
```

### TCP Three-Way Handshake

I captured an SMB TCP connection to port:

```text
445
```

and identified the TCP three-way handshake:

```text
SYN
SYN, ACK
ACK
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
27 September 2026 around 10:42 UK time.

SHOWS:
Wireshark TCP traffic on port 445.

Capture includes the new connection around the packets ending in:
52581 -> 445
445 -> 52581
52581 -> 445
-->

<!-- PASTE FILE: image.png - TCP 445 handshake -->

### SMB2

I also captured the actual SMB2 application traffic produced while Windows accessed network resources.

The capture included operations such as:

```text
Negotiate Protocol
Session Setup
Tree Connect
Create
Close
Tree Disconnect
Session Logoff
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
27 September 2026 around 10:52 UK time.

SHOWS:
Wireshark filter = smb2
Create Request
Create Response
Close Request
Close Response
-->

<!-- PASTE FILE: image.png - SMB2 Create and Close -->

This provided a useful view of what happens below File Explorer when a user accesses an SMB resource.

### DHCP DORA

I released and renewed CLIENT01's lease while Wireshark was capturing.

The complete DHCP process was visible:

```text
Discover
Offer
Request
ACK
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
27 September 2026 around 10:56 UK time.

SHOWS:
Wireshark filter = dhcp

DHCP Release
DHCP Discover
DHCP Offer
DHCP Request
DHCP ACK

Discover:
0.0.0.0 -> 255.255.255.255
-->

<!-- PASTE FILE: image.png - DHCP DORA -->

This also showed why a DHCP client initially uses:

```text
0.0.0.0
```

and broadcast:

```text
255.255.255.255
```

before it has its own IPv4 configuration.

---

## Jira Service Management

I created an:

```text
Enterprise IT Service Desk
```

in Jira Service Management so that I could document faults in a more realistic support workflow.

The general process I practised was:

```text
User reports issue
        ↓
Ticket created
        ↓
Assign / investigate
        ↓
Reproduce issue
        ↓
Identify root cause
        ↓
Apply fix
        ↓
Verify
        ↓
Document resolution
        ↓
Close ticket
```

I created several example tickets based on faults that could actually be reproduced in the Hyper-V lab.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
30 September 2026 around 20:25 UK time.

SHOWS:
Jira ticket list including:
Alex Morgan cannot open Settings or Control Panel
Daniel King lost network access
Sam Wilson cannot reach DC01 by internal name
James Hall Access Denied opening Finance folder
Sophie Brown cannot see Sales S drive
-->

<!-- PASTE FILE: image.png - Jira support ticket list -->

Rather than completing every fictional scenario, I selected several useful examples that demonstrated different troubleshooting skills.

---

## Unexpected Hyper-V Network Fault

While preparing the support scenarios, CLIENT01 unexpectedly lost communication with DC01.

This was not an intentionally created fault.

### Domain Authentication Failure

CLIENT01 was logged in as:

```text
CORP\alex.morgan
```

but:

```cmd
nltest /sc_query:corp.example.com
```

returned:

```text
ERROR_NO_LOGON_SERVERS
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
29 September 2026 around 19:06 UK time.

SHOWS:
whoami = corp\alex.morgan
LOGONSERVER = \\DC01
nltest /sc_query:corp.example.com
ERROR_NO_LOGON_SERVERS
-->

<!-- PASTE FILE: image.png - nltest no logon servers -->

DNS queries to DC01 also timed out.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
29 September 2026 around 19:08 UK time.

SHOWS:
nslookup dc01.corp.example.com
Server = Unknown
Address = 10.10.10.10
DNS request timed out
-->

<!-- PASTE FILE: image.png - DNS timeout during Hyper-V fault -->

I checked both systems' IPv4 configurations.

DC01 still had:

```text
IPv4:   10.10.10.10
Gateway: 10.10.10.1
DNS:     10.10.10.10 / ::1
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
29 September 2026 around 19:10 UK time.

SHOWS:
DC01 ipconfig
IPv4 = 10.10.10.10
Gateway = 10.10.10.1
DNS = ::1 and 10.10.10.10
MAC = 00-15-5D-01-68-00
-->

<!-- PASTE FILE: image.png - DC01 IP configuration -->

CLIENT01 also still had the expected DHCP configuration:

```text
IPv4:        10.10.10.100
Gateway:     10.10.10.1
DHCP Server: 10.10.10.10
DNS Server:  10.10.10.10
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
29 September 2026 around 19:10 UK time.

SHOWS:
CLIENT01 ipconfig
IPv4 = 10.10.10.100
Gateway = 10.10.10.1
DHCP Server = 10.10.10.10
DNS = 10.10.10.10
-->

<!-- PASTE FILE: image.png - CLIENT01 IP configuration -->

I also verified that the core domain services on DC01 were running:

```text
DNS
KDC
Netlogon
```

and that DNS was listening on UDP port 53.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
29 September 2026 around 19:11 UK time.

SHOWS:
Get-Service DNS,Netlogon,KDC
all Running

Get-NetUDPEndpoint -LocalPort 53
DNS listening on port 53
-->

<!-- PASTE FILE: image.png - domain services and UDP 53 -->

### ARP Worked While IP Communication Failed

One of the most useful clues came from testing ARP and ping together.

The client failed to ping DC01, but its ARP table still contained a MAC address for:

```text
10.10.10.10
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
29 September 2026 around 19:46 UK time.

SHOWS:
ping 10.10.10.10 = timeout

arp -a still contains:
10.10.10.1  -> 00-15-5D-01-68-01
10.10.10.10 -> 00-15-5D-01-68-00
-->

<!-- PASTE FILE: image.png - ping fails but ARP resolves -->

This showed that neighbour discovery was still occurring even though higher-level communication was failing.

### Hyper-V Adapter Investigation

I checked the virtual adapters from the Hyper-V host.

Both VMs were attached to:

```text
LabSwitch
```

and both reported an OK status.

The host showed:

```text
Client01 MAC: 00155D016802
DC01 MAC:     00155D016803
```

Both adapters were untagged and had no isolation configured.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
29 September 2026 around 19:59 UK time.

SHOWS:
Get-VMNetworkAdapter
Client01 LabSwitch 00155D016802 {Ok}
DC01 LabSwitch     00155D016803 {Ok}

Both VLAN = Untagged
IsolationMode = None
-->

<!-- PASTE FILE: image.png - Hyper-V adapter diagnostics -->

The important clue appeared when I compared that with the MAC address inside DC01.

Inside the guest operating system DC01 was using:

```text
00-15-5D-01-68-00
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
29 September 2026 around 20:04 UK time.

SHOWS:
getmac /v inside DC01
Ethernet physical address:
00-15-5D-01-68-00
-->

<!-- PASTE FILE: image.png - DC01 guest MAC -->

This did not match the dynamic MAC that Hyper-V was reporting for DC01.

After shutting down DC01 and configuring its virtual network adapter to use a consistent static MAC address matching the expected guest identity, communication returned.

After the change:

```text
CLIENT01 <-> DC01 connectivity restored
DNS restored
SMB restored
domain authentication restored
```

This was one of the most useful unexpected troubleshooting exercises in the lab because the IPv4 configuration looked correct and the normal services were running.

It forced me to investigate progressively through:

```text
IPv4
ARP
DNS
TCP
Windows services
Hyper-V switching
VLAN configuration
MAC addressing
```

rather than assuming the first visible symptom was the root cause.

---

## Support Scenario - Alex Morgan Account Lockout

One of my Jira tickets reported that Alex Morgan could not sign into CLIENT01.

I reproduced a real domain lockout using the account lockout policy configured earlier.

Windows displayed:

```text
The referenced account is currently locked out
and may not be logged on to.
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
30 September 2026 around 19:54 UK time.

SHOWS:
Windows login
alex morgan

"The referenced account is currently locked out and may not be logged on to."
-->

<!-- PASTE FILE: image.png - Alex locked-out login -->

I checked Alex's account in Active Directory Users and Computers.

The Account tab confirmed:

```text
Unlock account.
This account is currently locked out on this
Active Directory Domain Controller.
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
30 September 2026 around 19:55 UK time.

SHOWS:
alex morgan Properties > Account

"Unlock account. This account is currently locked out on this Active Directory Domain Controller."
-->

<!-- PASTE FILE: image.png - Alex ADUC lockout confirmation -->

I unlocked the account and verified successful login.

I then documented the resolution in Jira.

<!--
FILE NAME: Screenshot 2026-09-30 192243.png
EXACT CHAT FILE NAME.

SHOWS:
Jira Resolve dialog

Resolution = Done

Internal note:
Confirmed Alex Morgan's domain account was locked out in Active Directory.
Unlocked the account and verified successful sign-in to CLIENT01.
-->

<!-- PASTE FILE: Screenshot 2026-09-30 192243.png -->

The ticket then showed:

```text
Status:     Completed
Resolution: Done
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
30 September 2026 around 20:24 UK time.

SHOWS:
Enterprise IT Service Desk > All work

ITSD-1
Alex Morgan cannot sign in to CLIENT01

Status = Completed
Resolution = Done
-->

<!-- PASTE FILE: image.png - completed Alex Jira ticket -->

This provided a complete service-desk workflow:

```text
Ticket
  ↓
Reproduce fault
  ↓
Confirm root cause
  ↓
Fix in Active Directory
  ↓
Verify user login
  ↓
Document
  ↓
Resolve ticket
```

---

## Support Scenario - Sophie Brown Missing Sales Drive

Another Jira scenario involved Sophie Brown reporting that her normal Sales `S:` drive was missing.

### Reproducing the Fault

After removing the relevant departmental membership, File Explorer no longer showed the Sales drive.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
1 October 2026 around 19:18 UK time.

SHOWS:
File Explorer > This PC

Only:
Local Disk C:
DVD Drive D:

No Network Locations
No Sales S drive
-->

<!-- PASTE FILE: image.png - Sophie Sales drive missing -->

I checked the groups in Sophie's current Windows security token.

Searching specifically for:

```text
GG-Sales
```

returned no result.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
1 October 2026 around 19:22 UK time.

SHOWS:
whoami /groups | findstr /i "gg-sales"

No output returned.
-->

<!-- PASTE FILE: image.png - GG-Sales absent from current token -->

I then checked Sophie's Active Directory account directly.

Her `Member Of` tab showed only:

```text
Domain Users
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
1 October 2026 around 19:24 UK time.

SHOWS:
sophie brown Properties
Member Of

Domain Users only.
GG-Sales is absent.
-->

<!-- PASTE FILE: image.png - Sophie missing GG-Sales in AD -->

This identified the root cause.

The drive-map GPO used Item-Level Targeting against:

```text
GG-Sales
```

so without that membership Sophie did not meet the condition required for the `S:` mapping.

### Restoring the Group Membership

I added:

```text
GG-Sales
```

back to Sophie's account.

<!--
FILE NAME: Screenshot 2026-10-01 192441.png
EXACT CHAT FILE NAME.

SHOWS:
Select Groups
GG-Sales entered.
-->

<!-- PASTE FILE: Screenshot 2026-10-01 192441.png -->

I then verified the updated Active Directory membership.

<!--
FILE NAME: Screenshot 2026-10-01 192449.png
EXACT CHAT FILE NAME.

SHOWS:
sophie brown Properties > Member Of

Domain Users
GG-Sales
-->

<!-- PASTE FILE: Screenshot 2026-10-01 192449.png -->

However, the drive did not immediately reappear.

This demonstrated an important distinction:

```text
Active Directory membership has changed
```

does not automatically mean:

```text
the user's existing Windows logon token has changed
```

`gpupdate /force` refreshes Group Policy but does not rebuild the existing user security token.

I therefore completely signed Sophie out and logged her back in.

After the new login, `gpresult /r` showed the departmental drive-map GPO applying to Sophie.

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
1 October 2026 around 19:32 UK time.

SHOWS:
gpresult /r

USER SETTINGS:
CN=sophie brown,OU=Sales,OU=Company...

Applied Group Policy Objects:
GPO - Department Drive Maps
-->

<!-- PASTE FILE: image.png - Sophie's applied drive-map GPO -->

The Sales drive then returned:

```text
Sales (\\DC01) (S:)
```

<!--
FILE NAME: image.png
EXACT NAME RECEIVED BY CHAT.

CHAT UPLOAD:
1 October 2026 around 19:37 UK time.

SHOWS:
File Explorer > This PC

Network Locations:
Sales (\\DC01) (S:)
-->

<!-- PASTE FILE: image.png - Sophie Sales S drive restored -->

The full troubleshooting path was:

```text
Sales drive missing
        ↓
Check current Windows groups
        ↓
GG-Sales missing
        ↓
Check Active Directory
        ↓
Confirm missing membership
        ↓
Add GG-Sales
        ↓
Sign out / sign in
        ↓
New security token
        ↓
GPO item-level targeting matches
        ↓
Sales S drive restored
```

---

## What I Practised

- Hyper-V
- Windows Server 2025
- Windows 11 domain administration
- Active Directory Domain Services
- Organisational Units
- user administration
- security groups
- SMB file sharing
- share permissions
- NTFS permissions
- authorised and unauthorised access testing
- DNS
- DHCP
- DHCP scope configuration
- Group Policy
- Group Policy Preferences
- Item-Level Targeting
- mapped network drives
- user restrictions
- account lockout policies
- PowerShell fundamentals
- `Get-ADUser`
- `Get-ADGroup`
- `Get-ADGroupMember`
- new-starter administration
- Group Policy Result / RSOP
- Wireshark
- ICMP
- ARP
- DNS packet analysis
- TCP three-way handshake
- SMB2
- DHCP DORA
- Hyper-V virtual switching
- virtual NIC troubleshooting
- MAC-address troubleshooting
- Jira Service Management
- first-line support workflow
- incident investigation
- root-cause analysis
- ticket documentation

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
- [x] Group Policy Preferences
- [x] Item-Level Targeting
- [x] mapped departmental drives
- [x] HR user restriction GPO
- [x] domain account lockout policy
- [x] DHCP
- [x] DHCP client lease testing
- [x] DNS fault troubleshooting
- [x] PowerShell Active Directory queries
- [x] new-starter workflow
- [x] Wireshark packet analysis
- [x] ICMP capture
- [x] ARP capture
- [x] DNS capture
- [x] TCP handshake capture
- [x] SMB2 capture
- [x] DHCP DORA capture
- [x] Jira Service Management workflow
- [x] Alex account-lockout ticket
- [x] Sophie missing-drive ticket
- [x] Hyper-V network fault investigation
- [ ] Sam Wilson DNS Jira scenario
- [ ] final Mia/new-starter Jira documentation
- [ ] final README cleanup

---

## Next Steps

Before marking this Windows homelab as **v1.0 complete**, I plan to:

- complete the selected Sam Wilson DNS support ticket
- finish the final Jira documentation for the Mia Turner new-starter scenario
- replace the remaining image placeholders in this README with the actual GitHub-uploaded image links
- carry out one final README cleanup

After v1.0, my main lab focus will move back to **CCNA networking and Cisco Packet Tracer**.

# Enterprise IT Support Homelab

I built this lab in Hyper-V to get practical experience with Windows Server, Active Directory, DNS, domain-joined clients, file sharing and troubleshooting.

Rather than just installing Windows Server, I wanted to build something closer to a small company environment with IT, Finance, HR and Sales departments and then test it from a normal Windows client.
## At a Glance

**Completed:** October 2026 · **Focus:** Windows enterprise IT support and networking fundamentals

| Area | What I worked with |
|---|---|
| Infrastructure | Hyper-V, Windows Server 2025, Windows 11 and the `corp.example.com` Active Directory domain |
| Administration | OUs, users, security groups, PowerShell, Group Policy, mapped drives and new-starter onboarding |
| Core services | DNS, DHCP, SMB file shares and Share/NTFS permissions |
| Networking | Wireshark analysis of ICMP, ARP, DNS, TCP, SMB2 and DHCP DORA |
| Troubleshooting | DNS misconfiguration, account lockout, missing group membership and an unexpected Hyper-V virtual NIC/MAC issue |
| Service desk | Four Jira scenarios taken through investigation, fix, verification and resolution |

<sub>The sections below contain the configuration, testing and troubleshooting evidence from the completed lab.</sub>
---

## Lab Environment

| Device | Role | IP Address |
|---|---|---|
| Hyper-V Host | Lab gateway / NAT | 10.10.10.1 |
| DC01 | Windows Server 2025 / Domain Controller / DNS | 10.10.10.10 |
| CLIENT01 | Windows 11 domain workstation | DHCP - normally 10.10.10.100 |

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

I used Group Policy to manage departmental drives, user restrictions and domain account lockout settings centrally, then tested the results from CLIENT01.

### Department Drive Mapping

`GPO - Department Drive Maps` uses Group Policy Preferences with the **Update** action. Each mapping targets the corresponding departmental security group.

| Department | Drive | Share | Target group |
|---|---|---|---|
| Finance | F: | `\\DC01\Finance` | `GG-Finance` |
| HR | H: | `\\DC01\HR` | `GG-HR` |
| IT | I: | `\\DC01\IT` | `GG-IT` |
| Sales | S: | `\\DC01\Sales` | `GG-Sales` |

The evidence below shows the Finance mapping, a populated `CORP\GG-IT` targeting condition, all four mappings, and the GPO linked to the Company OU.

<!-- Evidence E109: Uploaded filename preserved in pasted README: Screenshot 2026-09-15 201531.png
Visible: Finance F drive, Update, \DC01\Finance. -->
![Finance F drive, Update, \DC01\Finance](<evidence/Screenshot 2026-09-15 201531.png>)

<!-- Evidence E124: Local original: Screenshot 2026-09-16 194333.png. Original chat upload filename not established.
Visible: Targeting Editor CORP\GG-IT selected. -->
![Targeting Editor CORP\GG-IT selected](<evidence/Screenshot 2026-09-16 194333.png>)

<!-- Evidence E128: Uploaded filename preserved in pasted README: Screenshot 2026-09-16 195658.png
Visible: All four departmental drive mappings. -->
![All four departmental drive mappings](<evidence/Screenshot 2026-09-16 195658.png>)

<!-- Evidence E107: Uploaded filename preserved in pasted README: Screenshot 2026-09-13 121117.png
Visible: Company OU drive-map GPO link enabled. -->
![Company OU drive-map GPO link enabled](<evidence/Screenshot 2026-09-13 121117.png>)

I refreshed policy with `gpupdate /force` and verified that an authorised Finance user received the F: drive in File Explorer.

<!-- Evidence E123: Uploaded filename preserved in pasted README: Screenshot 2026-09-16 193016.png
Visible: gpupdate succeeds for computer and user. -->
![gpupdate succeeds for computer and user](<evidence/Screenshot 2026-09-16 193016.png>)

<!-- Evidence E114: Uploaded filename preserved in pasted README: Screenshot 2026-09-15 202947.png
Visible: Finance F drive visible. -->
![Finance F drive visible](<evidence/Screenshot 2026-09-15 202947.png>)

### User Restriction GPO

I linked `GPO - User Restriction` to the HR OU and enabled **Prohibit access to Control Panel and PC settings**. Testing from the client produced the Windows restrictions message, confirming that the restriction applied.

<!-- Evidence E136: Uploaded filename preserved in pasted README: Screenshot 2026-09-17 195806.png
Visible: HR restriction link and domain account policy in GPMC. -->
![HR restriction link and domain account policy in GPMC](<evidence/Screenshot 2026-09-17 195806.png>)

<!-- Evidence E130: Uploaded filename preserved in pasted README: Screenshot 2026-09-17 190443.png
Visible: Prohibit Control Panel and PC settings enabled. -->
![Prohibit Control Panel and PC settings enabled](<evidence/Screenshot 2026-09-17 190443.png>)

<!-- Evidence E131: Uploaded filename preserved in pasted README: Screenshot 2026-09-17 193730.png
Visible: Windows restriction message. -->
![Windows restriction message](<evidence/Screenshot 2026-09-17 193730.png>)

### Domain Account Lockout Policy

The final domain policy locks an account after **5 invalid logon attempts**, with a **15-minute lockout** and **15-minute reset counter**.

The first effective-policy check did not match the intended settings. After correcting policy precedence, `net accounts /domain` confirmed the effective values of **5 / 15 / 15**. I also reproduced an actual lockout, unlocked the account in Active Directory and verified sign-in. The later Alex Jira scenario documents this as a completed support incident.

<!-- Evidence E133: Uploaded filename preserved in pasted README: Screenshot 2026-09-17 195307.png
Visible: Lockout policy 5 attempts, 15 minute duration and reset. -->
![Lockout policy 5 attempts, 15 minute duration and reset](<evidence/Screenshot 2026-09-17 195307.png>)

<!-- Evidence E137: Uploaded filename preserved in pasted README: Screenshot 2026-09-17 200232.png
Visible: Effective lockout policy corrected to 5/15/15. -->
![Effective lockout policy corrected to 5/15/15](<evidence/Screenshot 2026-09-17 200232.png>)

<!-- Evidence E140: Local original: Screenshot 2026-09-19 110318.png. Original chat upload filename not established.
Visible: Alex actual lockout during initial policy test. -->
![Alex actual lockout during initial policy test](<evidence/Screenshot 2026-09-19 110318.png>)

<!-- Evidence E142: Local original: Screenshot 2026-09-19 110444.png. Original chat upload filename not established.
Visible: Unlock account selected. -->
![Unlock account selected](<evidence/Screenshot 2026-09-19 110444.png>)

<!-- Evidence E143: Local original: Screenshot 2026-09-19 110738.png. Original chat upload filename not established.
Visible: Alex Welcome after initial policy test. -->
![Alex Welcome after initial policy test](<evidence/Screenshot 2026-09-19 110738.png>)

## DHCP and Client Networking

I installed and authorised DHCP on DC01 and configured the `10.10.10.0/24` lab scope.

| Setting | Completed configuration |
|---|---|
| Address pool | `10.10.10.100–10.10.10.200` |
| Subnet mask | `255.255.255.0` |
| 003 Router | `10.10.10.1` |
| 006 DNS Servers | `10.10.10.10` |
| 015 DNS Domain Name | `corp.example.com` |

CLIENT01 moved from its original static address to DHCP and normally received **10.10.10.100**. I checked the lease in DHCP Manager and verified the address, gateway, DHCP server and internal DNS server with `ipconfig /all`.

<!-- Evidence E152: Local original: Screenshot 2026-09-19 114821.png. Original chat upload filename not established.
Visible: Scope wizard corrected /24 pool .100-.200. -->
![Scope wizard corrected /24 pool .100-.200](<evidence/Screenshot 2026-09-19 114821.png>)

<!-- Evidence E159: Uploaded filename preserved in pasted README: Screenshot 2026-09-19 120055.png
Visible: Final DHCP address pool. -->
![Final DHCP address pool](<evidence/Screenshot 2026-09-19 120055.png>)

<!-- Evidence E158: Uploaded filename preserved in pasted README: Screenshot 2026-09-19 120033.png
Visible: Final router, DNS and domain scope options. -->
![Final router, DNS and domain scope options](<evidence/Screenshot 2026-09-19 120033.png>)

<!-- Evidence E172: Uploaded filename preserved in pasted README: Screenshot 2026-09-21 194649.png
Visible: CLIENT01 DHCP lease 10.10.10.100. -->
![CLIENT01 DHCP lease 10.10.10.100](<evidence/Screenshot 2026-09-21 194649.png>)

<!-- Evidence E178: Uploaded filename preserved in pasted README: Screenshot 2026-09-21 195852.png
Visible: Client DHCP configuration including DNS .10. -->
![Client DHCP configuration including DNS .10](<evidence/Screenshot 2026-09-21 195852.png>)

## PowerShell Administration

I used PowerShell alongside the management consoles to inspect domain users, groups, membership and account state.

```powershell
Get-ADUser -Filter *
Get-ADGroup -Filter *
Get-ADGroupMember "GG-IT"
Get-ADGroupMember "GG-Finance"
Get-ADUser "alex.morgan"
```

The results showed Alex Morgan and Sam Wilson in `GG-IT`, Emily Clarke and James Hall in `GG-Finance`, and Alex's enabled account in the IT OU.

<!-- Evidence E173: Uploaded filename preserved in pasted README: Screenshot 2026-09-21 195017.png
Visible: Get-ADUser -Filter *. -->
![Get-ADUser -Filter *](<evidence/Screenshot 2026-09-21 195017.png>)

<!-- Evidence E174: Uploaded filename preserved in pasted README: Screenshot 2026-09-21 195144.png
Visible: Get-ADGroup -Filter *. -->
![Get-ADGroup -Filter *](<evidence/Screenshot 2026-09-21 195144.png>)

<!-- Evidence E175: Uploaded filename preserved in pasted README: Screenshot 2026-09-21 195248.png
Visible: GG-IT members Alex and Sam. -->
![GG-IT members Alex and Sam](<evidence/Screenshot 2026-09-21 195248.png>)

<!-- Evidence E176: Uploaded filename preserved in pasted README: Screenshot 2026-09-21 195355.png
Visible: GG-Finance members Emily and James. -->
![GG-Finance members Emily and James](<evidence/Screenshot 2026-09-21 195355.png>)

<!-- Evidence E177: Uploaded filename preserved in pasted README: Screenshot 2026-09-21 195654.png
Visible: Alex account enabled and in IT OU. -->
![Alex account enabled and in IT OU](<evidence/Screenshot 2026-09-21 195654.png>)

## DNS Troubleshooting

I deliberately configured CLIENT01 to use `8.8.8.8` instead of the lab DNS server `10.10.10.10`.

The client could still ping DC01 by IP, but its private-domain queries went to the public resolver. `ipconfig /all` exposed the incorrect DNS setting even though the IP address had come from the correct DHCP server.

<!-- Evidence E179: Chat upload: image.png; source context: 21 September, around 20:01 2026. Matched local original: Screenshot 2026-09-21 200109.png
Visible: Fault injection: DNS manually 8.8.8.8. -->
![Fault injection: DNS manually 8.8.8.8](<evidence/Screenshot 2026-09-21 200109.png>)

<!-- Evidence E180: Chat upload: image.png; source context: 23 September, around 18:21 2026. Matched local original: Screenshot 2026-09-23 182133.png
Visible: Ping DC01 succeeds; DNS query goes to dns.google. -->
![Ping DC01 succeeds; DNS query goes to dns.google](<evidence/Screenshot 2026-09-23 182133.png>)

<!-- Evidence E182: Chat upload: image.png; source context: 23 September, around 18:33 2026. Matched local original: Screenshot 2026-09-23 183307.png
Visible: Client DNS 8.8.8.8 despite DHCP IP. -->
![Client DNS 8.8.8.8 despite DHCP IP](<evidence/Screenshot 2026-09-23 183307.png>)

I corrected the client to use DC01 for DNS and verified that `dc01.corp.example.com` resolved to **10.10.10.10**. The September troubleshooting included a direct internal-DNS setting; the final Sam ticket records restoring automatic DNS through DHCP.

The before/after output below demonstrates the resolver change and successful private-name resolution. It also shows why a successful ping alone does not establish that domain DNS is working.

<!-- Evidence E187: Chat upload: image.png; source context: 23 September, around 18:58 2026. Matched local original: Screenshot 2026-09-23 185736.png
Visible: Before/after DNS resolver and successful private A answer. -->
![Before/after DNS resolver and successful private A answer](<evidence/Screenshot 2026-09-23 185736.png>)

## New Starter Administration — Mia Turner

I completed an onboarding workflow for the fictional Sales user **Mia Turner**. I created the account with PowerShell, moved it into the Sales OU, set its password securely, enabled it and added the required Sales membership.

```powershell
New-ADUser -Name "Mia Turner" -SamAccountName "mia.turner" `
    -UserPrincipalName "mia.turner@corp.example.com"
Get-ADUser "mia.turner" | Move-ADObject `
    -TargetPath "OU=Sales,OU=Company,DC=corp,DC=example,DC=com"
```

The verification screenshots show the completed account state: **Enabled = True**, the **Sales OU**, and membership of **GG-Sales**.

<!-- Evidence E189: Chat upload: image.png; source context: 24 September, around 12:31 2026. Matched local original: Screenshot 2026-09-24 123051.png
Visible: New-ADUser creates Mia. -->
![New-ADUser creates Mia](<evidence/Screenshot 2026-09-24 123051.png>)

<!-- Evidence E194: Chat upload: image.png; source context: 24 September, around 13:03 2026. Matched local original: Screenshot 2026-09-24 130315.png
Visible: Mia enabled in Sales OU. -->
![Mia enabled in Sales OU](<evidence/Screenshot 2026-09-24 130315.png>)

<!-- Evidence E195: Chat upload: image.png; source context: 24 September, around 13:10 2026. Matched local original: Screenshot 2026-09-24 131009.png
Visible: GG-Sales membership includes Mia. -->
![GG-Sales membership includes Mia](<evidence/Screenshot 2026-09-24 131009.png>)

I checked the actual user's context with `gpresult /r`, confirmed Sales membership in the user token and verified the mapped Sales S: drive. The early group-result screenshot shows `GG-Sales` but no applied user GPO at that point; the working drive is the later outcome evidence.

<!-- Evidence E201: Chat upload: image.png; source context: 24 September, around 14:01 2026. Matched local original: Screenshot 2026-09-24 140143.png
Visible: gpresult identifies Mia and Sales OU. -->
![gpresult identifies Mia and Sales OU](<evidence/Screenshot 2026-09-24 140143.png>)

<!-- Evidence E202: Chat upload: image.png; source context: 24 September, around 14:05 2026. Matched local original: Screenshot 2026-09-24 140453.png
Visible: Mia token includes GG-Sales; Applied GPO list is N/A at this point. -->
![Mia token includes GG-Sales; Applied GPO list is N/A at this point](<evidence/Screenshot 2026-09-24 140453.png>)

<!-- Evidence E205: Chat upload: image.png; source context: 24 September, around 14:13 2026. Matched local original: Screenshot 2026-09-24 141318.png
Visible: Mia Sales S drive working. -->
![Mia Sales S drive working](<evidence/Screenshot 2026-09-24 141318.png>)

The final October verification and completed Jira request are documented below under Mia's support scenario.

## Wireshark Basics — Completed

I completed packet-capture exercises for **ICMP, ARP, DNS, the TCP three-way handshake, SMB2 and DHCP DORA**, using traffic generated in the lab.

| Exercise | Display filter | What I identified |
|---|---|---|
| ICMP | `icmp` | Echo requests and replies between CLIENT01 and DC01 |
| ARP | `arp` | IPv4-to-MAC address resolution |
| DNS | `dns.qry.name contains "dc01"` | Private hostname query and A-record answer |
| TCP | `tcp.port == 445` | SYN → SYN/ACK → ACK |
| SMB2 | `smb2` | Session and file-resource operations |
| DHCP | `dhcp` | Discover → Offer → Request → ACK |

### ICMP

The capture shows echo requests from `10.10.10.100` to `10.10.10.10` and replies in the opposite direction.

<!-- Evidence E217: Chat upload: image.png; source context: 26 September, around 09:14 2026. Matched local original: Screenshot 2026-09-26 091350.png
Visible: ICMP echo request/reply between .100 and .10. -->
![ICMP echo request/reply between .100 and .10](<evidence/Screenshot 2026-09-26 091350.png>)

### ARP

I identified the request **Who has 10.10.10.100? Tell 10.10.10.10** and the response identifying CLIENT01 as **00:15:5d:01:68:02**.

<!-- Evidence E219: Chat upload: image.png; source context: 26 September, around 09:24 2026. Matched local original: Screenshot 2026-09-26 092420.png
Visible: ARP request/reply identifying client MAC. -->
![ARP request/reply identifying client MAC](<evidence/Screenshot 2026-09-26 092420.png>)

### DNS

Filtering for `dc01` isolated the lookup of `dc01.corp.example.com` and the response containing **A 10.10.10.10**.

<!-- Evidence E221: Chat upload: image.png; source context: 27 September, around 10:26 2026. Matched local original: Screenshot 2026-09-27 102527.png
Visible: Filtered dc01 DNS query and A response. -->
![Filtered dc01 DNS query and A response](<evidence/Screenshot 2026-09-27 102527.png>)

### TCP Three-Way Handshake

Packets **2673–2675** show the complete handshake between client port **52581** and DC01 port **445**: SYN, SYN/ACK, then ACK. This separates transport connection establishment from the SMB operations that follow.

<!-- Evidence E222: Chat upload: image.png; source context: 27 September, around 10:42 2026. Matched local original: Screenshot 2026-09-27 104139.png
Visible: TCP 445 three-way handshake packets 2673-2675. -->
![TCP 445 three-way handshake packets 2673-2675](<evidence/Screenshot 2026-09-27 104139.png>)

### SMB2

I inspected SMB2 negotiation, session setup, tree connection and resource operations. The focused capture shows Create Request/Response and Close Request/Response pairs; the wider capture adds session context.

<!-- Evidence E224: Local original: Screenshot 2026-09-27 104800.png. Original chat upload filename not established.
Visible: SMB2 negotiate, session setup and tree connect. -->
![SMB2 negotiate, session setup and tree connect](<evidence/Screenshot 2026-09-27 104800.png>)

<!-- Evidence E226: Chat upload: image.png; source context: 27 September, around 10:52 2026. Matched local original: Screenshot 2026-09-27 105151.png
Visible: SMB2 Create and Close request/response. -->
![SMB2 Create and Close request/response](<evidence/Screenshot 2026-09-27 105151.png>)

### DHCP DORA

I released and renewed CLIENT01's lease while capturing. Following the Release, packets **98–101** show the full Discover, Offer, Request and ACK exchange with a shared transaction ID. The Discover uses `0.0.0.0` and the broadcast destination `255.255.255.255`.

<!-- Evidence E227: Chat upload: image.png; source context: 27 September, around 10:56 2026. Matched local original: Screenshot 2026-09-27 105539.png
Visible: DHCP Release followed by complete DORA. -->
![DHCP Release followed by complete DORA](<evidence/Screenshot 2026-09-27 105539.png>)

## Unexpected Hyper-V Network Fault — Resolved

While preparing the support scenarios, CLIENT01 unexpectedly lost communication with DC01. This was a real lab fault encountered during the work.

### Symptoms and Checks

`whoami` identified `CORP\alex.morgan`, but `nltest /sc_query:corp.example.com` returned **ERROR_NO_LOGON_SERVERS** and DNS requests to DC01 timed out. An existing session and a `LOGONSERVER` value were not enough to prove that the domain controller was currently reachable.

<!-- Evidence E245: Chat upload: image.png; source context: 29 September, around 19:06 2026. Matched local original: Screenshot 2026-09-29 190603.png
Visible: Logon server variable plus ERROR_NO_LOGON_SERVERS. -->
![Logon server variable plus ERROR_NO_LOGON_SERVERS](<evidence/Screenshot 2026-09-29 190603.png>)

<!-- Evidence E246: Chat upload: image.png; source context: 29 September, around 19:08 2026. Matched local original: Screenshot 2026-09-29 190753.png
Visible: DC01 DNS lookup times out. -->
![DC01 DNS lookup times out](<evidence/Screenshot 2026-09-29 190753.png>)

Both machines still had their expected IPv4 settings. DC01's DNS, KDC and Netlogon services were running, and DNS had UDP 53 listeners.

<!-- Evidence E247: Chat upload: image.png; source context: 29 September, around 19:10, DC01 2026. Matched local original: Screenshot 2026-09-29 190929.png
Visible: DC01 correct static IP and guest MAC. -->
![DC01 correct static IP and guest MAC](<evidence/Screenshot 2026-09-29 190929.png>)

<!-- Evidence E248: Chat upload: image.png; source context: 29 September, around 19:10, CLIENT01 2026. Matched local original: Screenshot 2026-09-29 191001.png
Visible: CLIENT01 correct DHCP IP and internal DNS. -->
![CLIENT01 correct DHCP IP and internal DNS](<evidence/Screenshot 2026-09-29 191001.png>)

<!-- Evidence E249: Chat upload: image.png; source context: 29 September, around 19:11 2026. Matched local original: Screenshot 2026-09-29 191052.png
Visible: DNS, KDC, Netlogon running; UDP 53 listeners. -->
![DNS, KDC, Netlogon running; UDP 53 listeners](<evidence/Screenshot 2026-09-29 191052.png>)

A failed ping alongside a DC01 entry in the client's ARP table prompted closer inspection of the virtual adapters. The cache entry was useful evidence to compare; it did not by itself prove working end-to-end communication.

<!-- Evidence E252: Chat upload: image.png; source context: 29 September, around 19:46 2026. Matched local original: Screenshot 2026-09-29 194609.png
Visible: Ping fails while ARP table contains DC01 entry. -->
![Ping fails while ARP table contains DC01 entry](<evidence/Screenshot 2026-09-29 194609.png>)

### Adapter Mismatch and Recovery

The host reported both VMs on `LabSwitch`, with untagged VLAN settings and no adapter isolation. However, the reported DC01 MAC addresses differed:

| View | DC01 MAC address |
|---|---|
| Hyper-V host | `00-15-5D-01-68-03` |
| Inside DC01 | `00-15-5D-01-68-00` |

This mismatch was the key clue in the adapter investigation.

<!-- Evidence E257: Chat upload: image.png; source context: 29 September, around 19:59 2026. Matched local original: Screenshot 2026-09-29 195825.png
Visible: Host VM MAC, VLAN and isolation diagnostics. -->
![Host VM MAC, VLAN and isolation diagnostics](<evidence/Screenshot 2026-09-29 195825.png>)

<!-- Evidence E259: Chat upload: image.png; source context: 29 September, around 20:04 2026. Matched local original: Screenshot 2026-09-29 200340.png
Visible: DC01 guest MAC ends 6800. -->
![DC01 guest MAC ends 6800](<evidence/Screenshot 2026-09-29 200340.png>)

I shut down DC01 and configured a consistent static virtual-adapter MAC matching the expected guest address. Communication returned, allowing DNS, SMB and domain-authentication testing to continue.

This exercise reinforced checking IPv4 settings, services, ARP entries and virtual-switch configuration before treating a DNS or sign-in symptom as the root cause. The screenshots document the investigation; the recovery outcome was confirmed during the lab work.

## Jira Service Management — Completed Support Scenarios

I used **Enterprise IT Service Desk** in Jira Service Management to practise a complete support workflow: record the reported symptom, investigate, apply a fix, verify the user outcome and document the resolution.

The four scenarios selected for this v1.0 portfolio are complete:

| User | Scenario | Completed outcome |
|---|---|---|
| Alex Morgan | Account lockout | Account unlocked; sign-in verified; ticket completed |
| Sophie Brown | Missing Sales S: drive | Sales membership restored; fresh sign-in; drive restored |
| Sam Wilson | Cannot resolve DC01 by internal name | Internal DNS restored through DHCP; name resolution verified; ticket completed |
| Mia Turner | New-starter access verification | Enabled Sales account, correct group, sign-in and S: drive verified; request resolved |

### Alex Morgan — Account Lockout

I reproduced a genuine domain lockout. Windows displayed **The referenced account is currently locked out and may not be logged on to**, and ADUC confirmed that the account was currently locked on the domain controller.

<!-- Evidence E265: Chat upload: image.png; source context: 30 September, around 19:54 2026. Matched local original: Screenshot 2026-09-30 185327.png
Visible: Alex genuine locked-out sign-in message. -->
![Alex genuine locked-out sign-in message](<evidence/Screenshot 2026-09-30 185327.png>)

<!-- Evidence E266: Chat upload: image.png; source context: 30 September, around 19:55 2026. Matched local original: Screenshot 2026-09-30 185402.png
Visible: ADUC confirms current lockout; unlock selected. -->
![ADUC confirms current lockout; unlock selected](<evidence/Screenshot 2026-09-30 185402.png>)

I unlocked Alex's account and verified successful sign-in to CLIENT01. The Jira resolution note records the fix and verification; the final ticket list shows **ITSD-1 — Completed / Done**.

<!-- Evidence E272: Uploaded filename preserved in pasted README: Screenshot 2026-09-30 192243.png
Visible: Alex resolution Done and internal note. -->
![Alex resolution Done and internal note](<evidence/Screenshot 2026-09-30 192243.png>)

<!-- Evidence E273: Chat upload: image.png; source context: 30 September, around 20:24 2026. Matched local original: Screenshot 2026-09-30 192328.png
Visible: Alex ITSD-1 Completed / Done. -->
![Alex ITSD-1 Completed / Done](<evidence/Screenshot 2026-09-30 192328.png>)

### Sophie Brown — Missing Sales Drive

Sophie's Sales S: drive was missing. Her current token lacked `GG-Sales`, and ADUC showed only `Domain Users`, explaining why she did not meet the group condition for the Sales mapping.

<!-- Evidence E275: Chat upload: image.png; source context: 1 October, around 19:18 2026. Matched local original: Screenshot 2026-10-01 191750.png
Visible: Sophie Sales drive absent. -->
![Sophie Sales drive absent](<evidence/Screenshot 2026-10-01 191750.png>)

<!-- Evidence E277: Chat upload: image.png; source context: 1 October, around 19:22 2026. Matched local original: Screenshot 2026-10-01 192123.png
Visible: Filtered token query returns no GG-Sales. -->
![Filtered token query returns no GG-Sales](<evidence/Screenshot 2026-10-01 192123.png>)

<!-- Evidence E278: Chat upload: image.png; source context: 1 October, around 19:24 2026. Matched local original: Screenshot 2026-10-01 192353.png
Visible: Sophie AD membership Domain Users only. -->
![Sophie AD membership Domain Users only](<evidence/Screenshot 2026-10-01 192353.png>)

I restored `GG-Sales` membership and checked the updated account. The existing session still lacked the group, so I fully signed Sophie out and back in to obtain a fresh security token. Refreshing Group Policy alone did not replace that token.

<!-- Evidence E280: Uploaded filename preserved in pasted README: Screenshot 2026-10-01 192441.png
Visible: Add Sophie to GG-Sales. -->
![Add Sophie to GG-Sales](<evidence/Screenshot 2026-10-01 192441.png>)

<!-- Evidence E281: Uploaded filename preserved in pasted README: Screenshot 2026-10-01 192449.png
Visible: Sophie AD membership corrected. -->
![Sophie AD membership corrected](<evidence/Screenshot 2026-10-01 192449.png>)

<!-- Evidence E283: Local original: Screenshot 2026-10-01 193033.png. Original chat upload filename not established.
Visible: Token still lacks GG-Sales before fresh sign-in. -->
![Token still lacks GG-Sales before fresh sign-in](<evidence/Screenshot 2026-10-01 193033.png>)

The drive-map policy was present in `gpresult /r`, and the final client check showed **Sales (\\DC01) (S:)** restored. The support scenario was completed with the verified user outcome.

<!-- Evidence E284: Chat upload: image.png; source context: 1 October, around 19:32 2026. Matched local original: Screenshot 2026-10-01 193111.png
Visible: Sophie drive-map GPO applied. -->
![Sophie drive-map GPO applied](<evidence/Screenshot 2026-10-01 193111.png>)

<!-- Evidence E285: Chat upload: image.png; source context: 1 October, around 19:37 2026. Matched local original: Screenshot 2026-10-01 193604.png
Visible: Sophie Sales S drive restored. -->
![Sophie Sales S drive restored](<evidence/Screenshot 2026-10-01 193604.png>)

### Sam Wilson — Internal DNS Resolution

Sam's scenario used the reproduced client DNS fault documented above: CLIENT01 pointed to `8.8.8.8` instead of the internal DNS server. I restored automatic DNS configuration through DHCP, confirmed `10.10.10.10` as the DNS server and verified resolution of `dc01.corp.example.com`.

I documented the cause, correction and verification in Jira and completed the ticket. The screenshot below captures the resolution note with **Done** selected; the earlier DNS screenshots provide the technical before/after evidence.

<!-- Evidence E290: Chat upload: image.png; source context: 3 October, Sam resolution 2026. Matched local original: Screenshot 2026-10-03 093223.png
Visible: Sam DNS resolution note, Done selected. -->
![Sam DNS resolution note, Done selected](<evidence/Screenshot 2026-10-03 093223.png>)

### Mia Turner — Final New-Starter Verification

For **ITSD-4**, I checked Mia's existing account rather than creating a duplicate. Final verification confirmed:

- The account was **enabled** and located in the **Sales OU**.
- Membership included **GG-Sales**.
- Mia could sign in to CLIENT01 and access the **Sales S: drive**.

The October screenshots below show the group membership, PowerShell account check and working drive.

<!-- Evidence E294: Chat upload: image.png; source context: 3 October, Mia group verification 2026. Matched local original: Screenshot 2026-10-03 095918.png
Visible: Mia Member Of includes GG-Sales. -->
![Mia Member Of includes GG-Sales](<evidence/Screenshot 2026-10-03 095918.png>)

<!-- Evidence E295: Chat upload: image.png; source context: 3 October, Mia account verification 2026. Matched local original: Screenshot 2026-10-03 100359.png
Visible: Mia enabled and in Sales OU via PowerShell. -->
![Mia enabled and in Sales OU via PowerShell](<evidence/Screenshot 2026-10-03 100359.png>)

<!-- Evidence E296: Chat upload: image.png; source context: 3 October, Mia drive verification 2026. Matched local original: Screenshot 2026-10-03 100532.png
Visible: Mia Sales S drive visible in final verification. -->
![Mia Sales S drive visible in final verification](<evidence/Screenshot 2026-10-03 100532.png>)

I recorded the completed checks in Jira. The final saved ticket shows **Resolved / Done**, with the internal note confirming that the new starter's access was ready for use.

<!-- Evidence E298: Chat upload: image.png; source context: 3 October, final Mia resolved ticket 2026. Matched local original: Screenshot 2026-10-03 101527.png
Visible: Mia ITSD-4 Resolved / Done with saved internal note. -->
![Mia ITSD-4 Resolved / Done with saved internal note](<evidence/Screenshot 2026-10-03 101527.png>)

## Skills Demonstrated

- Hyper-V internal switching, NAT and virtual-adapter troubleshooting.
- Windows Server 2025, Active Directory, OUs, users and departmental security groups.
- Windows 11 domain membership and user-side verification.
- SMB shares, Share/NTFS permissions and authorised/unauthorised access tests.
- Group Policy Preferences, group targeting, restrictions and account lockout.
- DHCP scope configuration, leases and internal DNS troubleshooting.
- PowerShell account, group, service and network checks.
- New-starter onboarding and access verification.
- Wireshark analysis of ICMP, ARP, DNS, TCP, SMB2 and DHCP DORA.
- Jira incident documentation, troubleshooting, verification and resolution.

## Project Complete — v1.0

**Completed: 3 October 2026.**

The Enterprise IT Support Homelab v1.0 is complete: infrastructure, Active Directory, departmental access, Group Policy, DHCP/DNS, PowerShell administration, Mia's onboarding, all six Wireshark basics exercises, Hyper-V troubleshooting and the four selected Jira support scenarios.

This project demonstrates a complete build, test, troubleshoot and document cycle using a small Windows enterprise environment. My future study focus is **CCNA networking and Cisco Packet Tracer**.

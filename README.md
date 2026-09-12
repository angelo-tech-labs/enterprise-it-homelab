# Enterprise IT Support Homelab

A practical Windows IT support homelab built in Hyper-V to simulate a small business environment and develop hands-on Windows administration, Active Directory, networking and troubleshooting skills.

## Technologies

- Hyper-V
- Windows Server 2025
- Windows 11 Pro
- Active Directory Domain Services
- DNS
- SMB file sharing
- NTFS permissions
- PowerShell
- TCP/IP networking

---

## Lab Environment

| Device | Role | IP Address |
|---|---|---|
| DC01 | Domain Controller / DNS Server | 10.10.10.10 |
| CLIENT01 | Domain-joined Windows 11 workstation | 10.10.10.x |
| Hyper-V Host | Lab gateway / NAT | 10.10.10.1 |

**Domain:** `corp.example.com`

**Network:** `10.10.10.0/24`

---

# Active Directory

I created a fictional company structure in Active Directory with separate organisational units for:

- IT
- Finance
- HR
- Sales
- Computers
- Groups

Departmental Global Security Groups were also created:

- `GG-IT`
- `GG-Finance`
- `GG-HR`
- `GG-Sales`

Users were placed into their relevant department groups so access can be managed through groups rather than assigning permissions directly to individual users.


## Active Directory Structure

📸 **SCREENSHOT 1 GOES HERE**

**Use:** the screenshot showing the `Company` OU expanded with:

- Computers
- Finance
- Groups
- HR
- IT
- Sales

and the `GG-Finance`, `GG-HR`, `GG-IT`, `GG-Sales` groups visible.

**Do not use four separate screenshots for each department.**


<!-- Screenshot 1: Active Directory OU and group structure -->



## Security Group Membership

Users were assigned to departmental Global Security Groups.

For example, IT users were added to:

`GG-IT`

This makes administration easier because access is assigned to the group instead of directly to individual user accounts.


📸 **SCREENSHOT 2 GOES HERE**

**Use:** your screenshot showing:

`GG-IT Properties → Members`

with:

- Alex Morgan
- Sam Wilson

visible as members.


<!-- Screenshot 2: GG-IT group membership -->



---

# Departmental File Shares

I created departmental folders on DC01:

```text
C:\Company Shares\
├── Finance
├── HR
├── IT
└── Sales

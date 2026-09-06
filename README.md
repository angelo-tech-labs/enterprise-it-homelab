# Enterprise IT Support Homelab

A practical Windows IT support lab built in Hyper-V to simulate a small business environment and develop hands-on first-line IT support, Windows administration and networking skills.

## Technologies

- Hyper-V
- Windows Server 2025
- Windows 11 Pro
- Active Directory Domain Services (AD DS)
- DNS
- PowerShell
- TCP/IP networking

## Lab Environment

| Device | Role | IP Address |
|---|---|---|
| DC01 | Domain Controller / DNS Server | 10.10.10.10 |
| CLIENT01 | Windows 11 workstation | To be configured |
| Hyper-V Host | Lab gateway / NAT | 10.10.10.1 |

**Domain:** `corp.example.com`  
**Lab network:** `10.10.10.0/24`

## Active Directory Structure

Departments:

- IT
- Finance
- Sales
- HR

Security groups:

- GG-IT
- GG-Finance
- GG-Sales
- GG-HR

Test users have been created and assigned to their relevant departmental security groups.

## Current Progress

- [x] Created Hyper-V lab network
- [x] Installed Windows Server 2025
- [x] Installed Active Directory Domain Services
- [x] Created `corp.example.com`
- [x] Created organisational units
- [x] Created users and security groups
- [x] Installed Windows 11 Pro CLIENT01
- [ ] Configure CLIENT01 networking
- [ ] Join CLIENT01 to the domain
- [ ] Configure file shares and permissions
- [ ] Configure Group Policy
- [ ] Configure DHCP
- [ ] Create troubleshooting scenarios
- [ ] Document first-line support tickets

## Project Goals

This lab will be used to practise:

- Active Directory user and group administration
- Windows domain joining
- Password resets and account lockouts
- New starter and leaver processes
- NTFS and shared-folder permissions
- Group Policy
- DNS and DHCP troubleshooting
- Basic TCP/IP troubleshooting
- PowerShell administration
- Realistic first-line IT support scenarios

# Configuring-Windows-Active-Directory-homelab
A self-contained lab for building, configuring, and securing a Windows Active Directory environment from scratch. Built to practice real-world sysadmin and security-analyst skills — domain setup, identity management, Group Policy, and common AD attack/defense scenarios — without touching production systems.  


# create directories
mkdir -p docs scripts/powerShell .github/workflows

# README.md
cat > README.md <<'EOF'
# Configuring a Windows Active Directory Home Lab

A self-contained lab for building, configuring, and securing a Windows Active Directory environment from scratch. Use this lab to practice real-world sysadmin and security-analyst skills — domain setup, identity management, Group Policy, logging, and common AD attack/defense scenarios — in an isolated environment.

Status: Documentation-first — the primary lab guide is in the PDF. This repo is a place to store a Markdown TOC, quickstart scripts, and automation in the future.

Quick links
- Lab guide (PDF): Configuring an Active Directory Home Lab.pdf

TL;DR (what you'll build)
- 1 Domain Controller (Windows Server)
- Several member workstations (Windows 10/11)
- Optional attacker host (Kali/Windows)
- Optional logging host (Sysmon/WEF → SIEM)
- DNS + AD DS, example OUs and GPOs for hardening and logging

Learning objectives
- Install and promote a Windows Server to an AD Domain Controller
- Configure DNS, time sync, and basic services for AD
- Design OUs and Group Policy: password policy, LAPS, auditing
- Implement logging (Sysmon + Windows Event Forwarding) and simple detections
- Run defensive and offensive exercises (Kerberoast, AS-REP, DCSync) and map detection artifacts

Prerequisites
- A host machine capable of running multiple VMs (Hyper-V, VMware, or VirtualBox)
- Windows Server ISO (2019/2022 recommended) and Windows client ISOs
- Sufficient CPU/RAM/disk to host VMs (recommend: 16+ GB RAM, 4+ CPU cores)
- Network isolation for safety (host-only, NAT, or isolated VLAN)

Quickstart
1. Clone the repo:
   git clone https://github.com/MK-ultra98-creator/Configuring-Windows-Active-Directory-homelab.git
2. Open the lab guide:
   - On macOS: open "Configuring an Active Directory Home Lab.pdf"
   - On Linux: xdg-open "Configuring an Active Directory Home Lab.pdf"
   - On Windows: double-click the PDF or use `start` in cmd
3. Follow the PDF’s "Topology" and "Prerequisites" sections to create your VMs.
4. Optional: Check the `docs/TOC.md` and `scripts/powerShell/` for copy/paste PowerShell automation and quick reference.

Repository roadmap (suggested)
- /docs
  - TOC.md (extracted table of contents)
  - quickstart.md (copy/paste steps)
- /scripts/powerShell
  - install-domain.ps1 (sample AD install commands)
  - create-sample-users.ps1
- Add LICENSE (MIT) and CONTRIBUTING.md

Contributing
- PRs welcome: open an issue first if you'd like to add automation or exercises.
- If you submit scripts, please include:
  - short description
  - tested OS/versions
  - idempotency notes and safety disclaimers

License
- This repository currently has an MIT license file added.

Contact
- Maintainer: MK-ultra98-creator (GitHub)
EOF

# LICENSE
cat > LICENSE <<'EOF'
MIT License

Copyright (c) 2026 MK-ultra98-creator

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
EOF

# .gitignore
cat > .gitignore <<'EOF'
# Ignore common VM and temp files

# VirtualBox
*.vbox
*.vdi
*.vbox-prev

# Vagrant
.vagrant/

# VMware
*.vmx
*.vmdk
*.nvram
*.vmem

# Hyper-V
Virtual Hard Disks/

# ISOs and installers
*.iso
*.img
*.ova

# Logs and runtime
*.log
*.tmp
*.temp

# Windows
Thumbs.db
desktop.ini

# macOS
.DS_Store

# PowerShell temp files
*.ps1xml

# Docs
*.pdf

# Node (if added later)
node_modules/

# Misc
.env
EOF

# docs/TOC.md
cat > docs/TOC.md <<'EOF'
# Approximate Table of Contents

This file is an extracted/approximated table of contents for the PDF "Configuring an Active Directory Home Lab.pdf". It was created to provide a quick, searchable TOC inside the repository. If you need an exact page-by-page TOC, I can extract it verbatim from the PDF and update this file.

1. Introduction
   - Learning objectives
   - Lab safety and ethics

2. Lab Overview
   - Topology diagram
   - Host and VM roles
   - Network isolation and addressing

3. Required Software & Downloads
   - Windows Server (ISO recommendations)
   - Windows client images
   - Hypervisor options (Hyper-V, VMware, VirtualBox)

4. VM Inventory & Sizing
   - Domain Controller (DC)
   - Member servers (file server, SQL, jump box)
   - Client workstations
   - Optional attacker and logging hosts

5. VM Deployment
   - Host network setup (virtual switches)
   - VM creation steps
   - Disk, CPU, memory recommendations

6. Domain Installation (AD DS)
   - Install AD DS role
   - Promote server to Domain Controller (Install-ADDSForest)
   - DNS configuration and time sync

7. Post-install Configuration
   - OU structure and sample accounts
   - Service accounts and delegation
   - Backup and snapshot guidance

8. Group Policy
   - Password and account policies (Default Domain Policy)
   - Kerberos settings
   - LAPS deployment
   - Audit policy and advanced auditing
   - Deploying Sysmon and log forwarding via GPO

9. Hardening & Monitoring
   - Patch management suggestions
   - Windows Firewall and SMB hardening
   - Sysmon configuration and WEF (Windows Event Forwarding)
   - SIEM ingestion recommendations

10. Offensive Exercises
    - Enumeration techniques (PowerView, BloodHound)
    - Credential theft: Kerberoast, AS-REP roast
    - Lateral movement: Pass-the-Hash, SMB/WinRM techniques
    - Privilege escalation and DCSync

11. Defensive Exercises
    - Detection rules and hunt tasks
    - Playbooks for containment & response
    - GPO mitigations and recommended controls

12. Cleanup & Maintenance
    - Reverting snapshots
    - Safe lab teardown

13. Appendices
    - PowerShell snippets
    - Useful tools (Impacket, Rubeus, Mimikatz, BloodHound)
    - References and further reading
EOF

# scripts/powerShell/install-domain.ps1
cat > scripts/powerShell/install-domain.ps1 <<'EOF'
<# 
.SYNOPSIS
Example PowerShell snippets to assist in building an AD lab. These are meant as starting points and SHOULD be reviewed before running in any environment. Run on isolated test VMs only.
#>

<# ---------------------------------------------
  install-domain.ps1
  Promote a Windows Server to a new AD forest/domain.
  Run on the server you want to become the Domain Controller.
  This script is an example — interactive prompts or secure handling of passwords
  should be used in production. Always test in a disposable VM.
---------------------------------------------#>

Param(
    [string]$DomainName = "corp.local",
    [string]$NetbiosName = "CORP",
    [string]$SafeModePassword = "P@ssw0rd"
)

# 1) Install AD DS role and management tools
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# 2) Promote to a new forest (this will reboot the server)
# Convert the plain password to a secure string — replace with secure handling.
$secPass = ConvertTo-SecureString $SafeModePassword -AsPlainText -Force

# The Install-ADDSForest cmdlet will make this server a Domain Controller.
# NOTE: This is potentially destructive if run in the wrong environment.
Install-ADDSForest -DomainName $DomainName -DomainNetbiosName $NetbiosName -SafeModeAdministratorPassword $secPass -InstallDNS -Force

# After promotion the server will restart automatically.

<##>
EOF

# scripts/powerShell/create-sample-users.ps1
cat > scripts/powerShell/create-sample-users.ps1 <<'EOF'
<# 
create-sample-users.ps1
Create sample OUs, groups, and users for the lab.
Run these from a machine with RSAT/AD PowerShell modules and network access to the domain.
#>

Import-Module ActiveDirectory

# Create OUs
New-ADOrganizationalUnit -Name "Workstations" -Path "DC=corp,DC=local" -ProtectedFromAccidentalDeletion:$false
New-ADOrganizationalUnit -Name "Servers" -Path "DC=corp,DC=local" -ProtectedFromAccidentalDeletion:$false
New-ADOrganizationalUnit -Name "Users" -Path "DC=corp,DC=local" -ProtectedFromAccidentalDeletion:$false

# Create a sample user (change password and parameters as needed)
$pw = ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force
New-ADUser -Name "Alice Admin" -SamAccountName "alice" -UserPrincipalName "alice@corp.local" -AccountPassword $pw -Enabled $true -Path "OU=Users,DC=corp,DC=local"

# Create a security group and add the user
New-ADGroup -Name "IT-Admins" -GroupScope Global -GroupCategory Security -Path "OU=Groups,DC=corp,DC=local"
Add-ADGroupMember -Identity "IT-Admins" -Members "alice"

# Add the group to Domain Admins for lab scenarios (use caution — destructive if used on production!)
Add-ADGroupMember -Identity "Domain Admins" -Members "IT-Admins"

<##>
EOF

# scripts/powerShell/sysmon-install.ps1
cat > scripts/powerShell/sysmon-install.ps1 <<'EOF'
<# 
sysmon-install.ps1
Example: install Sysmon and apply a configuration file.
This requires the Sysinternals Sysmon executable and a config file (download or craft one).
#>

Param(
    [string]$SysmonUrl = "https://download.sysinternals.com/files/Sysmon.zip",
    [string]$ConfigUrl = "https://raw.githubusercontent.com/SigmaHQ/sigma/master/sysmon/sysmonconfig.xml",
    [string]$TempDir = "$env:TEMP\sysmon-install"
)

New-Item -ItemType Directory -Path $TempDir -Force | Out-Null

# Download and extract Sysmon (manual step recommended for verification)
$zip = "$TempDir\sysmon.zip"
Invoke-WebRequest -Uri $SysmonUrl -OutFile $zip
Expand-Archive -Path $zip -DestinationPath $TempDir -Force

# Download a sample config (review before use)
$config = "$TempDir\sysmonconfig.xml"
Invoke-WebRequest -Uri $ConfigUrl -OutFile $config

# Install Sysmon (requires elevated privileges)
# sysmon.exe is typically in the extracted folder, adjust path as needed.
$sysmonExe = Get-ChildItem -Path $TempDir -Filter sysmon*.exe -Recurse | Select-Object -First 1
if ($sysmonExe) {
    & $sysmonExe.FullName -accepteula -i $config
    Write-Host "Sysmon installed with configuration: $config"
} else {
    Write-Error "Sysmon executable not found in $TempDir. Please extract manually and re-run."
}

<##>
EOF

# .github workflow
cat > .github/workflows/markdownlint.yml <<'EOF'
name: Markdown lint

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  markdownlint:
    name: Lint Markdown
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Lint Markdown with super-linter
        uses: github/super-linter@v4
        env:
          VALIDATE_MARKDOWN: true
          DEFAULT_BRANCH: main
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
EOF

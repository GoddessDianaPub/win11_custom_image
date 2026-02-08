# Quick Reference Guide - Windows 11 Image Deployment

## Command Reference

### Sysprep

**GUI Method:**
1. Navigate to `C:\Windows\System32\Sysprep`
2. Run `sysprep.exe`
3. Configure:
   - System Cleanup Action: OOBE
   - ✅ Generalize checkbox
   - Shutdown Options: Shutdown

**Command Line:**
```cmd
cd C:\Windows\System32\Sysprep
sysprep.exe /oobe /generalize /shutdown
```

**With Answer File:**
```cmd
sysprep.exe /oobe /generalize /shutdown /unattend:C:\Windows\System32\Sysprep\unattend.xml
```

---

### Store App Removal

**Before Sysprep:**
```powershell
# PowerShell as Administrator
Get-AppxPackage -AllUsers | Where-Object {$_.NonRemovable -eq $false -and $_.IsFramework -eq $false} | Remove-AppxPackage -AllUsers
```

**Selective Removal:**
```powershell
# List all apps
Get-AppxPackage -AllUsers | Select Name, PackageFullName

# Remove specific app
Get-AppxPackage -Name "PackageName" -AllUsers | Remove-AppxPackage -AllUsers
```

---

### DISM Commands

**Check Drive Letters (in WinPE):**
```cmd
diskpart
list volume
exit
```

**Capture Image:**
```cmd
# Fast compression (recommended)
dism /Capture-Image /ImageFile:F:\install.wim /CaptureDir:C:\ /Name:"Windows 11 Pro Custom" /Compress:fast

# Maximum compression
dism /Capture-Image /ImageFile:F:\install.wim /CaptureDir:C:\ /Name:"Windows 11 Pro Custom" /Compress:max

# No compression
dism /Capture-Image /ImageFile:F:\install.wim /CaptureDir:C:\ /Name:"Windows 11 Pro Custom"
```

**Get Image Information:**
```cmd
dism /Get-WimInfo /WimFile:F:\install.wim /index:1
```

**Get Detailed Information:**
```cmd
dism /Get-WimInfo /WimFile:F:\install.wim /index:1 /Detailed
```

**Mount Image (Advanced):**
```cmd
mkdir C:\mount
dism /Mount-Wim /WimFile:F:\install.wim /index:1 /MountDir:C:\mount

# Make changes to mounted image
# ...

dism /Unmount-Wim /MountDir:C:\mount /Commit
```

---

### oscdimg - Create Bootable ISO

**Navigate to Tool:**
```cmd
cd "C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\amd64\Oscdimg"
```

**Create Bootable ISO (Single Command):**
```cmd
oscdimg.exe -m -o -u2 -udfver102 -bootdata:2#p0,e,b"C:\Win11_Custom\boot\etfsboot.com"#pEF,e,b"C:\Win11_Custom\efi\microsoft\boot\efisys.bin" "C:\Win11_Custom" "C:\Win11_Custom_ISO.iso"
```

**Parameter Explanation:**
- `-m` = Ignore maximum image size
- `-o` = Optimize storage for duplicate files
- `-u2` = UDF file system
- `-udfver102` = UDF version 1.02
- `-bootdata:2#...` = BIOS and UEFI boot support
- First path = Source folder
- Last path = Destination ISO

---

### Windows Activation

**Set Product Key:**
```cmd
slmgr /ipk XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
```

**Activate Windows:**
```cmd
slmgr /ato
```

**Check Activation Status:**
```cmd
slmgr /xpr
```

**Display Detailed Info:**
```cmd
slmgr /dlv
```

**Remove Product Key:**
```cmd
slmgr /upk
```

---

### System Management

**Rename Computer:**
```powershell
# PowerShell
Rename-Computer -NewName "COMPUTER-NAME"
Restart-Computer
```

**Or via Command Prompt:**
```cmd
wmic computersystem where name="%computername%" call rename name="NEW-NAME"
shutdown /r /t 0
```

**System Information:**
```cmd
systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
```

**Check Windows Version:**
```cmd
winver
```

---

### Disk Management

**List Disks and Volumes:**
```cmd
diskpart
list disk
list volume
```

**Clean Disk:**
```cmd
diskpart
list disk
select disk 0
clean
exit
```

**Create Partition:**
```cmd
diskpart
select disk 0
clean
create partition primary
format fs=ntfs quick
assign letter=C
exit
```

---

### Log File Viewing

**View Sysprep Logs:**
```cmd
notepad C:\Windows\System32\Sysprep\Panther\setuperr.log
notepad C:\Windows\System32\Sysprep\Panther\setupact.log
```

**View DISM Logs:**
```cmd
# In WinPE
type X:\Windows\Logs\DISM\dism.log | more

# In Windows
notepad C:\Windows\Logs\DISM\dism.log
```

**View Setup Logs:**
```cmd
notepad C:\Windows\Panther\setuperr.log
notepad C:\Windows\Panther\setupact.log
```

---

## File Paths Reference

### Critical System Paths

| Item | Path |
|------|------|
| Sysprep executable | `C:\Windows\System32\Sysprep\sysprep.exe` |
| Answer file location | `C:\Windows\System32\Sysprep\unattend.xml` |
| Sysprep logs | `C:\Windows\System32\Sysprep\Panther\` |
| DISM logs (WinPE) | `X:\Windows\Logs\DISM\dism.log` |
| DISM logs (Windows) | `C:\Windows\Logs\DISM\dism.log` |
| Setup logs | `C:\Windows\Panther\` |
| oscdimg tool | `C:\Program Files (x86)\Windows Kits\10\...` |

### ISO Structure

```
Win11_Custom\
├── boot\
│   └── etfsboot.com
├── efi\
│   └── microsoft\
│       └── boot\
│           └── efisys.bin
├── sources\
│   ├── install.wim     ← Replace this file
│   └── boot.wim
├── bootmgr
├── bootmgr.efi
└── setup.exe
```

---

## Boot Keys by Manufacturer

| Manufacturer | Boot Menu | BIOS/UEFI Setup |
|--------------|-----------|-----------------|
| Lenovo | F12, F1 | F1, F2 |
| Dell | F12 | F2 |
| HP | F9, Esc | F10, Esc |
| ASUS | F8, Esc | F2, Del |
| Acer | F12 | F2, Del |
| MSI | F11 | Del |
| Gigabyte | F12 | Del |
| Microsoft Surface | Vol Down | Vol Up |

**General Tips:**
- Start pressing immediately when computer powers on
- Press repeatedly, don't hold
- Watch for on-screen prompts

---

## Rufus Configuration

### Recommended Settings for Windows 11

| Setting | Value |
|---------|-------|
| Device | Your USB drive |
| Boot selection | SELECT → Browse to ISO |
| Partition scheme | **GPT** |
| Target system | **UEFI (non CSM)** |
| Volume label | Descriptive name |
| File system | FAT32 (≤32GB) or NTFS (>32GB) |
| Cluster size | Default (4096 bytes) |

### Legacy Systems (Pre-2012)

| Setting | Value |
|---------|-------|
| Partition scheme | MBR |
| Target system | BIOS or UEFI-CSM |

---

## Expected File Sizes

| Item | Size |
|------|------|
| Windows 11 ISO (original) | 5-6 GB |
| Captured WIM (compressed) | 15-25 GB |
| Captured WIM (uncompressed) | 50-70 GB |
| Custom ISO with apps | 6-10 GB |
| Bootable USB | 6-10 GB |

### Image Quality Indicators

| Metric | Base Windows | With Apps |
|--------|--------------|-----------|
| Files | ~150,000 | 200,000+ |
| Directories | ~30,000 | 50,000+ |
| Size (uncompressed) | ~20 GB | 50-70 GB |

**If your numbers are significantly lower, applications may not be included.**

---

## Time Estimates

| Task | Duration |
|------|----------|
| Clean Windows install | 20-30 min |
| Windows Updates | 30-60 min |
| Application installation | 20-60 min |
| System configuration | 15-30 min |
| Sysprep execution | 2-5 min |
| DISM capture (fast) | 20-40 min |
| DISM capture (max) | 40-90 min |
| Copy ISO files | 5-10 min |
| Create ISO (oscdimg) | 5-10 min |
| Rufus USB creation | 10-15 min |
| Deployment to computer | 25-40 min |
| **Total process** | **4-6 hours** |

---

## unattend.xml Templates

### Minimal Template

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
    <settings pass="generalize">
        <component name="Microsoft-Windows-PnpSysprep" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
            <PersistAllDeviceInstalls>true</PersistAllDeviceInstalls>
        </component>
    </settings>
    <settings pass="oobeSystem">
        <component name="Microsoft-Windows-International-Core" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
            <InputLocale>en-US</InputLocale>
            <SystemLocale>en-US</SystemLocale>
            <UILanguage>en-US</UILanguage>
            <UserLocale>en-US</UserLocale>
        </component>
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
            <OOBE>
                <HideEULAPage>true</HideEULAPage>
                <ProtectYourPC>3</ProtectYourPC>
                <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
            </OOBE>
        </component>
    </settings>
</unattend>
```

### Common Locale Codes

| Region | Code |
|--------|------|
| US English | en-US |
| UK English | en-GB |
| French (France) | fr-FR |
| German (Germany) | de-DE |
| Spanish (Spain) | es-ES |
| Italian (Italy) | it-IT |
| Japanese (Japan) | ja-JP |

**Multiple Keyboards:**
```xml
<InputLocale>en-US;fr-FR;de-DE</InputLocale>
```

---

## Deployment Checklist

### Pre-Deployment

- [ ] Custom ISO created and verified
- [ ] Bootable USB created
- [ ] Target computer BIOS configured
  - [ ] Boot mode: UEFI
  - [ ] Secure Boot: Enabled (or disabled if issues)
  - [ ] USB boot: Enabled
- [ ] Product key available
- [ ] Network connectivity for activation
- [ ] Backup of existing data (if applicable)

### During Deployment

- [ ] Boot from USB successful
- [ ] All partitions deleted (for clean install)
- [ ] Correct partition selected
- [ ] Installation completed without errors
- [ ] Computer restarted successfully

### Post-Deployment

- [ ] Computer renamed
- [ ] Windows activated
- [ ] Windows Updates installed
- [ ] Antivirus installed
- [ ] Domain joined (if applicable)
- [ ] User account configured
- [ ] All applications tested
- [ ] Peripherals tested (printer, scanner)
- [ ] Network connectivity verified
- [ ] Deployment documented

---

## Diagnostic Commands

### System Information

```powershell
# PowerShell commands

# Get Windows version
Get-ComputerInfo | Select WindowsProductName, WindowsVersion, OsHardwareAbstractionLayer

# Get installed applications
Get-Package

# Get Windows features
Get-WindowsOptionalFeature -Online | Where-Object State -eq "Enabled"

# Check activation
Get-CimInstance -ClassName SoftwareLicensingProduct | Where-Object PartialProductKey
```

### Network Diagnostics

```cmd
# Command Prompt

# Basic network info
ipconfig /all

# DNS cache
ipconfig /displaydns

# Release/renew IP
ipconfig /release
ipconfig /renew

# Test connectivity
ping 8.8.8.8
ping google.com

# Trace route
tracert google.com

# Network statistics
netstat -an
```

### Event Logs

```powershell
# View recent errors
Get-EventLog -LogName System -EntryType Error -Newest 20

# View application errors
Get-EventLog -LogName Application -EntryType Error -Newest 20

# Search for specific event
Get-EventLog -LogName System | Where-Object {$_.EventID -eq 7000}
```

---

## Common Error Codes

| Code | Hex | Meaning | Context |
|------|-----|---------|---------|
| 2 | 0x80070002 | File not found | DISM, general file operations |
| 3 | 0x80070003 | Path not found | DISM capture, oscdimg |
| 5 | 0x80070005 | Access denied | Permissions, registry |
| 15634 | 0x3CF2 | Package error | Sysprep, Store apps |
| 1235 | 0x4D3 | Request aborted | DISM capture cancelled |

### Activation Error Codes

| Code | Meaning |
|------|---------|
| 0xC004F074 | Key Management Service unavailable |
| 0xC004C003 | Activation server busy |
| 0xC004F050 | Product key invalid |
| 0xC004F025 | Access denied (requires elevation) |

---

## PowerShell vs Command Prompt

### When to Use PowerShell

- Remove Windows Store apps
- System information queries
- WMI/CIM operations
- Complex filtering and scripting

### When to Use Command Prompt

- DISM operations
- Sysprep
- diskpart
- Traditional Windows utilities (slmgr, etc.)

**Note:** Most cmd commands work in PowerShell, but not vice versa.

---

## Recovery Options

### Boot to Recovery Environment

**From Windows:**
1. Hold Shift while clicking Restart
2. Troubleshoot → Advanced Options

**From Boot:**
1. Power on
2. Force shutdown when Windows logo appears
3. Repeat 2-3 times
4. Windows automatically enters recovery

### WinPE Command Prompt

Useful for:
- Image capture
- Partition management
- Boot repair
- File recovery

### Safe Mode

**Access:**
1. Settings → Update & Security → Recovery
2. Advanced startup → Restart now
3. Troubleshoot → Advanced → Startup Settings
4. Restart → Press 4 for Safe Mode

---

## Network Share Deployment (Optional)

### Setup Network Location

```cmd
# Map network drive
net use Z: \\server\images /user:domain\username password

# Copy WIM to network
copy C:\install.wim Z:\Windows11\install.wim

# Access from other computers
net use Z: \\server\images /user:domain\username password
```

### Deploy from Network

1. Boot target computer to WinPE
2. Connect to network
3. Map network drive
4. Run DISM to apply image from network location

**Note:** Requires appropriate network infrastructure.

---

## Version Control Template

### Document Each Image Version

```
Image: Win11_Pro_2026-02
Created: February 5, 2026
Creator: [Name]
Windows Build: 26200.7705

Applications:
- [List applications and versions]

Configuration:
- [List major configurations]

Changes from previous:
- [Document changes]

Known issues:
- [Document any issues]

Tested on:
- [List hardware models tested]
```

---

## Links and Resources

### Microsoft Official

- **Windows 11 ISO:** https://www.microsoft.com/software-download/windows11
- **Windows ADK:** https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install
- **DISM Reference:** https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism-reference
- **Sysprep Reference:** https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--system-preparation--overview

### Third-Party Tools

- **Rufus:** https://rufus.ie
- **WinPE Builder:** Part of Windows ADK

### Community

- **r/sysadmin:** Reddit community
- **r/Windows11:** Reddit community
- **Microsoft Tech Community:** https://techcommunity.microsoft.com

---

**Document Version:** 1.0  
**Last Updated:** February 2026

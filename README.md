# Windows 11 Custom Image Creation Guide

## Overview
This guide provides a complete technical methodology for creating custom Windows 11 Pro deployment images using only built-in Windows tools and free Microsoft utilities. This approach is ideal for organizations that need standardized deployments without enterprise infrastructure like MDT, SCCM, or Intune.

**Target Audience:** IT administrators, system engineers, technical support teams

**Cost:** $0 (uses only free Microsoft tools)

**Infrastructure Required:** None (USB-based deployment)

---

## Table of Contents
- [Technical Overview](#technical-overview)
- [Prerequisites](#prerequisites)
- [Phase 1: Prepare Base System](#phase-1-prepare-base-system)
- [Phase 2: Sysprep and Image Capture](#phase-2-sysprep-and-image-capture)
- [Phase 3: Create Bootable ISO](#phase-3-create-bootable-iso)
- [Phase 4: Deployment](#phase-4-deployment)
- [Technical Considerations](#technical-considerations)
- [Troubleshooting](#troubleshooting)

---

## Technical Overview

### What This Process Does

1. **Base System Preparation:** Clean Windows 11 installation with applications and configurations
2. **Generalization:** Uses Sysprep to remove computer-specific information
3. **Image Capture:** Uses DISM to capture the system into a WIM file
4. **ISO Creation:** Builds bootable ISO with custom WIM using oscdimg
5. **Deployment:** Distributes via bootable USB media

### Tools Used

All tools are free and from Microsoft:

- **Sysprep** (Built into Windows) - System generalization
- **DISM** (Built into Windows) - Image capture and management
- **Windows ADK** - Deployment toolkit (oscdimg)
- **Rufus** (Open source) - Bootable USB creation

### Process Flow

```
Clean Windows Install
        ↓
Install Applications & Configure
        ↓
Create Answer File (unattend.xml)
        ↓
Run Sysprep (Generalize)
        ↓
Boot into WinPE Recovery
        ↓
Capture Image with DISM
        ↓
Replace install.wim in ISO
        ↓
Create Bootable ISO with oscdimg
        ↓
Create Bootable USB with Rufus
        ↓
Deploy to Target Computers
```

---

## Prerequisites

### Hardware Requirements

**Source Computer (for image creation):**
- Windows 11 compatible hardware
- Minimum 8GB RAM
- 100GB free disk space
- Internet connection

**Working Computer (for ISO creation):**
- Any Windows 10/11 PC
- 100GB free disk space
- Administrator access

**Target Computers (deployment):**
- UEFI firmware
- Secure Boot capable
- Meets Windows 11 hardware requirements

### Required Materials

1. **USB Drives:**
   - USB 1 (32GB+) - Temporary storage for captured WIM file
   - USB 2 (32GB+) - Windows installation/deployment media
   - USB 3 (32GB+) - Windows bootable drive

2. **Software:**
   - Windows 11 Professional ISO
   - Windows ADK
   - Rufus

### Time Requirements

- **Total Process:** 4-6 hours (first time)
- **Subsequent Iterations:** 2-3 hours
- **Per-Computer Deployment:** 25-40 minutes

---

## Phase 1: Prepare Base System

### Step 1: Clean Windows 11 Installation

#### 1.1 Create Bootable Installation USB

1. Download Windows 11 ISO from Microsoft
2. Use Rufus to create bootable USB:
   - **Partition scheme:** GPT
   - **Target system:** UEFI (non CSM)
   - **File system:** FAT32 or NTFS

#### 1.2 Install Windows

1. Boot from USB (typically F12 for boot menu)
2. Select installation language and preferences
3. Choose "Custom: Install Windows only (advanced)"
4. Delete all existing partitions (for clean install)
5. Create new partition (Windows creates necessary structure automatically)
6. Complete installation

#### 1.3 Initial Setup (OOBE)

**Critical:** Create a **temporary local account** during setup:
- Offline account: windows 11 have only online account as an option. Open CMD: Press Shift + F10 > Type "start ms-cxh:localonly" > Enter
- Username: Generic (e.g., "Admin", "Setup", "ImageBuild")
- Password: Temporary password (or leave it empty for no password)
- Continue with the installation

**Note:** This account will be removed during Sysprep; end users will create their own accounts during deployment.

#### 1.4 Windows Updates

Install ALL available Windows updates:
1. Settings → Windows Update → Check for updates
2. Install all updates
3. Restart as needed
4. Repeat until no updates available

**Why this matters:** Updates are included in the image, reducing deployment time and ensuring security patches.

### Step 2: Install Applications

Install your required applications in the desired order.

#### Installation Best Practices

1. **Use silent/unattended installers when possible:**
   - Reduces manual intervention
   - Ensures consistent installation
   - Example: `installer.exe /S` or `/quiet`

2. **Install to default locations:**
   - Avoid custom paths that may conflict on target systems
   - Use standard Program Files directories

3. **Don't activate software in the image:**
   - Product keys should not be embedded
   - Activate post-deployment on each computer

4. **Test each application:**
   - Launch after installation to verify it works
   - Check for first-run prompts that need handling

#### What NOT to Include

❌ **Antivirus software** - Install post-deployment (licensing and signature update issues)
❌ **Product keys embedded in image** - Security risk
❌ **User-specific configurations** - Will be removed by Sysprep
❌ **Personal files or data** - Security and privacy concerns
❌ **Computer-specific drivers** (unless targeting identical hardware)

### Step 3: System Configuration

Configure system-wide settings that will persist through Sysprep.

#### Recommended Configurations

1. **File Explorer Settings:**
   - Show file extensions
   - Configure default view preferences

2. **Desktop Settings:**
   - Default wallpaper (if corporate branding)
   - Icon arrangement preferences

3. **Power Settings:**
   - Configure via `powercfg` command
   - Set appropriate timeouts for corporate environment

4. **Regional Settings:**
   - Time zone (can be overridden during deployment)
   - Date/time formats
   - Currency

#### Registry Modifications

If making registry changes:
- **HKLM (HKEY_LOCAL_MACHINE):** ✅ Persists through Sysprep
- **HKCU (HKEY_CURRENT_USER):** ❌ Removed during Sysprep
- Test on target hardware before deploying

**Note:** Document all registry changes for troubleshooting and future image updates.

### Step 4: Remove Unnecessary Components

#### Remove Bloatware

Remove pre-installed Windows apps that aren't needed:

```powershell
# PowerShell as Administrator
Get-AppxPackage -AllUsers | Where-Object {$_.NonRemovable -eq $false -and $_.IsFramework -eq $false} | Remove-AppxPackage -AllUsers
```

**What this does:**
- Removes removable Windows Store apps
- Keeps essential system components
- Prevents Sysprep errors

**Optional:** Remove specific apps individually for more control.

#### Clean Up Temporary Files

1. Run Disk Cleanup (cleanmgr.exe)
2. Empty Recycle Bin
3. Clear Downloads folder
4. Remove installation files for applications
5. Clear browser caches and temporary data

---

## Phase 2: Sysprep and Image Capture

### Step 5: Create Sysprep Answer File

The answer file (unattend.xml) automates portions of deployment and configures system settings.

#### 5.1 Create unattend.xml

1. Open Notepad as Administrator
2. Copy the XML template below
3. Customize for your environment (see customization options)
4. Save as: `C:\Windows\System32\Sysprep\unattend.xml`
   - **Important:** Save as "All Files" type, not .txt
   - Encoding: UTF-8

#### Basic unattend.xml Template

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
    <settings pass="generalize">
        <component name="Microsoft-Windows-PnpSysprep" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State">
            <PersistAllDeviceInstalls>true</PersistAllDeviceInstalls>
        </component>
    </settings>
    <settings pass="oobeSystem">
        <component name="Microsoft-Windows-International-Core" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <InputLocale>en-US</InputLocale>
            <SystemLocale>en-US</SystemLocale>
            <UILanguage>en-US</UILanguage>
            <UserLocale>en-US</UserLocale>
        </component>
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <OOBE>
                <HideEULAPage>true</HideEULAPage>
                <ProtectYourPC>3</ProtectYourPC>
                <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
            </OOBE>
        </component>
    </settings>
</unattend>
```

#### Customization Options

**Locale Settings:**
- `<InputLocale>` - Keyboard layout(s), separate multiple with semicolons
- `<SystemLocale>` - Regional format
- `<UILanguage>` - Windows interface language
- `<UserLocale>` - User regional settings

**Common Locale Codes:**
- US English: `en-US`
- UK English: `en-GB`
- Multiple keyboards: `en-US;de-DE` (English + German)

**OOBE Settings:**
- `<HideEULAPage>true</HideEULAPage>` - Skip license agreement screen
- `<ProtectYourPC>3</ProtectYourPC>` - Privacy settings (3 = minimal telemetry)
- `<HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>` - Skip WiFi setup

**Driver Persistence:**
- `<PersistAllDeviceInstalls>true</PersistAllDeviceInstalls>` - Keep all installed drivers

### Step 6: Remove Problematic Applications

Before Sysprep, remove Windows Store apps that can cause failures:

```powershell
# PowerShell as Administrator
Get-AppxPackage -AllUsers | Where-Object {$_.NonRemovable -eq $false -and $_.IsFramework -eq $false} | Remove-AppxPackage -AllUsers
```

**Why:** Some user-installed Store apps prevent Sysprep from generalizing the system.

### Step 7: Run Sysprep

#### 7.1 Execute Sysprep

1. Navigate to: `C:\Windows\System32\Sysprep`
2. Double-click `sysprep.exe`
3. Configure Sysprep window:
   - **System Cleanup Action:** Enter System Out-of-Box Experience (OOBE)
   - **Generalize:** ✅ **CHECK THIS BOX** (critical!)
   - **Shutdown Options:** Shutdown

4. Click **OK**
5. Wait 2-5 minutes for Sysprep to complete
6. Computer will shut down automatically

#### What Sysprep Does

- Removes computer-specific information (SID, computer name, etc.)
- Removes user accounts
- Removes activation data
- Prepares system for deployment to different hardware
- Preserves installed applications and system-wide settings

#### ⚠️ CRITICAL

**After Sysprep shuts down the computer:**
- **DO NOT** boot into Windows normally
- **DO NOT** turn on the computer without USB ready
- If Windows boots and completes OOBE, the Sysprep is "consumed" and you must start over

### Step 8: Boot into Windows Recovery Environment

#### 8.1 Prepare for Image Capture

1. Ensure computer is shut down (from Sysprep)
2. Insert Windows 11 installation USB
3. Insert second USB (40GB+) for storing captured image

#### 8.2 Boot from USB

1. Turn on computer
2. **Immediately press F12** (or manufacturer's boot key)
3. Select USB drive from boot menu
4. Windows Setup loads

#### 8.3 Access Command Prompt

1. Windows Setup appears → Select language → Click "Next"
2. **DO NOT** click "Install now"
3. Click **"Repair your computer"** (bottom left corner)
4. Choose an option → **Troubleshoot**
5. Advanced options → **Command Prompt**
6. Command Prompt window opens

### Step 9: Identify Drive Letters

Drive letters in Windows PE (recovery environment) differ from normal Windows.

#### 9.1 Use diskpart to List Volumes

```cmd
diskpart
list volume
```

#### 9.2 Identify Drives

Look for:
- **Windows partition:** Usually large NTFS volume (200-500 GB)
- **USB installation media:** May show as removable (16-32 GB)
- **Image storage USB:** Your 40GB+ USB drive

Example output:
```
Volume 0   C:            NTFS   476 GB   (Your Windows installation)
Volume 1                 FAT32  200 MB   (EFI System Partition)
Volume 2                 NTFS   735 MB   (Recovery)
Volume 3   D:  Win11_USB NTFS    29 GB   (Installation USB)
Volume 5   F:  Storage   exFAT  233 GB   (Your image storage USB)
```

#### 9.3 Exit diskpart

```cmd
exit
```

**Note down:**
- Windows drive letter (e.g., C:)
- Image storage USB drive letter (e.g., F:)

### Step 10: Capture the Windows Image

#### 10.1 DISM Capture Command

```cmd
dism /Capture-Image /ImageFile:F:\install.wim /CaptureDir:C:\ /Name:"Windows 11 Pro Custom" /Compress:fast
```

**Parameters:**
- `/ImageFile:F:\install.wim` - Destination file path (use your USB drive letter)
- `/CaptureDir:C:\` - Source directory (your Windows installation drive)
- `/Name:"Windows 11 Pro Custom"` - Image name (descriptive)
- `/Compress:fast` - Compression level

**Compression Options:**
- `fast` - Faster, larger file (~15-20 GB) - **Recommended**
- `max` - Slower, smaller file (~12-15 GB)
- None - Fastest, largest file (~50-65 GB)

#### 10.2 Wait for Completion

- Process takes 20-40 minutes (with fast compression)
- Progress shown as percentage
- **Do not interrupt** - this could corrupt the image

#### 10.3 Verify Success

Look for: `The operation completed successfully.`

File created: `F:\install.wim` (or your specified path)

#### 10.4 Shutdown

```cmd
wpeutil shutdown
```

Remove both USB drives.

### Step 11: Verify Captured Image

On your working computer, verify the image quality:

```cmd
dism /Get-WimInfo /WimFile:F:\install.wim /index:1
```

**Check for:**
- **Size:** 50-65 GB uncompressed (indicates applications included)
- **Files:** 200,000+ files (standard Windows ~150,000)
- **Directories:** 50,000+ directories
- **Edition:** Professional
- **Architecture:** x64

**If numbers are significantly lower, applications may not be included - recapture needed.**

---

## Phase 3: Create Bootable ISO

### Step 12: Extract Original Windows ISO

On your working computer:

#### 12.1 Mount ISO

1. Right-click Windows 11 ISO file
2. Select "Mount"
3. Windows assigns a drive letter (e.g., E:)

#### 12.2 Create Working Directory

```cmd
mkdir C:\Win11_Custom
```

#### 12.3 Copy All Files

1. Open mounted ISO drive in File Explorer
2. Select all files (Ctrl+A)
3. Copy (Ctrl+C)
4. Navigate to `C:\Win11_Custom`
5. Paste (Ctrl+V)
6. Wait 5-10 minutes for copy to complete

#### 12.4 Unmount ISO

1. Right-click mounted ISO drive
2. Select "Eject"

### Step 13: Replace install.wim

#### 13.1 Navigate to sources Folder

```cmd
cd C:\Win11_Custom\sources
```

#### 13.2 Remove Original install.wim

```cmd
del install.wim
```

Or rename as backup:
```cmd
ren install.wim install_original.wim
```

#### 13.3 Copy Custom install.wim

1. Insert your image storage USB (with custom install.wim)
2. Copy the WIM file:
   ```cmd
   copy F:\install.wim C:\Win11_Custom\sources\
   ```
   (Replace F: with your USB drive letter)
3. Wait 5-15 minutes for copy (large file)

### Step 14: Install Windows ADK

#### 14.1 Download Windows ADK

- URL: https://go.microsoft.com/fwlink/?linkid=2271337
- Download: `adksetup.exe`
- Size: ~500 MB download, ~2 GB installed

#### 14.2 Install ADK

1. Run `adksetup.exe` as Administrator
2. Choose installation location (default is fine)
3. **Select features screen:**
   - ✅ **Check ONLY:** "Deployment Tools"
   - ❌ Uncheck all others
4. Click Install
5. Wait ~10 minutes

**Why only Deployment Tools:** Contains oscdimg.exe needed for ISO creation.

### Step 15: Create Bootable ISO

#### 15.1 Open Command Prompt as Administrator

1. Press Windows key
2. Type: `cmd`
3. Right-click "Command Prompt"
4. Select "Run as administrator"

#### 15.2 Navigate to oscdimg

```cmd
cd "C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\amd64\Oscdimg"
```

#### 15.3 Create Bootable ISO

**Single command (all one line):**

```cmd
oscdimg.exe -m -o -u2 -udfver102 -bootdata:2#p0,e,b"C:\Win11_Custom\boot\etfsboot.com"#pEF,e,b"C:\Win11_Custom\efi\microsoft\boot\efisys.bin" "C:\Win11_Custom" "C:\Win11_Custom_ISO.iso"
```

**Parameters explained:**
- `-m` - Ignore maximum image size warnings
- `-o` - Optimize duplicate files storage
- `-u2` - UDF file system
- `-udfver102` - UDF version 1.02
- `-bootdata:2#...` - Dual boot (BIOS + UEFI)
- Source: `C:\Win11_Custom`
- Destination: `C:\Win11_Custom_ISO.iso`

#### 15.4 Wait for Completion

- Process takes 5-10 minutes
- Progress shown in console
- Final ISO size: ~6-8 GB

#### 15.5 Verify ISO

Check file exists: `C:\Win11_Custom_ISO.iso`

**Your custom bootable ISO is ready!**

---

## Phase 4: Deployment

### Step 16: Create Bootable USB

#### 16.1 Use Rufus

1. Download Rufus from https://rufus.ie
2. Run Rufus (portable, no installation needed)
3. Insert USB drive (32GB minimum, will be erased)

#### 16.2 Rufus Configuration

| Setting | Value |
|---------|-------|
| **Device** | Your USB drive |
| **Boot selection** | Click SELECT → Choose `C:\Win11_Custom_ISO.iso` |
| **Partition scheme** | GPT |
| **Target system** | UEFI (non CSM) |
| **Volume label** | Descriptive name |
| **File system** | FAT32 (for USB ≤32GB) or NTFS (>32GB) |
| **Cluster size** | Default |

#### 16.3 Create USB

1. Click **START**
2. If prompted about download required, click Yes
3. Confirm warning about data deletion
4. Wait 10-15 minutes
5. "READY" appears when complete
6. Eject USB safely

**Your deployment USB is ready!**

### Step 17: Deploy to Target Computers

#### 17.1 Boot from USB

1. Insert bootable USB into target computer
2. Power on computer
3. Press boot menu key (F12 for most manufacturers)
4. Select USB drive from boot menu
5. Press Enter

#### 17.2 Windows Installation Process

1. **Setup Begins:**
   - Select language and keyboard
   - Click "Next"
   - Click "Install now"

2. **Product Key:**
   - Enter Windows 11 Pro product key
   - Or click "I don't have a product key" (activate later)

3. **Select Edition:**
   - Choose "Windows 11 Pro"
   - Click "Next"

4. **License Agreement:**
   - Accept terms
   - Click "Next"

5. **Installation Type:**
   - Select "Custom: Install Windows only (advanced)"

6. **Partition Setup:**
   
   **For Clean Install (Recommended):**
   - Select each partition
   - Click "Delete"
   - Repeat until only "Unallocated Space" remains
   - Click "New"
   - Click "Apply"
   - Windows creates required partitions automatically
   - Select largest partition (Type: Primary)
   - Click "Next"

   **For Keeping Data:**
   - Format only the Windows partition
   - Or install to different partition

7. **Installation:**
   - Files copy (~10 minutes)
   - Features and updates install (~15 minutes)
   - Computer restarts several times automatically
   - **All pre-installed applications install automatically**

8. **Out of Box Experience (OOBE):**
   - **Region:** Select country/region
   - **Keyboard:** Confirm keyboard layout(s)
   - **Network:** Connect to WiFi/Ethernet (or skip)
   - **Account Type:**
     - Sign in with Microsoft Account, OR
     - Create local account (offline account)
   - **Username:** Create user account
   - **Password:** Set password
   - **Security Questions:** Answer 3 questions (for local accounts)
   - **Privacy:** Configure privacy settings
   - **Optional:** Set up Windows Hello PIN

9. **Desktop:**
   - First login may take 2-3 minutes
   - Desktop appears with all applications installed
   - ✅ Deployment complete!

#### 17.3 Typical Deployment Timeline

| Phase | Duration |
|-------|----------|
| Files copying | 10 min |
| Installation | 15 min |
| OOBE setup | 5 min |
| First boot | 2-3 min |
| **Total** | **~30-35 minutes** |

### Step 18: Post-Deployment Configuration

Complete these tasks on each deployed computer:

#### Essential Tasks

1. **Rename Computer:**
   - Settings → System → About → Rename this PC
   - Use naming convention (e.g., DEPT-USER-001)
   - Restart required

2. **Activate Windows:**
   - Settings → System → Activation
   - Enter product key if not done during installation
   - Or verify automatic activation (for OEM hardware)

3. **Windows Updates:**
   - Settings → Windows Update
   - Check for updates
   - Install any updates released after image creation

4. **Install Antivirus:**
   - Install corporate antivirus solution
   - **Never include antivirus in image** (causes licensing issues)

5. **Join Domain (if applicable):**
   - Settings → System → About → Advanced system settings
   - Computer Name tab → Change
   - Join to Active Directory domain

6. **Configure User Profile:**
   - Set up email accounts
   - Sign into applications
   - Configure user preferences

7. **Verify Functionality:**
   - Test all installed applications
   - Verify network connectivity
   - Test peripherals (printer, scanner)
   - Check hardware functionality

#### Optional Tasks

- Install additional user-specific software
- Configure VPN connections
- Set up shared network drives
- Configure backup solutions
- Apply Group Policy (if domain-joined)

---

## Technical Considerations

### Image Size Management

**Expected Sizes:**
- Windows 11 base: ~10-15 GB
- With applications: +5-10 GB per major suite
- Captured WIM (compressed): 15-25 GB
- Captured WIM (uncompressed): 50-70 GB
- Final ISO: 6-10 GB

**Tips for Smaller Images:**
- Remove unnecessary Windows features
- Use maximum DISM compression
- Clean temporary files before capture
- Remove language packs not needed

### Hardware Compatibility

**Best Practice:** Create separate images for significantly different hardware types:
- Different CPU architectures
- Different graphics solutions (integrated vs. discrete)
- Different storage types (NVMe vs. SATA)

**Universal Image Approach:**
- Capture on most generic hardware
- Include common drivers
- Let Windows Update handle specific drivers post-deployment
- Use vendor management tools (Dell Command Update, Lenovo Vantage)

### Sysprep Limitations

- **Reset Counter:** Sysprep can run 3-8 times before reaching limit
- **If limit reached:** Fresh Windows installation required
- **Check counter:** 
  ```cmd
  reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SoftwareProtectionPlatform" /v SkipRearm
  ```

### Security Considerations

**Do NOT Include in Image:**
- ❌ Product keys (Windows, Office, applications)
- ❌ Passwords or credentials
- ❌ Certificates (unless corporate PKI certificates)
- ❌ User data or personal files
- ❌ Browser profiles with saved passwords
- ❌ Activated antivirus (licensing issues)

**Do Include:**
- ✅ Latest security patches
- ✅ Hardened default configurations
- ✅ Corporate security policies
- ✅ Certificate authorities (if needed)

### Maintenance Strategy

**Update Image Regularly:**
- Monthly: For security-critical environments
- Quarterly: Standard recommendation
- Bi-annually: Minimum acceptable

**Version Control:**
- Name images with date: `Win11_Pro_2026-02.wim`
- Document changes in each version
- Keep at least 2 previous versions as backup
- Test new images on pilot hardware before mass deployment

### Legal and Licensing

- Ensure proper Windows 11 Pro licensing for all deployed computers
- Verify application licensing permits this deployment method
- Some software licenses prohibit imaging
- Consult with software vendors when uncertain
- Volume licensing may have specific imaging rights

---

## Troubleshooting

### Common Issues

#### Sysprep Fails

**Error:** "Sysprep was not able to validate your Windows installation"

**Cause:** Windows Store apps blocking generalization

**Solution:**
```powershell
Get-AppxPackage -AllUsers | Where-Object {$_.NonRemovable -eq $false -and $_.IsFramework -eq $false} | Remove-AppxPackage -AllUsers
```

**Check logs:** `C:\Windows\System32\Sysprep\Panther\setuperr.log`

---

#### DISM Capture Error

**Error:** "The system cannot find the path specified" (Error code: 3 or 0x80070003)

**Cause:** Incorrect drive letter in command

**Solution:**
1. Run `diskpart` → `list volume` to verify drive letters
2. Update DISM command with correct letters
3. Ensure destination USB has sufficient space

---

#### Computer Boots After Sysprep

**Problem:** Computer boots into Windows instead of staying off

**Impact:** Sysprep generalization consumed, cannot capture image

**Solution:**
- Must start over from clean Windows install
- Prevention: Have USB ready before running Sysprep, boot immediately after shutdown

---

#### USB Won't Boot

**Symptoms:** Boot menu doesn't show USB, or computer ignores it

**Solutions:**
1. Check BIOS/UEFI settings:
   - Enable USB boot
   - Disable Secure Boot (temporarily, if needed)
   - Set boot mode to UEFI (not Legacy)
2. Recreate USB with Rufus using GPT + UEFI settings
3. Try different USB port
4. Update system firmware

---

#### Applications Missing After Deployment

**Diagnosis:**
```cmd
dism /Get-WimInfo /WimFile:F:\install.wim /index:1
```

Check file count - should be 200,000+ if apps included

**Solution:**
- If low file count: Recapture image
- If WIM correct: Recreate ISO and bootable USB
- Verify install.wim copied to correct location in ISO

---

### Log Files

**Sysprep Logs:**
- Location: `C:\Windows\System32\Sysprep\Panther\`
- Files: `setuperr.log`, `setupact.log`

**DISM Logs:**
- WinPE: `X:\Windows\Logs\DISM\dism.log`
- Windows: `C:\Windows\Logs\DISM\dism.log`

**Windows Setup Logs:**
- Location: `C:\Windows\Panther\`
- Files: `setupact.log`, `setuperr.log`

---

## Additional Resources

### Microsoft Documentation
- [Windows 11 Deployment](https://learn.microsoft.com/en-us/windows/deployment/)
- [Sysprep Technical Reference](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--system-preparation--overview)
- [DISM Reference](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism---deployment-image-servicing-and-management-technical-reference-for-windows)

### Download Links
- **Windows ADK:** https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install
- **Rufus:** https://rufus.ie
- **Windows 11 ISO:** https://www.microsoft.com/software-download/windows11

---

## License

This documentation is provided as-is for educational and professional use. All Microsoft products and tools are subject to their respective licenses and terms of service.

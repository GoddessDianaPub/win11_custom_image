# Troubleshooting Guide - Windows 11 Custom Image Deployment

## Overview

This guide addresses common technical issues encountered during Windows 11 custom image creation and deployment. Issues are organized by deployment phase for quick reference.

---

## Table of Contents

- [Sysprep Phase Issues](#sysprep-phase-issues)
- [Image Capture Issues](#image-capture-issues)
- [ISO Creation Issues](#iso-creation-issues)
- [Deployment Issues](#deployment-issues)
- [Post-Deployment Issues](#post-deployment-issues)
- [Log File Analysis](#log-file-analysis)
- [Error Code Reference](#error-code-reference)

---

## Sysprep Phase Issues

### Issue: Sysprep Fails with App Package Error

**Error Message:**
```
Package [PackageName] was installed for a user, but not provisioned for all users.
This package will not function properly in the sysprep image.
Failed to remove apps for the current user: 0x80073cf2
```

**Cause:**  
Windows Store applications installed for current user only prevent Sysprep generalization.

**Solution:**
```powershell
# Open PowerShell as Administrator
Get-AppxPackage -AllUsers | Where-Object {$_.NonRemovable -eq $false -and $_.IsFramework -eq $false} | Remove-AppxPackage -AllUsers
```

**Prevention:**  
Run this command before Sysprep to proactively remove potentially problematic apps.

**Log Location:** `C:\Windows\System32\Sysprep\Panther\setuperr.log`

---

### Issue: Sysprep Completes But Computer Boots Normally

**Symptoms:**
- Sysprep appeared to complete successfully
- Computer shut down as expected
- Upon next boot, Windows starts normally instead of OOBE

**What Happened:**  
The sysprepped system booted into Windows, "consuming" the generalized state. The image cannot be captured.

**Recovery Options:**

**Option 1 - If Caught Early:**
If you notice Windows starting to boot (loading screen visible):
1. Force shutdown immediately (hold power button 10 seconds)
2. Boot from USB recovery environment
3. Attempt image capture
4. May or may not work depending on how far boot progressed

**Option 2 - If Setup Completed:**
Must start over:
1. Reinstall Windows from scratch
2. Reinstall all applications
3. Reconfigure settings
4. Run Sysprep again
5. Boot into recovery immediately after Sysprep shutdown

**Prevention:**
- Have bootable USB inserted before running Sysprep
- Power on immediately after Sysprep shutdown
- Press boot menu key (F12) immediately
- Don't leave computer unattended after Sysprep

---

### Issue: "Windows could not finish configuring the system"

**Cause:**  
- Corrupted unattend.xml file
- Invalid XML syntax
- Conflicting settings in answer file

**Solution:**
1. Check unattend.xml for syntax errors
2. Validate XML using online validator
3. Try without unattend.xml (remove or rename file)
4. Review Sysprep logs for specific error

**Common XML Errors:**
- Missing closing tags
- Incorrect namespace declarations
- Invalid locale codes
- Typos in component names

---

### Issue: Sysprep Reaches Reset Limit

**Error:** "A fatal error occurred while trying to sysprep the machine"

**Cause:**  
Sysprep has a limited reset counter (typically 3-8 times per Windows installation).

**Check Current Count:**
```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SoftwareProtectionPlatform" /v SkipRearm
```

**Solutions:**

**Temporary (Not Recommended):**
```cmd
reg add "HKLM\SYSTEM\Setup\Status\SysprepStatus" /v GeneralizationState /t REG_DWORD /d 7 /f
```

**Permanent:**
Fresh Windows installation required - counter cannot be permanently reset.

---

## Image Capture Issues

### Issue: DISM Error "The system cannot find the path specified"

**Error Code:** 3 (0x80070003)

**Symptoms:**
```
Error: 3
The system cannot find the path specified.
The DISM log file can be found at X:\windows\Logs\DISM\dism.log
```

**Cause:**  
Drive letters in Windows PE differ from normal Windows environment.

**Diagnosis:**
```cmd
diskpart
list volume
exit
```

**Solution:**

1. **Identify correct drive letters:**
   - Windows partition: Usually largest NTFS volume
   - Destination USB: Your image storage drive

2. **Update DISM command:**
   ```cmd
   dism /Capture-Image /ImageFile:F:\install.wim /CaptureDir:C:\ /Name:"Custom" /Compress:fast
   ```
   Replace drive letters with actual values from `list volume`

3. **Verify destination has space:**
   - Minimum 20GB free recommended
   - Check with `dir F:` (replace F: with your drive)

---

### Issue: DISM Capture Extremely Slow

**Symptoms:**
- Progress stuck at same percentage for >30 minutes
- Estimated time keeps increasing
- Process not responding to Ctrl+C

**Causes:**
1. USB 2.0 connection (slow transfer speeds)
2. Failing USB drive
3. Maximum compression taking excessive time
4. Bad sectors on source or destination drive

**Solutions:**

**Immediate:**
1. Wait - max compression can take 60-90 minutes normally
2. Check if progress is actually moving (even 1% every 5-10 minutes is progress)

**For Next Attempt:**
1. **Use faster compression:**
   ```cmd
   dism /Capture-Image /ImageFile:F:\install.wim /CaptureDir:C:\ /Name:"Custom" /Compress:fast
   ```
   
2. **Verify USB connection:**
   - Use USB 3.0 drive in USB 3.0 port (blue port)
   - Try different USB port
   - Check cable connection

3. **Test USB drive:**
   - Try different high-quality USB drive
   - Avoid cheap/old drives

4. **Check disk health:**
   ```cmd
   # Before capture, in WinPE
   chkdsk C: /scan
   ```

---

### Issue: "Not enough space" Error During Capture

**Error:** Insufficient disk space

**Cause:**  
Destination USB drive doesn't have enough free space for the WIM file.

**Expected Sizes:**
- Fast compression: 15-20 GB
- Max compression: 10-15 GB  
- No compression: 50-70 GB

**Solution:**
1. Use larger USB drive (64GB+ recommended)
2. Clear space on current USB
3. Use faster compression to reduce size
4. Split image across multiple drives (advanced)

---

### Issue: Captured Image Missing Applications

**Symptoms:**
- DISM capture completes successfully
- WIM file created
- After deployment, applications are missing

**Diagnosis:**
```cmd
dism /Get-WimInfo /WimFile:F:\install.wim /index:1
```

**Check:**
- **File count:** Should be 200,000+ (base Windows ~150,000)
- **Directory count:** Should be 50,000+
- **Size (uncompressed):** Should be 50-70 GB with apps

**If Numbers Are Low:**

**Root Causes:**
1. Applications weren't actually installed before Sysprep
2. Sysprep removed applications (shouldn't happen with normal apps)
3. Wrong partition captured
4. Capture stopped prematurely

**Solution:**
Must recapture:
1. Boot sysprepped system into audit mode (if possible)
2. Or start over with fresh installation
3. Verify applications before Sysprep next time
4. Document installed applications for verification

---

## ISO Creation Issues

### Issue: "Smart App Control blocked a file"

**Symptoms:**
Windows blocks mounting or using the ISO file.

**Error:** "This file was blocked because files of this type from the internet can be dangerous"

**Solution:**

**Method 1 - Unblock File:**
1. Right-click ISO file
2. Properties
3. General tab → Check "Unblock" at bottom
4. Apply → OK
5. Try mounting again

**Method 2 - Disable Smart App Control (Temporary):**
1. Windows Security → App & browser control
2. Smart App Control settings
3. Turn off (temporarily)
4. Complete ISO operations
5. Re-enable afterwards

---

### Issue: oscdimg Not Found

**Error:**
```
'oscdimg' is not recognized as an internal or external command,
operable program or batch file.
```

**Cause:**  
Windows ADK not installed, or not navigated to correct directory.

**Solution:**

**Verify Installation:**
```cmd
dir "C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\amd64\Oscdimg\oscdimg.exe"
```

**If File Doesn't Exist:**
1. Install Windows ADK with Deployment Tools component
2. Ensure "Deployment Tools" feature selected during installation

**If File Exists:**
Navigate to directory first:
```cmd
cd "C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\amd64\Oscdimg"
```

Then run oscdimg command from this directory.

---

### Issue: oscdimg Fails - Cannot Find Boot File

**Error:**
```
Unable to open "C:\Win11_Custom\boot\etfsboot.com"
Access is denied
```

**Causes:**
1. Files not copied correctly from original ISO
2. Wrong path in oscdimg command
3. Insufficient permissions

**Solution:**

**Verify Boot Files Exist:**
```cmd
dir "C:\Win11_Custom\boot\etfsboot.com"
dir "C:\Win11_Custom\efi\microsoft\boot\efisys.bin"
```

**If Missing:**
1. Recopy all files from original ISO
2. Don't skip any files or folders
3. Ensure hidden and system files copied

**If Present:**
1. Run Command Prompt as Administrator
2. Verify path in oscdimg command matches actual folder location
3. Check for typos in folder name

---

### Issue: Created ISO Not Bootable

**Symptoms:**
- ISO created successfully
- Rufus completes without errors
- USB doesn't boot on target computer

**Diagnosis Steps:**

**1. Verify ISO Structure:**
```cmd
# Mount ISO and check for:
dir E:\boot
dir E:\efi
dir E:\sources\install.wim
```

Should contain all these folders/files.

**2. Check Rufus Settings:**
- Partition scheme: GPT
- Target system: UEFI (non CSM)
- Boot selection: Your ISO file

**3. Test on Different Computer:**
Some computers have specific boot requirements.

**Solutions:**
1. Recreate ISO with oscdimg
2. Verify oscdimg command has correct boot file paths
3. Use different USB drive
4. Update target computer firmware
5. Check BIOS settings (UEFI mode, Secure Boot)

---

## Deployment Issues

### Issue: USB Drive Not Detected in Boot Menu

**Symptoms:**
- USB inserted
- Press boot menu key (F12)
- USB drive not listed

**Solutions:**

**1. BIOS/UEFI Settings:**
- Enter BIOS setup (F2, Del, F1 - varies by manufacturer)
- Check: USB boot enabled
- Check: Boot mode = UEFI (not Legacy)
- Check: Secure Boot = Can be enabled or disabled
- Save and exit

**2. Try Different USB Port:**
- Front ports vs rear ports
- USB 2.0 (black) sometimes more compatible than USB 3.0 (blue) for booting
- Avoid USB-C ports if having issues

**3. Recreate Bootable USB:**
- Use Rufus with correct settings
- GPT + UEFI for modern computers (2012+)
- Try different USB drive if available

**4. Update Firmware:**
- Download latest BIOS/UEFI from manufacturer
- Flash update
- May resolve boot compatibility

---

### Issue: "Windows cannot be installed to this disk"

**Full Error:** "Windows cannot be installed to this disk. The selected disk is of the GPT partition style."

**Cause:**  
Mismatch between partition style (GPT vs MBR) and boot mode (UEFI vs Legacy).

**Solution:**

**Recommended - Delete All Partitions:**
1. In partition screen, select each partition
2. Click "Delete"
3. Repeat for all partitions
4. Select "Unallocated Space"
5. Click "New"
6. Click "Apply"
7. Windows creates correct partition structure automatically
8. Select primary partition
9. Click "Next"

**Alternative - Change Boot Mode:**
1. Restart to BIOS/UEFI
2. Change boot mode from Legacy to UEFI
3. Save and exit
4. Boot from USB again

**For Modern Computers:** Always use GPT + UEFI.

---

### Issue: Installation Fails at Specific Percentage

**Common Failure Points:**
- "Getting files ready" (0-30%)
- "Getting devices ready" (30-60%)
- "Getting ready" (60-100%)

**Causes:**
1. Corrupted installation media
2. Failing storage drive
3. Incompatible hardware
4. Bad RAM

**Diagnosis:**

**Test Installation Media:**
1. Try installing on different computer
2. If works elsewhere: Target computer hardware issue
3. If fails everywhere: Recreate ISO and USB

**Test Storage Drive:**
```cmd
# In Command Prompt during setup (Shift+F10)
diskpart
list disk
select disk 0
clean
# Then restart installation
```

**Test RAM:**
- Remove RAM modules one at a time
- Try installation with each module individually
- Identifies faulty RAM

**Hardware Compatibility:**
- Verify TPM 2.0 present and enabled
- Check Windows 11 hardware requirements
- Update BIOS/UEFI firmware

---

### Issue: Installation Completes But Won't Boot

**Symptoms:**
- Installation finishes successfully
- Computer restarts
- Shows "No bootable device" or similar error

**Causes:**
1. Boot order incorrect
2. Secure Boot issues
3. Boot loader corruption
4. Wrong disk selected

**Solution:**

**1. Remove USB Drive:**
After installation, USB should be removed before first boot.

**2. Check Boot Order:**
- Enter BIOS
- Ensure internal drive first in boot order
- Save and exit

**3. Repair Boot Loader:**
```cmd
# Boot from USB → Repair your computer → Command Prompt
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd
```

**4. Verify Correct Disk:**
- Some systems have multiple drives
- May have installed to wrong disk
- Check which disk in BIOS

---

## Post-Deployment Issues

### Issue: Windows Activation Fails

**Symptoms:**
- "Windows is not activated"
- "We can't activate Windows on this device"
- Error codes: 0xC004F074, 0xC004C003, etc.

**Solutions:**

**Manual Activation:**
```cmd
# Open Command Prompt as Administrator
slmgr /ipk XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
slmgr /ato
```

**Troubleshoot Activation:**
1. Settings → System → Activation → Troubleshoot
2. Follow wizard instructions
3. May require Microsoft account sign-in

**For OEM Computers:**
- Should activate automatically when connected to internet
- Digital license tied to hardware
- Wait 24 hours and check again

**For Volume License:**
- Ensure connected to corporate network
- KMS server should activate automatically
- Contact IT if issues persist

**Last Resort:**
- Phone activation: `slui 4`
- Follow prompts for manual activation

---

### Issue: Drivers Missing (Device Manager Yellow Exclamation)

**Symptoms:**
- Devices not working
- Yellow exclamation marks in Device Manager
- Unknown devices listed

**Solutions:**

**1. Windows Update:**
```
Settings → Windows Update → Check for updates
```
Installs most common drivers automatically.

**2. Manufacturer Tools:**
- Use vendor-specific update tools (if available)
- Download drivers from manufacturer website
- Install in order: Chipset → Storage → Network → Graphics → Audio

**3. Manual Driver Installation:**
1. Identify unknown device in Device Manager
2. Right-click → Properties → Details tab
3. Property: Hardware IDs
4. Google the hardware ID to identify device
5. Download driver from manufacturer

**4. For Future Images:**
- Capture image on same hardware model as targets
- Or create hardware-specific images
- Or use driver injection techniques (advanced)

---

### Issue: Applications Don't Launch or Crash

**Common Causes:**

**1. Missing Runtime/Framework:**
- Verify .NET versions installed
- Install Visual C++ Redistributables if needed
- Check application requirements

**2. Windows Updates Needed:**
- Install all available Windows Updates
- Restart and check again

**3. Permissions Issues:**
- Run application as Administrator (test)
- Check user account permissions
- Verify application installation directory permissions

**4. Corrupted Installation:**
- Uninstall application
- Clear leftover files/registry entries
- Reinstall fresh copy
- Or use repair installation if available

---

### Issue: Custom Configurations Not Applied

**Symptoms:**
- Expected settings not present
- System reverted to defaults
- Configurations missing

**Causes:**

**User-Specific vs System-Wide:**
- Settings in HKCU (current user registry) are removed by Sysprep
- Only HKLM (local machine) settings persist
- User profile settings don't transfer

**Solutions for Future:**

**System-Wide Settings:**
- Configure in HKLM registry hive
- Use Group Policy (if domain environment)
- Apply via startup scripts
- Or accept manual configuration post-deployment

**Default User Profile:**
- Modify default user profile before Sysprep (advanced)
- Settings apply to all new users created
- Requires careful implementation

---

## Log File Analysis

### Sysprep Logs

**Location:** `C:\Windows\System32\Sysprep\Panther\`

**Key Files:**
- `setuperr.log` - Errors and warnings
- `setupact.log` - All actions performed

**How to Read:**
1. Open in Notepad or text editor
2. Search for "Error" or "Failed"
3. Note the component and package causing issue
4. Look for preceding context to understand failure

**Common Errors:**
- Package installation failures
- Registry access denied
- Driver installation issues

---

### DISM Logs

**Location:**
- WinPE: `X:\Windows\Logs\DISM\dism.log`
- Windows: `C:\Windows\Logs\DISM\dism.log`

**Analysis:**
```
# View last 50 lines
type X:\Windows\Logs\DISM\dism.log | more
```

**Common Errors:**
- `HRESULT=0x80070003` - Path not found
- `HRESULT=0x80070002` - File not found
- Access denied errors

---

### Windows Setup Logs

**Location:** `C:\Windows\Panther\`

**Key Files:**
- `setupact.log` - Setup actions
- `setuperr.log` - Setup errors
- `unattend.xml` - Answer file used (if present)

**When to Check:**
- Installation failures
- OOBE errors
- Driver installation issues

---

## Error Code Reference

| Error Code | Hex Code | Meaning | Common Context |
|------------|----------|---------|----------------|
| 2 | 0x80070002 | File not found | DISM, file operations |
| 3 | 0x80070003 | Path not found | DISM capture, oscdimg |
| 5 | 0x80070005 | Access denied | Permissions, Sysprep |
| 15634 | 0x3CF2 | Package error | Sysprep, Store apps |
| 1235 | 0x4D3 | Request aborted | DISM, user cancelled |

**When Encountering Errors:**
1. Note exact error code
2. Check relevant log files
3. Search online: "Windows deployment error [code]"
4. Check Microsoft documentation

---

## Prevention Best Practices

### Pre-Sysprep Checklist

- [ ] All applications installed and tested
- [ ] Windows fully updated
- [ ] Store apps removed
- [ ] Temporary files cleaned
- [ ] unattend.xml created and validated
- [ ] Backup of system created (optional)
- [ ] Two USB drives ready
- [ ] Tested one application launch each

### Pre-Capture Checklist

- [ ] Sysprep completed successfully
- [ ] Computer shut down (not restarted)
- [ ] Bootable USB inserted
- [ ] Storage USB inserted
- [ ] Know correct boot key for system
- [ ] Verified free space on storage USB

### Pre-Deployment Checklist

- [ ] Custom ISO created
- [ ] ISO integrity verified
- [ ] Bootable USB created with Rufus
- [ ] Target computer BIOS configured
- [ ] Product key available
- [ ] Network connectivity for activation
- [ ] Backup of important data (if applicable)

---

## Getting Additional Help

### Resources

**Official Documentation:**
- Microsoft Learn: Windows Deployment
- TechNet: Windows Sysprep
- Microsoft Q&A Forums

**Community Forums:**
- Reddit: r/sysadmin, r/Windows11
- Microsoft Tech Community
- Spiceworks Community

**When Asking for Help:**

Provide:
1. Exact error message
2. Error code (if present)
3. Phase where error occurred (Sysprep, Capture, Deploy)
4. Relevant log file excerpts
5. Windows version and build
6. Hardware specifications
7. Steps already attempted

---

## Emergency Recovery Procedures

### Can't Boot After Deployment

**Boot from USB → Repair → Command Prompt:**

```cmd
# Repair boot files
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd

# Repair Windows
dism /image:C:\ /cleanup-image /restorehealth
sfc /scannow /offbootdir=C:\ /offwindir=C:\Windows
```

### Corrupted Image Need to Redeploy

**Quick Redeploy:**
1. Boot from USB
2. Delete all partitions
3. Install fresh
4. Faster than troubleshooting in many cases

### System Unstable After Image Deployment

**Safe Mode Diagnosis:**
1. Boot to Safe Mode (press Shift while clicking Restart)
2. Troubleshoot → Advanced Options → Startup Settings → Safe Mode
3. Identify problematic driver/application
4. Remove and test

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Maintained by:** IT Community

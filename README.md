# Northstar Windows Imaging Lab (Sysprep + DISM)

Lab 8 of my **Northstar Information Systems (NIS)** IT home lab. I built a Windows 11 Enterprise reference VM, customized it, generalized it with Sysprep, captured it to a `.wim` file with DISM, and deployed that image to a blank VM. Everything used built-in Windows tools from the Windows 11 ISO; I did not install the Windows ADK or MDT.

This is the same capture-and-deploy process a desktop support team uses to give every new workstation the same standard build.

## Skills demonstrated

- Building a Windows 11 reference image in **Audit mode** (Ctrl+Shift+F3 at OOBE)
- Customizing the image (VirtualBox Guest Additions, OEM support info in the registry, an image stamp file) with a PowerShell script
- Generalizing with **Sysprep** (`/generalize /oobe /shutdown`)
- Booting the Windows Setup environment (Shift+F10) and using **DiskPart** to identify and prepare disks
- Capturing a custom image with **DISM `/Capture-Image`**
- Partitioning a blank UEFI/GPT disk (EFI, MSR and Windows partitions)
- Deploying with **DISM `/Apply-Image`** and making the disk bootable with **`bcdboot`**
- Verifying the deployment and troubleshooting a DISM failure (error 112)

## Environment

| Item | Details |
|---|---|
| Host | i7-11700K, 16 GB RAM, VirtualBox 7 |
| REF01 | Windows 11 Enterprise (26H2 eval), 4 GB RAM, 2 vCPU, 64 GB disk, EFI + TPM 2.0 + Secure Boot, **network not attached** |
| NEW01 | Blank VM with the same specs as REF01 |
| IMAGES.vdi | 32 GB second virtual disk holding the captured `.wim`, moved from REF01 to NEW01 |
| Tools | Sysprep, DiskPart, DISM and bcdboot from the Windows 11 ISO |

REF01 stayed offline the whole time so Windows Update and Store app updates couldn't break Sysprep.

## Workflow

```
REF01 (Audit mode) --> customize --> Sysprep generalize --> boot ISO --> DISM capture --> NIS-Win11.wim
                                                                                             |
NEW01 (blank disk) <-- bcdboot <-- DISM apply <-- DiskPart (GPT) <-- boot ISO <-------------+
```

### 1. Build the reference VM (REF01)
- Created REF01 with "Skip Unattended Installation" checked so I could control OOBE.
- Installed Windows 11 Enterprise. At the first OOBE screen I pressed **Ctrl+Shift+F3** to boot into **Audit mode** as the built-in Administrator, so no user profile ends up in the image.

### 2. Customize the image
- Installed VirtualBox Guest Additions.
- Ran [`scripts/Customize-REF01.ps1`](scripts/Customize-REF01.ps1), which:
  - sets Northstar **OEM support information** (Settings > System > About > Support)
  - writes `C:\Northstar\ImageInfo.txt` with the image name, source VM and build date, so the deployed machine can be traced back to REF01

### 3. Generalize with Sysprep
- Sysprep set to **Enter System OOBE**, **Generalize** checked, **Shutdown**.
- Took a VirtualBox snapshot (`Sysprepped - ready to capture`) as a rollback point, because booting a generalized VM back into Windows uses up the generalized state.

### 4. Capture with DISM
Booted REF01 from the Windows 11 ISO and pressed **Shift+F10** at the Setup screen. Then I prepared the second disk:

```
diskpart
list disk
list vol
select disk 1
create partition primary
format fs=ntfs quick label=IMAGES
assign letter=I
exit
```

Captured the Windows volume:

```
mkdir I:\Scratch
dism /capture-image /imagefile:I:\NIS-Win11.wim /capturedir:C:\ /name:"NIS Win11 v1.0" /compress:max /checkintegrity /verify /scratchdir:I:\Scratch
dism /get-imageinfo /imagefile:I:\NIS-Win11.wim
```

### 5. Deploy to NEW01
Attached IMAGES.vdi to a blank NEW01 VM, booted from the ISO and pressed Shift+F10. Then I partitioned the blank disk with a UEFI/GPT layout:

```
diskpart
select disk 0
clean
convert gpt
create partition efi size=100
format quick fs=fat32 label="System"
assign letter=S
create partition msr size=16
create partition primary
format quick fs=ntfs label="Windows"
assign letter=W
exit
```

Applied the image and created the boot files. In this boot session the IMAGES disk had the letter **C:**, as explained under Troubleshooting:

```
mkdir C:\Scratch
dism /apply-image /imagefile:C:\NIS-Win11.wim /index:1 /applydir:W:\ /scratchdir:C:\Scratch
bcdboot W:\Windows /s S: /f UEFI
wpeutil shutdown
```

### 6. Verify
- NEW01 booted from its own disk straight into a fresh **OOBE**, as a generalized image should.
- After OOBE:
  - the **Northstar support info** from REF01 appeared under Settings > About
  - `C:\Northstar\ImageInfo.txt` showed `Built on: REF01`
  - the computer had a new random name, showing Sysprep gave it a new identity
  - the registry `ImageState` was `IMAGE_STATE_COMPLETE`

```powershell
hostname; Get-Content C:\Northstar\ImageInfo.txt; (Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Setup\State').ImageState
Rename-Computer -NewName NEW01 -Restart
```

## Troubleshooting

### DISM error 112: "There is not enough space on the disk"
- **Symptom:** The capture failed at 46% with error 112, even though the I: drive had about 31 GB free and the compressed image is much smaller than that.
- **Cause:** By default DISM writes temporary files to the scratch directory on **X:**, the Setup environment's small RAM disk. A full Windows 11 capture filled it up.
- **Fix:** I created `I:\Scratch` and reran the capture with `/scratchdir:I:\Scratch`. It completed successfully. I added `/scratchdir` to the apply step as well.

### Drive letters change in the Setup environment
- On REF01 the Windows volume was C: and IMAGES was I:. On NEW01, IMAGES came up as **C:**, because the Setup environment assigns letters in whatever order it finds volumes.
- **Lesson:** Always run `list vol`, or search for the file, before running DISM. I used this one-liner:

```
for %d in (C D E F G H I J) do @if exist %d:\NIS-Win11.wim echo IMAGES is %d:
```

### No clipboard in the Setup environment
- Guest Additions doesn't run in the Setup environment, so I typed every command by hand.
- After deployment, Guest Additions was already in the image, so Shared Clipboard worked on NEW01.

## Screenshots

| # | File | Shows |
|---|---|---|
| 01 | [01-ref01-vm-settings.png](screenshots/01-ref01-vm-settings.png) | REF01 VM configuration |
| 02 | [02-ref01-setup-disk.png](screenshots/02-ref01-setup-disk.png) | Windows Setup disk selection |
| 03 | [03-ref01-audit-mode.png](screenshots/03-ref01-audit-mode.png) | Audit mode desktop with the Sysprep window |
| 04 | [04-ref01-guest-additions.png](screenshots/04-ref01-guest-additions.png) | Guest Additions installed |
| 05 | [05-ref01-customize-script.png](screenshots/05-ref01-customize-script.png) | Customization script and output |
| 06 | [06-ref01-oem-support-info.png](screenshots/06-ref01-oem-support-info.png) | Northstar support info on REF01 |
| 07 | [07-ref01-sysprep-options.png](screenshots/07-ref01-sysprep-options.png) | Sysprep: OOBE + Generalize + Shutdown |
| 08 | [08-ref01-storage-images-disk.png](screenshots/08-ref01-storage-images-disk.png) | REF01 storage: OS disk, IMAGES disk, ISO |
| 09 | [09-ref01-winpe-diskpart-list.png](screenshots/09-ref01-winpe-diskpart-list.png) | DiskPart `list disk` / `list vol` |
| 10 | [10-ref01-diskpart-format-images.png](screenshots/10-ref01-diskpart-format-images.png) | IMAGES volume formatted as I: |
| 11 | [11-ref01-dism-capture-complete.png](screenshots/11-ref01-dism-capture-complete.png) | DISM capture completed |
| 12 | [12-ref01-wim-imageinfo.png](screenshots/12-ref01-wim-imageinfo.png) | `dism /get-imageinfo` output |
| 13 | [13-new01-storage.png](screenshots/13-new01-storage.png) | NEW01 storage with IMAGES attached |
| 14 | [14-new01-diskpart-partitions.png](screenshots/14-new01-diskpart-partitions.png) | GPT partitions (S: and W:) |
| 15 | [15-new01-dism-apply-complete.png](screenshots/15-new01-dism-apply-complete.png) | DISM apply completed |
| 16 | [16-new01-bcdboot.png](screenshots/16-new01-bcdboot.png) | Boot files created |
| 17 | [17-new01-first-boot-oobe.png](screenshots/17-new01-first-boot-oobe.png) | NEW01 first boot into OOBE |
| 18 | [18-new01-oem-support-info.png](screenshots/18-new01-oem-support-info.png) | Northstar support info carried over to NEW01 |
| 19 | [19-new01-verify-image.png](screenshots/19-new01-verify-image.png) | Hostname, ImageInfo.txt, IMAGE_STATE_COMPLETE |

No passwords appear in any screenshot.

## What I'd do in production
- Use **MDT** or **Configuration Manager** (or Windows Autopilot for cloud-managed devices) instead of manual DISM commands.
- Use an **unattend.xml** answer file to automate OOBE, set the computer name and join the domain (`corp.northstar.local`).
- Add a **Recovery** partition (WinRE) to the deployment layout.
- Keep the reference image patched on a regular schedule and version the `.wim` files (v1.0, v1.1, and so on).

## Related Northstar labs
- [it-helpdesk-lab](https://github.com/SreekrishnaSiddi/it-helpdesk-lab): AD domain, PowerShell onboarding, shares, printers, GPO, RDP
- [northstar-pfsense-openvpn](https://github.com/SreekrishnaSiddi/northstar-pfsense-openvpn): pfSense firewall and OpenVPN remote access
- [northstar-wazuh-siem](https://github.com/SreekrishnaSiddi/northstar-wazuh-siem): Wazuh SIEM with agents on DC01 and CLIENT01
- [microsoft-365-azure-helpdesk-lab](https://github.com/SreekrishnaSiddi/microsoft-365-azure-helpdesk-lab): Entra ID and Microsoft 365 administration

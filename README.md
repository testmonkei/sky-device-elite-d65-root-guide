# Sky Device Elite D65 — MT6739 — Android 13 — Root Guide
### First Known Root | Magisk v26.4 | mtkclient | Kali Linux VM

---

## Section 0A — VM USB Setup (CRITICAL — Read First)

This is the most important section of the entire guide. If your VM does not
grab the device before Windows does, nothing else will work. BROM mode is
impossible if Windows sees the device first.

### Requirements
- VirtualBox 7.x
- VirtualBox Extension Pack (REQUIRED — USB passthrough won't work without it)
- USB 3.0 (xHCI) controller enabled
- Kali Linux installed to hard disk (NOT running from live ISO)

### Recommended VM Specs
```
RAM:        8192 MB (8GB)
Processors: 4
Storage:    80GB VDI dynamically allocated
USB:        xHCI (USB 3.0)
Boot Order: Hard Disk first, Optical second
```

### Step 1 — Enable USB 3.0 controller
```
VirtualBox → Your VM → Settings → USB
Enable USB Controller → Select USB 3.0 (xHCI)
```

### Step 2 — Add USB filter manually
```
VirtualBox → Your VM → Settings → USB → Click + icon

Name:          MediaTek
Vendor ID:     0e8d
Product ID:    (leave completely blank)
Manufacturer:  (leave completely blank)
Serial Number: (leave completely blank)
```

### Why Product ID must stay blank
The device changes its USB identity during the process:
```
Mode 1 (Normal):     Shows as SkyDevice Elite_D65 in Windows
Mode 2 (Preloader):  Shows as 0e8d:0000 MediaTek in lsusb
Mode 3 (BROM):       Shows as 0e8d:mt6227 in lsusb
```
The only consistent value across all three modes is vendor ID 0e8d.
That is all the filter needs.

### Step 3 — Verify in Kali
```bash
lsusb
# Look for: 0e8d MediaTek
```

> ⚠️ WARNING: Windows will ALWAYS grab the device first and open File Explorer.
> Do NOT try Devices → USB → SkyDevice in the VirtualBox menu.
> This does NOT work for BROM mode.
> The USB filter is the ONLY reliable method.

---

## Section 0B — Initial Kali VM Setup (Do This Before Anything Else)

### Step 1 — Update system FIRST
Before anything else update your system. Skipping this causes Guest Additions
failures and kernel mismatches:
```bash
sudo apt update
sudo apt upgrade -y
```
> ⚠️ If you see kernel updates during upgrade reboot before continuing:
```bash
sudo reboot
```

### Step 2 — Fix broken packages (if needed)
If you hit any errors during update run these in order:
```bash
sudo dpkg --configure -a
sudo apt --fix-broken install -y
sudo apt update && sudo apt upgrade -y
```

### Step 3 — Install Guest Additions
Guest Additions fixes screen resolution, performance, clipboard, and mouse
integration. Do not skip this.

First install required dependencies:
```bash
sudo apt install build-essential dkms linux-headers-$(uname -r) -y
```

Then in VirtualBox menu:
```
Devices → Insert Guest Additions CD Image
```

Then mount and run:
```bash
sudo mkdir -p /media/cdrom
sudo mount /dev/cdrom /media/cdrom
cd /media/cdrom
sudo sh VBoxLinuxAdditions.run
```

Reboot:
```bash
sudo reboot
```

### Step 4 — Fix fonts (if you see boxes instead of text)
```bash
sudo apt install fonts-dejavu fonts-noto fonts-noto-color-emoji -y
sudo reboot
```

### Step 5 — Install base essentials
```bash
sudo apt install git python3 python3-pip build-essential -y
```

### Step 6 — Set up clean working environment
```bash
mkdir ~/lab ~/scripts ~/notes
```

### Correct order every time
```
1. apt update && upgrade
2. reboot (if kernel updated)
3. install Guest Additions
4. reboot
5. install essentials
6. done
```

---

## Section 1 — ADB and Fastboot Configuration

> 💡 NOTE: This guide uses ZSH but all commands work the same in Bash.
> The only difference is where you save your config file:
> ZSH → ~/.zshrc  |  Bash → ~/.bashrc
> All terminal commands throughout this guide are identical for both shells.

---

### Install ADB and Fastboot
```bash
sudo apt install adb fastboot -y
```

### Step 1 — Start ADB
```bash
adb kill-server
adb start-server
```

### Step 2 — Verify ADB is running
```bash
adb devices
```
Expected output:
```
List of devices attached
```

### Step 3 — Prepare your phone
On the device:
- Enable Developer Options (Settings → About Phone → tap Build Number 7 times)
- Enable USB Debugging
- Set USB mode to File Transfer (MTP)

### Step 4 — Check device again
```bash
adb devices
```
> ⚠️ First time connecting: look at your phone and tap **Allow USB Debugging**

### Step 5 — Fix permissions (Kali udev rules)
```bash
sudo nano /etc/udev/rules.d/51-android.rules
```
Paste this line:
```
SUBSYSTEM=="usb", MODE="0666", GROUP="plugdev"
```
Save and exit, then run:
```bash
sudo chmod a+r /etc/udev/rules.d/51-android.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```
> ⚠️ You only need to do this ONCE. You do not need to reload udev rules
> every time you start Kali. This is a one time setup.

### Step 6 — Restart ADB after rules
```bash
adb kill-server
adb start-server
adb devices
```

### Step 7 — Test shell access
```bash
adb shell
```
Expected: device shell prompt

### Step 8 — Verify device info
```bash
adb shell getprop ro.product.model
adb shell getprop ro.build.version.release
adb shell getprop ro.board.platform
adb shell getprop ro.product.cpu.abi
```
> ⚠️ IMPORTANT: If ro.product.cpu.abi returns armeabi-v7a your device is
> 32-bit. This affects which version of Magisk you use. See Section 6.

### Step 9 — Test file access
```bash
adb shell ls /sdcard
```

---

### Fastboot Setup

### Step 10 — Reboot to bootloader
```bash
adb reboot bootloader
```

### Step 11 — Verify fastboot
```bash
fastboot devices
```
Expected:
```
device serial number
```

---

### Optional — Shell Aliases (saves time)

Open your shell config file:
```bash
# ZSH
nano ~/.zshrc

# Bash
nano ~/.bashrc
```

Add these lines:
```bash
alias adbstart='adb kill-server && adb start-server'
alias update='sudo apt update && sudo apt upgrade -y'
```

Save and reload:
```bash
source ~/.zshrc   # or source ~/.bashrc
```

---

### Common Issues

| Problem | Cause | Fix |
|--------|-------|-----|
| Device not showing | Bad cable | Use a DATA cable not a charge only cable |
| Unauthorized | Prompt not accepted | Accept Allow USB Debugging on phone |
| Kali not seeing device | udev rules | Follow Step 5 |
| Works on Windows not Kali | USB passthrough | Check VM USB filter from Section 0A |

---

### Final State Checklist
```
✅ ADB working
✅ Device detected
✅ Shell accessible
✅ Fastboot accessible
✅ udev rules configured (one time only)
```

---

## Section 2 — Verify MediaTek Device

> 💡 GOAL: Confirm Kali can see the phone, ADB works, Fastboot works,
> and identify your exact device configuration before touching anything.

> 💡 TIP: You can also verify your device info using the **DevCheck** app
> from the Play Store. It shows chipset, CPU, RAM and more at a glance.
> However always confirm critical info like slot and bootloader state
> using ADB commands — they are the most accurate source.

---

### Step 1 — Check USB detection
```bash
lsusb
```
Expected: Android / MediaTek / phone related USB device

If nothing shows:
- Try a DATA cable not a charge only cable
- Try another USB port

> ⚠️ If you try Devices → USB → SkyDevice Elite_D65US in the VirtualBox
> menu bar and get a "can't connect" error, it means Windows grabbed the
> device first. Do NOT rely on this method.
> Instead enable USB Tethering on the phone:
> Settings → Hotspot & Tethering → USB Tethering → toggle ON
> This is also why the USB filter in Section 0A is critical —
> it automatically passes the device to Kali before Windows can claim it.

---

### Step 2 — Restart ADB server
```bash
adb kill-server
adb start-server
```

---

### Step 3 — Check ADB device list
```bash
adb devices
```
Expected:
```
List of devices attached
SERIAL_NUMBER    device
```

**If device shows unauthorized:**
```
SERIAL_NUMBER    unauthorized
```
Fix: Look at phone screen and tap **Allow USB Debugging** then run:
```bash
adb devices
```

**If no device shows:**
- Set USB mode on phone to File Transfer / MTP
- If MTP fails enable USB Tethering:
  Settings → Hotspot & Tethering → USB Tethering → toggle ON
```bash
adb kill-server
adb start-server
adb devices
```

---

### Step 4 — Enter ADB shell
```bash
adb shell
```
Expected shell prompt:
```
Elite_D65:/ $
```
Exit shell:
```bash
exit
```

---

### Step 5 — Verify device info
```bash
adb shell getprop ro.product.model
adb shell getprop ro.product.name
adb shell getprop ro.product.device
adb shell getprop ro.board.platform
adb shell getprop ro.hardware
adb shell getprop ro.soc.model
```
Expected output for this device:
```
Elite D65
Elite_D65US
Elite_D65
mt6739
mt6739
MT6739
```

---

### Step 6 — Verify build info
```bash
adb shell getprop ro.build.display.id
adb shell getprop ro.build.fingerprint
adb shell getprop ro.build.version.release
adb shell getprop ro.build.version.sdk
```
Expected output for this device:
```
SkyDevices_Elite D65_V01_20240911
SkyDevices/Elite_D65US/Elite_D65:13/TP1A.220624.014/mp1V1121119:user/release-keys
13
33
```

---

### Step 7 — Verify bootloader state
```bash
adb shell getprop ro.boot.vbmeta.device_state
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop ro.boot.flash.locked
adb shell getprop sys.oem_unlock_allowed
```
Expected output for this device:
```
unlocked
orange
0
1
```
What this means:
```
unlocked = bootloader is unlocked
orange   = verified boot warning state (normal after unlock)
0        = flash not locked
1        = OEM unlock allowed
```

---

### Step 8 — Check active slot
```bash
adb shell getprop ro.boot.slot_suffix
adb shell getprop ro.boot.slot
```
Expected:
```
_a
a
```
> ⚠️ Note your active slot. You will need this when flashing with mtkclient.
> Always flash to your active slot.

---

### Step 9 — Verify partition list
```bash
adb shell ls /dev/block/by-name
```
Look for:
```
boot_a
boot_b
vbmeta_a
vbmeta_b
super
```

---

### Step 10 — Verify boot partition links
```bash
adb shell ls -l /dev/block/by-name/boot_a
adb shell ls -l /dev/block/by-name/boot_b
```
Expected output for this device:
```
boot_a → /dev/block/mmcblk0p27
boot_b → /dev/block/mmcblk0p42
```

---

### Step 11 — Check root status
```bash
adb shell
su
id
```
If rooted:
```
uid=0(root)
```
If not rooted:
```
su: inaccessible or not found
```
Exit:
```bash
exit
exit
```

---

### Step 12 — Optional: Check kernel info
```bash
adb shell uname -a
adb shell cat /proc/version
```
> 💡 This shows kernel version and build info only.
> It does NOT dump boot.img.

---

### Step 13 — Fastboot verify
```bash
adb reboot bootloader
fastboot devices
```
Expected:
```
SERIAL_NUMBER    fastboot
```

---

### Step 14 — Check fastboot variables
```bash
fastboot getvar unlocked
fastboot getvar current-slot
fastboot getvar all
```
Expected:
```
unlocked: yes
current-slot: a
```

**If fastboot shows no device:**
- Try another USB port
- Try another data cable
- Check: `lsusb`
- VirtualBox users: check USB filter from Section 0A

---

### Step 15 — Reboot back to Android
```bash
fastboot reboot
```

---

### Verified Device Summary
```
Model:            Sky Devices Elite D65
Variant:          Elite_D65US
Chipset:          MT6739
Android:          13
SDK:              33
Active slot:      _a
Bootloader:       unlocked
Verified boot:    orange
boot_a:           /dev/block/mmcblk0p27
boot_b:           /dev/block/mmcblk0p42
```

---

## Section 3 — Unlock Bootloader

> ⚠️ CRITICAL WARNING — READ BEFORE PROCEEDING:
> Unlocking the bootloader will FACTORY RESET your device.
> ALL data will be wiped. Back up anything important first.

---

### About .ab Backups
You can create an ADB backup (.ab file) before unlocking:
```bash
adb backup -apk -shared -all -f backup.ab
```
> ⚠️ WARNING: .ab backups can be unreliable. Some apps may not
> restore properly. Additionally apps that use root detection
> such as banking apps, Google Pay, and streaming services may
> deny access or block features after restore. Financial companies
> do this to protect their customers from fraud and unauthorized
> access. Manage your expectations with backups.

---

### Step 1 — Check bootloader state
From Section 2 Step 7 check your output:
```
unlocked = skip to Section 4
locked   = continue below
```

---

### Step 2 — Enable OEM Unlocking
On the device:
```
Settings → About Phone → tap Build Number 7 times
Settings → Developer Options → OEM Unlocking → toggle ON
```

---

### Step 3 — Reboot to bootloader
```bash
adb reboot bootloader
```

---

### Step 4 — Verify correct unlock command for your device

> 💡 NOTE: The fastboot oem unlock command can vary depending on
> the device. Before proceeding verify the correct unlock command
> for your specific device by running:
```bash
fastboot oem help
```
or:
```bash
fastboot --help
```
> This will show you the available fastboot commands supported by
> your device so you can confirm the correct unlock command.

---

### Step 5 — Unlock bootloader
```bash
fastboot oem unlock
```
> ⚠️ A prompt will appear on the phone screen asking you to confirm.
> Use volume buttons to select YES and power button to confirm.
> This will immediately wipe all data and reboot.

---

### Step 6 — After the wipe
The device will factory reset and reboot. You must now:
- Set up the phone from scratch
- Re-enable Developer Options
- Re-enable USB Debugging
- Set USB mode or USB Tethering

> 💡 The bootloader remains unlocked permanently even after factory
> reset. You do not need to unlock it again.

---

### Step 7 — The orange warning screen
Every time you boot the device you will see an orange warning screen
saying the device is not trusted or cannot be verified. This is
completely normal after unlocking. Do not panic.

```
Orange state = bootloader unlocked = working as expected
```

You can look up "Android orange state bootloader" online to see
exactly what the screen looks like and confirm this is normal.

---

### Step 8 — Verify unlock was successful
```bash
adb reboot bootloader
fastboot getvar unlocked
```
Expected:
```
unlocked: yes
```

---

### Re-verify device state from Section 2 Step 7
```bash
adb shell getprop ro.boot.vbmeta.device_state
adb shell getprop ro.boot.verifiedbootstate
```
Expected:
```
unlocked
orange
```

---

## Section 4 — MTKclient Setup + BROM Mode

> 💡 This section is where most people give up. Every error and fix
> is documented here so you don't have to figure it out the hard way.

> 💡 This is also where the USB filter you set up in Section 0A becomes
> critical. BROM mode is extremely timing sensitive and the device
> changes USB identity during this process. Without the USB filter
> set to vendor ID 0e8d Kali will miss the BROM window completely
> and mtkclient will never connect. If you skipped Section 0A go
> back and do it now before continuing.

---

### Step 1 — Install dependencies
```bash
sudo apt update
sudo apt install git python3 python3-venv python3-pip libusb-1.0-0 -y
```

---

### Step 2 — Download mtkclient
```bash
cd ~/Downloads
git clone https://github.com/bkerler/mtkclient mtkclient-main
cd mtkclient-main
```

---

### Step 3 — Create Python virtual environment
```bash
python3 -m venv venv
source venv/bin/activate
```

> 💡 Why venv? Kali Linux blocks direct pip installs with this error:
> `externally-managed-environment`
> The venv bypasses this completely. Always activate it before
> running any mtkclient commands.

---

### Step 4 — Install requirements
```bash
pip install -r requirements.txt
```

> ⚠️ If you get `externally-managed-environment` error:
> You forgot to activate the venv first. Run this:
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

---

### Step 5 — Fix libfuse error (Kali specific)

When you first run mtkclient you may hit this error:
```
OSError: Unable to find libfuse
```

**Why this happens:**
Kali only has `libfuse3.so.4` but mtkclient expects `libfuse.so.2`

**Fix:**
```bash
sudo ln -s /usr/lib/x86_64-linux-gnu/libfuse3.so.4 /usr/lib/x86_64-linux-gnu/libfuse.so.2
sudo ldconfig
```

---

### Step 6 — BROM connection sequence

> ⚠️ CRITICAL: The command must be running BEFORE you plug in the device.
> This is the most common mistake people make.

**Follow this sequence every single time:**
```
1. Phone OFF
2. USB unplugged
3. Run your command first
4. Hold VOL UP + VOL DOWN
5. Plug in USB
6. Hold buttons for 10-15 seconds
```

---

### Understanding the payload files

> 💡 Inside your mtkclient-main folder there is a directory:
> mtkclient/payloads/
>
> In there you will find:
>
> mt6739_payload.bin (or your device MT# payload)
> → Exploit payload used during BROM stage
> → Use the one that matches YOUR chipset number
>
> generic_preloader_dump_payload.bin
> → Used to dump the preloader from the device
>
> preloader_k39tv1_bsp_1g_k419.bin
> → This was the output from THIS specific device
> → Your device will generate a different filename
> → Whatever YOUR dumppreloader outputs is YOUR
>   preloader — use that exact filename for all
>   commands going forward
> → You can verify the correct preloader filename
>   for your device by searching online for your
>   chipset + preloader

> ⚠️ WARNING: Be very cautious about using DA files
> found online from other sources. Use ONLY the payload
> files that come with mtkclient-main. These are the
> ones that have been tested and verified to work.

---

### Step 7 — Test mtkclient
```bash
python3 mtk.py printgpt
```

> ⚠️ Common mistake — wrong command format:
> WRONG: `python3 mtk payload`
> RIGHT: `python3 mtk.py <command>`
> The `.py` extension is required every time.

---

### Step 8 — Dump preloader (CRITICAL STEP)

If printgpt gives you this error:
```
No preloader given
Stage wasn't executed
```

This means DRAM is not initialized and the DA cannot run.

**Fix — dump the preloader first:**
```bash
python3 mtk.py dumppreloader
```

Output:
```
preloader_k39tv1_bsp_1g_k419.bin  ← yours will be different
```

> 💡 This was the key breakthrough. The preloader initializes
> device memory (DRAM). Without it nothing else works.
> Every command from this point requires --preloader flag.

---

### Step 9 — Read partition table
```bash
python3 mtk.py printgpt --preloader preloader_k39tv1_bsp_1g_k419.bin
```

Expected output includes:
```
boot_a
boot_b
```

---

### Step 10 — Extract boot image
```bash
python3 mtk.py r boot_a boot.img --preloader preloader_k39tv1_bsp_1g_k419.bin
```

Verify:
```bash
file boot.img
```
Expected:
```
Android bootimg ✔
```

---

### Step 11 — Check boot_b slot
```bash
python3 mtk.py r boot_b boot_b.img --preloader preloader_k39tv1_bsp_1g_k419.bin
```

Verify:
```bash
hexdump -C boot_b.img | head
```

If output is all zeros:
```
00 00 00 00 ...
```
> 💡 On the Sky Device Elite D65 boot_b is empty. boot_a is your
> active slot and the only one you need to flash for rooting.
>
> However if you plan to build a custom recovery like TWRP or just
> want a complete backup of all partitions you can keep boot_b.img.
> Otherwise you can safely ignore it and delete it.

---

### Step 12 — Backup immediately
```bash
cp boot.img boot_backup.img
cp boot_b.img boot_b_backup.img
```

> ⚠️ Do NOT skip this. If anything goes wrong during flashing
> you can restore from this backup using mtkclient.

---

### About the USB device name changing

During the mtkclient process you may notice the device name
changes in lsusb:
```
MT6739 → MT6227
```
This is completely normal. It is standard USB re-enumeration
during the DA stage. The vendor ID 0e8d stays the same
which is why the USB filter from Section 0A keeps working
throughout the entire process.

---

### Common Issues Summary

| Problem | Cause | Fix |
|--------|-------|-----|
| externally-managed-environment | pip blocked by Kali | Use venv |
| Unable to find libfuse | Wrong libfuse version | Create symlink Step 5 |
| No such file or directory: mtk | Missing .py extension | Use python3 mtk.py |
| No preloader given | DRAM not initialized | Run dumppreloader first |
| Device not responding | Wrong connection sequence | Run command BEFORE plugging in |
| Windows grabbing device | VM USB passthrough | USB filter Section 0A |

---

## Section 6 — Magisk v26.4

> ⚠️ CRITICAL: Magisk version matters enormously on MT6739.
> This device is 32-bit (armeabi-v7a). Magisk v27 and newer
> dropped 32-bit support. You MUST use Magisk v26.4 specifically.
>
> Download here:
> https://github.com/topjohnwu/Magisk/releases/tag/v26.4

---

### Starting point
Before continuing verify you have:
```
✔ boot.img extracted via mtkclient
✔ verified: file boot.img → Android bootimg
✔ backup created: boot_backup.img
```

---

### Step 1 — Turn device ON for ADB

> ⚠️ Critical state change reminder:
> ```
> ADB commands = phone ON
> MTK commands = phone OFF
> ```
> This is one of the most common mistakes people make.
> You just finished mtkclient with phone OFF.
> Now turn the phone ON before continuing.

```bash
adb devices
```
If needed:
- Enable USB Debugging
- Accept RSA fingerprint prompt on phone

---

### Step 2 — Push boot.img to device
```bash
adb push boot.img /sdcard/
```

---

### Step 3 — Install Magisk v26.4
```bash
adb install Magisk-v26.4.apk
```

---

### Step 4 — Patch boot.img on device

On the phone:
```
Open Magisk app
→ Tap Install
→ Tap Select and Patch a File
→ Navigate to boot.img on sdcard
→ Tap Let's Go
```

Output file will be saved to:
```
/storage/emulated/0/Download/magisk_patched_[random].img
```

---

### Step 5 — Pull patched image back to PC
```bash
adb pull /sdcard/Download/magisk_patched_*.img
```

---

### Step 6 — Critical state change

> ⚠️ You must now switch states again:
> ```
> FROM: ADB → phone ON
> TO:   MTK → phone OFF
> ```
> Turn phone OFF before continuing.

---

### Step 7 — Flash patched boot image

Use the BROM connection sequence from Section 4:
```
1. Phone OFF
2. USB unplugged
3. Run command first
4. Hold VOL UP + VOL DOWN
5. Plug in USB
6. Hold 10-15 seconds
```

```bash
python3 mtk.py w boot_a magisk_patched_[yourfile].img --preloader preloader_k39tv1_bsp_1g_k419.bin
```

> 💡 Remember: use YOUR preloader filename from Section 4
> and YOUR magisk_patched filename from Step 5.

---

### Step 8 — Boot device normally

Power on the device normally. Open Magisk app.
It should show Magisk as installed with version number.

---

### Step 9 — Verify root
```bash
adb shell
su
id
```
Expected:
```
uid=0(root)
```

---

### About init_boot (important note)

Some newer Android builds use `init_boot.img` instead of `boot.img`
for Magisk patching. If root fails after following all steps check
if your device has an init_boot partition:
```bash
adb shell ls /dev/block/by-name | grep init_boot
```
If it exists you may need to patch init_boot instead of boot.

---

### Magisk version troubleshooting

| Symptom | Cause | Fix |
|--------|-------|-----|
| Bootloop after flash | Wrong Magisk version or AVB | Restore boot_backup.img and retry |
| Device boots but no root | Patch didn't apply | Try re-patching with v26.4 |
| Unable to unpack error | Version incompatibility | Confirm you are using v26.4 |

**If bootloop occurs — restore original boot:**
```bash
python3 mtk.py w boot_a boot_backup.img --preloader preloader_k39tv1_bsp_1g_k419.bin
```
Then:
1. Re-patch with Magisk v26.4
2. Re-flash
3. Reboot

---

### Magisk version notes for MT6739

```
Magisk v26.4 → recommended, tested, works on 32-bit ✔
Magisk v27+  → dropped 32-bit support, will fail
Magisk <v25  → sometimes useful for problematic devices
```

> 💡 The Magisk version was not the only issue on MT6739 but
> it IS the most common failure point. Always use v26.4 for
> this device. If you use a newer version and it fails,
> do not waste time debugging — just downgrade to v26.4.

---

### Final state checklist
```
✔ boot.img extracted and verified
✔ Magisk v26.4 installed on device
✔ boot.img patched successfully
✔ magisk_patched.img flashed via mtkclient
✔ Device boots normally
✔ Magisk shows installed
✔ root verified: uid=0(root)
✔ Device rooted!
```

---

## Credits
- mtkclient by bkerler — https://github.com/bkerler/mtkclient
- Magisk by topjohnwu — https://github.com/topjohnwu/Magisk
- Guide by [testmonkei]

---

## Tested On
```
Device:    Sky Device Elite D65
Build:     SkyDevice_EliteD65_V01
Android:   13 (Tiramisu)
SDK:       33
Chipset:   MT6739
Date:      April 2026
Status:    Fully rooted ✔
```

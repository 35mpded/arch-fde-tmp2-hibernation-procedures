#  LUKS (EXT4) + TPM2 (aka. AUTOUNLOCK) + SWAP TO FILE WITH HIBERNATION (aka. SUSPEND TO DISK) PROCEDURES
**WARNING: DATA LOSS CAN OCCUR IF ANYTHING GOES WRONG DURING THE SETUP! 
MAKE SURE TO HAVE A BACKUP OF YOUR DATA STORED ON A SAFE PLACE (PREFERABLY ON EXTERNAL STORAGE)**

These procedures were document with the goal of helping other people achieve a setup similar to Windows Bitlocker + TPM and avoid all of the fuss in doing so. 
As a person used to Windows Bitlocker + TPM (one of the better features of Windows over Linux in terms of data protection) and that shoulder surfing, recoding devices in public places or even worse - hidden such (personally in my opinion) outweigh the risk of cold-boot attacks, I decided I need the same implementation on Linux to avoid the password prompts, but still have a decent protection. Turns out, it ain't that easy to implement on Linux, which is a shame honestly. It ain't perfect but what is in terms of security anyway? 

Before starting with the below procedures there are several things you should be aware of.
* This guide requires a decent understanding of Linux, namely arch, but do not be afraid to try it out if you are new to linux, this experience can be a good teacher, just make sure to do a lot of research of each step and prepare yourself with patience (for a lot of reinstalls)!

*  **This will not work on non-arch based distros!**

* **Do your own research about the advantages and disadvantage over TPM and standard FDE**. Using TPM is not the most secure setup in the world but neither is FDE without TPM 

* **These procedures are made to work on Arch distros which use the Arch Linux Calamares Installer.** This is not to say the principle isn't the same but changes such as custom partition layout will require additional changes which will not be covered in this guide. 

* Both procedures (swap to file with hibernation and tpm2) are independent from one another and can be used as standalone guides/implementations if need be.

* Secure Boot will not be in the Scope of this setup. If I ever find the time to implement a working procedure for it, I'll update the guide. In the meantime, if you want to contribute please go ahead, I'll give all credits.

* **The TPM2 procedure may not work on some devices** 

# INTRODUCTION & REQUIREMENTS
**Tested on:**

-   OS: Garuda, EndevourOS and ArcoLinux. I expect all other ARCH distros with calamares to work as well without additional steps.
- Device Manufacturer: Lenovo


**Requirements:**

-   Not being afraid of linux and breaking it...
- Hardware TPM (most modern laptops and dekstop motherboards have it)
-   TPM chip enabled in UEFI settings
- UEFI (duh)
- GPT partitioning with ext4 (ext3 should work as well but I haven't tried it. **btrfs will not!**)
- OS installation using the following setup done via Calamares:

_**Warnings:**_

-   _Custom partition layout should will work but you need to adjust the crypttab and TPM bindings accordingly._
-   _This setup can work with Secure Boot if you sign your EFI image, but this guide will not go there..._
-   _If you are stuck on the loading screen press ESC (culprit is usually the playmouth hook)._
- btrfs will not work without additional steps which are, yet, not covered in this procedures.  
- If the procedures fail you will have a broken OS (at best), thus it is strongly suggested to backup your data (external storage) and follow the procedures on a clean slate.

# HIBERNATION PROCEDURE
**WARNING: DO NOT REBOOT YOUR SYSTEM UNTIL YOU'VE FINISHED ALL THE STEPS OR OTHERWISE SPECIFICALLY STATED TO REBOOT IN THE FOLLOWING SECTIONS. 
FAILING TO COMPLY MAY LEAD TO UNBOTABLE SYSTEM AND DATA LOSS!**


## SETUP

### 1. RESIZE AND ENABLE SWAP
See if any swap files are already in use by executing (output will list any swap files already active):
`sudo swapon -s`

To turn off swap by using:
`sudo swapoff -a`

The following will create a swap (mkswap) file with a specific size (dd), secure it by setting the permissions (chmod), and enable the swap (swapon) at `/swapfile`:
```
sudo dd if=/dev/zero of=/swapfile bs=1M count=1000
```

> dd will create a file "swapfile" with the size of 1GB (1M block size * 1000
> blocks). 
> if = input file   
> of = output file  
> bs = block size   
> count = multiplier of blocks

```
sudo mkswap /swapfile
sudo chmod 600 /swapfile
sudo swapon /swapfile
sudo mount -a
```
**Note:** 
Adjust the swapfile size according to your needs, use the following baseline for the size: 
 If you use suspend to disk (hibernation) it needs  to store the entire conents of RAM in the swapfile on disk so the swap should be big enough. 
  if you do not want to suspend to disk (hibernation) then the swapfile can be smaller.
***

### 2. UUID & PHYISICAL OFFSET OF THE SWAP
In order to use the SWAP file we need to update the grub config ``GRUB_CMDLINE_LINUX_DEFAULT`` line with 2 new parameters (`resume=` and `resume_offset`) and respective values for (`_swap_device_` and `_swap_file_offset_`).
* `resume=_swap_device_`
"\_swap\_device\_" is just a placeholder for the volume UUID where the swap file resides and it follows the same format as for the [root parameter](https://wiki.archlinux.org/title/Persistent_block_device_naming#Kernel_parameters "Persistent block device naming").
* `resume_offset=_swap_file_offset_`
"\_swap\_file\_offset\_" is just a placeholder for the phyisical offset of the swapfile created earlier in the **RESIZE AND ENABLE SWAP** section.

Following the below sections take a note of the **UUID** of the partition where the `_swap_file_` is located and the `_swap_file_` **phyisical offset**. 

**2.1 TO FIND THE UUID:** 
To find the LUKS volume UUID, on which the `_swap_file_` is located, run the following command (It will print out the volumes and their respective UUIDs): 
```
sudo blkid | grep -i "luks"
```
 If you have more than one luks device listed from the above command something is wrong with your setup and it is beyond the scope of this guide!
 
**Notes:**

One could also use the below command to find the UUID.
The required volume will contain look like this `/dev/mapper/luks-`**`84111cd6-8cd0-4b5e-86e8-599aefe9d94c`** but the bolded alphanumerical character will be different.
```
sudo blkid
```

**2.2 FIND THE OFFSET**
To find the find the `resume_offset=` of `_swap_file_`  run the commands:

> **Warning:** Do not try to use the filefrag tool, on [Btrfs](https://wiki.archlinux.org/title/Btrfs "Btrfs") the "physical"
> offset you get from filefrag is not the real physical offset on disk;
> there is a virtual disk address space in order to support multiple
> devices. [[1]](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation_into_swap_file_on_Btrfs)[[3]](https://bugzilla.kernel.org/show_bug.cgi?id=202803)

```
sudo filefrag -v /swapfile | tr -d '.' | cut -d ":" -f 1,3 | grep ' 0\:\|ext\:' | awk '{print $2}'
```


**Notes:**
If the above command fails, run the following`filefrag -v  _swap_file_`
The output is in a table format and the required value is located in the first row of the `physical_offset` column. 

For example:
```
Filesystem type is: ef53
File size of /swapfile is 4294967296 (1048576 blocks of 4096 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..       0:      38912..     38912:      1:            
   1:        1..   22527:      38913..     61439:  22527:             unwritten
   2:    22528..   53247:     899072..    929791:  30720:      61440: unwritten
```
In the example the value of `_swap_file_offset_` is the first `38912` with the two periods.



***
### 3. UPDATING GRUB & RAMDISK

**3.1 UPDATE THE GRUB CONFIG**

Edit the grub configuration by running the below command
```
sudo nano /etc/default/grub 
```
* Find the line containing `GRUB_CMDLINE_LINUX_DEFAULT=` and appended the`resume=` and `resume_offset=` parameters with their respective values. 
Example: `"quiet loglevel=3 `**`resume=/dev/mapper/luks-84111cd6-8cd0-4b5e-86e8-599aefe9d94c resume_offset=45731840`**` nowatchdog nvme_load=yes"`
*	Save the file once done

Generate the grub configuration file by executing the following command:
```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

**3.2 UPDATE THE RAMDISK**

Edit the ramdisk configuration file by executing the below command:
```
sudo nano /etc/mkinitcpio.conf
```
*	Find the line containing `HOOKS=` and append the **`resume`** hook. 
Example: `HOOKS="base udev autodetect moconf block keyboard keymap consolefront filestystem `**`resume`**` fsck"`.
* Save the file once done. 


Generate the initramfs:
```
sudo mkinitcpio -P 
```
**Reboot and test!** 
After the system is rebooted, you can try to hibernate via the hibernate button. When you start your machine again,, if all is working, it should continue from where it left off. If hibernation is not working double check for typos and if you followed the steps correctly.


# TPM2
**WARNING: DO NOT REBOOT YOUR SYSTEM UNTIL YOU'VE FINISHED ALL THE STEPS OR OTHERWISE SPECIFICALLY STATED TO REBOOT IN THE FOLLOWING SECTIONS. 
FAILING TO COMPLY MAY LEAD TO UNBOTABLE SYSTEM AND DATA LOSS!**

## 1 PREPARATIONS
**Important note before starting the setup:** Down the line (step 4.) you have to chose between 2 methods of TPM unlocking, whatever you chose, stick with it and skip the section of the opposite method, both methods cannot be implemented simultaneously! ***The author strongly recommends "Method 1 - Clevis"***

1. Edit the file `/etc/crypttab` and change from `/crypto_keyfile.bin luks` to `none discard`
Example: 

2. Delete the file `/crypto_keyfile.bin`

3. Edit the intial ramdisk conf file `/etc/mkinitcpio.conf` and change this line from: `FILES="/crypto_keyfile.bin"` to: `#FILES="/crypto_keyfile.bin"`

4. Edit `/etc/mkinitcpio.conf` and find the line which starts with `HOOKS=`,  just before the `encrypt` hook append the respective hook (`clevis` or `encrypt-tpm`) based of which method you would like to use.

	* **Example for Method 1 - Clevis:**</br>
	`HOOKS="base udev autodetect modconf block keyboard keymap consolefont clevis encrypt filesystems"`

	* **Example for Method 2 - Manual**</br>
	`HOOKS="base udev autodetect modconf block keyboard keymap consolefont encrypt-tpm encrypt filesystems"`

**NOTE**
 -  Whatever method you chose, remove the `plymouth` hook in your config if you have it, the hook is reported to induce bugs.

Proceed to the next section (**section 1.1** or for manual **section 1.2**)

## 1.1 METHOD 1 - CLEVIS
If you would like to use the manual method  (1.2 METHOD 2 - MANUAL) skip this section. If you otherwise want to use this method, go back to the section **1 PREPARATIONS**, make the required adjustments and return here.

**Setup**:

1. Install the following packages.
	```
	pacman --needed -S clevis tpm2-tools luksmeta libpwquality
	```
2. Add `clevis` binding to your LUKS device </br>
**Note:** Set the [PCR registers](https://wiki.archlinux.org/index.php/Trusted_Platform_Module#Accessing_PCR_registers) based on your paranoia setting...
	```
	clevis luks bind -d <device> tpm2 '{"pcr_bank":"sha256","pcr_ids":"0,1,2,4,7,8"}'
	```
3. Install the `clevis` hook from [mkinitcpio-clevis-hook](https://github.com/kishorv06/arch-mkinitcpio-clevis-hook)
	```
	git clone https://github.com/kishorv06/arch-mkinitcpio-clevis-hook.git
	cd arch-mkinitcpio-clevis-hook
	./install.sh
	```

4. Regenerate `initramfs` image.
	```
	mkinitcpio -P
	```
5. Reboot your system. Now your disk should get decrypted using the key from TPM.

**Note:**
If integrity on your system is changed, you will get prompted to manually enter the password for decryption since TPM will not be able to unseal the key.

It is actually recommended to test this.
1. Open your UEFI settings. 
2. Find the TPM settings (most common location is in security menu/tab).
3. Delete the keys.
4. Boot.  Now you will be notified that the TPM key could not be unsealed, and you will be prompted to enter a password for decryption, to fix this follow the next section **"Clevis Binding"**.

Proceed to section **3 GENERATE A UNIFIED KERNEL IMAGE**

## 1.2 METHOD 2 - MANUAL 

If you would like to use the clevis method  (1.1 METHOD 1 - CLEVIS) skip this section. If you otherwise want to use this method, go back to the section **1 PREPARATIONS**, make the required adjustments and return here.

1. Create the key
```
dd if=/dev/random of=/root/secret.bin bs=32 count=1
```

2. Add the key to luks
```
cryptsetup luksAddKey /dev/<luksDevice> /root/secret.bin
```

3. Add key to TPM
```
tpm2_createpolicy --policy-pcr -l sha1:0,2,4,7 -L policy.digest
tpm2_createprimary -C e -g sha1 -G rsa -c primary.context
tpm2_create -g sha256 -u obj.pub -r obj.priv -C primary.context -L policy.digest -a "noda|adminwithpolicy|fixedparent|fixedtpm" -i /root/secret.bin
tpm2_load -C primary.context -u obj.pub -r obj.priv -c load.context
tpm2_evictcontrol -C o -c load.context 0x81000000
rm load.context obj.priv obj.pub policy.digest primary.context
```

4. Unseal:
```sh
tpm2_unseal -c 0x81000000 -p pcr:sha1:0,7 -o /crypto_keyfile.bin
```
5. Install the hooks:</br>
```
git clone https://github.com/SubXi/GarudaLinux-FDE_and_TPM-Guide.git
chmod +x install.sh
sudo ./install.sh
```
6. Rebuild inital ramdisk 
```
mkinitcpio -P
```
7. Generate an Unified Kernel Image as per Section 3  .</br>

**Note:**
After reboot, you should get prompted to input the password manually. This behavior is expected since you changed your EFI image. Remove the key from TPM using the below command and redo step 3: 
```
tpm2_evictcontrol -C o -c 0x81000000
```

## 3 GENERATE A UNIFIED KERNEL IMAGE

**root (/) partition** is encrypted and it is safe to assume, without further verification, that it is not tampered with.  Unfortunately, EFI system partition (ESP) is not encrypted and anyone with a live CD/USB can tamper with it, meaning the ESP cannot be trusted thus we need to verify it some way. This verification though, by defaults, is not possible - when the UEFI loads the boot manager it will append the hash of the boot manager into PCR4, however, the bootmanager is not TPM-aware, meaning the kernel, initramfs and/or the kernel comand lines will not be verified. this essentially allows an attacker to use another kernel to gain access to the system. There is a solution to this problem provided by a systemd-boot feature known as Unified Kernel Image [[1]](https://wiki.archlinux.org/title/Unified_kernel_image), it allows to combine the bootloader, kernel, initramfs and the kernel command lines into a single file which will then be loaded as the "boot manager" which removes the need for an intermediate bootloader such as GRUB [[2]](https://wiki.archlinux.org/title/EFISTUB). End result is that if anything changes then the PCR4 will change!

**Notes:**
Although efibootmgr can  achieve the same result, it is not recommended to use it.  "Some firmwares do not pass command line parameters from the boot entries in NVRAM to the EFI binaries.[[1]](https://bbs.archlinux.org/viewtopic.php?id=178154)  In that case, the kernel and parameters can be combined into a  [unified kernel image](https://wiki.archlinux.org/title/Unified_kernel_image "Unified kernel image"), then create a boot entry with an  [.efi file](https://wiki.archlinux.org/title/EFISTUB#efibootmgr_with_.efi_file)."

1. Create a Unified Kernel Image:
	```
	objcopy \
	    --add-section .osrel="/usr/lib/os-release" --change-section-vma .osrel=0x20000 \
	    --add-section .cmdline="/proc/cmdline" --change-section-vma .cmdline=0x30000 \
	    --add-section .linux="/boot/vmlinuz-linux-zen" --change-section-vma .linux=0x40000 \
	    --add-section .initrd="/boot/initramfs-linux-zen.img" --change-section-vma .initrd=0x3000000 \
	    "/usr/lib/systemd/boot/efi/linuxx64.efi.stub" "/boot/efi/EFI/Garuda/grubx64.efi"
	```
	
	**Note:**
	* Change `/boot/efi/EFI/`**`_distro_name_`**`/grubx64.efi` to your distro, for example:`/boot/efi/EFI/Garuda/grubx64.efi`
	* If you are using LTS  or ZEN or whatever! Change `vmlinuz-linux-zen.img` and `initramfs-linux-zen.img` to `vmlinuz-linux-lts.img` and `initramfs-linux-lts.img` or  to whatever you are using.
2. Reboot your system. You will be prompted for password!  That is because you’ve just changed the EFI image and your PCR4 changed. (If it worked, then you’re in serious trouble as the TPM or the UEFI BIOS is not doing what it should.)

3. Generate the TPM binding
	(For METHOD 1 CLEVIS) Follow the steps for **Regenerate Clevis Binding** in section **4 COMMON CLEVIS COMMANDS**

5. Reboot and this time you should boot via TPM UNLOCK

## 4 COMMON CLEVIS COMMANDS
**Regenerate Clevis Binding**</br>
To regenerate a Clevis binding after changes in system's configuration which result in different PCR values:

1. Find the slot used for the Clevis pin (usually the last one)
`cryptsetup luksDump <luskDevice>`
2. Regenerate the Clevis binding, run:
`clevis luks regen -d <luksDevice> -s <keySlot>`
3. Reboot, the disk now should be decrypted using the key from TPM.


**Remove Clevis Binding**</br>
To remove a Clevis binding:

1. Find the slot used for the Clevis pin
`cryptsetup luksDump <luksDevice>`
2. Remove the Clevis binding, run:
`clevis luks unbind -d <luksDevice> -s <keySlot>`

## 5 SOME OTHER IMPORTANT NOTES ON DISK ENCRYPTION AND TPM2:
If the system unexpectedly asks for LUKS password after reboot it may indicate that your system was compromised.

Always test your system to see if the TPM is handled properly.</br> 
I suggest the following procedure:
1. Bind the TPM keys, test if you successfully boot without issues.

   A. If you don't boot: troubleshoot.
   
   B. If you do boot: continue to the next step.
   
2. Clear/delete the keys stored on TPM, preferably by using the UEFI menu.

   A. If the system boots: you have a serious issue and you should troubleshoot.
   
   B. If the system prompts for password input: you are good to go just regenerate the TPM binding.

***
Other:
1.	https://wiki.archlinux.org/title/Unified_kernel_image
2. https://wiki.archlinux.org/title/EFISTUB

Sources and relevant material:

https://wiki.archlinux.org/index.php/Trusted_Platform_Module

https://github.com/pawitp/arch-luks-tpm

https://pawitp.medium.com/full-disk-encryption-on-arch-linux-backed-by-tpm-2-0-c0892cab9704

https://github.com/kishorv06/arch-mkinitcpio-clevis-hook

https://bentley.link/secureboot/

https://github.com/archont00/arch-linux-luks-tpm-boot

https://github.com/saucepan14/TPMSecuredArch

https://github.com/electrickite/mkinitcpio-tpm-hook

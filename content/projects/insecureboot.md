---
title: "InsecureBoot"
date: 2025-01-01
categories: 
  - "projects"
tags: 
  - "binary-patching"
  - "bios-modding"
  - "permanent-secure-boot-disable"
  - "secure-boot"
  - "secure-boot-bypass"
  - "uefi-patching"
---

# Secure Boot bypass via firmware patching

## Background

Running unsigned or stealthily running self-signed EFI modules on a motherboard with properly implemented and enabled Secure Boot is theoretically impossible. Unsigned code gets blocked, and self-signed or hash-enrolled code allows for detection. To circumvent these restrictions, patching the return codes of the signature verification routines allows Secure Boot to remain active while allowing unsigned code execution.

Software can identify unauthorized EFI modules by parsing the firmware's trusted database (db) and checking for untrusted certificates or hashes, such as self-signed certificates or unknown hashes. If either is found, the software assumes the system has executed non-standard modules during the current boot. These modules could range from those designed to undermine system security to harmless ones, such as a Linux Unified Kernel Image.

This solution arose from my need to dual boot Linux and Windows while playing games protected by Riot Games' Vanguard, without constantly toggling Secure Boot in the UEFI firmware settings. To achieve this, I chose the path of least resistance: patching my firmware. Thanks, Riot Games, for forcing Linux users to compromise their firmware security.

## History

### SecureFakePkg

Previously, game integrity protection software accepted a hooked GetVariable that returned a fake Secure Boot status (as implemented in [SecureFakePkg](https://github.com/SamuelTulach/SecureFakePkg)). Kernel-mode software now detects hooks on GetVariable, requiring a new solution to enable free dual booting.

### MOKList

Previously, users could enroll their own keys or hashes into the MOKList, allowing self-signed binaries to be loaded and pass secure boot. MOKList is now checked for any non-OEM keys and users are blocked if unrecognized keys are found, harming legitimate Linux dual-booters once again.

### PatchBoot

Samuel Tulach's [PatchBoot](https://github.com/SamuelTulach/PatchBoot/) implements a modified [DxeImageVerificationHandler](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeImageVerificationLib/DxeImageVerificationLib.c#L1659) that returns EFI\_SUCCESS early. Requires manual extraction of SecurityStubDxe and identification of the image verification handler, followed by a byte patch of the handler.

## Alternative Method

To simplify the process, I propose an alternative method for patching SecurityStubDxe: replacing all occurrences of `EFI_SECURITY_VIOLATION` with `EFI_SUCCESS`. By referencing the [EDK II source code](https://github.com/tianocore/edk2) or Akira Moroo's [convenient list](https://retrage.github.io/2019/11/26/efi-status-code.html/), we find the following hex equivalents:

```
EFI_SECURITY_VIOLATION = 0x800000000000001A
EFI_SUCCESS = 0x0
```

With these values, we can ensure any loaded module always succeeds by replacing all instances of `0x800000000000001A` with `0x0000000000000000`. After performing some endian conversion, we can create a UEFIPatch-compatible patch to apply the changes in a single step.  
`1A00000000000080 -> 0000000000000000`

For some firmware implementations, replacing `EFI_ACCESS_DENIED` with `EFI_SUCCESS` may also be required, depending on platform policy. I have not yet tested this because the need has not arisen on my hardware.  
`0F00000000000080 -> 0000000000000000`

## Simplified PatchBoot Method

Manually extracting the SecurityStubDxe PE image and then patching it manually is error prone, to address this, I have created a signature based patch for that appears to work for a good chunk of motherboards. The following signature can be used to identify [DxeImageVerificationHandler](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeImageVerificationLib/DxeImageVerificationLib.c#L1659): 

IDA style:  
```hex
48 8B C4 48 89 58 08 48 89 ? 18 48 89 ? 20 48 89 50 10 ? 41 54 41 55 41 56 41 57 48
```

UEFIPatch style:  
```patch
F80697E9-7FD6-4665-8646-88E33EF71DFC 10 P:488BC4488958084889..184889..2048895010..415441554156415748:4831C0C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3
```

This patch replaces the start of [DxeImageVerificationHandler](https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeImageVerificationLib/DxeImageVerificationLib.c#L1659) with:  
```
xor rax, rax   
ret
```

Which returns 0 (`EFI_SUCCESS`) right away, the rest is padding it with `ret` because the UEFIPatch behaves better when we maintain the file size, and changing the binary size when patching is asking for trouble.

### Disclaimer

Patching your firmware is risky and can render your computer unbootable if done incorrectly. [UEFIPatch](https://github.com/LongSoft/UEFITool/tree/old_engine/UEFIPatch) has an issue with [breaking certain firmware images](https://github.com/LongSoft/UEFITool/issues/192), so you should pay attention to any warnings it produces. **Proceed with caution, and ensure you have an SPI flash recovery method.**

## Patching

To patch your firmware, I've provided a browser compatible [UEFIPatch](https://github.com/LongSoft/UEFITool/tree/old_engine/UEFIPatch), available at the link below. The tool performs all work locally on the device and no binaries are uploaded. Provide your firmware file, select your flavor of InsecureBoot and then "Patch".

[Online UEFIPatch](https://uefipatch.virtfunc.com/)

Alternatively, use your own version of UEFIPatch with the below patch, saving `patches.txt` to the directory of the UEFIPatch binary.

patches.txt (Alternative Method)

```patch
# Replace EFI_SECURITY_VIOLATION and EFI_ACCESS_DENIED with EFI_SUCCESS in SecurityStubDxe
F80697E9-7FD6-4665-8646-88E33EF71DFC 10 P:1A00000000000080:0000000000000000F80697E9-7FD6-4665-8646-88E33EF71DFC 10 P:0F00000000000080:0000000000000000
```

patches.txt (Simplified Patchboot Method)

```patch
# Return EFI_SUCCESS early in DxeImageVerificationHandler
F80697E9-7FD6-4665-8646-88E33EF71DFC 10 P:488BC4488958084889..184889..2048895010..415441554156415748:4831C0C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3
```

UEFIPatch usage:

```shell
UEFIPatch <UEFI_ROM_FILE> patches.txt
```

## Compatibility

This patch is not compatible with [AMD PSB](https://labs.ioactive.com/2024/02/exploring-amd-platform-secure-boot.html) or [Intel BootGuard](https://github.com/flothrone/bootguard/blob/master/Intel%20BootGuard%20final.pdf), unless you have already created a bypass for those. Attempting to load unsigned motherboard firmware will prevent the motherboard from booting. Before flashing patched firmware, verify that the firmware signature enforcement mechanism for your platform is not enabled. See [BootGuard / PSB Checker](https://virtfunc.com/projects/bootguard-psb-checker/) for a hastily written, and likely bug ridden tool to check this on Windows.

| CPU Vendor | Signature Enforcement Mechanism | Linux Status Check Procedure | Windows Status Check Proceedure |
| --- | --- | --- | --- |
| Intel | BootGuard | [https://github.com/felixsinger/bootguard-status](https://github.com/felixsinger/bootguard-status?tab=readme-ov-file#okay-how-can-i-check-bootguard-status) | [BootGuard / PSB Checker](https://virtfunc.com/projects/bootguard-psb-checker/) |
| AMD | Platform Secure Boot (PSB) | [https://github.com/mkopec/psb\_status](https://github.com/mkopec/psb_status) | [BootGuard / PSB Checker](https://virtfunc.com/projects/bootguard-psb-checker/) |

I believe this method to be compatible with most AMD AMI based firmware, and have tested it on 2 different models of MSI motherboards, AM4 and AM5. I have also done a dry run of the patching process on some Intel AMI and Insyde firmware, and the patching process does complete. However, I do not possess any intel hardware to verify that the resultant firmware is bootable or bypasses secure boot.

## Flashing

Given that this firmware does not have a valid signature, depending on your motherboard vendor, you may have issues flashing it.

| Vendor | Patching Resource |
| --- | --- |
| MSI | [https://virtfunc.com/info/msi-motherboard-flash-methods/](https://virtfunc.com/info/msi-motherboard-flash-methods/) |
| All others | [Win-Raid forum](https://winraid.level1techs.com/) |

I may write more guides for other vendors if I have time.

## Detection

After transitioning into virtual mode, SecurityStubDxe does not have a virtual memory mapping, because it is not a runtime driver. Data still may exist in memory, but is not valid.

Software could theoretically verify Secure Boot functionality by dropping an unsigned EFI executable that sets a flag on the system and then chainloads the OS bootloader. To silently execute the test, the software could also rewrite the BootNext variable to ensure the executable loads. However, due to poor UEFI Specification implementation on many motherboards, this approach often results in unbootable PCs. This method would also flag many motherboards that shipped with faulty Secure Boot implementations, looking at you MSI.

### ROM Analysis

Dumping of firmware remains the only viable detection method, and comes with its own issues related to compatibility, such as system lockup. Protecting your firmware against dumping remains an exercise for the reader.

[Many motherboards](https://us.msi.com/blog/statement-on-secure-boot) previously shipped with "Allow All" execution policies in Secure Boot settings, so by default, they exhibited behavior similar to that of this patch.

### TPM PCRs

If your TPM is enabled, loading different EFI executables will affect the PCRs. This alone is not a detection vector because PCRs only represent a hash of the computer's state and can change with different hardware and firmware configurations (such as updating firmware or changing UEFI settings). However, you will almost certainly get a different hash for your PCRs when loading your own UEFI code.  
  
While the PCRs themselves are not a detection vector, they are a hash of the logs that have been added to TPM event log. On a system with properly functioning measured boot, any loaded EFI module will produce an entry in the log.

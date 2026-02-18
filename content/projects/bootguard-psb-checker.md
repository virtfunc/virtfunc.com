---
title: "BootGuard / PSB Checker"
date: 2025-01-03
categories: 
  - "projects"
tags: 
  - "bootguard"
  - "platform-secure-boot"
  - "rweverything"
---

# Windows tool to check if motherboard has BootGuard or PSB enabled.

## Motivation

Before flashing unsigned firmware, one must be sure that BootGuard / PSB is not enabled on the platform in question. Identifying the state of these features on Windows can be a little annoying, so I built a simple program to read the relevant registers, using RwEverything’s kernel driver.

## Under the hood

The program identifies the CPU vendor, checks for Hyper-V and then proceeds if no Hyper-V and Intel processor are detected. If they both are, it gives the user the option to continue, which is risky.

On Intel processors only a single MSR must be read, then the status can be determined by looking MSR\_0x13A\[29:28\].

| MSR\_0x13A | BootGuard Status |
| --- | --- |
| `0x00000000` | None |
| `0x10000000` | Verified |
| `0x20000000` | Measured |
| `0x30000000` | Verified + Measured |

On AMD, the process is a little bit more complicated. PCI I/O must be performed using Index / Data ports on the PCI root complex. Writing [`0x03810994`](#references) to index port 0xB8, a `DWORD` can be read back from port `0xBC` that shows the status of PSB.

The status of PSB can be determined by the looking at the 24th bit: `(output >> 24) & 1)`

In order to get access to the registers to obtain this data, kernel mode access is needed. On Windows this requires a signed driver, or a self signed driver in test mode without secure boot. In the interest of ease of use, I opted to go the signed driver route, however I’m not paying Microsoft to sign my own driver, so I decided on RwDrv from RWEverything.

This driver is very useful because it provides a very convenient way to interface with the kernel, at the cost of being closed source and on the Microsoft vulnerable driver blocklist. Thus, disabling the blocklist is required to use the tool.

### Reversing RwEverything

Due to lack of source access, RwEverything's calls its kernel driver must be reverse engineered.

Using IDA Pro or any other disassembler, `DeviceIoControl` can be found in the imports of Rw.exe

![](/images/bootguard-psb-checker/kernel32_deviceioctl.png)

After finding the import, cross references to `DeviceIoControl` can be listed.  

![](/images/bootguard-psb-checker/deviceioctl_xrefs.png)

Here, `PciWriteDword` is going to be selected. The function takes 3 unsigned chars (`BYTE`), an unsigned short (`WORD`), and an unsigned long (`DWORD`).

![](/images/bootguard-psb-checker/pci_write_dword_asm.png)

After selecting PciWriteDword, the disassembly appears, and can be decompiled by pressing F5.

![](/images/bootguard-psb-checker/pci_write_dword_calling_c.png)

The decompiled code is alright, but not ideal, and we can see the 3 `BYTE` arguments (`a3`, `a5`, `a6`), the `WORD` argument (`a11`) and the `DWORD` argument (`a12`). By cleaning the above code up, we get the following code and command structure.

```
char __fastcall TReadWrite::PciWriteDword(TReadWrite *this, BYTE bus, BYTE dev, BYTE func, WORD reg, DWORD value)
{
  HANDLE *Instance; // rax
  DWORD BytesReturned; // [rsp+4Ch] [rbp-1Ch] BYREF
  CMD_PCI_RW_DWORD cmd; // [rsp+50h] [rbp-18h] BYREF

  cmd.bus = bus ;
  cmd.dev = dev ;
  cmd.func = func ;
  cmd.offset = reg;
  cmd.value = value;
  Instance = TRwDrv::GetInstance(this);
  DeviceIoControl(*Instance, 0x222844u, &cmd, 0xCu, &cmd, 0xCu, &BytesReturned, 0LL);
  return 1;
}
```

```
struct CMD_PCI_RW_DWORD // sizeof=0xC
00000000 {                                       
00000000     BYTE bus;                           
00000001     BYTE dev;                        
00000002     BYTE func;                         
00000003     // padding byte
00000004     WORD offset;                     
00000006     // padding byte
00000007     // padding byte
00000008     DWORD value;                    
0000000C };
```

Given the above, it is now easy to see how to create a usermode program to call the driver to write a `DWORD` to the PCI bus. The same steps must also be completed for other required calls.

## Status

- **This program is experimental and may cause a bluescreen**. **Save your data before executing.**
    - Disable Hyper-V/other hypervisors if you bluescreen.

- I don't actually know that it works.
    - None of my systems have PSB or BootGuard.
    
    - On both of my systems without PSB/BootGuard, it detects no PSB/BootGuard.

## Source / Binaries

GitHub: [https://github.com/](https://github.com/vanishingfork/psb_bg)[VirtFunc](https://github.com/VirtFunc/psb_bg)[/psb\_bg](https://github.com/vanishingfork/psb_bg)  
Binary Download: [https://github.com/VirtFunc/psb\_bg/releases/latest/download/psb\_bg.exe](https://github.com/VirtFunc/bootguard_psb_check/releases/latest/download/psb_bg_check.exe)

## References

- BootGuard detection method: [https://trmm.net/Security\_tools](https://trmm.net/Security_tools)

- PSB detection method: [https://github.com/mkopec/psb\_status/blob/master/psb\_status.sh](https://github.com/mkopec/psb_status/blob/master/psb_status.sh)
    - Uses coreboot method:
        - get\_psb\_status(): [https://github.com/coreboot/coreboot/blob/17848b65c38c32fa9630925ca8a15203a0617788/src/soc/amd/common/block/psp/psb.c#L86](https://github.com/coreboot/coreboot/blob/17848b65c38c32fa9630925ca8a15203a0617788/src/soc/amd/common/block/psp/psb.c#L86)
        
        - smn\_read32(): [https://github.com/coreboot/coreboot/blob/add685507b760a384e9b078a5695d86e8f10a1e9/src/soc/amd/common/block/smn/smn.c#L12](https://github.com/coreboot/coreboot/blob/add685507b760a384e9b078a5695d86e8f10a1e9/src/soc/amd/common/block/smn/smn.c#L12)

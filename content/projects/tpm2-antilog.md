---
title: "TPM2 AntiLog"
date: 2025-11-26
categories: 
  - "projects"
tags: 
  - "bios-modding"
  - "tcg"
  - "tpm2"
  - "uefi-patching"
---

TCG logs can be used to verify the boot chain against tampering, and are often used by software to check that the early boot sequence has not been tampered with, and thus that the kernel is (likely) intact.

However because the root of trust is often the SPI flash itself, such a system is vulnerable to patching of the routines that log and extend the TPM2 Platform Configuration Registers (PCRs). This post will discuss a simple patch that prevents logging of UEFI image hashes and extension of the PCRs related the boot sequence.

Given a firmware binary, the location of the code that performs the measurement must be identified. An easy way I have found to do this is to reference the [EDKII](https://github.com/tianocore/edk2/) source code and find functions relating to measurement, log creation and PCR extension. `[Tcg2HashLogExtendEvent](https://github.com/tianocore/edk2/blob/ad5030d73171d9aae30ee74c4b2cd41686b93bcd/SecurityPkg/Tcg/Tcg2Dxe/Tcg2Dxe.c#L1310)` is a function that resides in `Tcg2Dxe`, so analysis should be started by dumping the PE with this name or its associated GUID using a tool that can parse a UEFI firmware image, such as UEFITool.

`EFI_STATUS   EFIAPI   Tcg2HashLogExtendEvent (   IN EFI_TCG2_PROTOCOL *This,   IN UINT64 Flags,   IN EFI_PHYSICAL_ADDRESS DataToHash,   IN UINT64 DataToHashLen,   IN EFI_TCG2_EVENT *Event   )`

After extracting this binary, the image can be analyzed. I will be using IDA Pro with [efiXplorer](https://github.com/binarly-io/efiXplorer/) to more easily manage the GUIDs and automatic location of boot services functions. After performing analysis with efiXplorer, a list of multiple protocols interfaces was produced. `AMI_PROTOCOL_INTERNAL_HLXE` looked like a good place to start analysis.

![](/images/tpm2-antilog/efiXplorer_protocols.png)

Some other interfaces of interest were also found in the same call to  
`gBS->InstallMultipleProtocolInterfaces`.

![](/images/tpm2-antilog/interfaces_of_interest.png)

![](/images/tpm2-antilog/struct_ptr.png)

Given that this is an interface, it likely has other functions, but the function pointer at offset 0 of the pointer appears to perform hashing of the images because of a call to a function in `AMI_DXE_HASH_INTERFACE`.  

![](/images/tpm2-antilog/AMI_DXE_HASH_INTERFACE_caller.png)

Given this discovery, I am going to refer to this function as `AMI_PROTOCOL_INTERNAL_HLXE`, HLXE being short for Hash Log eXtend Event.

The function responsible for measuring the images in the DXE phase when they are loaded has been identified, and now creating the aforementioned patch is simple. By synchronizing the hex view with the disassembly, we can copy the function bytes to create a crude signature for this firmware.

![](/images/tpm2-antilog/hex_view.png)

The resultant bytes are:

`48 8B C4 48 89 58 10 4C 89 40 18 48 89 48 08 55 56 57 41 54 41 55 41 56 41 57 48 83 EC`

This appears to work for many modern AMI binaries, but for some older ones, the function signature is vastly different:

`4C 89 4C 24 20 4C 89 44 24 18 48 89 54 24 10 48 89 4C 24 08 48 81 EC E8 01 00 00 48 C7`

In either case the function can be patched out by replacing these bytes with the following instructions, padded with extra C3 (ret) to match the length of the search bytes. Obviously we do not care about the code we are overwriting because it will never be executed under our patch.

`48 31 C0 xor rax, rax   C3 ret`

The final patch, after converting to a format UEFIPatch can use, is as follows:

`# Return EFI_SUCCESS early in AMI's internal HashLogExtendEvent function.   39045756-FCA3-49BD-8DAE-C7BAE8389AFF 10 P:488BC4488958104C8940184889480855565741544155415641574883EC:4831C0C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3   39045756-FCA3-49BD-8DAE-C7BAE8389AFF 10 P:4C894C24204C89442418488954241048894C24084881ECE801000048C7:4831C0C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3C3`

This patch functions as expected on my MSI AM5 motherboard, here are the results of the PCRs after the patch. The firmware PCR remains but does not produce any meaningful logs related to the boot chain. It was redacted because I do not want these specific firmware images to be used for fingerprinting. The measurement of the firmware occurs in a different part of the boot chain, thus the PCR is populated.

![](/images/tpm2-antilog/final_pcrs.png)

A wasm based UEFIPatch (with this patch available to apply in a few clicks) is available at [https://uefipatch.virtfunc.com/](https://uefipatch.virtfunc.com/)

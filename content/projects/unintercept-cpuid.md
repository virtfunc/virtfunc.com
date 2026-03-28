---
title: "Unintercept CPUID"
date: 2026-03-25
categories: 
  - "projects"
tags: 
  - "kvm"
  - "svm"
  - "linux-kernel"
  - "virtual-machine"
  - "timing-detection"
---

## Introduction

Most hypervisors intercept the CPUID instruction to present a modified feature set to the guest. This behavior is critical for use cases like live migration across different hardware models, and disabling features that the emulator does not handle.

However, this interception introduces a detectable side effect. Timing attacks are a common method for virtual machine (VM) detection because VM exits are relatively expensive operations. On Intel CPUs that support Virtual Machine Extensions ([VMX](https://en.wikipedia.org/wiki/X86_virtualization#Intel_virtualization_(VT-x))), executing CPUID unconditionally causes a VM exit. Without manual mitigation of the transition overhead, the resulting latency serves as a flag of hypervisor presence.

However, AMD CPUs do not impose this restriction on guests virtualized under the Secure Virtual Machine ([SVM](https://en.wikipedia.org/wiki/X86_virtualization#AMD_virtualization_(AMD-V))) extension. An SVM guest can be configured to bypass `CPUID` interception by clearing bit 18 at offset `0xC` in the Virtual Machine Control Block (VMCB) ([Appendix B of AMD document 24593](https://docs.amd.com/api/khub/documents/68GKiN0gMEd6bMddsmhPwg/content?#G27.1021954.9Y)).

## Challenges in Implementation

Flipping this bit in the VMCB should, in theory, make a hypervisor immune to `CPUID` timing checks. However, bypassing the intercept passes the raw silicon data and `CPUID` leaves directly to the guest, which introduces several critical issues:

1.  **Topology Mismatches:** The host CPU topology may differ from the hardware allocated to the VM, such as running a 6-core VM on an 8-core host.
2.  **Feature Set Inconsistency:** Passing through raw data may change the feature set presented to the virtualized OS, which occasionally leads to system instability if changed at runtime.
3.  **Boot Failures:** The VM may fail to boot entirely. This is caused by CET not being implemented in current QEMU versions (v10.2.x), as well as some MSR issues.

The first issue is resolvable by matching the VM core count to the host and utilizing [libvirt pinning](https://libvirt.org/formatdomain.html#cpu-tuning) ensure the cores do not move between different physical cores and confuse the kernel.

The second issue can be addressed using tools like [arch\_enum](https://github.com/daaximus/arch_enum) to identify differences between host and guest features. Emulators like QEMU often expose emulated Intel features by default for AMD CPUs; these must be disabled to maintain stealth and stability. Depending on the method used for patching disabling intercepts on CPUID, this may not be needed.

The third issue is the most significant hurdle, as a VM that cannot boot is of questionable utility.

## Solution Theory

To work on fixing the third issue, we first need to patch QEMU to support CET for KVM guests. Recent linux kernels already have CET support in KVM, but QEMU does not virtualize it. A patch to enable support is available [here](https://patchwork.ozlabs.org/series/485044/mbox/).

A hacky solution involves disabling `CPUID` intercepts at runtime only after the guest VM has successfully initialized all CPU cores. While one could create a KVM parameter to toggle intercepts manually, doing so at every boot is tedious and error prone.

A more automated approach follows this logic:

  * Enable `CPUID` exits during the initial guest start sequence.
  * Hook a VM exit that reliably occurs after all cores have been initialized.
  * From that hook, loop through all vCPUs to disable their `CPUID` intercepts.
  * Conduct runtime tests without the performance overhead of `CPUID` transitions.
  * Restore the intercepts upon a guest reboot to ensure the next boot sequence succeeds.

## Implementation

Enabling `CPUID` exits during the initial startup sequence is the first—and only—action performed for us. After the cores are initialized, Windows appears to enumerate CPUID leaves starting from 0 up to the maximum value returned in `EAX` when `CPUID` is invoked with `EAX = 0` (which indicates the highest supported leaf). According to the AMD manual, leaves `0x00000008` through `0x0000000A` are reserved, meaning calls to `CPUID` with these values do not provide useful information and are most likely part of a simple enumeration loop.

Any leaf in this range is valid, so here lets choose `0x00000008` as our trigger point to disable CPUID intercepts. It is worth noting that this appears to be executed after boot start drivers have loaded. A simple CPUID hook that catches this could be implemented as:
```
#define RESERVED_LEAF 0x00000008
static int windows_boot_hook(struct kvm_vcpu *vcpu)
{
    u32 eax = kvm_rax_read(vcpu);
    int ret = kvm_emulate_cpuid(vcpu);
    if (eax == RESERVED_LEAF) svm_set_cpuid_intercept_for_all_vcpus(vcpu, false);
    return ret;
}
```
Here `svm_set_cpuid_intercept_for_all_vcpus()` is a function that sets a flag to update the CPUID intercept, as well as sets the desired state of the intercept. Then after setting the flags for all the cores, it kicks the CPUs so they have a chance to process the update. 

A reset/reboot detection hook could be implemented like below. `svm_set_segment()` gets called in cases other than processor resets, so we should do some filtering to make sure the core is the Bootstrap Processor (BSP), and is being reset.
```
static void svm_set_segment(struct kvm_vcpu *vcpu, struct kvm_segment *var, int seg)
{
  ...
    if (unlikely(vcpu->vcpu_id == 0 && seg == VCPU_SREG_CS &&
        var->base == 0xffff0000)) {//detect BSP x86 reset
          printk(KERN_INFO "kvm_amd: BSP reset detected\n");
          svm_set_cpuid_intercept_for_all_vcpus(vcpu, true);
    }
  ...
}
```
To actually update the state of the intercept, something like the following is used. The flag controlling the CPUID intercept update exists as a performance optimization: since `struct vcpu_svm` is already part of the fast path and typically cached, using a flag avoids making a comparison against the VMCB intercept region, and an unconditional `vmcb_mark_dirty()` call on every `svm_vcpu_run()`.
```
static __no_kcsan fastpath_t svm_vcpu_run(struct kvm_vcpu *vcpu, u64 run_flags)
{
  ...
    if (unlikely(svm->cpuid_update_req)) { // simple check instead of cmp against vcpu_svm
      svm->cpuid_update_req = false; // clear request
      if (svm->cpuid_intercept) svm_set_intercept(svm, INTERCEPT_CPUID);
      else svm_clr_intercept(svm, INTERCEPT_CPUID);
      vmcb_mark_dirty(svm->vmcb, VMCB_INTERCEPTS);
    } 
  ...
}
```
## Results
Using this patch, it is possible to fully bypass CPUID based timing attacks. With correctly configured firmware and QEMU, it is possible to obtain 0/91 detections on the latest version of [VMAware](https://github.com/kernelwernel/VMAware).

![](/images/unintercept-cpuid/vmaware_bypass.png)

## Alternatives
A simpler method involving passing through host `CPUID` for the entire VM runtime is possible, but outside the scope of this article. Using this method, the configuration of the guest `CPUID` in QEMU is completely irrelevant, because only what the silicon reports will be seen when executing `CPUID`.
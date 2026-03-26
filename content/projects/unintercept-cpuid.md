---
title: "Unintercept CPUID"
date: 2025-06-04
categories: 
  - "projects"
tags: 
  - "kvm"
  - "svm"
  - "linux-kernel"
  - "virtual-machine"
  - "timing-detection"
---

Timing attacks continue to be a common vector for VM detection, as VM exits are very time intensive. The time taken by VM exits can be measured using various timers to as a flag for hypervisor presence, if abnormally large.

On an Intel CPU, which provides the [VMX](include link) extension for virtualization, CPUID unconditionally causes a VM exit, so without manual timer mitigations the overhead will always be present, and as a consequnce, can be identified with a timing check.

Most hypervisors intercept the CPUID instruction, causing a VM exit. This is commonly done to present a different feature set to the guest, which has many use cases, for example, live migration.

However, AMD CPUs do not impose such a restriction on guests virtualized under their own virtualization extension, [SVM](include link). An SVM guest can be configured to not intercept CPUID by clearing bit 18 at offset 0xC in the Virtual Machine Control Block (VMCB). [include link]

So, great, with this knowledge we can go ahead and flip that bit in the VMCB and our hypervisor will be immune to CPUID based timing checks, right?

If only it were that simple.


By flipping that bit, we are passing through our silicon's raw CPUID leaves and data. This causes several issues:

1. Your CPU topology may differ from the hardware passed through to your VM, for example: a 6 core VM on an 8 core host.

2. You may have changed the feature set you are presenting to the virtualized OS, potentially causing instability.

3. Your VM fails to boot. (I believe this to be caused by mismatched APIC ids. The ACPI tables do not match the hardware.)

The first issue can be resolved by giving the virtual machine the same number of cores as the host. For performance it is also usually best to pin the cores to match the host topology. See [libvirt pinning](link).

The second issue can be resolved by using a tool such as [arch_enum](https://github.com/daaximus/arch_enum) identify feature differences between the host and the guest, using this information, features that do not match the host can be disabled. The default configuration of emulators such as QEMU often expose emulated Intel features and their corrosponding CPUID leaves. These should be disabled for stealth and stability.

The third and final issue is the one that scares most people away instantly. The VM does not boot.

A solution that I have found sufficient for my use, (although a little bit of a hack), is to disable CPUID intercepts at runtime, after the guest VM has had a chance to initialize all the cores of the VM. One can create a KVM parameter to toggle the intercepts of CPUID at will, and this works well, but is a little bit cumbersome to toggle the interception on and off every time at boot.

A better solution I have found is the following:

Enable CPUID exits at guest start.





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

## Introduction

Timing attacks remain a prevalent vector for Virtual Machine (VM) detection because VM exits are computationally expensive. On Intel CPUs providing the **[VMX](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)** (Virtual Machine Extensions) architecture, the `CPUID` instruction unconditionally triggers a VM exit. Without manual mitigation of the transition overhead, the resulting latency serves as a flag of hypervisor presence.

Most hypervisors intercept `CPUID` to present a modified feature set to the guest. This is essential for various use cases, such as live migration between different hardware models.

However, AMD CPUs do not impose this restriction on guests virtualized under the **[SVM](https://www.amd.com/system/files/TechDocs/24593.pdf)** (Secure Virtual Machine) extension. An SVM guest can be configured to bypass `CPUID` interception by clearing bit 18 at offset 0xC in the Virtual Machine Control Block (VMCB) **[See Page 737 of the APM](https://docs.amd.com/api/khub/documents/68GKiN0gMEd6bMddsmhPwg/content?#G27.1021954.9Y)**.

## Challenges in Implementation

Flipping this bit in the VMCB should, in theory, make a hypervisor immune to `CPUID` timing checks. However, bypassing the intercept passes the raw silicon data and `CPUID` leaves directly to the guest, which introduces several critical issues:

1.  **Topology Mismatches:** The host CPU topology may differ from the hardware allocated to the VM, such as running a 6-core VM on an 8-core host.
2.  **Feature Set Inconsistency:** Passing through raw data may change the feature set presented to the virtualized OS, which occasionally leads to system instability.
3.  **Boot Failures:** The VM may fail to boot entirely. I believe this is caused by mismatched APIC IDs where the ACPI tables do not align with the underlying hardware.

The first issue is resolvable by matching the VM core count to the host and utilizing **[libvirt pinning](https://libvirt.org/formatdomain.html#cpu-tuning)** to align the topologies.

The second issue can be addressed using tools like **[arch\_enum](https://github.com/daaximus/arch_enum)** to identify differences between host and guest features. Emulators like QEMU often expose emulated Intel features by default; these must be disabled to maintain stealth and stability.

The third issue is the most significant hurdle, as a VM that cannot boot is of questionable utility.

# A Solution
## Theory

A practical solution involves disabling `CPUID` intercepts at runtime only after the guest VM has successfully initialized all CPU cores. While one could create a KVM parameter to toggle intercepts manually, doing so at every boot is inefficient.

A more automated approach follows this logic:

  * Enable `CPUID` exits during the initial guest start sequence.
  * Hook a VM exit that reliably occurs after all cores have been initialized.
  * From that hook, loop through all vCPUs to disable their `CPUID` intercepts.
  * Conduct runtime tests without the performance overhead of `CPUID` transitions.
  * Restore the intercepts upon a guest reboot to ensure the next boot sequence succeeds.

## Implementation






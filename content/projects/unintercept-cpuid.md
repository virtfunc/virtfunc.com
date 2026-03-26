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

Most hypervisors intercept the CPUID instruction, causing a VM exit. CPUID is an instruction that typically is accessible to ring 3, makin



1
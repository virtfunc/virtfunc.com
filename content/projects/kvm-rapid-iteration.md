---
title: "KVM Rapid Iteration"
date: 2025-06-04
categories: 
  - "projects"
tags: 
  - "kvm"
  - "iterate-module"
  - "linux-recompile"
  - "rapid-iteration"
---

Rebuilding a the full linux kernel when modifying KVM for the purposes of patching VM exit RDTSC timings is a tedious process. Thankfully, the linux kernel is modular and can be built in pieces and incrementally upgraded.

This can be accomplished relatively safely by following the rough steps outlined below:

1. Fully build the kernel once, and loading this built kernel.

2. Compile only KVM as a module.

3. Kill all running VMs.

4. Remove existing KVM modules.

5. Load freshly built KVM modules.

6. Restart killed VMs.

On first use, the script may perform a full kernel build if it cannot find `vmlinux` in the source folder. Subsequent iterations skip the full build, reducing iteration time from ~10 minutes to approximately 5 seconds, plus the time it takes to reboot your virtual machines. 

To use the script:

1. Clone a copy of [AutoVirt](https://github.com/Scrut1ny/AutoVirt). (commit `c69721f` was tested.)

2. Drop the script in the `modules` folder.

3. Build the kernel by executing the kernel script in `modules` folder.

4. Make your changes to the linux kernel KVM source code on disk. 

5. Run the iteration script after killing all VMs.

6. Restart your VMs.

Note that a full build will revert to the tag specified. For iterative builds, the script uses the current source files on disk directly, no Git operations are performed.

# Source:  
Available at: [https://gist.github.com/virtfunc/2278f2f6f1d486521cb635813ff8dde7](https://gist.github.com/virtfunc/2278f2f6f1d486521cb635813ff8dde7)

Caveats and assumptions:

- Hardcoded to use `kvm_amd`, but should work on Intel processors with simple modifications.

- Expects to be present in the `modules` folder of an [AutoVirt](https://github.com/Scrut1ny/AutoVirt) git project. (Tested against commit `c69721f`.)

- Assumes Arch Linux, other distros untested.

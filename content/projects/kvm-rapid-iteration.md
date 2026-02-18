---
title: "KVM Rapid Iteration"
date: 2025-06-04
categories: 
  - "projects"
---

Rebuilding a the full linux kernel when modifying KVM for the purposes of patching VM exit RDTSC timings is a tedious process. Thankfully, the linux kernel is modular and can be built in pieces and incrementally upgraded.

This can be accomplished relatively safely by following the rough steps outlined below:

1. Fully building the kernel once, and loading this built kernel.

3. Compiling only KVM as a module.

5. Killing all running VMs.

7. Removing existing KVM modules.

9. Loading freshly built KVM modules.

On first use (or with the `--full` argument), the script will perform a full kernel build — this is necessary at least once. Subsequent iterations typically skip the full build, reducing iteration time from minutes to approximately 5 seconds, plus the time it takes to reboot your virtual machines. To use the script, simply make your changes to the source code and run it. Note that a full build will overwrite your modifications unless they are included as a userpatch. For iterative builds, the script uses the current source files on disk directly—no Git operations are performed.

Source:  
Available at: [https://gist.github.com/virtfunc/2278f2f6f1d486521cb635813ff8dde7](https://gist.github.com/virtfunc/2278f2f6f1d486521cb635813ff8dde7)

Caveats and assumptions:

- Only tested on AMD.

- Expects to be present in the `modules` folder of a [AutoVirt](https://github.com/Scrut1ny/AutoVirt) git project.

- Assumes Arch Linux, other distros untested.

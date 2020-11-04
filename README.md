# CMPE 283 Assignment2

*Haasitha Pidaparthi*

In this assignment you will learn how to modify processor instruction behavior inside the KVM hypervisor. Your assignment is to modify the CPUID emulation code in KVM to report back additional information when a special CPUID “leaf function” is called.

### Build the Kernel
* Clone the Linux repository using the following link: git clone https://github.com/torvalds/linux.git
```
haasi@haasi-vm:~$ git clone https://github.com/torvalds/linux.git
Cloning into 'linux'...
remote: Enumerating objects: 7749332, done.
remote: Total 7749332 (delta 0), reused 0 (delta 0), pack-reused 7749332
Receiving objects: 100% (7749332/7749332), 2.90 GiB | 5.67 MiB/s, done.
Resolving deltas: 100% (6431938/6431938), done.
Checking connectivity... done.
Checking out files: 100% (70695/70695), done.
```
* Follow the sequence of instruction
```
haasi@haasi-vm:~$ sudo bash
[sudo] password for haasi: 
root@haasi-vm:/home/haasi# apt-get install build-essential kernel-package fakeroot libncurses5-dev libssl-dev ccache bison flex libelf-dev
root@haasi-vm:/home/haasi# uname -a
Linux haasi-vm 5.4.0-52-generic #57-Ubuntu SMP Thu Oct 15 10:57:00 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
root@haasi-vm:/home/haasi# cp /boot/config-4.15.0-112-generic ./.config
root@haasi-vm:/home/haasi# cd linux/
root@haasi-vm:/home/haasi/linux# make oldconfig
root@haasi-vm:/home/haasi/linux# make && make modules && make install && make modules-install
root@haasi-vm:/home/haasi/linux# reboot
```
### Changes to the Source Code 
For CPUID leaf function %eax=0x4FFFFFFF:
* Return the total number of exits (all types) in %eax
* Return the high 32 bits of the total time spent processing all exits in %ebx
* Return the low 32 bits of the total time spent processing all exits in %ecx
  * %ebx and %ecx return values are measured in processor cycles

1. In cpuid.c (linux/arch/x86/kvm), add the following changes to the end of the file
```
if (eax == 0x4fffffff) {
  printk(KERN_INFO "Updating registers");
  kvm_cpuid(vcpu, &eax, &ebx, &ecx, &edx, true);
  printk(KERN_INFO "Updating exit counter, EAX = %llu", atomic64_read(&exit_counter));
  eax = atomic64_read(&exit_counter);
  printk(KERN_INFO "Number of exits in EAX = %u", eax);

  // Return high 32 bits of all exits in %ebx
  printk(KERN_INFO "Exit duration = %llu", atomic64_read(&exit_duration));
  ebx = (atomic64_read(&exit_duration) >> 32);
  printk(KERN_INFO "Updated exit duration for EBX = %u", ebx);

  // Return low 32 bits of all exits in %ecx
  ecx = (atomic64_read(&exit_duration) & 0xFFFFFFFF);
  printk(KERN_INFO "Updated exit duration for ECX = %u", ecx);
} else {
    kvm_cpuid(vcpu, &eax, &ebx, &ecx, &edx, false);
}
```
2. vmx.c (linux/arch/x86/kvm/vmx) - add exit counter to vmx_handle_exit() function
3. Rebuild the code using the following command: make && make modules && make install && make modules-install

### Install KVM
```
root@haasi-vm:/home/haasi/linux# sudo apt-get install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils
root@haasi-vm:/home/haasi/linux# sudo apt-get install virt-viewer
root@haasi-vm:/home/haasi/linux# sudo apt-get install virt-manager
```
* Download Ubuntu ISO image (Ubuntu 20.04.1 LTS)
* Using the installed Vitual Machine Manager (virt-manager), create a new VM using Ubuntu iso
  * Use this VM to run and complie the program
  
**Comment on the frequency of exits – does the number of exits increase at a stable rate? Or are there more exits performed during certain VM operations? Approximately how many exits does a full VM boot entail?**
* The number of exits were increasing each time the test file was executed. 
* The number of exits for the full test VM boot, from first build, to running on KVM approximately took 1200000 to 3600000. 

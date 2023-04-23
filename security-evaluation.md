# Security Evaluation Guide

DuVisor is able to prevent host kernel from crashing even if the user-level virtualization module is attacked.
As mentioned in the **Table 6** of our paper, we have tested and emulated the following CVEs: they could crash DuVisor itself, but the host kernel can continue to execute other programs (including DuVisor VMs) normally.

This guide shows how to evaluate the security of DuVisor by simulating representative KVM CVEs on it.

<!--ts-->
* [Security Evaluation Guide](#security-evaluation-guide)
   * [Prerequisite and Building](#prerequisite-and-building)
   * [<a href="https://nvd.nist.gov/vuln/detail/CVE-2017-12188" rel="nofollow">CVE-2017-12188 Memory Virtualization</a>](#cve-2017-12188-memory-virtualization)
   * [<a href="https://nvd.nist.gov/vuln/detail/CVE-2018-16882" rel="nofollow">CVE-2018-16882 Interrupt Virtualization</a>](#cve-2018-16882-interrupt-virtualization)
   * [<a href="https://nvd.nist.gov/vuln/detail/CVE-2016-8630" rel="nofollow">CVE-2016-8630 ISA Emulation</a>](#cve-2016-8630-isa-emulation)
   * [<a href="https://nvd.nist.gov/vuln/detail/CVE-2020-8834" rel="nofollow">CVE-2020-8834 VM Exit Handling</a>](#cve-2020-8834-vm-exit-handling)
   * [<a href="https://nvd.nist.gov/vuln/detail/CVE-2016-5412" rel="nofollow">CVE-2016-5412 Para-Virtualization</a>](#cve-2016-5412-para-virtualization)
   * [<a href="https://nvd.nist.gov/vuln/detail/CVE-2019-6974" rel="nofollow">CVE-2019-6974 Device Virtualization</a>](#cve-2019-6974-device-virtualization)
<!--te-->

## Prerequisite and Building

Hardware requirements:

* CPU: Commodity CPU with >= 4 cores which is able to run qemu. Architecture is not limitted.
* Memory: >8GB

First, clone the open-sourced DuVisor repository:

```bash
git clone https://github.com/IPADS-DuVisor/DuVisor -b security-ae
cd DuVisor
git submodule update --init --recursive
```

Other steps for preparation of the execution environment for security evaluation for DuVisor are the same as [Prerequisite](https://github.com/IPADS-DuVisor/DuVisor/tree/security-ae#prerequisite) and [Build DuVisor for QEMU](https://github.com/IPADS-DuVisor/DuVisor/tree/security-ae#build-duvisor-for-qemu).

All the following CVEs can be tested without rebooting or shutting down the qemu host VM. Therefore, you may boot the host VM only once.

If you use docker build in last step:

```bash
# pwd is root of the DuVisor repo
# Boot host VM
./scripts/build/docker_exec_wrapper.sh ./scripts/run/example-boot.sh

# Enter the directory storing the scripts
cd duvisor
```

If you use native build in last step:

```bash
# pwd is root of the DuVisor repo
# Boot host VM
./scripts/run/example-boot.sh

# Enter the directory storing the scripts
cd duvisor
```

## [CVE-2017-12188 Memory Virtualization](https://nvd.nist.gov/vuln/detail/CVE-2017-12188)

Description:

> arch/x86/kvm/mmu.c in the Linux kernel through 4.13.5, when nested virtualisation is used, does not properly traverse guest pagetable entries to resolve a guest virtual address, which allows L1 guest OS users to execute arbitrary code on the host OS or cause a denial of service (incorrect index during page walking, and host OS crash), aka an "MMU potential **stack buffer overrun**."

DuVisor does not support nested virtualization, but we emulate a **stack buffer overrun** in [stage-2 page fault handling](https://github.com/IPADS-DuVisor/DuVisor/blob/security-ae/src/duvisor/src/vcpu/virtualcpu.rs#L612) which would randomly crash DuVisor.

Execute the following command to emulate this CVE:
```bash
./boot-cve.sh 1

# Generate a large amount of stage-2 page fault
mkdir tmp
mount -t tmpfs tmpfs /tmp
dd if=/dev/zero of=/tmp/tmp bs=1G count=2
```

It would crash with output like the following:
```txt
Emulating CVE-2017-12188 (stack buffer overrun) in memory virtualizaion!
```

The host kernel can still run normally. You can continue to boot DuVisor VMs test other CVEs.

## [CVE-2018-16882 Interrupt Virtualization](https://nvd.nist.gov/vuln/detail/CVE-2018-16882)

Description:

> A **use-after-free** issue was found in the way the Linux kernel's KVM hypervisor processed posted interrupts when nested(=1) virtualization is enabled. In nested_get_vmcs12_pages(), in case of an error while processing posted interrupt address, it unmaps the 'pi_desc_page' without resetting 'pi_desc' descriptor address, which is later used in pi_test_and_clear_on(). A guest user/process could use this flaw to crash the host kernel resulting in DoS or potentially gain privileged access to a system. Kernel versions before 4.14.91 and before 4.19.13 are vulnerable.

We emulate a **use-after-free** in [interrupt virtualization](https://github.com/IPADS-DuVisor/DuVisor/blob/security-ae/src/duvisor/src/devices/vplic.rs#L71) which would randomly crash DuVisor.

Execute the following command to emulate this CVE:
```bash
./boot-cve.sh 2

# Execute ls to get a lot of interrupts from tty device
ls
```

It would crash with output like the following:

```txt
Emulating CVE-2018-16882 (use-after-free) in posted interrupt!
```

## [CVE-2016-8630 ISA Emulation](https://nvd.nist.gov/vuln/detail/CVE-2016-8630)

Description:

> The x86_decode_insn function in arch/x86/kvm/emulate.c in the Linux kernel before 4.8.7, when KVM is enabled, allows local users to cause a denial of service (host OS crash) via a certain use of a ModR/M byte in an **undefined instruction**.

We emulate an **undefined instruction** in [ISA emulation](https://github.com/IPADS-DuVisor/DuVisor/blob/security-ae/src/duvisor/src/vcpu/virtualcpu.rs#L223) which would randomly crash DuVisor.

Execute the following command to emulate this CVE:
```bash
./boot-cve.sh 3

# Wait and it would crash randomly.
```

It would crash with output like the following:

```txt
Emulating CVE-2016-8630 (undefined instruction) in ISA emulation!
```

## [CVE-2020-8834 VM Exit Handling](https://nvd.nist.gov/vuln/detail/CVE-2020-8834)

Description:

> KVM in the Linux kernel on Power8 processors has a conflicting use of HSTATE_HOST_R1 to store r1 state in kvmppc_hv_entry plus in kvmppc_{save,restore}_tm, leading to a **stack corruption**. Because of this, an attacker with the ability run code in kernel space of a guest VM can cause the host kernel to panic.

We emulate a **stack corruption** in [VM exit handling](https://github.com/IPADS-DuVisor/DuVisor/blob/security-ae/src/duvisor/src/vcpu/virtualcpu.rs#L757) which would randomly crash DuVisor.

Execute the following command to emulate this CVE:
```bash
./boot-cve.sh 4

# Wait and it would crash randomly.
```

It would crash with output like the following:

```txt
Emulating CVE-2020-8834 (stack corruption) in VM exit!
```

## [CVE-2016-5412 Para-Virtualization](https://nvd.nist.gov/vuln/detail/CVE-2016-5412)

Description:

> arch/powerpc/kvm/book3s_hv_rmhandlers.S in the Linux kernel through 4.7 on PowerPC platforms, when CONFIG_KVM_BOOK3S_64_HV is enabled, allows guest OS users to cause a denial of service (host OS **infinite loop**) by making a H_CEDE hypercall during the existence of a suspended transaction.

We emulate a **infinite loop** by calling `panic` in [para-virtualization](https://github.com/IPADS-DuVisor/DuVisor/blob/security-ae/src/duvisor/src/plat/opensbi/emulation.rs#L158) which would randomly crash DuVisor.

Execute the following command to emulate this CVE:
```bash
./boot-cve.sh 5

# Wait and it would crash randomly.
```

It would crash with output like the following:

```txt
Emulating CVE-2016-5412 (infinite loop) in para-virtualization!
```

## [CVE-2019-6974 Device Virtualization](https://nvd.nist.gov/vuln/detail/CVE-2019-6974)

Description:

> In the Linux kernel before 4.20.8, kvm_ioctl_create_device in virt/kvm/kvm_main.c mishandles reference counting because of a race condition, leading to a **use-after-free**.

We emulate a **use-after-free** in [device virtualization](https://github.com/IPADS-DuVisor/DuVisor/blob/security-ae/src/devices/src/serial.rs#L199) which would randomly crash DuVisor.

Execute the following command to emulate this CVE:
```bash
./boot-cve.sh 6

# DuVisor would crash if the tty device prints "DV-cve".
echo DV-cve
```

It would crash with output like the following:

```txt
Emulating CVE-2019-6974 (use-after-free) in device virtualization!
```

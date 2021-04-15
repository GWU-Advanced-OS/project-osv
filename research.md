## 1. Design and Implementation
## 2.
## 3. Security (Sarah)

The following details the information we have found with regard to the security of OSv and a limited analysis.

### OSv | Through Lens of Basic OS Security Principles

The security of OSv relies heavily on its claimed "reduced attack surface" and the isolation it gets from the hypervisor. Some of the security principles discussed in lecture are violated by OSv in the name of simplicity and efficiency. In some cases, the violation leaves OSv vulnerable, but in others the security principle itself makes little sense and seems unnecessary within the context of OSv.

#### Least Privilege
> "Each protection domain should have access to the minimal set of resources required to complete its task." - Textbook

- OSv has a single protection domain shared by the kernel and application.
    - Internally, it violates the principle of least privilege because everything is in ring 0
- OSv is a virtual machine to run a single application
    - The threat posed by violation of least privilege in OSv to the *host* it is running on can (generally) be reduced to the security of the hypervisor.

#### Economy of Mechanism
> "Keep the design and implementation as simple and small as possible." - Textbook

- OSv, by nature, is relatively small compared to other systems.
- Executing a single app and nothing more is (in general) fairly simple.
- OSv bases its C library on `musl` which is fairly common. This increases the chance a developer is familiar with the C library implementations and reduces the lines of code in their C library implementation.
- Differences from common implementations (absence of spin-locks, single protection domain, etc.) do introduce complexity for those unfamiliar with unikernels

#### Least Common Mechanism
> "Minimize the number of services that are shared between two principals." - Textbook

- Shared address space and resources between kernel and application in OSv seems to violate this principle
- On the other hand, only one application can run in OSv so "negatively impacting another principal" could
only refer to the kernel
  - if the "negative impact" causes errors in the kernel or crashes OSv, then the app has really only affected itself because it is the only thing running on the system
  - if the "negative impact" on the kernel has the potential to affect the host system, then the security is reduced to the security of the hypervisor and the host system
- Least common mechanism isn't the most useful metric for assessing security within OSv because OSv runs a single application at a time

#### Separation of Privilege
> "If the privilege to accomplish some task can be split up among different isolated protection domains such that they can coordinate to accomplish the task, system security will be increased." - Textbook

- Does not exist in OSv as kernel and app share single protection domain

#### Minimal Trusted Computing Base (TCB)

- Basically violated by design in OSv

#### Defense in Depth
> "This is concerned with, intuitively, encouraging the construction of as many barriers as possible, over which attackers need to jump." - Textbook

- OSv runs unmodified applications.
- To make changes to the system, the unikernel must be destroyed and rebuilt with the modifications.
- Though it is not a barrier within a running OSv system, it poses significant difficulty for anyone trying to modify a running system.

### OSv | Through Lens of Unikernel Security

Most of the security-focused research on OSv discusses the security of unikernels in general and includes a few specific notes about OSv. The following is a summary of the security pros and cons from one research paper followed by their relation to OSv. For issues where OSv is not specifically discussed in the paper we reviewed the source code to determine what mitigation strateies OSv employs.  

#### Background | Security in Unikernels

Summary of [[1](https://github.com/GWU-Advanced-OS/project-osv/blob/38e574e131b138e70942dd647a3fe22a56f58bc4/research/1911.06260.pdf)]:
- Legacy Unikernels
  - Focus on supporting unmodified software by implementing a subset of POSIX
  - Some re-implement Linux system call interfaces (includes OSv)
- Isolation: Software running on Unikernel is
  - more isolated from host/hypervisor than a container
  - less isolated from host/hypervisor than a full VM
- Security Overview
  - Unikernels that do not implement shells inherently protect themselves from attacks relying on the use of shell scripts
  - Lack of support for some system calls can add to Unikernel security. An attacker might then need detailed information about current state of the system (such as memory layout) to succeed.
  - Many unikernels simply rely on their limited nature to make claims about a reduced attack surface.
  - Smaller implementations can create vulnerabilities that might not have existed on more fleshed out systems
- Common Unikernel vulnerabilities and Limitations
  - Lack of Address Space Layout Randomization (ASLR)
  - Single Protection Ring
  - No guard pages to cause seg faults
  - Debugging is almost impossible while running
- Possible solutions
  - Increase entropy, disallow reuse of seeds
  - Host Hardening
    - limit access to host applications with small attack surface, protection against overflows, ASLR, and ecnryption
  - Library Hardening
    - support `_FORTIFY_SOURCE` macro to force common bound checks in C library implementations
  - Network Hardening
    - disable unnecessary services

#### Unikernel Security Topics in OSv

- **Protection Rings**
  - OSv has 1 protection ring and everything (kernel and app) runs in ring 0.
  - OSv does not use protection rings to increase security.
- **Guard Pages**
  - The OSv implementation generates page faults and this can be seen throughout memory allocation code and the `mmu` files, so it is doing (at least some) checks on memory access and usage.
  - [Issue 143](https://github.com/cloudius-systems/osv/issues/143) confirmed my suspicion that thread stacks have guard pages. The issue suggests a lazy allocation for threads (because they often use less than their default stack allocation) but it is still open. I don't believe any of the proposed changes have affected the use of guard pages in the current implementation.
- **Library Hardening**
  - [[1](https://github.com/GWU-Advanced-OS/project-osv/blob/38e574e131b138e70942dd647a3fe22a56f58bc4/research/1911.06260.pdf)] references the support for the `_FORTIFY_SOURCE` macro as an option for library hardening in unikernels. The macro forces the use of bound checking in common functions.
  - OSv claims (in the [libc README](https://github.com/cloudius-systems/osv/blob/3af01e018f0a5d360d55cba47051264cc6990fe5/libc/README.md)) to have implemented `FORTIFY`.
  - Indeed, in [`libc/__read_chk.c`](https://github.com/cloudius-systems/osv/blob/3af01e018f0a5d360d55cba47051264cc6990fe5/libc/__read_chk.c#L7-L9) (the read function called if `_FORTIFY_SOURCE` is set) a bound check is implemented. If the check fails, the [`libc/internal/_chk_fail.cc`](https://github.com/cloudius-systems/osv/blob/3af01e018f0a5d360d55cba47051264cc6990fe5/libc/internal/_chk_fail.cc) is called which aborts due to the failed check.
- **Network Hardening**
  - By design, OSv does not come with all the services of a general purpose OS.
  - Unfortunately, it has not done enough.
  - "... in OSv the REST API used to control the Unikernl can replace the command line, read and write files and directories. Exposing it to attackers gives them command execution, Local and Remote File Inclusion." - [[1](https://github.com/GWU-Advanced-OS/project-osv/blob/38e574e131b138e70942dd647a3fe22a56f58bc4/research/1911.06260.pdf)]

# References

1. [A Security Perspective on Unikernels](https://github.com/GWU-Advanced-OS/project-osv/blob/38e574e131b138e70942dd647a3fe22a56f58bc4/research/1911.06260.pdf)

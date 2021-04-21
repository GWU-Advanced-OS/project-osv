## Technology

The cloud design allows OSv to simplify the stack, both looking down (towards the hardware) and looking up (towards the application).

In this section, we will talk about the core technologies being used in OSv.

### 1. Platform

OSv could be run based on several hardwares including Xen, KVM and VMware. In those environments, OSv dramatically simplifies the I/O stack. Because the hypervisor multiplexes the hardware into different applications, the application runs in the operating kernel's address space and no longer to consider the hardware, which increases the efficiency o system calls and context switches.
the operating system using OSv no longer needs to do this.

### 2. Java Virtual Machine integration

JVM has the advantage that allows OSv to expose hardware features, such as the page tables directly to JVM. Because of that, JVM could manage the memory more efficiently in several aspects. At first, JVM could use large pages directly and use the page tables to track memory modifications and aid with garbage collection. Also, JVM using OSv remaps the memory instead of copying large swept object. Moreover, JVM could achieve the cooperative scheduling and prioritization of critical GC threads.

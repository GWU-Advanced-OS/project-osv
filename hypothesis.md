## Questions

1. Summarize the project, what it is, what its goals are, and why it exists.
2. What is the target domain of the system? Where is it valuable, and where is it not a good fit? These are all implemented in an engineering domain, thus are the product of trade-offs. No system solves all problems (despite the claims of marketing material).
3. What are the "modules" of the system (see early lectures), and how do they relate? Where are isolation boundaries present? How do the modules communicate with each other? What performance implications does this structure have? 
4. What are the core abstractions that the system aims to provide. How and why do they depart from other systems?
5. In what conditions is the performance of the system "good" and in which is it "bad"? How does its performance compare to a Linux baseline (this discussion can be quantitative or qualitative)?
6. What are the core technologies involved, and how are they composed?
7. What are the security properties of the system? How does it adhere to the principles for secure system design? What is the reference monitor in the system, and how does it provide complete mediation, tamperproof-ness, and how does it argue trustworthiness?
8. What optimizations exist in the system? What are the "key operations" that the system treats as a fast-path that deserve optimization? How does it go about optimizing them?
9. Subjective: What do you like, and what don't you like about the system? What could be done better were you to re-design it?


## Hypothesis

1. Most cloud systems run some general version of Linux. However, Linux can be very bloated with isolation features that the hypervisor already gives, especially when a user only wants to run a single application. OSv offers a way to boot a single linux application with less overhead and more memory to give to the application. 

2. The target domain of OSv is in the Cloud Computing and Virtual Machine realm. On a physical machine a single OS would handle isolation, between multiple untrusted applications. However, on the cloud the hypervisor handles a lot of the isolation. This means there is an unnesecary cost to provide this isolation which OSv takes advantage of. On the other hand, OSv would not be a good idea on a physical machine or something like a Desktop OS. We can only run a single application. 

3. OSv has many [components](https://github.com/cloudius-systems/osv/wiki/Components-of-OSv) in order to allow any Linux application to be able to run on OSv. This also provides a good summary of the goals of OSv.  We summarize the following components here: 

- **Virtual hardware drivers**
OSv needs to be able to support a minimal set of virtual hardware. OSv supports some traditional PC hardware as well as some paravirtual drivers. 

- **Filesystem**
OSv uses a file system similar to Unix "VFS" to allow application to interact with a filesystem. 

- **The Loader**
OSv needs to bootup before running. The loader loads and compresses the OSv kernel into memory. 

- **The Linker**
OSv runs applications. Therefore a linker is needed to map the application into memory and provide a map with symbols and where they are so that the code is runnable. Something to note here is that only dynamically-linked executables are able to be run. 


- **Memory Management**
OSv needs to support applications that run things like mmap or malloc. A memory manger is needed to support these operations and decide how many pages to give and when to revoke the pages. 

- **Scheduler**
OSv needs to support threads. A thread scheduler will support multiplexing threads on CPUs, guarantee fairness and provide load balancing. It is said that the scheduler is quite different than Linux. 

- **Synchronization**
OSv needs to suport mutexs. However, here they do not use Spinlocks and instead use a "lock-free" mutex and lock-free algorithms. 

- **Supporting C library**
OSv needs to support Linux application who rely on the C library. Hence they need to implement traditional Linux system calls and glibc calls. 

- **Network Stack**
OSv needs to support applications that want to use the network. OSv has an entire TCP/IP network stack. 

- **DHCP client**
OSv needs to be able to support gateway applications. Therefore a DHCP client is needed to be implemented in OSv. 

TODO: Explain tradeoffs of providing or not providing some modules. How do these modules communicate

4. OSv tries to provide a standard Linux API so that native Linux applications can be run using OSv. However, we can take advantage of the fact that only one application is being run in OSv. For example there are APIs in OSv that give applications direct accsess to a page table. This can be extremely benifical to something like Java's Garbage collector. The paper can explain it a lot better than I can: 

> The Hotspot JVM uses a data structure called a card table [22] to track write accesses to references to objects. To update this card table to mark memory containing that reference as dirty, the code generated by the JVM has to be followed by a “write barrier”. This additional code causes both extra instructions and cache line bounces. However, the MMU already tracksn write access to memory. By giving the JVM access to the MMU, we can track reference modifications without a separate card table or write barriers.

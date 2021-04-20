\newpage

# Target Domain

In this section we will discuss where OSv is a good fit, and where OSv might not be a good fit.
We will dive into two possible scenarios and see how different aspects of OSv might hinder or help a certain situation.

## A Memcached application
Memcached is an in memory key value system. It allows us to not have to visit the "slow" path of acsessing the database every time, instead we can accsess a fast cache that has information that we use often. This is a *good* use case for OSv for the following reasons: 

- It is a single application system
- It is memory intensive

### Single Application Systems
Single applications like memcached are a good fit for OSv because they only rely on a single address space. When we fuse an application to OSv there is only one address space that an application can acsess. This means we can get added benifits that system calls are as fast as regular function calls. 

### Memory Intensive
Memcached is a memory intensive system as it focuses on caching. Memory intensive operations are fast in OSv because we only have one address space. This means we can do things like giving an application direct accsess to page tables which can tremendously speed up memory operations. 

## Running multiple applications on baremetal
However, there are cases where running OSv for an application may not be the best choice. An example of this would be running multiple applications on bare metal hardware. There are two main reasons why this might be a *bad* use case for OSv: 

- There is no hypervisor
- There are multiple applications

### Hypervisor
It is required that OSv is running on top of a hypervisor like Qemu or KVM. This is because OSv is focused on running applications on the cloud which have a hypervisor. 

### Multiple Applications
OSv will not provide memory abstractions such as seperation of address spaces into a kernel and userspace. If there are no memory abstractions all applications will have acsess to all other applications memory. Therefore running OSv for multiple applications is a bad idea. 

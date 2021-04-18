## 1. Design and Implementation (Cuidi Wei)
### 1. Memory
OSv runs an application with the kernel and threads sharing a single space. It means the threads and kernel use the same tables, which make system calls as efficient as function calls and also make context switches quicker. When OSv is going to run an application, it will create the share space.
```c
shared_app_t application::run_and_join(const std::string& command,
                      const std::vector<std::string>& args,
                      bool new_program,
                      const std::unordered_map<std::string, std::string> *env,
                      waiter* setup_waiter,
                      const std::string& main_function_name,
                      std::function<void()> post_main)
{
    auto app = std::make_shared<application>(command, args, new_program, env,
                                             main_function_name, post_main);
    app->start_and_join(setup_waiter);
    return app;
}
```
Also, OSv uses the huge page so that it could reduce the number of TLB misses.
In this case, the OSv couldn’t do the isolation. The isolation is managed by the hypervisor.

## 2. Applications & Build Process (Graham)

### Applications 
In this section I will dive deep into a variety of applications that can be run with OSv, and if there is necessary glue that needs to be used for each application to run. I will also look to see how different application have different needs from OSv and if that effects their build process. 

#### 1. ImageMagick

- ImageMagick is a C image manipulation library. It has the ability to convert images into different sizes and add special effects to images.

- It is very important to add as least as possible to ImageMagick, because the libraries are very old and have undergone extensive testing. 

##### Building ImageMagick

In order to build ImageMagick we can look at the [build file](https://github.com/cloudius-systems/osv-apps/blob/43e5d7d35c91048164c7cd58750f15dc53ddef12/ImageMagick/Makefile) that we are not actually building from source, but instead building from a library that is already pre installed on our computer. We can notice 2 distinct steps: 

- Creating the usr manifest
- Converting the library to a shared object that can be booted in OSv

##### Creating a shared object
Creating a shared object from a library is actually quite complex. From the [convert.c](https://github.com/cloudius-systems/osv-apps/blob/43e5d7d35c91048164c7cd58750f15dc53ddef12/ImageMagick/convert.c#L4) file that is provided we can see that we are essentially loading all that we can from the library into the compiled c file. We can see the compilation command as 

```bash
	cc -shared -o convert.so -fPIC `pkg-config --cflags ImageMagick` convert.c `pkg-config --libs MagickWand`
```

To me it looks like we are essentially compiling the library into some position independent code which is then used as a shared object. 

From this we can see what we have to do in order to compile something like a library that a host machine has into a Uni-kernel

#### 2. XApps 
- XApps are essentially just application that use the windows manager. We can think of this as widgets like a desktop clock or a calculator. 

- Xapps are special in this case because they require the use of the display of the host machine. We will explore the work that needs to be done in order to specify how to do this. 

##### Building our XApps
In order to get the object to build into OSv it seems we only need to specify the path that our apps would normally be on the host machine: 

```Makefile
xclock_path := $(shell which xclock)
xeyes_path := $(shell which xeyes)
xlogo_path := $(shell which xlogo)
```

##### Running our XApps
Although it seems pretty easy to build our XApps, we need to have some special considerations when running our XApps in our Unikernel. Our Unikernel will use [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) in order to communicate with our host machine and display our XApps. 

When we are actually running our App we use the following code: 

```python
common_prefix = '--env=XAUTHORITY=/.Xauthority --env=DISPLAY=192.168.122.1:0.0'

default = api.run(cmdline='%s /xclock -digital -fg red -bg black -update 1 -face "Arial Black-25:bold"' % common_prefix)
```

Here we can see that we get our permissions from the host machine locating in the XAuth directory. We also see that we set a display, this is the port we are sending our packets to in order to display our applications. We can see this is our machine because we are using `192` which means the computer is in our network. 

As we can see from running XApps each application can have its own build needs which can lead to each Uni-kernel build getting more and more complex. 


#### 3. OSv-Memcached 
- Memcached is a memory caching system for small pieces of data (like in a browser). Memcached can prevent unnecessary database queries.

- Memcached is one of OSv's bragging points. It is focused on memory and key value stores which OSv does really fast as it only has to deal with one address space as well as several other optimizations. 

##### 2 types of Memcached
We have 2 options when deciding to build Memcached with OSv. 
- Default Memcached source code

- Specialized Memcached (OSv Memcached) which is a lot faster. 

In this Memcached we examine how we can achieve faster operations by taking advantage of being in a Uni kernel. Then we will examine how we build this specialized Memcached. 

##### The Shrinker API 
Shrinker is something that OSv implements beyond the Linux API. Essentially Shrinker takes advantage that we are allowed truly dynamic memory on a Uni-kernel. On a regular Operating System most systems like a dynamic cache must statically determine their size before hand. This can be limiting as when there are boosts of memory. Sometimes we do not have enough cache entries and sometimes we have allocated too much memory. 

The Shrinker API fixes this issue by allowing us to dynamically increase or decrease our memory. We are allowed to increase our memory to the entire memory of the system or image. 

#### Using the Shrinker API

In the OSv Memcached source we can see we use the [API](https://github.com/vladzcloudius/osv-memcached/blob/master/memcached-udp.cc#L297). The first thing that we do is grab the Shrinker lock: 

```c
WITH_LOCK(_locked_shrinker) {

```

Once we have the lock we can simply update our cache sizes: 

```c
it->second.lru_link->mem_size = memory_needed;
```

This way we can be way more efficient with our caches and dynamically size them based on our needs. 

#### The OSv build process
The build process for OSv is quite complex. From the build file we can see that we have a monstrous 2169 line makefile. However, we can break down the build process in two simple steps. 

- Building the kernel
- Building and fusing our application

##### Building the kernel
In order to investigate what is going on to build the kernel we will look at this [Makefile](https://github.com/cloudius-systems/osv/blob/master/Makefile). We can see the first thing we want to do is detect the architecture so we can build everything 
```Makefile

detect_arch = $(word 1, $(shell { echo "x64        __x86_64__";  \
                                  echo "aarch64    __aarch64__"; \
                       } | $1 -E -xc - | grep ' 1$$'))

```

We then set an output directory. All of our compiled code will go into a single directory: 

```Makefile
out = build/$(mode).$(arch)
```

Then we need to get the correct C++ headers to essentially build musl libc and the core of the kernel: 

```Makefile
CXX_INCLUDES = $(shell $(CXX) -E -xc++ - -v </dev/null 2>&1 | awk '/^End/ {exit} /^ .*c\+\+/ {print "-isystem" $$0}')
```

We also set up some interesting parameters where our Uni-kernel will actually be in memory and how big our TLS size our applications will have.

```Makefile
kernel_base := 0x40080000
kernel_vm_base := $(kernel_base)
app_local_exec_tls_size := 0x40
```

We then compile everything together using shared libraries 

```Makefile
libgcc_eh.a := $(shell $(CC) -print-file-name=libgcc_eh.a)

###########################

$(objects:%=$(out)/%) $(drivers:%=$(out)/%) $(out)/arch/$(arch)/boot.o $(out)/loader.o $(out)/runtime.o: COMMON += -fno-pie
```

Through a long process of building library files is how we build our kernel. 

##### Fusing our application
We then have to fuse our application with our Unikernel. In order to investigate this we will look through [module.py](https://github.com/cloudius-systems/osv/blob/master/scripts/module.py). 

From this we can see there are two steps to our build process: 

- Adding our application and its manifest to a list 

- Compiling our module


##### Compiling our module
When we compile modules in our kernel we have to use make_modules method in [module.py](https://github.com/cloudius-systems/osv/blob/master/scripts/module.py). 

```python
def make_modules(modules, args):
    for module in modules:
        if os.path.exists(os.path.join(module.local_path, 'Makefile')):
            if subprocess.call(make_cmd('module', j=args.j, jobserver=args.jobserver_fds),
                               shell=True, cwd=module.local_path):
                raise Exception('make failed for ' + module.name)
```

Here we simply call the Makefile in the directory of where the application is.

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

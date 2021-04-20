\newpage

# OSv

Most cloud systems run a general-purpose version of a Linux operating system. In many cases, the traditional operating system has more features than what is required by what it runs and duplicates isolation features already provided by the cloud systems hypervisor. The excess provided by the system comes at a high cost.

OSv supports the execution of a single Linux application with less overhead than a traditional operating system. It runs as a virtual machine, thus relying on the host's hypervisor to provide isolation. In some sense, OSv aims at being the happy medium between containers and robust virtual machines by providing the isolation of a virtual machine in conjunction with the efficiency of a container. 

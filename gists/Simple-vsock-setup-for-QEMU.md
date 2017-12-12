# Simple vsock setup for QEMU

## Configuration

Host Kernel:     rawhide  4.13.0-0.rc6.git4.2.fc28.x86_64 (on Fedora 24)

QEMU is mainline built from sources:        QEMU emulator version 2.10.50 (v2.10.0-105-g223cd0e)

Guest: clear-17460-kvm.img (which has vsock support)


# Launching the VM

## First install the vsock driver
```
modprobe vhost_vsock
```

## Launch QEMU

```
export VMN=3
export IMAGE=clear-17460-kvm.img
/usr/local/bin/qemu-system-x86_64 \
    -device vhost-vsock-pci,id=vhost-vsock-pci0,guest-cid=${VMN} \
    -enable-kvm \
    -bios OVMF.fd \
    -smp sockets=1,cpus=4,cores=2 -cpu host \
    -m 1024 \
    -vga none -nographic \
    -drive file="$IMAGE",if=virtio,aio=threads,format=raw \
    -netdev user,id=mynet0,hostfwd=tcp::${VMN}0022-:22,hostfwd=tcp::${VMN}2375-:2375 \
    -device virtio-net-pci,netdev=mynet0 \
    -debugcon file:debug.log -global isa-debugcon.iobase=0x402 $@
```

# How test the vsock connection using socat

Here the CID of the VM is set to 3 and the port set to 1024

In the VM
```
socat - SOCKET-LISTEN:40:0:x00x00x00x04x00x00x03x00x00x00x00x00x00x00
Note: CID = 3
```
On the host 
```
socat - SOCKET-CONNECT:40:0:x00x00x00x04x00x00x02x00x00x00x00x00x00x00
CID = 2 //Hypervisor CID
```

# Minimal console over vsock using socat
VM:
Without auth:
```
socat SOCKET-LISTEN:40:0:x00x00x00x04x00x00x03x00x00x00x00x00x00x00,reuseaddr,fork EXEC:bash,pty,stderr,setsid,sigint,sane,ctty,echo=0 
```
With auth:
```
socat SOCKET-LISTEN:40:0:x00x00x00x04x00x00x03x00x00x00x00x00x00x00,reuseaddr,fork EXEC:login,pty,stderr,setsid,sigint,sane,ctty,echo=0
```


On the host
```
socat - SOCKET-CONNECT:40:0:x00x00x00x04x00x00x03x00x00x00x00x00x00x00
```

#ssh over vsock using socat

VM:
```
socat SOCKET-LISTEN:40:0:x0000xFFxFFx0000x03x00000000000000,reuseaddr,fork TCP:localhost:22
```

VMM:
```
sudo socat TCP4-LISTEN:2222,reuseaddr,fork SOCKET-CONNECT:40:0:x0000xFFxFFx0000x03x00000000000000
```

Now you can ssh into the VM from the host on port 2222

```
ssh root@localhost -p 2222
```
# Using specific ports

socat does not support vsock right now, so you will need to use the generic socket address option to interact with it
http://www.dest-unreach.org/socat/doc/socat-genericsocket.html

To generate the appropriate generic socket option you can use this simple C program

```
#include <sys/socket.h>
#include <linux/vm_sockets.h>
#include <stdio.h>
#include <string.h>

#define GUEST_CID 3

int main(void)
{
        int i;
        char buf[16];
        struct sockaddr_vm sa = {
                            .svm_family = AF_VSOCK,
                            .svm_cid = GUEST_CID, 
                            .svm_port = 1024,
                       };

        printf("VM:\n");
        memcpy(buf, &sa, sizeof(sa));
        for(i=2;i<sizeof(sa);i++) {
                printf("x%02x", (unsigned char)buf[i]);
        }
        printf("\n");
        
        sa.svm_cid = 2;

        printf("VMM:\n");
        memcpy(buf, &sa, sizeof(sa));
        for(i=2;i<sizeof(sa);i++) {
                printf("x%02x", (unsigned char)buf[i]);
        }
        printf("\n");       
}

```

# Testing with a go program
Testing with: https://github.com/mdlayher/vsock/tree/master/cmd/vscp

On the host you run: 
vscp -v -r -p 1024 cpuinfo.txt

On the guest:
vscp -v -s -c 2 -p 1024 /proc/cpuinfo

And you will see the file contents show up


# zD: A Scalable Zero-Drop Network Stack at End Hosts
zD is a new architecture for handling congestion of scheduled buffers. The key idea is to apply backpressure from packet queues, which can overflow and drop packets, to source buffers from whch packets are dispatched. It provides a layer between the sender buffer and the scheduled buffer. We implement zD in the linux kernel to handle back-
pressure for two cases: 1) when the queues and traffic sources are within the kernel stack (i.e., in the same virtual or physical
machine), and 2) when the traffic sources are in the virtual machine and the queues are in the hypervisor. Using zD requires compiling and installing the linux kernel: https://github.com/zD-linux/linux-4.14.67

## What motivated the work?
For years, improved chips added more cores rather than more capacity per core. From the perspective of the networking stack,
this meant that rather than having to serve a few connections per machine, new networking stacks have to cope with requirements in the tens of thousands of connections per machine. Processing of egress traffic in such stacks relies on holding packets in a cascade of queues pending their processing and eventual scheduling to be transmitted on the wire. Congestion of scheduled queues at end hosts typically incurs packet drops which lead to high CPU cost as well as degradation in network performance. we show
that by augmenting existing architectures , CPU and network performance can be significantly improved under high loads

## Getting started
We show how to install and configure zD with an example case. 

Requirement: Two machines that are connected and be able to ping each other.

Experiment setting:
Physical machine A hosts a VM that generates large number of flows and send them to physical machine B. Users can choose to use zD only on VM a or use zD on both VM a and physical machine A.

Steps:
1. Configure the sender (machine A)
    1. Fetch and install your hypervisor kernel. We provide two versions, one with tbf implemented (physical-tbf) and another with pfifo implemented (physical-pfifo).
        ```
        git clone --single-branch --branch physical-pfifo https://github.com/zD-linux/linux-4.14.67.git
        ```
        Configure linux kernel features and compile the kernel:
        ```
        cd linux-4.14.67
        nconfig
        make -j 16
        ```
        copy the compiled kernel image to /boot (or other directory according to the grub configuration).
        
      Note: here's the .config file we use https://github.com/zD-linux/zD-configuration/tree/master/kernel-config. We only turned on necessary modules to reduce the compilation time and optimize the networking stack performance. Person compiling the code should take care of changing other parameters according to their hardware configurations. 
    
    2. Install and configure the hypervisor. Note: we use QEMU v. 2.12: https://download.qemu.org/ for our experiment. While zD is not depedent on hypervisor, the configuration script we provide to configure and start the VM is tested with QEMU v.2.12.
  
    3. Fetch and compile kernel for VM
        ```
        git clone --single-branch --branch vm-pfifo https://github.com/zD-linux/linux-4.14.67.git
        ```
        Configure linux kernel features and compile the kernel:
        ```
        cd linux-4.14.67
        nconfig
        make -j 16
        ```
    4. start the VM and change the networking configuration. 
        ```
        qemu-system-x86_64      -m 10240 -smp 6 -no-acpi                                                                                                \
                        -enable-kvm     -snapshot                                                                                       \
                        -nographic -serial mon:stdio                                                                            \
                        -kernel <*path to the compiled kernel*>                                                                            \
                        -append "root=/dev/vda3 console=ttyS0"                                                                  \
                        -drive file=<*path to the VM image*>,if=none,id=disk0,format=qcow2,cache=none                                                 \
                        -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=disk0,id=disk0,bootindex=1                     \
                        -netdev tap,id=foo,ifname=tapfoo,vhost=on                                       \
                        -device virtio-net,netdev=foo,mac=54:52:00:00:00:02                                                                             \
                        -monitor telnet::22222,server,nowait
        ```
          Configure the VM IP address so it is in the same subnet as machine B. Set up the bridge on machine A so the VM can      access the outwide network. 
          ```
          brctl addbr br-zd
          ifconfig br-zd <ip address assgiend to machine A> netmask 255.255.255.0 up
          brctl addif br-zd tapfoo
          brctl addif br-bptest <interface connected with machine B>
          ```

    5. Configure kernel for large number of flows
      ```
      ulimit -n 1000000
      ```
       
2. Configure receiving machine B
    1. download and install neper. neper is a Linux networking performance tool. https://github.com/google/neper.git
    2. configurre kernel for a large number of flows
        ```
        ulimit -n 1000000
        ```
        

3. Start an experiment
   1. Script to start neper at receiver
      ```
      cd ./neper
      ./tcp_stream -F 4000 -T 20
      ```
   2. Start monitoring
      monitor CPU:
      ```
      apt-get install dstat
      dstat -C
      ```
      monitor number of retransmission
      ```
      netstat -s | grep retrans
      ```
      monitor throughput: neper will report the total throughput
      monitor latency: tcpdump
      ```
      tcpdump -i <interface> host <host IP> -B 10000000 -s 68 -w <pcap file>
      ```
   3. Script to run neper at sender
      ```
      cd ./neper
      ./tcp_stream -c -H 192.168.4.200 -F 10000 -T 20 -B 1024 -l 200
      ```




## Contact
For questions/comments please email yzhao389 at gatech dot edu

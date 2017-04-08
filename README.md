## PISCES Simulation Environment

Following provides instructions on how to setup virtual machines for the PISCES simulation environment along with
some example P4 programs.

### Setup Virtual Machines (VMs)

There are three virtual machines: Switch, Generator, and Receiver. The Generator
sends traffic to the switch on its `eth1` interface, the switch then processes the packet
based on the configured P4 program and sends it out to the receiver on its `eth2` interface.
The receiver receives the traffic and displays stats on the screen.

Add the `vagrant` box.

```bash
$ vagrant box add pisces-ubuntu-trusty64 http://www.cs.princeton.edu/~mshahbaz/projects/pisces/vagrant/ubuntu-trusty64.box
```

Clone the `vagrant` repository.

```bash
$ git clone https://github.com/P4-vSwitch/vagrant.git
$ cd vagrant
```

Bring up virtual machines.

```bash
$ vagrant up
```

> Note: Make sure that you have enough memory on the host to run the setup (switch ~ 4G, generator/receiver ~ 2G each). 
> Also, VMs are only compatible with VirtualBox, so please make sure this provider is available to vagrant.
>
> It can take vagrant 10-15 mins to start up the VMs. So sit back, relax, and wait for the setup to complete. :-)

## Examples

#### 1. Simple Layer-2 Switch

We show how to build a simple layer-2 switch using PISCES. We use the
[`l2_switch.p4`](https://github.com/p4lang/p4factory/tree/master/targets/l2_switch) program provided with the
[p4lang/p4factory](https://github.com/p4lang/p4factory) repository.

Log into the switch VM.

```bash
$ vagrant ssh switch
```

##### a. Compiling `l2_switch.p4`

First, make sure that DPDK environment variables are populated in the switch VM.

```bash
$ export RTE_SDK=/home/vagrant/ovs/deps/dpdk
$ export RTE_TARGET=x86_64-native-linuxapp-gcc
$ export DPDK_DIR=$RTE_SDK
$ export DPDK_BUILD=$DPDK_DIR/$RTE_TARGET/
```

Compile the `l2_switch.p4` program. Specify `/vagrant/examples/l2_switch/l2_switch.p4` for the `p4inputfile` flag.

```bash
$ cd ~/ovs
$ sudo ./configure --with-dpdk=$DPDK_BUILD CFLAGS="-g -O2 -Wno-cast-align" \
                   p4inputfile=/vagrant/examples/l2_switch/l2_switch.p4 \
                   p4outputdir=./include/p4/src
$ sudo make clean
$ sudo make -j 2
```

##### b. Running `l2_switch.p4`

###### Run `ovsdb-server`

Open a new terminal, log into the switch VM and run `ovsdb-server`.

```bash
$ cd ~/ovs/ovsdb
$ sudo ./ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock \
                      --remote=db:Open_vSwitch,Open_vSwitch,manager_options --pidfile
```

###### Run `ovs-vswitchd`

In another terminal, log into the switch VM and run `ovs-vswitchd`.

```bash
$ cd ~/ovs/vswitchd
$ sudo ./ovs-vswitchd --dpdk -c 0x1 -n 4 -- unix:/usr/local/var/run/openvswitch/db.sock --pidfile
```

###### Create an OVS bridge

Open a third terminal, log into the switch VM and run the following commands to create a new OVS bridge.

```bash
$ cd ~/ovs/utilities
$ sudo ./ovs-vsctl --no-wait init
$ sudo ./ovs-vsctl add-br br0 -- set bridge br0 datapath_type=netdev
$ sudo ./ovs-vsctl set bridge br0 protocols=OpenFlow15
$ sudo ./ovs-vsctl add-port br0 dpdk0 -- set Interface dpdk0 type=dpdk
$ sudo ./ovs-vsctl add-port br0 dpdk1 -- set Interface dpdk1 type=dpdk
```

> Note: <br>
> This needs to be done only once. These changes persist across reboots.
>
> Also, to delete the bridge run:
> ```bash
> $ sudo ./ovs-vsctl del-br br0
> ```
>
> And to display bridge settings run:
> ```bash
> $ sudo ./ovs-vsctl show
> ```

###### Install flow rules

```bash
$ cd /vagrant/examples/l2_switch/
$ sudo ./l2_switch.sh
```

###### Send and receive test traffic

Open a new terminal and log into the generator VM.

```bash
$ vagrant ssh generator
```

Go to the `pktgen` directory.

```bash
$ cd ~/pktgen
```

Run `pktgen` as follows:

```bash
$ sudo ./app/app/x86_64-native-linuxapp-gcc/app/pktgen -c 0x3 -n 4 -- -P -m "1.0" -f /vagrant/examples/l2_switch/generator.pkt
```

Similarly, in another terminal log into the receiver VM.

```bash
$ vagrant ssh receiver
```

Run `pktgen` as follows:

```bash
$ cd ~/pktgen
$ sudo ./app/app/x86_64-native-linuxapp-gcc/app/pktgen -c 0x3 -n 4 -- -P -m "1.0" -f /vagrant/examples/l2_switch/receiver.pkt
```

Now go back to the `pktgen` interface, running on the generator VM, and start sending traffic.

```bash
$ start 0
```

> To stop sending traffic, type:
> ```bash
> $ stop 0
> ```

On the receiver side, you will start seeing stats (e.g., packet RX counts) on the `pktgen` interface.

> You can also verify if the switch is forwarding traffic by dumping flows on the switch VM.
> Open a new terminal and log into the switch VM.
> ```bash
> $ vagrant ssh switch
> ```
>
> Now run the following commands:
> ```
> $ cd ~/ovs/utilities
> $ sudo ./ovs-ofctl --protocols=OpenFlow15 dump-flows br0
> ```

You should see non-zero byte and packet counts in the dumped flow rules.

#### 2. Simple Router

To build a simple router, follow the same steps as above with following changes.

###### a. P4 program:

`/vagrant/examples/simple_router/simple_router.p4`

> When compiling the switch, make sure to run `sudo make clean` before running `sudo make -j 2`.

###### b. Flow rules:

`/vagrant/examples/simple_router/simple_router.sh`

###### c. Send and test traffic

Use the following commands on the generator VM.

```bash
 $ cd ~/pktgen
 $ sudo ./app/app/x86_64-native-linuxapp-gcc/app/pktgen -c 0x3 -n 4 -- -P -m "1.0" -f /vagrant/examples/simple_router/generator.pkt
 ```

And the following on the receiver VM.

```bash
 $ cd ~/pktgen
 $ sudo ./app/app/x86_64-native-linuxapp-gcc/app/pktgen -c 0x3 -n 4 -- -P -m "1.0" -f /vagrant/examples/simple_router/receiver.pkt
 ```

---

Enjoy!

For more information please visit: <br>
[pisces.cs.princeton.edu](http://pisces.cs.princeton.edu)

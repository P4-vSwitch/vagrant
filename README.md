## PISCES Test Environment

Following provides instructions on how to setup the PISCES test environment.

### Setup Virtual Machines (VMs)

```bash
$ git clone https://github.com/P4-vSwitch/vagrant.git
$ cd vagrant
$ vagrant up
```

#### 1. Switch

```bash
$ vagrant ssh switch
```

##### Clone repositories

###### a. p4-hlir

```bash
$ cd ~/
$ git clone https://github.com/p4lang/p4-hlir.git
$ cd p4-hlir/
$ sudo python setup.py install
```

###### b. p4c-behavioral

```bash
$ cd ~/
$ git clone https://github.com/P4-vSwitch/p4c-behavioral.git
$ cd p4c-behavioral/
$ git checkout ovs
$ sudo python setup.py install
```

###### c. ovs

```bash
$ cd ~/
$ git clone https://github.com/P4-vSwitch/ovs.git
$ cd ovs/
$ git checkout p4
$ git submodule update --init
```

##### Build DPDK

```bash
$ cd ~/ovs/deps/dpdk
$ patch -p1 -N < ../../setup-scripts/patches/dpdk.patch
$ make -j 2 install T=x86_64-native-linuxapp-gcc
```

###### a. Setup DPDK-specific environment variables

```bash
$ cd ~/
$ export RTE_SDK=~/ovs/deps/dpdk
$ export RTE_TARGET=x86_64-native-linuxapp-gcc

$ export DPDK_DIR=$RTE_SDK
$ export DPDK_BUILD=$DPDK_DIR/$RTE_TARGET/
```

> You may want to append these variables in the `~/.bashrc` file. This way you don't have to execute these whenever you
> open a new terminal.

###### b. Setup DPDK kernel modules

```bash
$ cd ~/
$ sudo modprobe uio
$ sudo insmod $RTE_SDK/$RTE_TARGET/kmod/igb_uio.ko
$ sudo insmod $RTE_SDK/$RTE_TARGET/kmod/rte_kni.ko "lo_mode=lo_mode_ring"
```

###### c. Add eth1 and eth2 interfaces to DPDK

```bash
$ sudo ifconfig eth1 down
$ sudo $RTE_SDK/tools/dpdk_nic_bind.py -b igb_uio eth1

$ sudo ifconfig eth2 down
$ sudo $RTE_SDK/tools/dpdk_nic_bind.py -b igb_uio eth2
```

> To view these interfaces run the following command:
> ```bash
> $ $RTE_SDK/tools/dpdk_nic_bind.py --status
> ```

##### Build OVS

```bash
$ cd ~/ovs/
$ ./boot.sh
$ ./configure --with-dpdk=$DPDK_BUILD CFLAGS="-g -O2 -Wno-cast-align" \
              p4inputfile=./include/p4/examples/l2_switch/l2_switch.p4 \
              p4outputdir=./include/p4/src
$ make -j 2
```

###### a. Configure OVS database

```bash
$ sudo mkdir -p /usr/local/etc/openvswitch
$ sudo mkdir -p /usr/local/var/run/openvswitch
$ cd ~/ovs/ovsdb/
$ sudo ./ovsdb-tool create /usr/local/etc/openvswitch/conf.db ../vswitchd/vswitch.ovsschema
```

Initialize OVS database using ovs-vsctl (first time only)

```bash
$ sudo ./ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock --remote=db:Open_vSwitch,Open_vSwitch,manager_options --pidfile
```

In another terminal, run the following command:

```bash
$ cd ~/ovs/utilities
$ sudo ./ovs-vsctl --no-wait init
```

Once configured, go back to the terminal where `ovsdb-server` is running and close it using CTRL+C.


##### Configure Huge Pages

```bash
$ cd ~/
$ sudo mkdir -p /mnt/huge
$ (mount | grep hugetlbfs) > /dev/null || sudo mount -t hugetlbfs nodev /mnt/huge
```

Log in as `root`.

```bash
$ su
```

> The password for `root` is `vagrant`.

After logging into `root`, run the following command:

```bash
$ echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
```

Then log out `root`.

```bash
$ exit
```

> You can verify if huge pages are configured properly using the following command:
> ```bash
> $ grep -i huge /proc/meminfo
> ```

#### 2. Generator / Receiver


```bash
$ vagrant ssh generator
```

##### Clone MoonGen

```bash
$ cd ~/
$ git clone https://github.com/P4-vSwitch/MoonGen.git
$ cd MoonGen/
$ git checkout dpdk2.1
$ git submodule update --init
```

##### Build LuaJIT

```bash
$ cd ~/MoonGen/deps/luajit
$ make -j 2 'CFLAGS=-DLUAJIT_NUMMODE=2 -DLUAJIT_ENABLE_LUA52COMPAT'
$ make install DESTDIR=$(pwd)
```

##### Build DPDK

```bash
$ cd ~/MoonGen/deps/dpdk
$ patch -p1 -N < ../../setup-scripts/patches/dpdk-config.patch
$ make -j 2 install T=x86_64-native-linuxapp-gcc
```

##### Build MoonGen

```bash
$ cd ~/MoonGen/build
$ cmake ..
$ make
```

## Examples

#### 1. A Layer-2 Switch


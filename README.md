## PISCES Test Environment

Following provides instructions on how to setup virtual machines along with some example P4 programs.

### Setup Virtual Machines (VMs)

There are three virtual machines: (1) Switch, (2) Generator, and (3) Receiver. The Generator
sends traffic, using MoonGen, to the switch on switch's `eth1` interface, the switch then
processes the packet based on the configured P4 program and sends it out to the receiver
on its `eth2` interface. The receiver, also using MoonGen, receives the traffic and displays
the stats on the screen.

The setup is configured using `vagrant`. We first clone the `vagrant` repository that we
have created for this setup.

```bash
$ git clone https://github.com/P4-vSwitch/vagrant.git
$ cd vagrant
```

And start virtual machines.

```bash
$ vagrant up
```

> Note: it can take vagrant 10-15 mins to start up the VMs. So sit back, relax, and wait for the setup to complete. :-)

## Examples

#### 1. A Layer-2 Switch


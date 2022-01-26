# Turing Pi 2 Cluster

[![CI](https://github.com/geerlingguy/turing-pi-2-cluster/workflows/CI/badge.svg?branch=master&event=push)](https://github.com/geerlingguy/turing-pi-2-cluster/actions?query=workflow%3ACI)

<p align="center"><a href="https://www.youtube.com/watch?v=kgVz4-SEhbE"><img src="images/turing-pi-2-hero.jpg?raw=true" width="500" height="auto" alt="Turing Pi 2 - Raspberry Pi Compute Module Cluster" /></a></p>

This repository will contain examples and automation used in Turing Pi 2-related videos on [Jeff Geerling's YouTube channel](https://www.youtube.com/c/JeffGeerling).

You might also be interested in another Raspberry-Pi cluster I've maintained for years, the [Raspberry Pi Dramble](https://www.pidramble.com), which is a Kubernetes Pi cluster in my basement that hosts [www.pidramble.com](https://www.pidramble.com).

## Usage

  1. Make sure you have [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) installed.
  2. Copy the `example.hosts.ini` inventory file to `hosts.ini`. Make sure it has the `control_plane` and `node`s configured correctly (for my examples I named my nodes `turing-node[1-4].local`).
  3. Modify the `config.yml` file to your liking.

### Raspberry Pi Setup

I am running Raspberry Pi OS (64-bit, lite) on a set of four Raspberry Pi Compute Module 4s with 8GB of RAM and no built-in eMMC. I am using [32 GB SanDisk Extreme microSD cards](https://amzn.to/3G35QbY) to boot each node.

I flashed Raspberry Pi OS to the Pis using Raspberry Pi Imager.

To make network discovery and integration easier, I edited the advanced configuration in Imager (press Shift + Ctrl + X), and set the following options:

  - Set hostname: `turing-node-1.local` (set to `2` for node 2, `3` for node 3, etc.)
  - Enable SSH: 'Allow public-key', and paste in my public SSH key(s)
  - Configure wifi: (ONLY on node 1) enter SSID and password for local WiFi network

After setting all those options, making sure only node 1 has WiFi configured, and the hostname is unique to each node (and matches what is in `hosts.ini`), I inserted the microSD cards into the respective Pis, and booted the cluster.

### SSH connection test

To test the SSH connection from my Ansible controller (my main workstation, where I'm running all the playbooks), I connected to each server individually, and accepted the hostkey:

```
ssh pi@turing-node-1.local
```

This ensures Ansible will also be able to connect via SSH in the following steps. You can test Ansible's connection with:

```
ansible all -m ping
```

It should respond with a 'SUCCESS' message for each node.

### Storage Configuration

**Warning**: This playbook is configured to set up a ZFS mirror volume on node 3, with two drives connected to the built-in SATA ports on the Turing Pi 2.

To disable this behavior, you can set `storage_configure: false` in `config.yml`.

To make sure the ZFS mirror volume is able to be created, log into node 3, and make sure your two SATA drives are wiped:

```
pi@turing-node-3:~ $ sudo wipefs --all --force /dev/sda?; sudo wipefs --all --force /dev/sda
pi@turing-node-3:~ $ sudo wipefs --all --force /dev/sdb?; sudo wipefs --all --force /dev/sdb
```

If you run `lsblk`, you should see `sda` and `sdb` have no partitions, and are ready to use:

```
pi@turing-node-3:~ $ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           8:0    0  1.8T  0 disk 
sdb           8:16   0  1.8T  0 disk 
```

### Static network configuration (optional, but recommended)

Because I am using my Turing Pi cluster both on-premise and remote (using a 4G LTE modem connected to the Pi in slot 1), I set it up on its own subnet (10.1.1.x). You can change the subnet that's used via the `ipv4_subnet_prefix` variable in `config.yml`.

To configure the local network for the Turing Pi cluster (this is optional—you can still use the rest of the configuration without a custom local network), run the playbook:

```
ansible-playbook networking.yml
```

After running the playbook, until a reboot, the Pis will still be accessible over their former DHCP-assigned IP address. After the nodes are rebooted, you will need to make sure your workstation is connected to an interface using the same subnet as the cluster (e.g. 10.1.1.x).

> Note: After the networking changes are made, since this playbook uses DNS names (e.g. `turing-node-1.local`) instead of IP addresses, your computer will still be able to connect to the nodes directly—assuming your network has IPv6 support. Pinging the nodes on their new IP addresses will _not_ work, however. For better network compatibility, it's recommended you set up a separate network interface on the Ansible controller that's on the same subnet as the Pis in the cluster:
> 
> On my Mac, I connected a second network interface and manually configured its IP address as `10.1.1.10`, with subnet mask `255.255.255.0`, and that way I could still access all the nodes via IP address or their hostnames (e.g. `turing-node-2.local`).

Because the cluster subnet needs its own router, node 1 is configured as a router, using `wlan0` as the primary interface for Internet traffic by default. The other nodes get their Internet access through node 1.

TODO: Document how to configure VPS and Pi for [reverse ssh tunnel](https://www.jeffgeerling.com/blog/2022/ssh-and-http-raspberry-pi-behind-cg-nat).

TODO: Add more documentation on `active_internet_interface` variable and how to switch to 4G LTE.

### Cluster configuration and K3s installation

Run the playbook:

```
ansible-playbook main.yml
```

### Upgrading the cluster

Run the upgrade playbook:

```
ansible-playbook upgrade.yml
```

### Benchmarking the cluster

See the README file within the `benchmark` folder.

### Shutting down the cluster

The safest way to shut down the cluster is to run the following command:

```
ansible all -B 500 -P 0 -a "shutdown now" -b
```

Then after you confirm the nodes are shut down, press the 'STM32_POWER' button to power down all the slots physically. Then you can switch off or disconnect your ATX power supply.

## Caveats

The Turing Pi 2 I'm using is a prototype, pre-production version of the board. Some of the things here may work well on mine but need changes to work on your own, once you have one.

## Author

The repository was created in 2021 by [Jeff Geerling](https://www.jeffgeerling.com), author of [Ansible for DevOps](https://www.ansiblefordevops.com), [Ansible for Kubernetes](https://www.ansibleforkubernetes.com), and [Kubernetes 101](https://www.kubernetes101book.com).

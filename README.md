# Turing Pi 2 Cluster

[![CI](https://github.com/geerlingguy/turing-pi-2-cluster/workflows/CI/badge.svg?branch=master&event=push)](https://github.com/geerlingguy/turing-pi-2-cluster/actions?query=workflow%3ACI)

<p align="center"><a href="https://www.youtube.com/watch?v=kgVz4-SEhbE"><img src="images/turing-pi-2-hero.jpg?raw=true" width="500" height="auto" alt="Turing Pi 2 - Raspberry Pi Compute Module Cluster" /></a></p>

This repository will contain examples and automation used in Turing Pi 2-related videos on [Jeff Geerling's YouTube channel](https://www.youtube.com/c/JeffGeerling).

You might also be interested in another Raspberry-Pi cluster I've maintained for years, the [Raspberry Pi Dramble](https://www.pidramble.com), which is a Kubernetes Pi cluster in my basement that hosts [www.pidramble.com](https://www.pidramble.com).

## Usage

  1. Make sure you have [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) installed.
  2. Copy the `example.hosts.ini` inventory file to `hosts.ini`. Make sure it has the `control_plane` and `node`s configured correctly (for my examples I named my nodes `turing-node[1-4].local`).
  3. Copy the `example.config.yml` file to `config.yml`, and modify the variables to your liking.

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

> **Warning**: This playbook is configured to set up a ZFS mirror volume on node 3, with two drives connected to the built-in SATA ports on the Turing Pi 2.

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

#### Switch between 4G LTE and WiFi (optional)

The network configuration defaults to an `active_internet_interface` of `wlan0`, meaning node 1 will route all Internet traffic for the cluster through it's WiFi interface.

Assuming you have a [working 4G card in slot 1](https://www.jeffgeerling.com/blog/2022/using-4g-lte-wireless-modems-on-raspberry-pi), you can switch node 1 to route through an alternate interface (e.g. `usb0`):

  1. Set `active_internet_interface: "usb0"` in your `config.yml`
  2. Run the networking playbook again: `ansible-playbook networking.yml`

You can switch back and forth between interfaces using the steps above.

#### Reverse SSH and HTTP tunnel configuration (optional)

For my own experimentation, I decided to run my Pi cluster 'off-grid', using a 4G LTE modem, as mentioned above.

Because my mobile network provider uses CG-NAT, there is no way to remotely access the cluster, or serve web traffic to the public internet from it, at least not out of the box.

I am using a reverse SSH tunnel to enable direct remote SSH and HTTP access. To set that up, I configured a VPS I run to use TCP Forwarding (see [this blog post for details](https://www.jeffgeerling.com/blog/2022/ssh-and-http-raspberry-pi-behind-cg-nat)), and I configured an SSH key so node 1 could connect to my VPS (e.g. `ssh my-vps-username@my-vps-hostname-or-ip`).

Then I set the `reverse_tunnel_enable` variable to `true` in my `config.yml`, and configured the VPS username and hostname options.

Doing that and running the `main.yml` playbook configures `autossh` on node 1, and will try to get a connection through to the VPS on ports 2222 (to node 1's port 22) and 8080 (to node 1's port 80).

After that's done, you should be able to log into the cluster _through_ your VPS with a command like:

```
$ ssh -p 2222 pi@[my-vps-hostname]
```

> Note: If autossh isn't working, it could be that it didn't exit cleanly, and a tunnel is still reserving the port on the remote VPS. That's often the case if you run `sudo systemctl status autossh` and see messages like `Warning: remote port forwarding failed for listen port 2222`.
>
> In that case, log into the remote VPS and run `pgrep ssh | xargs kill` to kill off all active SSH sessions, then `autossh` should pick back up again.

> **Warning**: Use this feature at your own risk. Security is your own responsibility, and for better protection, you should probably avoid directly exposing your cluster (e.g. by disabling the `GatewayPorts` option) so you can only access the cluster while already logged into your VPS).

### Cluster configuration and K3s installation

Run the playbook:

```
ansible-playbook main.yml
```

At the end of the playbook, there should be an instance of Drupal running on the cluster. If you log into node 1, you should be able to access it with `curl localhost`. Alternatively, if you have SSH tunnelling configured, you could access `http://[your-vps-ip-or-hostname]:8080/` and you'd see the site.

You can also log into node 1, switch to the root user account (`sudo su`), then use `kubectl` to manage the cluster (e.g. view Drupal pods with `kubectl get pods -n drupal`).

K3s' `kubeconfig` file is located at `/etc/rancher/k3s/k3s.yaml`. If you'd like to manage the cluster from other hosts (or using a tool like Lens), copy the contents of that file, replacing `localhost` with the IP address or hostname of the control plane node, and paste the contents into a file `~/.kube/config`.

### Upgrading the cluster

Run the upgrade playbook:

```
ansible-playbook upgrade.yml
```

### Monitoring the cluster

Prometheus and Grafana are used for monitoring. Grafana can be accessed via port forwarding (or you could choose to expose it another way).

To access Grafana:

  1. Make sure you set up a valid `~/.kube/config` file (see 'K3s installation' above).
  1. Run `kubectl port-forward service/cluster-monitoring-grafana :80`
  1. Grab the port that's output, and browse to `localhost:[port]`, and bingo! Grafana.

The default login is `admin` / `prom-operator`, but you can also get the secret with `kubectl get secret cluster-monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -D`.

### Benchmarking the cluster

See the README file within the `benchmark` folder.

### Shutting down the cluster

The safest way to shut down the cluster is to run the following command:

```
ansible all -B 500 -P 0 -a "shutdown now" -b
```

> Note: If using the SSH tunnel, you might want to run the command _first_ on worker nodes 2-4, _then_ on node 1. So first run `ansible 'all:!control_plane' [...]`, then run it again just for `control_plane`.

Then after you confirm the nodes are shut down (with K3s running, it can take a few minutes), press the 'STM32_POWER' button or a power button attached to the front panel connector to power down all the slots physically. Then you can switch off or disconnect your ATX power supply.

## Caveats

The Turing Pi 2 I'm using is a prototype, pre-production version of the board. If you have a production board, YMMV. You've been warned!

## Author

The repository was created in 2021 by [Jeff Geerling](https://www.jeffgeerling.com), author of [Ansible for DevOps](https://www.ansiblefordevops.com), [Ansible for Kubernetes](https://www.ansibleforkubernetes.com), and [Kubernetes 101](https://www.kubernetes101book.com).

# Turing Pi 2 Cluster

[![CI](https://github.com/geerlingguy/turing-pi-2-cluster/workflows/CI/badge.svg?branch=master&event=push)](https://github.com/geerlingguy/turing-pi-2-cluster/actions?query=workflow%3ACI)

<p align="center"><a href="https://www.youtube.com/watch?v=kgVz4-SEhbE"><img src="images/turing-pi-2-hero.jpg?raw=true" width="500" height="auto" alt="Turing Pi 2 - Raspberry Pi Compute Module Cluster" /></a></p>

This repository will contain examples and automation used in Turing Pi 2-related videos on [Jeff Geerling's YouTube channel](https://www.youtube.com/c/JeffGeerling).

You might also be interested in another Raspberry-Pi cluster I've maintained for years, the [Raspberry Pi Dramble](https://www.pidramble.com), which is a Kubernetes Pi cluster in my basement that hosts [www.pidramble.com](https://www.pidramble.com).

## Usage

  1. Make sure you have [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) installed.
  3. Copy the `example.hosts.ini` inventory file to `hosts.ini`. Make sure it has the `master` and `node`s configured correctly.
  5. Run the playbook:

     ```
     ansible-playbook main.yml
     ```

TODO.

### Benchmarking the cluster

Run the benchmarking playbook:

```
ansible-playbook benchmark.yml
```

TODO.

### Shutting down the cluster

There will be a safe shutdown option provided through the board's controller at some point, but right now the safest way to shut down the cluster is to run the following command:

```
ansible all -B 500 -P 0 -a "shutdown now" -b
```

Then after you confirm the nodes are shut down, press the 'STM32_POWER' button to power down all the slots physically. Then you can switch off or disconnect your ATX power supply.

## Caveats

The Turing Pi 2 I'm using is a prototype, pre-production version of the board. Some of the things here may work well on mine but need changes to work on your own, once you have one.

## Author

The repository was created in 2021 by [Jeff Geerling](https://www.jeffgeerling.com), author of [Ansible for DevOps](https://www.ansiblefordevops.com), [Ansible for Kubernetes](https://www.ansibleforkubernetes.com), and [Kubernetes 101](https://www.kubernetes101book.com).

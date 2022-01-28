# Cluster Benchmark

A common generic benchmark for clusters is Linpack, or HPL (High Performance Linpack), which is famous for its use in rankings in the TOP500 supercomputer list over the past few decades.

I wanted to see where my various Pi clusters would rank, historically, so I built this playbook which installs all the necessary tooling for HPL to run, connects all the nodes together via SSH, then runs the benchmark and outputs the result.

## Usage

Make sure you have Ansible installed, and make sure you've at _least_ run the `networking.yml` playbook in the main directory. Then run the benchmarking playbook inside this directory:

```
ansible-playbook main.yml
```

You should be able to log directly into any of the nodes (I did my tests on node 1), and run the following commands to kick off a benchmarking run:

```
cd ~/tmp/hpl-2.3/bin/rpi
mpirun -f cluster-hosts ./xhpl
```

> The configuration here is optimized for a 4-node Pi CM4 cluster with 8 GB of RAM on each module. Some settings like those in the `HPL.dat` file may need changes for different cluster layouts!

## Results

In my testing on Raspberry Pi OS Bullseye, in November 2021, I got the following results:

| Benchmark | Result | Wattage | Gflops/W |
| --- | --- | --- | --- |
| HPL (1.5 GHz default clock) | 44.942 Gflops | 24.5W | 1.83 Gflops/W |
| HPL (2.0 GHz overclock) | 51.327 Gflops | 33W | 1.54 Gflops/W |

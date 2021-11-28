# Cluster Benchmark

A common generic benchmark for clusters is Linpack, or HPL (High Performance Linpack), which is famous for its use in rankings in the TOP500 supercomputer list over the past few decades.

I wanted to see where my various Pi clusters would rank, historically, so I built this playbook which installs all the necessary tooling for HPL to run, connects all the nodes together via SSH, then runs the benchmark and outputs the result.

## Usage

Run the benchmarking playbook inside this directory:

```
ansible-playbook main.yml
```

TODO.

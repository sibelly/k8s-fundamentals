# Linux Namespaces and cgroup v2 Lab

This lab demonstrates some of the Linux primitives used to build containers. We will create an isolated environment with namespaces, attach its main process to a cgroup v2, apply CPU and memory limits, and observe the kernel metrics while generating load.

> Run this lab only in a disposable Linux VM. Root privileges are required.

## Prerequisites

- Linux with cgroup v2
- `stress`
- `pstree` (usually provided by the `psmisc` package)
- Root or `sudo` access

Confirm that the system is using cgroup v2:

```shell
stat -fc %T /sys/fs/cgroup
```

The expected output is:

```text
cgroup2fs
```

## Creating an Isolated Environment with `unshare`

The `unshare` command creates new namespaces for a process. Namespaces isolate resources such as process IDs, network interfaces, and mount tables.

In this example, we create new PID, mount, and network namespaces and start Bash inside the isolated environment:

```shell
sudo unshare -p -m -n -f --mount-proc bash
```

The options mean:

- `-p`: creates a new PID namespace. Processes inside it have their own view of process IDs.
- `-m`: creates a new mount namespace, which provides an isolated mount table. It does not create a separate root filesystem by itself.
- `-n`: creates a new network namespace with its own network stack and interfaces.
- `-f`: forks before launching the requested command. This is required for the new process to become PID 1 inside the PID namespace.
- `--mount-proc`: mounts a new `/proc` filesystem for the PID namespace, allowing processes inside it to see the namespace-specific process tree.

The Bash process now sees itself as PID 1 inside the namespace. The host, however, still identifies it by its global PID.

## Finding the Bash Process Created by `unshare`

Open another terminal on the host and find the `unshare` process:

```shell
ps -ef | grep '[u]nshare'
```

Use `pstree` to inspect its process tree:

```shell
pstree -p <UNSHARE_PID>
```

Replace `<UNSHARE_PID>` with the PID returned by the previous command. The Bash process should appear as a child of `unshare`:

```text
unshare(<UNSHARE_PID>)---bash(<BASH_GLOBAL_PID>)
```

Save `<BASH_GLOBAL_PID>`. We will use it to attach Bash to the cgroup.

## Limiting Resources with cgroup v2

A cgroup controls and accounts for the resources used by a group of processes. We will create one that limits CPU and memory usage.

### 1. Create the cgroup

```shell
sudo mkdir /sys/fs/cgroup/mycontainer
```

For convenience, define its path:

```shell
CG=/sys/fs/cgroup/mycontainer
```

### 2. Configure CPU and memory limits

Limit CPU usage to approximately 50% of one CPU core:

```shell
echo "50000 100000" | sudo tee "$CG/cpu.max"
```

`cpu.max` contains a quota and a period. In this example, processes may consume 50,000 microseconds of CPU time during each 100,000-microsecond period.

Limit memory usage to 100 MiB:

```shell
echo "100M" | sudo tee "$CG/memory.max"
```

### 3. Attach Bash to the cgroup

Write the global PID of the isolated Bash process to `cgroup.procs`:

```shell
echo <BASH_GLOBAL_PID> | sudo tee "$CG/cgroup.procs"
```

Bash is now running inside the namespaces and is also a member of the cgroup. New child processes started by this Bash shell inherit its cgroup membership and are subject to the configured limits.

You can confirm the attached processes with:

```shell
cat "$CG/cgroup.procs"
```

## Running a Process in the Isolated Environment

Return to the Bash shell running inside the namespaces and use `stress` to generate CPU and memory load:

```shell
stress --cpu 2 --vm 1 --vm-bytes 80M --timeout 30
```

This command starts two CPU workers and one memory worker that allocates 80 MiB for 30 seconds. The workload remains subject to the CPU and memory limits configured in the cgroup.

## Inspecting CPU Usage with `cpu.stat`

From the host, inspect the cgroup's CPU statistics:

```shell
cat /sys/fs/cgroup/mycontainer/cpu.stat
```

Important fields include:

- `usage_usec`: total CPU time used by all processes in the cgroup, in microseconds.
- `user_usec`: CPU time spent in user space.
- `system_usec`: CPU time spent in kernel space.
- `nr_periods`: number of CPU bandwidth enforcement periods.
- `nr_throttled`: number of periods in which the cgroup was throttled.
- `throttled_usec`: cumulative time during which the cgroup was throttled, in microseconds.

To convert `usage_usec` to seconds, divide it by 1,000,000.

### Calculating the Percentage of Throttled Periods

The ratio below represents the percentage of CPU enforcement periods in which throttling occurred:

```text
(nr_throttled / nr_periods) * 100
```

Calculate it with `awk`:

```shell
awk '
  $1 == "nr_periods"   { periods = $2 }
  $1 == "nr_throttled" { throttled = $2 }
  END {
    if (periods > 0) {
      printf "Throttled periods: %.2f%%\n", (throttled / periods) * 100
    } else {
      print "No CPU periods recorded yet"
    }
  }
' /sys/fs/cgroup/mycontainer/cpu.stat
```

This is not the percentage of wall-clock time spent throttled. It only shows the fraction of enforcement periods in which throttling occurred. For workload analysis, compare changes in these counters over a defined observation window.

### Monitoring CPU Statistics in Real Time

Use `watch` to refresh `cpu.stat` every second:

```shell
watch -n 1 cat /sys/fs/cgroup/mycontainer/cpu.stat
```

The counters should change while `stress` consumes CPU. If the workload attempts to exceed the quota, `nr_throttled` and `throttled_usec` should increase.

## Inspecting Memory Usage

The cgroup v2 memory controller exposes its state through files under the cgroup directory:

```shell
cat "$CG/memory.current"
cat "$CG/memory.max"
cat "$CG/memory.stat"
cat "$CG/memory.events"
```

- `memory.current`: current memory usage of the cgroup.
- `memory.max`: configured hard memory limit.
- `memory.stat`: detailed breakdown of the cgroup's memory usage.
- `memory.events`: counters for events such as reaching the limit or an OOM kill.

Monitor the main CPU and memory metrics together:

```shell
watch -n 1 "
  cat $CG/cpu.stat
  echo
  echo memory.current=\$(cat $CG/memory.current)
  echo memory.max=\$(cat $CG/memory.max)
  echo
  cat $CG/memory.events
"
```

## Tips and Cleanup

### Use a new cgroup for each test

Files such as `cpu.stat` contain cumulative counters. Creating a new cgroup for each experiment prevents previous executions from affecting the results.

### Remove the cgroup after the test

First, confirm that no processes remain in the cgroup:

```shell
cat /sys/fs/cgroup/mycontainer/cgroup.procs
```

If the file is empty, remove the cgroup:

```shell
sudo rmdir /sys/fs/cgroup/mycontainer
```

The directory cannot be removed while it still contains processes or child cgroups.

## What This Lab Demonstrates

- Namespaces isolate what a process can see.
- Cgroups control and account for what a process can consume.
- A process can simultaneously belong to multiple namespaces and a cgroup.
- Child processes inherit their parent's cgroup membership.
- Linux exposes cgroup configuration and metrics through files under `/sys/fs/cgroup`.
- Container runtimes automate these same kernel mechanisms when creating containers.

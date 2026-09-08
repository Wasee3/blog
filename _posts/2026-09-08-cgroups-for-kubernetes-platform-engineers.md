---
layout: post
title: "cgroups for Kubernetes Platform Engineers"
date: 2026-09-08 12:00:00 +0000
categories: linux kubernetes cgroups
---

You already know what `resources.requests` and `limits` do. This post is about what
they *are*: a handful of files under `/sys/fs/cgroup` that the kubelet writes, and
that the kernel enforces without ever asking Kubernetes' opinion.

Everything below was run on one throwaway VM. Every command and every block of
output is real — no invented numbers, and the surprises are left in, because the
surprises are the interesting part. All the scripts are inline, so you can paste
them and get your own numbers. At the end there is a ledger of every change made
to the host and proof that it was all undone.

---

## 0. Why cgroups exist at all

**A process is not a unit of resource control.** The pre-cgroups tool was
`setrlimit`, and its limits are per-process: `RLIMIT_AS` bounds one address space,
`RLIMIT_NPROC` bounds one user's process count. Fork twice and each child gets its
own fresh allowance. There was no kernel object meaning "this workload, all of it,
including whatever it spawns next".

**Namespaces isolate visibility, not consumption.** `chroot`, and later mount, PID
and network namespaces, change what a process can *see*. None of them stop it
eating the machine. A container is namespaces (what you can see) plus cgroups
(what you can use) plus capabilities and seccomp (what you can do). cgroups is
the half that does accounting and enforcement — and it is the half that
`resources:` compiles down to.

**The failure modes are specific, and you have met all of them.** One tenant's log
flush evicting everyone else's page cache. The OOM killer picking the largest RSS
rather than the process actually responsible. A batch job starving a latency-
sensitive service on the same box. A fork bomb in one container taking the node's
PID space with it, so you cannot even SSH in to fix it.

**Accounting matters as much as limiting.** Scheduling, chargeback, autoscaling
and capacity planning all need per-workload usage the kernel can vouch for, not a
sum over `ps` output that misses short-lived children.

The lineage: Google's "process containers" (2006) → merged as cgroups in Linux
2.6.24 (2008) → a decade of v1, where every controller had its own independent
hierarchy and they contradicted each other → cgroup v2's single unified hierarchy,
stable in 4.5 (2016) → systemd and Kubernetes standardising on it, with cgroup v2
going GA in Kubernetes 1.25.

The through-line for the rest of this post: **every field in `resources:` is a
write to a file under `/sys/fs/cgroup`, and you can do all of it by hand.**

---

## 1. The canonical documents

A note first, because it comes up: **there are no IETF RFCs for cgroups.** cgroups
is a Linux kernel subsystem, not a wire protocol — there was never anything to
put on the standards track. If someone hands you an "RFC number for cgroups", it
does not exist. What *does* exist is a small set of normative documents:

**Kernel and runtime**

| Document | Why you care |
|---|---|
| `Documentation/admin-guide/cgroup-v2.rst` | The actual specification of every file in this post, including the delegation and "no internal processes" rules. |
| OCI Runtime Spec, `config-linux.md` | How a container's `resources` block maps to cgroup files — the contract runc and crun implement. |
| `systemd.resource-control(5)`, `systemd.slice(5)` | `MemoryMax=`, `CPUQuota=`, `Delegate=`. The reason `cgroupDriver: systemd` exists. |
| `cgroups(7)`, `cgroup_namespaces(7)` | The man pages, including why a container sees `0::/`. |

**Kubernetes KEPs — the closest thing to an RFC process here**

| KEP | Subject | Why a platform engineer needs it |
|---|---|---|
| KEP-2254 | cgroup v2 support | GA in 1.25. The baseline everything else assumes. |
| KEP-4569 | cgroup v1 maintenance mode | The deprecation runway. Plan node upgrades around it. |
| KEP-2570 | Memory QoS | `memory.min` / `memory.high` from requests and limits; tiered protection by QoS class landed in v1.36. Turns OOM kills into latency. |
| KEP-2400 | Node memory swap | cgroup v2 only, `LimitedSwap` for Burstable pods. Changes what `limits.memory` means. |
| KEP-1287 | In-place pod resize | Beta in 1.33. Rewrites `cpu.max` / `memory.max` on a running container. |
| KEP-2837 | Pod-level resources | Limits at the pod slice rather than per container. |
| KEP-4205 | PSI | Pressure metrics, and node conditions/taints from them in phase 2. |
| KEP-3570 (+2625, 4540) | CPU Manager and its policy options | `cpuset.cpus` for Guaranteed pods; `full-pcpus-only`, `strict-cpu-reservation`. |

---

## 2. The host these ran on

Everything below ran here. Your numbers will differ; the shapes should not.

```text
== cgroup hierarchy ==
  /sys/fs/cgroup is: cgroup2fs
  [ ok ] unified cgroup v2 hierarchy
  [ ok ] no cgroup v1 controllers mounted
== controllers available at the root ==
  cgroup.controllers:     cpuset cpu io memory hugetlb pids rdma misc
  cgroup.subtree_control: cpu memory pids
  [ ok ] controller present: cpu
  [ ok ] controller present: memory
  [ ok ] controller present: pids
  [ ok ] controller present: cpuset
  [ ok ] controller present: io
  [ ok ] controller present: hugetlb
== kernel configuration ==
  [ ok ] CONFIG_CFS_BANDWIDTH=y
  [ ok ] CONFIG_PSI=y
  [ ok ] CONFIG_BLK_DEV_THROTTLING=y
  [ ok ] CONFIG_MEMCG=y
  [ ok ] CONFIG_CGROUP_PIDS=y
  [ ok ] CONFIG_CGROUP_HUGETLB=y
== privileges ==
  [ ok ] running as root

PREFLIGHT PASSED - this host can run every scenario.
```

The preflight script that produced it:

```bash
#!/usr/bin/env bash
# 00-preflight.sh - refuse to run the demos unless this host can actually
# demonstrate them. Everything after this assumes a v2-only hierarchy.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

fail=0
check() { # check <description> <condition-result>
    if [[ $2 == 0 ]]; then printf '  [ ok ] %s\n' "$1"
    else                   printf '  [FAIL] %s\n' "$1"; fail=1; fi
}

echo "== cgroup hierarchy =="
fstype=$(stat -fc %T $CG_ROOT)
echo "  /sys/fs/cgroup is: $fstype"
[[ $fstype == cgroup2fs ]]; check "unified cgroup v2 hierarchy" $?
mount | grep -q 'type cgroup ' ; [[ $? == 1 ]]; check "no cgroup v1 controllers mounted" $?

echo "== controllers available at the root =="
echo "  cgroup.controllers:     $(cat $CG_ROOT/cgroup.controllers)"
echo "  cgroup.subtree_control: $(cat $CG_ROOT/cgroup.subtree_control)"
for c in cpu memory pids cpuset io hugetlb; do
    grep -qw "$c" $CG_ROOT/cgroup.controllers; check "controller present: $c" $?
done

echo "== kernel configuration =="
cfg=/boot/config-$(uname -r)
for opt in CONFIG_CFS_BANDWIDTH=y CONFIG_PSI=y CONFIG_BLK_DEV_THROTTLING=y \
           CONFIG_MEMCG=y CONFIG_CGROUP_PIDS=y CONFIG_CGROUP_HUGETLB=y; do
    grep -qx "$opt" "$cfg"; check "$opt" $?
done

echo "== privileges =="
[[ $EUID -eq 0 ]]; check "running as root" $?

echo
[[ $fail == 0 ]] && echo "PREFLIGHT PASSED - this host can run every scenario." \
                || echo "PREFLIGHT FAILED - see [FAIL] lines above."
exit $fail
```

Two shared files are used by every scenario. The first is the helper library —
note `cg_run`, which exists because of a deadlock I hit, and `cg_wait_empty`,
which exists because `cgroup.kill` is asynchronous and `rmdir` races it:

```bash
#!/usr/bin/env bash
# lib.sh - shared helpers for the cgroup v2 demos.
# Sourced by every scenario script. Nothing here mutates the host without
# recording the change (and its exact undo command) in the ledger first.

set -uo pipefail

CG_ROOT=/sys/fs/cgroup            # cgroup v2 unified hierarchy
DEMO_ROOT=$CG_ROOT/blogdemo       # every cgroup we create lives under here
# Derived from this file's location, NOT $HOME: these scripts run under sudo,
# where $HOME is /root, and the ledger must stay with the project.
BLOG_DIR=${BLOG_DIR:-$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)}
STATE_DIR=$BLOG_DIR/state
LEDGER=$STATE_DIR/CHANGELOG-system.md

# --- change ledger --------------------------------------------------------
# record_change "<what changed>" "<exact command that undoes it>"
record_change() {
    local what=$1 undo=$2
    mkdir -p "$STATE_DIR"
    if [[ ! -f $LEDGER ]]; then
        printf '# System change ledger\n\n| when | change | revert command | status |\n|---|---|---|---|\n' > "$LEDGER"
    fi
    printf '| %s | %s | `%s` | PENDING |\n' "$(date -Is)" "$what" "$undo" >> "$LEDGER"
}

# Flip a ledger row to REVERTED once its undo command has actually run.
mark_reverted() {                  # mark_reverted <substring of the change text>
    local needle=${1//\//\\/}
    [[ -f $LEDGER ]] || return 0
    sed -i "\|$needle|s/| PENDING/| REVERTED/" "$LEDGER"
}

# --- cgroup helpers -------------------------------------------------------
# The v2 rules that bite you: a controller must be enabled in the PARENT's
# cgroup.subtree_control before a child can use it, and only leaf cgroups may
# hold processes ("no internal processes" rule).

cg_enable() {                      # cg_enable <cgroup-dir> <ctrl>...
    local dir=$1; shift
    local ctrl
    for ctrl in "$@"; do
        if ! grep -qw -- "$ctrl" "$dir/cgroup.subtree_control" 2>/dev/null; then
            echo "+$ctrl" > "$dir/cgroup.subtree_control"
        fi
    done
}

demo_root_init() {                 # create /sys/fs/cgroup/blogdemo once
    if [[ ! -d $DEMO_ROOT ]]; then
        mkdir "$DEMO_ROOT"
        record_change "created cgroup $DEMO_ROOT" "rmdir $DEMO_ROOT"
    fi
    cg_enable "$DEMO_ROOT" cpu memory pids
}

# Always hands back a PRISTINE cgroup: a leftover from an interrupted run has
# stale limits and possibly live processes, which silently corrupts results.
cg_create() {                      # cg_create <name> -> echoes the path
    demo_root_init
    local dir=$DEMO_ROOT/$1
    [[ -d $dir ]] && cg_destroy "$dir"
    mkdir "$dir"
    echo "$dir"
}

# Wait until a cgroup holds no processes. cgroup.kill is asynchronous: the
# write returns before the victims finish exiting, and rmdir on a cgroup that
# still has an exiting task fails with EBUSY. Skipping this wait is how you
# leave a live busy-loop behind and poison the NEXT measurement.
cg_wait_empty() {                  # cg_wait_empty <dir> [tries]
    local dir=$1 tries=${2:-50}
    while (( tries-- > 0 )); do
        [[ -s $dir/cgroup.procs ]] || return 0
        sleep 0.1
    done
    return 1
}

cg_destroy() {                     # kill everything inside, then remove
    local dir=$1 tries=30
    [[ -d $dir ]] || return 0
    [[ -e $dir/cgroup.kill ]] && echo 1 > "$dir/cgroup.kill" 2>/dev/null
    cg_wait_empty "$dir"
    while (( tries-- > 0 )); do
        rmdir "$dir" 2>/dev/null && return 0
        sleep 0.1
    done
    echo "WARNING: could not remove $dir (procs: $(wc -l < "$dir/cgroup.procs"))" >&2
    return 1
}

# Runs <cmd> inside <cgroup-dir> in the background and sets $CG_PID.
# NOTE: this deliberately does NOT echo the pid for $(...) capture - a
# backgrounded child inherits the substitution's stdout pipe, so command
# substitution would block until the child exits. Global variable instead.
cg_run() {                         # cg_run <cgroup-dir> <cmd>...
    local dir=$1; shift
    setsid bash -c 'echo $$ > "$1/cgroup.procs"; shift; exec "$@"' _ "$dir" "$@" &
    CG_PID=$!
    disown "$CG_PID" 2>/dev/null   # keep "Killed" job notices out of the transcript
}

# Run a command IN the cgroup in the foreground and return its exit status.
# Needed for OOM demos: the shell reports 137 (128 + SIGKILL) exactly as
# a container runtime reports an OOMKilled container.
cg_exec() {                        # cg_exec <cgroup-dir> <cmd>...
    local dir=$1; shift
    bash -c 'echo $$ > "$1/cgroup.procs"; shift; exec "$@"' _ "$dir" "$@"
}

cg_kill() {                        # kill every process in a cgroup, atomically
    [[ -e $1/cgroup.kill ]] && echo 1 > "$1/cgroup.kill"
    cg_wait_empty "$1"
}

show() {                           # show <file> - print a cgroup file with its name
    printf '%-28s %s\n' "${1#$CG_ROOT/}:" "$(tr '\n' ' ' < "$1")"
}

# --- load generators (no stress-ng on this host) --------------------------
burn_cpu() {                       # burn_cpu [threads] - busy loop(s)
    local n=${1:-1} i
    for ((i = 0; i < n; i++)); do
        while :; do :; done &
    done
}

# Allocate anonymous memory in <mb> 1 MiB chunks, touching every page.
MEM_HOG='
import sys, time
chunks, mb = [], int(sys.argv[1])
for i in range(mb):
    chunks.append(bytearray(1024 * 1024))
    for off in range(0, 1024 * 1024, 4096):
        chunks[-1][off] = 1
    print(f"allocated {i+1} MiB", flush=True)
time.sleep(float(sys.argv[2]) if len(sys.argv) > 2 else 0)
'

# Wrapped and exported so nested `bash -c` bodies can call it without the
# quoting gymnastics of re-embedding the python source.
mem_hog() { python3 -c "$MEM_HOG" "$@"; }
export MEM_HOG
export -f mem_hog
```

The second records every change made to the host, with the command that undoes it:

```bash
#!/usr/bin/env bash
# state-snapshot.sh {before|after}
# Captures every piece of host state the demos can touch, so the teardown can
# be proven complete by diffing the two snapshots.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

phase=${1:?"usage: state-snapshot.sh before|after"}
out=$STATE_DIR/$phase
mkdir -p "$out"

cat "$CG_ROOT/cgroup.subtree_control"            > "$out/root-subtree_control.txt"
ls "$CG_ROOT"                                    > "$out/cgroup-root-listing.txt"
sysctl vm.nr_hugepages                           > "$out/sysctl-hugepages.txt" 2>&1
grep -i -E 'HugePages_Total|SwapTotal' /proc/meminfo > "$out/meminfo-bits.txt"
swapon --show                                    > "$out/swapon.txt" 2>&1
cat /etc/fstab                                   > "$out/fstab.txt"
systemctl list-unit-files --state=enabled --no-pager --no-legend > "$out/enabled-units.txt" 2>&1
dpkg -l | awk '/^ii/{print $2, $3}' | sort       > "$out/packages.txt"
ls -d /sys/fs/cgroup/blogdemo /sys/fs/cgroup/kubepods.slice /var/lib/rancher \
      /usr/local/bin/k3s 2>/dev/null              > "$out/demo-artifacts.txt"
ps -eo comm= | sort | uniq -c | sort -rn         > "$out/process-summary.txt"

echo "snapshot '$phase' written to $out"
```

---

## Part 1 — The primitives, by hand

### 1. `cpu.max` — CFS bandwidth, and the myth of the exact quota

`limits.cpu: 200m` becomes `cpu.max = "20000 100000"`: 20 ms of CPU per 100 ms
period. When the quota runs out, every thread in the cgroup is stopped until the
next period begins. This is why a pod can show 20% CPU utilisation and still miss
its latency SLO — it is not slow, it is *stopped*, in 100 ms sawteeth.

The second half of this one surprised me. I expected the quota to be enforced
exactly and it is not: a workload free to migrate between CPUs deviates from its
quota in both directions, because bandwidth is handed to per-CPU run-queues in
slices rather than drawn from one global bucket. Pin it and the deviation vanishes.
That is a direct argument for the CPU Manager `static` policy, and I would not have
believed it without the measurement.

```bash
#!/usr/bin/env bash
# 01-cpu-max.sh - CFS bandwidth control: the knob behind `limits.cpu`.
# Part 1: a throttled workload is stopped for the rest of every 100ms period.
# Part 2: how accurately the quota is actually enforced - measured, not assumed.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

cg=$(cg_create cpu-max)
trap 'cg_destroy "$cg"' EXIT

sample() { awk '/^(usage_usec|nr_periods|nr_throttled|throttled_usec)/{printf "%s=%s ", $1, $2}' "$cg/cpu.stat"; }
report() {
    python3 - "$@" <<'PY'
import sys
label, b, a = sys.argv[1], sys.argv[2], sys.argv[3]
b = dict(kv.split('=') for kv in b.split()); a = dict(kv.split('=') for kv in a.split())
d = {k: int(a[k]) - int(b[k]) for k in a}
cores = d['usage_usec'] / (d['nr_periods'] * 1e5) if d['nr_periods'] else 0
pct   = 100 * d['nr_throttled'] // d['nr_periods'] if d['nr_periods'] else 0
print(f"  {label}: nr_periods={d['nr_periods']} nr_throttled={d['nr_throttled']} "
      f"throttled={d['throttled_usec']/1e6:.2f}s usage={d['usage_usec']/1e6:.2f}s")
print(f"  {' ' * len(label)}  -> throttled in {pct}% of periods, effective {cores:.3f} CPU")
PY
}

echo "### A fresh cgroup has no CPU limit"
show "$cg/cpu.max"                       # "max 100000" = unlimited

echo
echo "### Apply 0.2 CPU: quota 20000us per 100000us period == limits.cpu: 200m"
echo "20000 100000" > "$cg/cpu.max"
show "$cg/cpu.max"

for threads in 1 4; do
    echo
    echo "### $threads busy thread(s), 5s wall time, under the 0.2 CPU quota"
    b=$(sample)
    cg_run "$cg" bash -c "for i in \$(seq $threads); do while :; do :; done & done; wait"
    sleep 5
    cg_kill "$cg"
    a=$(sample)
    echo "  raw cpu.stat before: $b"
    echo "  raw cpu.stat after : $a"
    report "${threads}-thread" "$b" "$a"
done

# --- How exact is the quota, really? --------------------------------------
# Bandwidth is not enforced from one global bucket: each CPU's runqueue pulls
# a slice (kernel.sched_cfs_bandwidth_slice_us) from the pool. A task that
# migrates leaves unused slice behind on the old runqueue and pulls a fresh
# one on the new. So a MIGRATING workload does not track its quota exactly -
# in either direction. Pin it and the deviation disappears.
echo
echo "### Is the quota exact? 3 runs each, free to migrate vs pinned to CPU 0"
sysctl kernel.sched_cfs_bandwidth_slice_us
measure() {  # measure <quota-us> <pin|no> -> effective cores
    echo "$1 100000" > "$cg/cpu.max"
    local u0 u1 t0 t1
    u0=$(awk '/^usage_usec/{print $2}' "$cg/cpu.stat"); t0=$(date +%s.%N)
    if [[ $2 == pin ]]; then cg_run "$cg" taskset -c 0 bash -c 'while :; do :; done'
    else                     cg_run "$cg" bash -c 'while :; do :; done'; fi
    sleep 6
    cg_kill "$cg"
    u1=$(awk '/^usage_usec/{print $2}' "$cg/cpu.stat"); t1=$(date +%s.%N)
    python3 -c "print(f'{($u1-$u0)/1e6/($t1-$t0):.3f}')"
}
for quota in 20000 50000; do
    for mode in no pin; do
        vals=(); for i in 1 2 3; do vals+=("$(measure "$quota" "$mode")"); done
        printf '  quota=%.1f CPU  migrating=%-3s  measured: %s\n' \
               "$(python3 -c "print($quota/100000)")" "$([[ $mode == pin ]] && echo no || echo yes)" "${vals[*]}"
    done
done

echo
echo "### Kubernetes mapping"
echo "  limits.cpu: 200m -> kubelet writes '20000 100000' into the container's cpu.max."
echo "  nr_throttled / throttled_usec is what container_cpu_cfs_throttled_* exports."
echo "  Threads buy no throughput under a quota: they burn it faster, then all stop."
echo "  And the quota is only exact when the workload cannot migrate - which is"
echo "  precisely what the CPU Manager 'static' policy buys a Guaranteed pod."
```

```text
### A fresh cgroup has no CPU limit
blogdemo/cpu-max/cpu.max:    max 100000 

### Apply 0.2 CPU: quota 20000us per 100000us period == limits.cpu: 200m
blogdemo/cpu-max/cpu.max:    20000 100000 

### 1 busy thread(s), 5s wall time, under the 0.2 CPU quota
  raw cpu.stat before: usage_usec=0 nr_periods=0 nr_throttled=0 throttled_usec=0 
  raw cpu.stat after : usage_usec=1007222 nr_periods=51 nr_throttled=50 throttled_usec=3959893 
  1-thread: nr_periods=51 nr_throttled=50 throttled=3.96s usage=1.01s
            -> throttled in 98% of periods, effective 0.197 CPU

### 4 busy thread(s), 5s wall time, under the 0.2 CPU quota
  raw cpu.stat before: usage_usec=1007222 nr_periods=53 nr_throttled=50 throttled_usec=3959893 
  raw cpu.stat after : usage_usec=2235875 nr_periods=103 nr_throttled=100 throttled_usec=9770208 
  4-thread: nr_periods=50 nr_throttled=50 throttled=5.81s usage=1.23s
            -> throttled in 100% of periods, effective 0.246 CPU

### Is the quota exact? 3 runs each, free to migrate vs pinned to CPU 0
kernel.sched_cfs_bandwidth_slice_us = 5000
  quota=0.2 CPU  migrating=yes  measured: 0.230 0.198 0.200
  quota=0.2 CPU  migrating=no   measured: 0.198 0.198 0.198
  quota=0.5 CPU  migrating=yes  measured: 0.499 0.454 0.415
  quota=0.5 CPU  migrating=no   measured: 0.495 0.502 0.497

### Kubernetes mapping
  limits.cpu: 200m -> kubelet writes '20000 100000' into the container's cpu.max.
  nr_throttled / throttled_usec is what container_cpu_cfs_throttled_* exports.
  Threads buy no throughput under a quota: they burn it faster, then all stop.
  And the quota is only exact when the workload cannot migrate - which is
  precisely what the CPU Manager 'static' policy buys a Guaranteed pod.
```

### 2. `cpu.max.burst` — letting a spiky workload bank unused quota

The workload here averages 0.1 CPU against a 0.2 quota and is *still* throttled,
because its demand arrives in 20 ms spikes. Burst lets it carry a little unused
quota forward. Note the honest result: throttling drops but does not vanish.

```bash
#!/usr/bin/env bash
# 02-cpu-burst.sh - cpu.max.burst: let a bursty workload borrow unused quota.
# Bursty request handlers are the common victim of CFS throttling: they idle,
# then need a short spike. Burst lets them bank a little unused quota.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

cg=$(cg_create cpu-burst)
trap 'cg_destroy "$cg"' EXIT

echo "20000 100000" > "$cg/cpu.max"      # 0.2 CPU
echo "### Limit and burst defaults"
show "$cg/cpu.max"
show "$cg/cpu.max.burst"                 # 0 = no banking allowed

stats() { awk '/^(nr_periods|nr_throttled|nr_bursts|burst_usec|usage_usec)/{printf "%s=%s ", $1,$2}' "$cg/cpu.stat"; }

# A spiky workload: 20ms of CPU every 200ms. Mean demand is 0.1 CPU - well
# under the 0.2 quota - but each spike wants a full core briefly.
spiky() {
    cg_run "$cg" python3 -c '
import time
end = time.time() + 8
while time.time() < end:
    spin_until = time.time() + 0.02      # 20ms of solid CPU
    while time.time() < spin_until: pass
    time.sleep(0.18)                     # then idle
'
}

for burst in 0 20000; do
    echo
    echo "### cpu.max.burst = $burst"
    echo "$burst" > "$cg/cpu.max.burst"
    show "$cg/cpu.max.burst"
    b=$(stats); spiky; sleep 8; cg_kill "$cg"; a=$(stats)
    echo "  before: $b"
    echo "  after : $a"
    python3 - "$b" "$a" <<'PY'
import sys
b = dict(kv.split('=') for kv in sys.argv[1].split()); a = dict(kv.split('=') for kv in sys.argv[2].split())
d = {k: int(a[k]) - int(b[k]) for k in a}
print(f"  delta : periods={d['nr_periods']} throttled={d['nr_throttled']} "
      f"nr_bursts={d['nr_bursts']} burst_usec={d['burst_usec']} usage={d['usage_usec']/1e6:.2f}s")
PY
done

echo
echo "### Kubernetes mapping"
echo "  There is no burst field in a PodSpec - this is set by runtimes/annotations"
echo "  (Alibaba's CPU Burst, containerd's cpu.burst support). Worth knowing because"
echo "  it is the honest fix for 'we raised limits.cpu just to stop the throttling',"
echo "  which wastes the whole node's allocatable to smooth a 20ms spike."
```

```text
### Limit and burst defaults
blogdemo/cpu-burst/cpu.max:  20000 100000 
blogdemo/cpu-burst/cpu.max.burst: 0 

### cpu.max.burst = 0
blogdemo/cpu-burst/cpu.max.burst: 0 
  before: usage_usec=0 nr_periods=0 nr_throttled=0 nr_bursts=0 burst_usec=0 
  after : usage_usec=846827 nr_periods=81 nr_throttled=10 nr_bursts=0 burst_usec=0 
  delta : periods=81 throttled=10 nr_bursts=0 burst_usec=0 usage=0.85s

### cpu.max.burst = 20000
blogdemo/cpu-burst/cpu.max.burst: 20000 
  before: usage_usec=846827 nr_periods=82 nr_throttled=10 nr_bursts=0 burst_usec=0 
  after : usage_usec=1732967 nr_periods=163 nr_throttled=15 nr_bursts=26 burst_usec=38207 
  delta : periods=81 throttled=5 nr_bursts=26 burst_usec=38207 usage=0.89s

### Kubernetes mapping
  There is no burst field in a PodSpec - this is set by runtimes/annotations
  (Alibaba's CPU Burst, containerd's cpu.burst support). Worth knowing because
  it is the honest fix for 'we raised limits.cpu just to stop the throttling',
  which wastes the whole node's allocatable to smooth a 20ms spike.
```

### 3. `cpu.weight` — what `requests.cpu` actually buys

Weight is a share of *contended* CPU. The four cases here are the whole argument:
pinned to one CPU the configured 8:1 ratio holds; unpinned with one thread each on
a 4-CPU box the ratio is 1:1, because two threads on four CPUs never compete at
all; genuinely oversubscribed it reappears but diluted; and alone, a `weight=100`
cgroup takes the entire machine.

If you take one thing from this post: **`requests.cpu` is not a floor you are
guaranteed and not a cap you are held to. It only means anything under contention.**

```bash
#!/usr/bin/env bash
# 03-cpu-weight.sh - cpu.weight: the knob behind `requests.cpu`.
# Weight is a share of *contended* CPU, applied per run-queue. Three cases
# show what that really means - and why requests are so widely misread.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

demo_root_init
low=$(cg_create weight-low)
high=$(cg_create weight-high)
trap 'cg_destroy "$low"; cg_destroy "$high"' EXIT

echo "### Default weight is 100 for every new cgroup"
show "$low/cpu.weight"; show "$high/cpu.weight"

echo
echo "### Configure an 8:1 ratio"
echo 100 > "$low/cpu.weight"; echo 800 > "$high/cpu.weight"
show "$low/cpu.weight"; show "$high/cpu.weight"

NCPU=$(nproc)
usage() { awk '/^usage_usec/{print $2}' "$1/cpu.stat"; }
contend() {  # contend <threads-each> <pin|free> <label>
    local n=$1 mode=$2 label=$3 l0 h0 l1 h1
    l0=$(usage "$low"); h0=$(usage "$high")
    for cg in "$low" "$high"; do
        if [[ $mode == pin ]]; then
            cg_run "$cg" bash -c "for i in \$(seq $n); do taskset -c 0 bash -c 'while :; do :; done' & done; wait"
        else
            cg_run "$cg" bash -c "for i in \$(seq $n); do while :; do :; done & done; wait"
        fi
    done
    sleep 8
    cg_kill "$low"; cg_kill "$high"
    l1=$(usage "$low"); h1=$(usage "$high")
    python3 -c "
l, h = ($l1-$l0)/1e6, ($h1-$h0)/1e6
print(f'  $label')
print(f'    weight=100 used {l:5.2f}s CPU   weight=800 used {h:5.2f}s CPU   ratio {h/l:.2f}:1  (configured 8:1)')"
}

echo
echo "### Case 1: both pinned to CPU 0 - genuine contention for one run-queue"
contend 1 pin "1 thread each, both on CPU 0:"

echo
echo "### Case 2: unpinned, 1 thread each, on a ${NCPU}-CPU box"
contend 1 free "1 thread each, free to migrate:"
echo "    ==> 1:1. Two threads on $NCPU CPUs never compete, so weight is never consulted."

echo
echo "### Case 3: unpinned, 8 threads each - the machine is genuinely oversubscribed"
contend 8 free "8 threads each, free to migrate:"
echo "    ==> the ratio reappears, but diluted: weight is enforced per run-queue and"
echo "        the load balancer keeps moving threads between them."

echo
echo "### Case 4: no competition at all"
l0=$(usage "$low")
cg_run "$low" bash -c "for i in \$(seq $NCPU); do while :; do :; done & done; wait"
sleep 5; cg_kill "$low"
l1=$(usage "$low")
python3 -c "
l = ($l1-$l0)/1e6
print(f'    weight=100 cgroup alone: {l:.2f}s CPU over 5s wall = {l/5:.2f} cores')
print(f'    ==> the low-weight cgroup takes the WHOLE machine when nobody competes.')"

echo
echo "### Kubernetes mapping"
echo "  requests.cpu is a scheduling promise plus a contention share - never a cap."
echo "  kubelet: shares = max(2, milliCPU * 1024 / 1000); runc then converts to"
echo "  cpu.weight = 1 + ((shares - 2) * 9999) / 262142."
echo "  Consequences worth internalising:"
echo "   - A pod with requests.cpu: 100m can use every core on an idle node."
echo "   - Its share only binds when the node is genuinely oversubscribed."
echo "   - Ratios you configure are approached, not guaranteed, once threads migrate."
```

```text
### Default weight is 100 for every new cgroup
blogdemo/weight-low/cpu.weight: 100 
blogdemo/weight-high/cpu.weight: 100 

### Configure an 8:1 ratio
blogdemo/weight-low/cpu.weight: 100 
blogdemo/weight-high/cpu.weight: 800 

### Case 1: both pinned to CPU 0 - genuine contention for one run-queue
  1 thread each, both on CPU 0:
    weight=100 used  0.94s CPU   weight=800 used  7.06s CPU   ratio 7.50:1  (configured 8:1)

### Case 2: unpinned, 1 thread each, on a 4-CPU box
  1 thread each, free to migrate:
    weight=100 used  8.06s CPU   weight=800 used  8.16s CPU   ratio 1.01:1  (configured 8:1)
    ==> 1:1. Two threads on 4 CPUs never compete, so weight is never consulted.

### Case 3: unpinned, 8 threads each - the machine is genuinely oversubscribed
  8 threads each, free to migrate:
    weight=100 used  7.14s CPU   weight=800 used 24.98s CPU   ratio 3.50:1  (configured 8:1)
    ==> the ratio reappears, but diluted: weight is enforced per run-queue and
        the load balancer keeps moving threads between them.

### Case 4: no competition at all
    weight=100 cgroup alone: 18.28s CPU over 5s wall = 3.66 cores
    ==> the low-weight cgroup takes the WHOLE machine when nobody competes.

### Kubernetes mapping
  requests.cpu is a scheduling promise plus a contention share - never a cap.
  kubelet: shares = max(2, milliCPU * 1024 / 1000); runc then converts to
  cpu.weight = 1 + ((shares - 2) * 9999) / 262142.
  Consequences worth internalising:
   - A pod with requests.cpu: 100m can use every core on an idle node.
   - Its share only binds when the node is genuinely oversubscribed.
   - Ratios you configure are approached, not guaranteed, once threads migrate.
```

### 4. `cpuset` — placement, and the top-down enablement rule

Two lessons. First, the v2 rule that catches everyone: a controller must be
enabled in the *parent's* `cgroup.subtree_control` before children have its files
at all — which is why a fresh child cgroup here has zero `cpuset.*` files until
the root delegates it. Second, pinning: this is what CPU Manager's `static` policy
does for a Guaranteed pod with integer CPU limits, and per scenario 1 it is also
the only way to get an exactly-enforced quota.

```bash
#!/usr/bin/env bash
# 04-cpuset.sh - cpuset.cpus / cpuset.mems: placement, not bandwidth.
# Also demonstrates the v2 rule that trips everyone up: a controller must be
# enabled in the PARENT's cgroup.subtree_control before a child can use it.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

echo "### The v2 top-down enablement rule"
echo "  root cgroup.controllers:     $(cat $CG_ROOT/cgroup.controllers)"
echo "  root cgroup.subtree_control: $(cat $CG_ROOT/cgroup.subtree_control)"
echo "  ==> cpuset is AVAILABLE but not DELEGATED, so children have no cpuset.* files."

demo_root_init
cg=$(cg_create cpuset)
echo "  files in a fresh child before enabling: $(ls $cg | grep -c cpuset) cpuset.* files"

# Enabling cpuset at the root is a host change - record it before touching it.
orig_root_sc=$(cat "$CG_ROOT/cgroup.subtree_control")
cpuset_added=no
if ! grep -qw cpuset <<< "$orig_root_sc"; then
    record_change "enabled 'cpuset' in root cgroup.subtree_control (was: $orig_root_sc)" \
                  "echo -cpuset > $CG_ROOT/cgroup.subtree_control"
    echo "+cpuset" > "$CG_ROOT/cgroup.subtree_control"
    cpuset_added=yes
fi
cg_enable "$DEMO_ROOT" cpuset
restore() {
    cg_destroy "$cg"
    if [[ $cpuset_added == yes ]]; then
        echo "-cpuset" > "$DEMO_ROOT/cgroup.subtree_control" 2>/dev/null
        cg_destroy "$DEMO_ROOT" 2>/dev/null
        echo "-cpuset" > "$CG_ROOT/cgroup.subtree_control"
        mark_reverted "enabled 'cpuset' in root cgroup.subtree_control"
        echo "  restored root cgroup.subtree_control: $(cat $CG_ROOT/cgroup.subtree_control)"
    fi
}
trap restore EXIT

echo
echo "### After 'echo +cpuset > $CG_ROOT/cgroup.subtree_control'"
echo "  root cgroup.subtree_control: $(cat $CG_ROOT/cgroup.subtree_control)"
cg=$(cg_create cpuset)
show "$cg/cpuset.cpus"                    # empty = inherit everything
show "$cg/cpuset.cpus.effective"
show "$cg/cpuset.mems.effective"

echo
echo "### Pin the cgroup to CPU 2"
echo 2 > "$cg/cpuset.cpus"
show "$cg/cpuset.cpus"
show "$cg/cpuset.cpus.effective"

cg_run "$cg" bash -c 'while :; do :; done'
sleep 1
pid=$(head -1 "$cg/cgroup.procs")
echo "  worker pid=$pid"
echo "  /proc/$pid/status Cpus_allowed_list: $(awk '/Cpus_allowed_list/{print $2}' /proc/$pid/status)"
echo "  taskset -pc $pid: $(taskset -pc "$pid" | sed 's/.*: //')"
echo "  per-CPU time (5s of a busy loop pinned to CPU 2):"
sleep 5
awk '{printf "    %s\n", $0}' <(ps -o psr,pcpu,comm -p "$pid" --no-headers | sed 's/^ *//;s/^/psr(cpu)=/')
cg_kill "$cg"

echo
echo "### Widen to CPUs 0-1 and confirm the kernel moved it"
cg_run "$cg" bash -c 'while :; do :; done'
sleep 0.5
pid=$(head -1 "$cg/cgroup.procs")
echo 0-1 > "$cg/cpuset.cpus"
sleep 2
echo "  cpuset.cpus now: $(cat $cg/cpuset.cpus), task Cpus_allowed_list: $(awk '/Cpus_allowed_list/{print $2}' /proc/$pid/status)"
echo "  running on CPU: $(ps -o psr= -p $pid | tr -d ' ')"
cg_kill "$cg"

echo
echo "### NUMA placement"
echo "  cpuset.mems.effective: $(cat $cg/cpuset.mems.effective)  (this host has $(numactl -H | awk '/available:/{print $2}') NUMA node)"
echo "  On a multi-socket node this is what keeps a pod's memory local to its CPUs."

echo
echo "### Kubernetes mapping"
echo "  CPU Manager 'static' policy (KEP-3570) writes cpuset.cpus for Guaranteed pods"
echo "  with integer CPU limits - exclusive cores, no migration, no CFS dilution"
echo "  (see scenario 01, where pinning was the only way to get an exact quota)."
echo "  Memory Manager + Topology Manager do the same for cpuset.mems on NUMA nodes."
echo "  Everything else on the node keeps sharing the remaining shared pool."
```

```text
### The v2 top-down enablement rule
  root cgroup.controllers:     cpuset cpu io memory hugetlb pids rdma misc
  root cgroup.subtree_control: cpu memory pids
  ==> cpuset is AVAILABLE but not DELEGATED, so children have no cpuset.* files.
  files in a fresh child before enabling: 0 cpuset.* files

### After 'echo +cpuset > /sys/fs/cgroup/cgroup.subtree_control'
  root cgroup.subtree_control: cpuset cpu memory pids
blogdemo/cpuset/cpuset.cpus:  
blogdemo/cpuset/cpuset.cpus.effective: 0-3 
blogdemo/cpuset/cpuset.mems.effective: 0 

### Pin the cgroup to CPU 2
blogdemo/cpuset/cpuset.cpus: 2 
blogdemo/cpuset/cpuset.cpus.effective: 2 
  worker pid=172231
  /proc/172231/status Cpus_allowed_list: 2
  taskset -pc 172231: 2
  per-CPU time (5s of a busy loop pinned to CPU 2):
    psr(cpu)=2 99.1 bash

### Widen to CPUs 0-1 and confirm the kernel moved it
  cpuset.cpus now: 0-1, task Cpus_allowed_list: 0-1
  running on CPU: 1

### NUMA placement
  cpuset.mems.effective: 0  (this host has 1 NUMA node)
  On a multi-socket node this is what keeps a pod's memory local to its CPUs.

### Kubernetes mapping
  CPU Manager 'static' policy (KEP-3570) writes cpuset.cpus for Guaranteed pods
  with integer CPU limits - exclusive cores, no migration, no CFS dilution
  (see scenario 01, where pinning was the only way to get an exact quota).
  Memory Manager + Topology Manager do the same for cpuset.mems on NUMA nodes.
  Everything else on the node keeps sharing the remaining shared pool.
  restored root cgroup.subtree_control: cpu memory pids
```

### 5. `memory.max` — where OOMKilled and exit 137 come from

There is no soft landing. The allocation that crosses the limit triggers reclaim,
and if reclaim cannot free enough, the kernel SIGKILLs inside the cgroup. The
kubelet neither decides this nor can intervene; it reports the corpse. The exit
status below is literally 137 = 128 + SIGKILL, and the kernel's own `dmesg` line
names the cgroup.

Then `memory.oom.group`, which turns "one process died and the container is now
half-alive" into "the whole cgroup died at once" — 7 processes, one kill.

```bash
#!/usr/bin/env bash
# 05-memory-max.sh - memory.max: the knob behind `limits.memory`, and the
# kernel behaviour behind the OOMKilled/137 you see in kubectl describe.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

cg=$(cg_create mem-max)
trap 'cg_destroy "$cg"' EXIT

echo "### A fresh cgroup has no memory limit"
show "$cg/memory.max"
show "$cg/memory.current"

echo
echo "### Apply a 64 MiB hard limit (== limits.memory: 64Mi)"
echo $((64 * 1024 * 1024)) > "$cg/memory.max"
show "$cg/memory.max"

echo
echo "### memory.events before"
cat "$cg/memory.events" | sed 's/^/  /'

echo
echo "### Allocate 256 MiB of anonymous memory inside the 64 MiB cgroup"
set +e
cg_exec "$cg" python3 -c "$MEM_HOG" 256 2>&1 | grep -v 'Killed' | tail -3 | sed 's/^/  /'
rc=${PIPESTATUS[0]}
set -e
echo "  exit status: $rc   (137 = 128 + SIGKILL(9) - exactly what a runtime reports as OOMKilled)"

echo
echo "### memory.events after"
cat "$cg/memory.events" | sed 's/^/  /'
echo "  memory.peak: $(cat $cg/memory.peak 2>/dev/null) bytes"

echo
echo "### The kernel's own record of the kill"
dmesg | grep -i -E 'Memory cgroup out of memory|oom-kill' | tail -3 | sed 's/^/  /'

echo
echo "### memory.oom.group: kill the whole cgroup, not just the greediest task"
echo 1 > "$cg/memory.oom.group"
show "$cg/memory.oom.group"
# three siblings; one of them will trip the limit
set +e
CG=$cg cg_exec "$cg" bash -c '
    mem_hog 8 20 >/dev/null 2>&1 &
    mem_hog 8 20 >/dev/null 2>&1 &
    sleep 1
    echo "    processes in the cgroup before the kill: $(wc -l < $CG/cgroup.procs)"
    mem_hog 256 >/dev/null 2>&1
' 2>&1 | grep -v 'Killed'
echo "  exit status: ${PIPESTATUS[0]}"
set -e
sleep 0.5
echo "  processes left in the cgroup afterwards: $(wc -l < $cg/cgroup.procs)"
echo "  memory.events now:"
cat "$cg/memory.events" | sed 's/^/    /' 

echo
echo "### Kubernetes mapping"
echo "  limits.memory -> memory.max. There is no soft landing: the allocation that"
echo "  crosses the limit triggers reclaim, and if reclaim cannot free enough the"
echo "  kernel SIGKILLs. The kubelet does not decide this and cannot intervene -"
echo "  it only reports the corpse as OOMKilled with exit code 137."
echo "  memory.oom.group is why a runtime can lose every process in a container at"
echo "  once instead of leaving a half-dead container with its PID 1 still up."
```

```text
### A fresh cgroup has no memory limit
blogdemo/mem-max/memory.max: max 
blogdemo/mem-max/memory.current: 0 

### Apply a 64 MiB hard limit (== limits.memory: 64Mi)
blogdemo/mem-max/memory.max: 67108864 

### memory.events before
  low 0
  high 0
  max 0
  oom 0
  oom_kill 0
  oom_group_kill 0

### Allocate 256 MiB of anonymous memory inside the 64 MiB cgroup
  allocated 57 MiB
  allocated 58 MiB
  allocated 59 MiB
  exit status: 137   (137 = 128 + SIGKILL(9) - exactly what a runtime reports as OOMKilled)

### memory.events after
  low 0
  high 0
  max 35
  oom 1
  oom_kill 1
  oom_group_kill 0
  memory.peak: 67108864 bytes

### The kernel's own record of the kill
  [82692.141979] python3 invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=0
  [82692.143136] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=/,mems_allowed=0,oom_memcg=/blogdemo/mem-max,task_memcg=/blogdemo/mem-max,task=python3,pid=172759,uid=0
  [82692.143155] Memory cgroup out of memory: Killed process 172759 (python3) total-vm:79652kB, anon-rss:65280kB, file-rss:6656kB, shmem-rss:0kB, UID:0 pgtables:196kB oom_score_adj:0

### memory.oom.group: kill the whole cgroup, not just the greediest task
blogdemo/mem-max/memory.oom.group: 1 
    processes in the cgroup before the kill: 7
  exit status: 137
  processes left in the cgroup afterwards: 0
  memory.events now:
    low 0
    high 0
    max 54
    oom 2
    oom_kill 8
    oom_group_kill 1

### Kubernetes mapping
  limits.memory -> memory.max. There is no soft landing: the allocation that
  crosses the limit triggers reclaim, and if reclaim cannot free enough the
  kernel SIGKILLs. The kubelet does not decide this and cannot intervene -
  it only reports the corpse as OOMKilled with exit code 137.
  memory.oom.group is why a runtime can lose every process in a container at
  once instead of leaving a half-dead container with its PID 1 still up.
```

### 6. Page cache — why your memory graph is lying to you

The most useful scenario here. A 512 MiB file written and read inside a 256 MiB
cgroup pins `memory.current` at the limit — with a **working set of 1.2 MiB** and
zero OOM kills, because page cache is reclaimable. That gap is exactly
`container_memory_usage_bytes` versus `container_memory_working_set_bytes`, and it
is why alerting on the former against the limit pages you at 3am for a pod that is
perfectly healthy.

The contrast at the end matters: swap the page cache for anonymous memory and the
same limit kills immediately.

```bash
#!/usr/bin/env bash
# 06-working-set.sh - why memory.current is NOT what your pod is "using".
# Page cache is charged to the cgroup that faulted it in. It counts towards
# memory.max, it is reclaimable, and it is the reason memory graphs creep
# towards the limit and then flatten. This is the gap between
# container_memory_usage_bytes and container_memory_working_set_bytes.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

cg=$(cg_create workingset)
FILE=/var/tmp/cgroup-demo-pagecache.bin
trap 'cg_destroy "$cg"; rm -f "$FILE"' EXIT

LIMIT=$((256 * 1024 * 1024))
echo "$LIMIT" > "$cg/memory.max"
echo "### 256 MiB limit, nothing running yet"
show "$cg/memory.max"
show "$cg/memory.current"

stat_of() { awk -v k="$1" '$1 == k {print $2}' "$cg/memory.stat"; }
snapshot() {
    local cur anon file inactive active
    cur=$(cat "$cg/memory.current")
    anon=$(stat_of anon); file=$(stat_of file)
    inactive=$(stat_of inactive_file); active=$(stat_of active_file)
    python3 -c "
mb = lambda b: f'{int(b)/1048576:7.1f} MiB'
cur, anon, file_, inact, act = $cur, $anon, $file, $inactive, $active
print(f'    memory.current {mb(cur)}   anon {mb(anon)}   file(page cache) {mb(file_)}')
print(f'    inactive_file  {mb(inact)}   active_file {mb(act)}')
print(f'    working set    {mb(cur - inact)}  <-- memory.current - inactive_file')"
}

echo
echo "### Write a 512 MiB file from INSIDE the cgroup (twice the limit)"
cg_exec "$cg" dd if=/dev/zero of="$FILE" bs=1M count=512 2>&1 | tail -1 | sed 's/^/  /'
sync
echo "  after writing:"
snapshot

echo
echo "### Read the whole file back in, again from inside the cgroup"
cg_exec "$cg" dd if="$FILE" of=/dev/null bs=1M 2>&1 | tail -1 | sed 's/^/  /'
echo "  after reading:"
snapshot

echo
echo "### Did it get OOM-killed for exceeding a 256 MiB limit with a 512 MiB file?"
cat "$cg/memory.events" | sed 's/^/    /'
echo "  ==> oom_kill is 0. Page cache is reclaimable: the kernel evicts clean pages"
echo "      instead of killing. 'max' counts the times the limit forced reclaim."

echo
echo "### memory.reclaim: ask the kernel to give it back (proactive reclaim)"
echo "  memory.current before: $(( $(cat $cg/memory.current) / 1048576 )) MiB"
echo $((200 * 1024 * 1024)) > "$cg/memory.reclaim" 2>/dev/null || echo "  (memory.reclaim write returned $?)"
echo "  memory.current after : $(( $(cat $cg/memory.current) / 1048576 )) MiB"
snapshot

echo
echo "### Now the same limit, but with ANONYMOUS memory (not reclaimable)"
cg2=$(cg_create workingset-anon)
echo $((64 * 1024 * 1024)) > "$cg2/memory.max"
set +e
cg_exec "$cg2" python3 -c "$MEM_HOG" 256 >/dev/null 2>&1
echo "  exit status: $?  (anonymous memory cannot be dropped, so this one dies)"
set -e
cg_destroy "$cg2"

echo
echo "### Kubernetes mapping"
echo "  container_memory_usage_bytes      = memory.current  (includes page cache)"
echo "  container_memory_working_set_bytes = memory.current - inactive_file"
echo "  The kubelet evicts on WORKING SET, not on memory.current."
echo "  Practical consequences:"
echo "   - A pod that writes logs or reads large files will show usage climbing to"
echo "     its limit. That is normal and is not a leak."
echo "   - Alerting on container_memory_usage_bytes / limit produces false pages."
echo "   - But cache is not free: reclaim costs CPU and IO latency, and a workload"
echo "     whose hot file set exceeds its limit will thrash rather than die."
```

```text
### 256 MiB limit, nothing running yet
blogdemo/workingset/memory.max: 268435456 
blogdemo/workingset/memory.current: 0 

### Write a 512 MiB file from INSIDE the cgroup (twice the limit)
  536870912 bytes (537 MB, 512 MiB) copied, 7.25122 s, 74.0 MB/s
  after writing:
    memory.current   254.8 MiB   anon     0.0 MiB   file(page cache)   246.8 MiB
    inactive_file    246.8 MiB   active_file     0.1 MiB
    working set        8.0 MiB  <-- memory.current - inactive_file

### Read the whole file back in, again from inside the cgroup
  536870912 bytes (537 MB, 512 MiB) copied, 2.96127 s, 181 MB/s
  after reading:
    memory.current   254.8 MiB   anon     0.0 MiB   file(page cache)   253.6 MiB
    inactive_file    253.5 MiB   active_file     0.1 MiB
    working set        1.2 MiB  <-- memory.current - inactive_file

### Did it get OOM-killed for exceeding a 256 MiB limit with a 512 MiB file?
    low 0
    high 0
    max 3082
    oom 0
    oom_kill 0
    oom_group_kill 0
  ==> oom_kill is 0. Page cache is reclaimable: the kernel evicts clean pages
      instead of killing. 'max' counts the times the limit forced reclaim.

### memory.reclaim: ask the kernel to give it back (proactive reclaim)
  memory.current before: 254 MiB
  memory.current after : 54 MiB
    memory.current    54.8 MiB   anon     0.0 MiB   file(page cache)    53.6 MiB
    inactive_file     53.5 MiB   active_file     0.1 MiB
    working set        1.2 MiB  <-- memory.current - inactive_file

### Now the same limit, but with ANONYMOUS memory (not reclaimable)
  exit status: 137  (anonymous memory cannot be dropped, so this one dies)

### Kubernetes mapping
  container_memory_usage_bytes      = memory.current  (includes page cache)
  container_memory_working_set_bytes = memory.current - inactive_file
  The kubelet evicts on WORKING SET, not on memory.current.
  Practical consequences:
   - A pod that writes logs or reads large files will show usage climbing to
     its limit. That is normal and is not a leak.
   - Alerting on container_memory_usage_bytes / limit produces false pages.
   - But cache is not free: reclaim costs CPU and IO latency, and a workload
     whose hot file set exceeds its limit will thrash rather than die.
```

### 7. `memory.high` and `memory.min` — pressure instead of death

`memory.max` kills; `memory.high` throttles the allocator and reclaims. Same
overshoot, 127× slower instead of dead, `oom_kill` still 0.

Look closely at `pgscan` versus `pgsteal`: the kernel scanned 10,334 pages and
reclaimed 165. With no swap, anonymous memory cannot be evicted, so reclaim burns
CPU and frees nothing and all the kernel can do is stall the allocator. That is
`memory.high` as a brake with no disc — and it is why MemoryQoS and NodeSwap keep
being discussed in the same breath.

Then `memory.min`: under a squeezed parent, the protected sibling keeps 99 MiB
while the unprotected one is reclaimed to 0.

```bash
#!/usr/bin/env bash
# 07-memory-high.sh - memory.high and memory.min: pressure instead of death.
# memory.max kills. memory.high throttles the allocator and reclaims, turning
# an OOM into latency. memory.min makes a slice of memory unreclaimable.
# Together these are what KEP-2570 (MemoryQoS) wires up from requests/limits.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

# ---------- Part 1: memory.high throttles instead of killing ----------------
cg=$(cg_create mem-high)
trap 'cg_destroy "$cg"' EXIT

echo "### memory.high = 64 MiB, memory.max left unlimited"
echo $((64 * 1024 * 1024)) > "$cg/memory.high"
show "$cg/memory.high"
show "$cg/memory.max"

echo
echo "### Allocate 200 MiB anonymous - far past the high watermark"
echo "  (bounded to 20s: with no swap there is nothing reclaimable, so the"
echo "   kernel just keeps stalling the allocator. That is the whole point.)"
got=$(timeout 20 bash -c 'echo $$ > "$1/cgroup.procs"; shift; exec "$@"' _ "$cg" \
        python3 -c "$MEM_HOG" 200 2>/dev/null | grep -c '^allocated')
echo "  allocated in 20s under memory.high=64Mi: $got MiB"
echo "  memory.peak (high-water mark while it ran): $(( $(cat $cg/memory.peak) / 1048576 )) MiB"
echo "  memory.events:"
sed 's/^/    /' "$cg/memory.events"
echo "  ==> 'high' counts allocation-throttling events. 'oom_kill' is still 0:"
echo "      the workload is being slowed to a crawl, not killed."
awk '/^(pgscan |pgsteal |pgmajfault)/{printf "    %-12s %s\n", $1, $2}' "$cg/memory.stat"
echo "  ==> note the ratio: the kernel SCANNED thousands of pages and managed to"
echo "      STEAL almost none. This host has no swap, so anonymous pages cannot be"
echo "      evicted - reclaim burns CPU and frees nothing, and all the kernel can"
echo "      do is keep stalling the allocator. memory.high without swap is a brake"
echo "      with no disc, which is why KEP-2400 (NodeSwap) and KEP-2570 (MemoryQoS)"
echo "      are always discussed together."
cg_kill "$cg"

echo
echo "### The same 200 MiB with memory.high removed (memory.max = 512Mi)"
echo max                      > "$cg/memory.high"
echo $((512 * 1024 * 1024))   > "$cg/memory.max"
t0=$(date +%s.%N)
cg_exec "$cg" python3 -c "$MEM_HOG" 200 >/dev/null 2>&1
t1=$(date +%s.%N)
python3 -c "
t = $t1 - $t0
print(f'  completed 200 MiB in {t:.2f}s = {200/t:.0f} MiB/s unthrottled')
print(f'  throttled run managed $got MiB in 20s = {$got/20:.1f} MiB/s')
print(f'  ==> memory.high slowed allocation by roughly {(200/t)/max($got/20, 0.01):.0f}x')"
cg_destroy "$cg"

# ---------- Part 2: memory.min protects against reclaim ---------------------
# Parent with a hard limit, three children competing underneath it.
echo
echo "### memory.min: who loses their page cache when the parent is squeezed?"
parent=$(cg_create qos)
cg_enable "$parent" memory
prot=$parent/protected;   mkdir -p "$prot"
unprot=$parent/unprotected; mkdir -p "$unprot"
greedy=$parent/greedy;    mkdir -p "$greedy"
F1=/var/tmp/cgroup-demo-prot.bin
F2=/var/tmp/cgroup-demo-unprot.bin
cleanup2() { for d in "$prot" "$unprot" "$greedy"; do cg_destroy "$d"; done
             cg_destroy "$parent"; rm -f "$F1" "$F2"; }
trap 'cleanup2; cg_destroy "$cg"' EXIT

echo $((300 * 1024 * 1024)) > "$parent/memory.max"
echo $((100 * 1024 * 1024)) > "$prot/memory.min"      # protected floor
echo 0                      > "$unprot/memory.min"    # no protection
echo "  parent memory.max     : $(( $(cat $parent/memory.max) / 1048576 )) MiB"
echo "  protected memory.min  : $(( $(cat $prot/memory.min) / 1048576 )) MiB"
echo "  unprotected memory.min: $(( $(cat $unprot/memory.min) / 1048576 )) MiB"

dd if=/dev/urandom of="$F1" bs=1M count=100 status=none
dd if=/dev/urandom of="$F2" bs=1M count=100 status=none
sync; echo 3 > /proc/sys/vm/drop_caches      # start from a cold cache

cg_exec "$prot"   dd if="$F1" of=/dev/null bs=1M status=none
cg_exec "$unprot" dd if="$F2" of=/dev/null bs=1M status=none
echo
echo "  after both cached 100 MiB each:"
echo "    protected  memory.current: $(( $(cat $prot/memory.current) / 1048576 )) MiB"
echo "    unprotected memory.current: $(( $(cat $unprot/memory.current) / 1048576 )) MiB"

echo
echo "  now a greedy sibling allocates 250 MiB inside the same 300 MiB parent..."
set +e
cg_exec "$greedy" python3 -c "$MEM_HOG" 250 3 >/dev/null 2>&1
set -e
echo "  after the squeeze:"
echo "    protected  memory.current: $(( $(cat $prot/memory.current) / 1048576 )) MiB"
echo "    unprotected memory.current: $(( $(cat $unprot/memory.current) / 1048576 )) MiB"
echo "    protected  memory.events low: $(awk '/^low /{print $2}' $prot/memory.events)"
echo "    unprotected memory.events low: $(awk '/^low /{print $2}' $unprot/memory.events)"
echo "  ==> reclaim takes from the unprotected sibling first; the protected one"
echo "      keeps its floor because memory.min is unreclaimable."

echo
echo "### Kubernetes mapping (KEP-2570, MemoryQoS)"
echo "  memory.min  <- requests.memory        (a floor the kernel will not reclaim)"
echo "  memory.high <- limits.memory * memoryThrottlingFactor"
echo "  memory.max  <- limits.memory          (the hard kill line)"
echo "  With MemoryQoS on, a pod that overshoots gets SLOW before it gets KILLED,"
echo "  and a pod within its requests is protected from noisy neighbours on the"
echo "  same node. Without it, requests.memory is only a scheduling number and"
echo "  the first thing you learn about the limit is exit code 137."
```

```text
### memory.high = 64 MiB, memory.max left unlimited
blogdemo/mem-high/memory.high: 67108864 
blogdemo/mem-high/memory.max: max 

### Allocate 200 MiB anonymous - far past the high watermark
  (bounded to 20s: with no swap there is nothing reclaimable, so the
   kernel just keeps stalling the allocator. That is the whole point.)
  allocated in 20s under memory.high=64Mi: 65 MiB
  memory.peak (high-water mark while it ran): 70 MiB
  memory.events:
    low 0
    high 1080
    max 0
    oom 0
    oom_kill 0
    oom_group_kill 0
  ==> 'high' counts allocation-throttling events. 'oom_kill' is still 0:
      the workload is being slowed to a crawl, not killed.
    pgscan       10334
    pgsteal      165
    pgmajfault   0
  ==> note the ratio: the kernel SCANNED thousands of pages and managed to
      STEAL almost none. This host has no swap, so anonymous pages cannot be
      evicted - reclaim burns CPU and frees nothing, and all the kernel can
      do is keep stalling the allocator. memory.high without swap is a brake
      with no disc, which is why KEP-2400 (NodeSwap) and KEP-2570 (MemoryQoS)
      are always discussed together.

### The same 200 MiB with memory.high removed (memory.max = 512Mi)
  completed 200 MiB in 0.48s = 414 MiB/s unthrottled
  throttled run managed 65 MiB in 20s = 3.2 MiB/s
  ==> memory.high slowed allocation by roughly 127x

### memory.min: who loses their page cache when the parent is squeezed?
  parent memory.max     : 300 MiB
  protected memory.min  : 100 MiB
  unprotected memory.min: 0 MiB

  after both cached 100 MiB each:
    protected  memory.current: 100 MiB
    unprotected memory.current: 100 MiB

  now a greedy sibling allocates 250 MiB inside the same 300 MiB parent...
  after the squeeze:
    protected  memory.current: 99 MiB
    unprotected memory.current: 0 MiB
    protected  memory.events low: 0
    unprotected memory.events low: 0
  ==> reclaim takes from the unprotected sibling first; the protected one
      keeps its floor because memory.min is unreclaimable.

### Kubernetes mapping (KEP-2570, MemoryQoS)
  memory.min  <- requests.memory        (a floor the kernel will not reclaim)
  memory.high <- limits.memory * memoryThrottlingFactor
  memory.max  <- limits.memory          (the hard kill line)
  With MemoryQoS on, a pod that overshoots gets SLOW before it gets KILLED,
  and a pod within its requests is protected from noisy neighbours on the
  same node. Without it, requests.memory is only a scheduling number and
  the first thing you learn about the limit is exit code 137.
```

### 8. `memory.swap.max` — the difference between slow and dead

Same 384 MiB workload, same 128 MiB limit, run twice. With swap: **exit 0**. With
`memory.swap.max = 0`: **exit 137**. Swap is the only variable.

This also fixes scenario 7: with somewhere to put the pages, `memory.high`
throttling goes from 65 MiB/20s to the full 200 MiB, and `pgsteal` goes from 165
to 37,068.

This scenario creates a 2 GiB swapfile. It is recorded in the ledger, never added
to `/etc/fstab`, and removed at the end — you can see `SwapTotal` back at 0 in the
output.

```bash
#!/usr/bin/env bash
# 08-swap.sh - memory.swap.max: the difference between "slow" and "dead".
# HOST CHANGE: creates a 2 GiB swapfile. Recorded in the ledger and removed
# at the end of this script; revert.sh will also clean it up if we die early.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

SWAPFILE=/swapfile-cgroupdemo
SIZE_MB=2048

echo "### This host starts with no swap at all"
swapon --show || echo "  (swapon --show prints nothing: no swap devices)"
grep SwapTotal /proc/meminfo | sed 's/^/  /'

echo
echo "### Create and enable a ${SIZE_MB} MiB swapfile"
record_change "created and enabled swapfile $SWAPFILE (${SIZE_MB} MiB)" \
              "swapoff $SWAPFILE; rm -f $SWAPFILE"
dd if=/dev/zero of="$SWAPFILE" bs=1M count=$SIZE_MB status=none
chmod 600 "$SWAPFILE"
mkswap "$SWAPFILE" | sed 's/^/  /'
swapon "$SWAPFILE"
swapon --show | sed 's/^/  /'
grep SwapTotal /proc/meminfo | sed 's/^/  /'
echo "  NOTE: not added to /etc/fstab - this does not survive a reboot by design."

cleanup() {
    cg_destroy "$cg" 2>/dev/null
    swapoff "$SWAPFILE" 2>/dev/null
    rm -f "$SWAPFILE"
    mark_reverted "created and enabled swapfile $SWAPFILE"
    echo
    echo "### Host restored"
    echo "  SwapTotal now: $(awk '/SwapTotal/{print $2, $3}' /proc/meminfo)"
    echo "  $SWAPFILE exists: $([[ -e $SWAPFILE ]] && echo yes || echo no)"
}
cg=$(cg_create swap)
trap cleanup EXIT

echo
echo "### A 128 MiB cgroup, swap allowed (memory.swap.max defaults to max)"
echo $((128 * 1024 * 1024)) > "$cg/memory.max"
show "$cg/memory.max"
show "$cg/memory.swap.max"

echo
echo "### Allocate 384 MiB anonymous - three times the memory limit"
t0=$(date +%s.%N)
set +e
cg_exec "$cg" python3 -c "$MEM_HOG" 384 2 >/dev/null 2>&1
rc=$?
set -e
t1=$(date +%s.%N)
printf '  exit status: %s   wall time: %.2fs\n' "$rc" "$(python3 -c "print($t1-$t0)")"
echo "  memory.peak      : $(( $(cat $cg/memory.peak) / 1048576 )) MiB (capped at the limit)"
echo "  memory.swap.peak : $(( $(cat $cg/memory.swap.peak) / 1048576 )) MiB (the overflow went to swap)"
echo "  memory.events oom_kill: $(awk '/^oom_kill /{print $2}' $cg/memory.events)"
echo "  ==> it SURVIVED. The pages that did not fit were written to swap."

echo
echo "### Now forbid swap for this cgroup: memory.swap.max = 0"
cg_destroy "$cg"; cg=$(cg_create swap)
echo $((128 * 1024 * 1024)) > "$cg/memory.max"
echo 0 > "$cg/memory.swap.max"
show "$cg/memory.swap.max"
t0=$(date +%s.%N)
set +e
cg_exec "$cg" python3 -c "$MEM_HOG" 384 2 >/dev/null 2>&1
rc=$?
set -e
t1=$(date +%s.%N)
printf '  exit status: %s   wall time: %.2fs\n' "$rc" "$(python3 -c "print($t1-$t0)")"
echo "  memory.swap.peak: $(( $(cat $cg/memory.swap.peak) / 1048576 )) MiB"
echo "  memory.events oom_kill: $(awk '/^oom_kill /{print $2}' $cg/memory.events)"
echo "  ==> identical workload, identical limit. Swap is the only difference"
echo "      between exit 0 and exit 137."

echo
echo "### And now memory.high actually works (compare scenario 07)"
cg_destroy "$cg"; cg=$(cg_create swap)
echo $((64 * 1024 * 1024)) > "$cg/memory.high"
got=$(timeout 20 bash -c 'echo $$ > "$1/cgroup.procs"; shift; exec "$@"' _ "$cg" \
        python3 -c "$MEM_HOG" 200 2>/dev/null | grep -c '^allocated')
echo "  allocated in 20s under memory.high=64Mi WITH swap: $got MiB"
echo "  (scenario 07 managed 65 MiB in 20s on the same box with no swap)"
awk '/^(pgscan |pgsteal )/{printf "    %-10s %s\n", $1, $2}' "$cg/memory.stat"
echo "  memory.swap.current: $(( $(cat $cg/memory.swap.current) / 1048576 )) MiB"
echo "  ==> now reclaim has somewhere to put the pages, so throttling reclaims"
echo "      instead of just stalling."

echo
echo "### Kubernetes mapping (KEP-2400, NodeSwap)"
echo "  Swap on nodes was forbidden for years precisely because it makes"
echo "  limits.memory soft: a pod over its limit gets slow instead of killed,"
echo "  and the kubelet's accounting no longer matches RSS."
echo "  NodeSwap re-introduces it deliberately: cgroup v2 only, with"
echo "  swapBehavior LimitedSwap giving Burstable pods a swap allowance"
echo "  proportional to their memory request, while Guaranteed and BestEffort"
echo "  pods get none. memory.swap.max is the file that implements it."
```

```text
### This host starts with no swap at all
  SwapTotal:             0 kB

### Create and enable a 2048 MiB swapfile
  Setting up swapspace version 1, size = 2 GiB (2147479552 bytes)
  no label, UUID=0cdeff3a-f454-41dc-9657-f13a93048e0f
  NAME                 TYPE SIZE USED PRIO
  /swapfile-cgroupdemo file   2G   0B   -2
  SwapTotal:       2097148 kB
  NOTE: not added to /etc/fstab - this does not survive a reboot by design.

### A 128 MiB cgroup, swap allowed (memory.swap.max defaults to max)
blogdemo/swap/memory.max:    134217728 
blogdemo/swap/memory.swap.max: max 

### Allocate 384 MiB anonymous - three times the memory limit
  exit status: 0   wall time: 12.00s
  memory.peak      : 128 MiB (capped at the limit)
  memory.swap.peak : 267 MiB (the overflow went to swap)
  memory.events oom_kill: 0
  ==> it SURVIVED. The pages that did not fit were written to swap.

### Now forbid swap for this cgroup: memory.swap.max = 0
blogdemo/swap/memory.swap.max: 0 
  exit status: 137   wall time: 0.73s
  memory.swap.peak: 0 MiB
  memory.events oom_kill: 1
  ==> identical workload, identical limit. Swap is the only difference
      between exit 0 and exit 137.

### And now memory.high actually works (compare scenario 07)
  allocated in 20s under memory.high=64Mi WITH swap: 200 MiB
  (scenario 07 managed 65 MiB in 20s on the same box with no swap)
    pgscan     86645
    pgsteal    37068
  memory.swap.current: 0 MiB
  ==> now reclaim has somewhere to put the pages, so throttling reclaims
      instead of just stalling.

### Kubernetes mapping (KEP-2400, NodeSwap)
  Swap on nodes was forbidden for years precisely because it makes
  limits.memory soft: a pod over its limit gets slow instead of killed,
  and the kubelet's accounting no longer matches RSS.
  NodeSwap re-introduces it deliberately: cgroup v2 only, with
  swapBehavior LimitedSwap giving Burstable pods a swap allowance
  proportional to their memory request, while Guaranteed and BestEffort
  pods get none. memory.swap.max is the file that implements it.

### Host restored
  SwapTotal now: 0 kB
  /swapfile-cgroupdemo exists: no
```

### 9. `pids.max` — the cheapest blast-radius control you have

19 forks succeed, 81 are refused with `EAGAIN`, the host never sees them. Note the
detail in the script: bash cannot detect a failed fork from `cmd &`, because
backgrounding always "succeeds" — you have to call `fork()` directly to see the
refusal. Without this limit a runaway container exhausts `kernel.pid_max` and
`fork()` starts failing for `sshd` too, which is how you lose the node.

```bash
#!/usr/bin/env bash
# 09-pids.sh - pids.max: the cheapest blast-radius control on a node.
# A fork bomb in one cgroup should not cost you the kernel's PID space.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

cg=$(cg_create pids)
trap 'cg_destroy "$cg"' EXIT

echo "### Defaults"
show "$cg/pids.max"
show "$cg/pids.current"
echo "  host-wide pid_max: $(cat /proc/sys/kernel/pid_max)"

echo
echo "### Cap the cgroup at 20 processes"
echo 20 > "$cg/pids.max"
show "$cg/pids.max"

echo
echo "### A process tries to fork 100 children inside a cgroup capped at 20"
# bash cannot report a failed fork from `cmd &` (backgrounding always
# "succeeds"), so fork() directly and catch EAGAIN.
cg_exec "$cg" python3 -c '
import os, sys, time
made = failed = 0
for i in range(100):
    try:
        if os.fork() == 0:
            time.sleep(30)
            os._exit(0)
        made += 1
    except OSError as e:
        failed += 1
        if failed == 1:
            first = f"{type(e).__name__}: {e}"
print(f"    fork() succeeded: {made}")
print(f"    fork() refused  : {failed}")
print(f"    first refusal   : {first}")
' 2>&1 | sed 's/^/  /'

echo "  pids.current: $(cat $cg/pids.current)   pids.peak: $(cat $cg/pids.peak)"
echo "  pids.events:"
sed 's/^/    /' "$cg/pids.events"
echo "  ==> 'max' counts fork() attempts the kernel refused with EAGAIN."

echo
echo "### The host is unharmed - it never saw those 80 processes"
echo "  total processes on the host: $(ps -e --no-headers | wc -l)"
cg_kill "$cg"

echo
echo "### What a fork bomb looks like WITHOUT the limit (still capped by us at 200)"
echo 200 > "$cg/pids.max"
cg_exec "$cg" bash -c 'for i in $(seq 500); do sleep 30 & done 2>/dev/null; true'
echo "  pids.peak reached: $(cat $cg/pids.peak) (clamped by pids.max=200, not by the host)"
cg_kill "$cg"

echo
echo "### Kubernetes mapping"
echo "  kubelet --pod-max-pids / podPidsLimit -> pids.max on the pod cgroup."
echo "  SupportNodePidsLimit reserves PIDs for the node itself the same way."
echo "  Without it, one runaway container exhausts kernel.pid_max and you cannot"
echo "  even ssh in to fix it - fork() fails for sshd too. This is a node-level"
echo "  availability control, not a per-app nicety."
```

```text
### Defaults
blogdemo/pids/pids.max:      max 
blogdemo/pids/pids.current:  0 
  host-wide pid_max: 4194304

### Cap the cgroup at 20 processes
blogdemo/pids/pids.max:      20 

### A process tries to fork 100 children inside a cgroup capped at 20
      fork() succeeded: 19
      fork() refused  : 81
      first refusal   : BlockingIOError: [Errno 11] Resource temporarily unavailable
  pids.current: 0   pids.peak: 20
  pids.events:
    max 81
  ==> 'max' counts fork() attempts the kernel refused with EAGAIN.

### The host is unharmed - it never saw those 80 processes
  total processes on the host: 160

### What a fork bomb looks like WITHOUT the limit (still capped by us at 200)
  pids.peak reached: 200 (clamped by pids.max=200, not by the host)

### Kubernetes mapping
  kubelet --pod-max-pids / podPidsLimit -> pids.max on the pod cgroup.
  SupportNodePidsLimit reserves PIDs for the node itself the same way.
  Without it, one runaway container exhausts kernel.pid_max and you cannot
  even ssh in to fix it - fork() fails for sshd too. This is a node-level
  availability control, not a per-app nicety.
```

### 10. `io.max` — the controller Kubernetes never gave you

Writes pinned from 100.9 MiB/s to exactly 4.0 MiB/s. It works, it is per-cgroup,
and **there is no PodSpec field for it**. `ephemeral-storage` is a capacity quota,
not a rate limit. One pod doing a restore can starve every other pod on the node
for disk and nothing in the Kubernetes API will stop it. The OCI spec has
`blockIO`, containerd can set it, Kubernetes never surfaced it.

```bash
#!/usr/bin/env bash
# 10-io-max.sh - io.max: per-cgroup block IO limits.
# The controller Kubernetes never exposed. Worth knowing precisely because
# there is no PodSpec field for it: disk is the unprotected resource.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

FILE=/var/tmp/cgroup-demo-io.bin

# Which block device backs /var/tmp? io limits are keyed by MAJ:MIN of the
# whole disk, not the partition.
PART=$(df --output=source /var/tmp | tail -1)
DISK=$(lsblk -no PKNAME "$PART" 2>/dev/null | head -1)
[[ -z $DISK ]] && DISK=$(basename "$PART")
DEVNO=$(lsblk -no MAJ:MIN "/dev/$DISK" | head -1 | tr -d ' ')
echo "### Target device"
echo "  /var/tmp is on $PART, whose disk is /dev/$DISK = $DEVNO"
echo "  rotational: $(cat /sys/block/$DISK/queue/rotational)  scheduler: $(cat /sys/block/$DISK/queue/scheduler)"

echo
echo "### The io controller is available but not delegated"
echo "  root cgroup.controllers:     $(cat $CG_ROOT/cgroup.controllers)"
echo "  root cgroup.subtree_control: $(cat $CG_ROOT/cgroup.subtree_control)"

demo_root_init
io_added=no
if ! grep -qw io "$CG_ROOT/cgroup.subtree_control"; then
    record_change "enabled 'io' in root cgroup.subtree_control" \
                  "echo -io > $CG_ROOT/cgroup.subtree_control"
    echo "+io" > "$CG_ROOT/cgroup.subtree_control"
    io_added=yes
fi
cg_enable "$DEMO_ROOT" io
cg=$(cg_create io)
restore() {
    cg_destroy "$cg"
    rm -f "$FILE"
    if [[ $io_added == yes ]]; then
        echo "-io" > "$DEMO_ROOT/cgroup.subtree_control" 2>/dev/null
        cg_destroy "$DEMO_ROOT" 2>/dev/null
        echo "-io" > "$CG_ROOT/cgroup.subtree_control"
        mark_reverted "enabled 'io' in root cgroup.subtree_control"
        echo "  restored root cgroup.subtree_control: $(cat $CG_ROOT/cgroup.subtree_control)"
    fi
}
trap restore EXIT
echo "  after enabling: $(cat $CG_ROOT/cgroup.subtree_control)"
show "$cg/io.max"

timed_write() {  # timed_write <MiB> <label>
    local mb=$1 label=$2 t0 t1
    t0=$(date +%s.%N)
    cg_exec "$cg" dd if=/dev/zero of="$FILE" bs=1M count="$mb" oflag=direct status=none
    t1=$(date +%s.%N)
    python3 -c "
t = $t1 - $t0
print(f'  {\"$label\":<22} {$mb} MiB in {t:5.2f}s = {$mb/t:6.1f} MiB/s')"
}

echo
echo "### Unthrottled direct write"
timed_write 64 "no limit:"

echo
echo "### Now cap the cgroup at 4 MiB/s of writes"
echo "$DEVNO wbps=4194304" > "$cg/io.max"
show "$cg/io.max"
timed_write 16 "wbps=4MiB/s:"

echo
echo "### Cap reads at 4 MiB/s too, then read the file back"
echo "$DEVNO rbps=4194304 wbps=4194304" > "$cg/io.max"
show "$cg/io.max"
t0=$(date +%s.%N)
cg_exec "$cg" dd if="$FILE" of=/dev/null bs=1M count=16 iflag=direct status=none
t1=$(date +%s.%N)
python3 -c "
t = $t1 - $t0
print(f'  {\"rbps=4MiB/s:\":<22} 16 MiB in {t:5.2f}s = {16/t:6.1f} MiB/s')"

echo
echo "### Per-cgroup IO accounting"
echo "  this cgroup's io.stat:"
sed 's/^/    /' "$cg/io.stat"
echo "  io.pressure: $(head -1 $cg/io.pressure)"

echo
echo "### The buffered-write caveat"
echo "  io.max throttles the process submitting the IO. A BUFFERED write returns"
echo "  as soon as it hits page cache; the actual disk write happens later in"
echo "  kernel writeback threads. cgroup writeback attributes those back to the"
echo "  owning cgroup (CONFIG_CGROUP_WRITEBACK), but the throttling a workload"
echo "  feels is far less direct than with O_DIRECT above."

echo
echo "### Kubernetes mapping - the gap"
echo "  There is NO PodSpec field for disk bandwidth or IOPS. ephemeral-storage"
echo "  is a capacity quota enforced by the kubelet, not a rate limit."
echo "  Consequences on a shared node:"
echo "   - One pod doing a large restore or log flush can starve every other pod"
echo "     on the node for disk, and nothing in the API will stop it."
echo "   - The OCI runtime spec DOES have blockIO limits, and containerd can set"
echo "     them, but Kubernetes never surfaced them."
echo "   - Options are: local PVs on separate devices, io.max via a node agent,"
echo "     or io.cost/io.weight for proportional sharing. All of them are outside"
echo "     the PodSpec, which is why disk noisy-neighbours remain a real problem."
```

```text
### Target device
  /var/tmp is on /dev/sda2, whose disk is /dev/sda = 8:0
  rotational: 1  scheduler: none [mq-deadline] 

### The io controller is available but not delegated
  root cgroup.controllers:     cpuset cpu io memory hugetlb pids rdma misc
  root cgroup.subtree_control: cpu memory pids
  after enabling: cpu io memory pids
blogdemo/io/io.max:          

### Unthrottled direct write
  no limit:              64 MiB in  0.63s =  100.9 MiB/s

### Now cap the cgroup at 4 MiB/s of writes
blogdemo/io/io.max:          8:0 rbps=max wbps=4194304 riops=max wiops=max 
  wbps=4MiB/s:           16 MiB in  4.00s =    4.0 MiB/s

### Cap reads at 4 MiB/s too, then read the file back
blogdemo/io/io.max:          8:0 rbps=4194304 wbps=4194304 riops=max wiops=max 
  rbps=4MiB/s:           16 MiB in  4.07s =    3.9 MiB/s

### Per-cgroup IO accounting
  this cgroup's io.stat:
    8:0 rbytes=16777216 wbytes=83886080 rios=32 wios=160 dbytes=0 dios=0
  io.pressure: some avg10=41.01 avg60=8.99 avg300=1.94 total=6721507

### The buffered-write caveat
  io.max throttles the process submitting the IO. A BUFFERED write returns
  as soon as it hits page cache; the actual disk write happens later in
  kernel writeback threads. cgroup writeback attributes those back to the
  owning cgroup (CONFIG_CGROUP_WRITEBACK), but the throttling a workload
  feels is far less direct than with O_DIRECT above.

### Kubernetes mapping - the gap
  There is NO PodSpec field for disk bandwidth or IOPS. ephemeral-storage
  is a capacity quota enforced by the kubelet, not a rate limit.
  Consequences on a shared node:
   - One pod doing a large restore or log flush can starve every other pod
     on the node for disk, and nothing in the API will stop it.
   - The OCI runtime spec DOES have blockIO limits, and containerd can set
     them, but Kubernetes never surfaced them.
   - Options are: local PVs on separate devices, io.max via a node agent,
     or io.cost/io.weight for proportional sharing. All of them are outside
     the PodSpec, which is why disk noisy-neighbours remain a real problem.
  restored root cgroup.subtree_control: cpu memory pids
```

### 11. `cgroup.freeze` and `cgroup.kill` — whole-tree operations

CPU time goes 7055 ms → 0 ms frozen → 6084 ms thawed, and one write to
`cgroup.kill` removes an entire process tree with no PID walking and no race
against `fork()`. Freeze is what container pause and CRIU checkpoint/restore are
built on; kill is how a runtime tears a container down atomically.

```bash
#!/usr/bin/env bash
# 11-freeze-kill.sh - cgroup.freeze and cgroup.kill: whole-tree operations.
# Two things that are painful with signals and trivial with cgroups: stop an
# entire process tree consistently, and kill one with no escapees.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

cg=$(cg_create freezer)
trap 'cg_destroy "$cg"' EXIT

echo "### Start a small tree: a parent and 3 children, all counting"
cg_exec "$cg" bash -c 'for i in 1 2 3; do (while :; do :; done) & done; disown -a' 2>/dev/null
sleep 1
echo "  processes in cgroup: $(wc -l < $cg/cgroup.procs)"
show "$cg/cgroup.freeze"
echo "  cgroup.events: $(tr '\n' ' ' < $cg/cgroup.events)"

usage() { awk '/^usage_usec/{print $2}' "$cg/cpu.stat"; }

echo
echo "### Running: CPU time advances"
u0=$(usage); sleep 2; u1=$(usage)
echo "  cpu usage delta over 2s: $(( (u1 - u0) / 1000 )) ms"

echo
echo "### echo 1 > cgroup.freeze"
echo 1 > "$cg/cgroup.freeze"
sleep 0.5
echo "  cgroup.events: $(tr '\n' ' ' < $cg/cgroup.events)"
echo "  process states: $(for p in $(cat $cg/cgroup.procs); do ps -o stat= -p $p; done | tr -d ' ' | sort | uniq -c | tr '\n' ' ')"
u0=$(usage); sleep 2; u1=$(usage)
echo "  cpu usage delta over 2s while frozen: $(( (u1 - u0) / 1000 )) ms"
echo "  ==> frozen tasks are not stopped by SIGSTOP; they are parked by the"
echo "      freezer and are not signallable-away by the workload itself."

echo
echo "### echo 0 > cgroup.freeze (thaw)"
echo 0 > "$cg/cgroup.freeze"
sleep 0.5
echo "  cgroup.events: $(tr '\n' ' ' < $cg/cgroup.events)"
u0=$(usage); sleep 2; u1=$(usage)
echo "  cpu usage delta over 2s after thaw: $(( (u1 - u0) / 1000 )) ms"

echo
echo "### echo 1 > cgroup.kill - one write, whole tree, no escapees"
echo "  before: $(wc -l < $cg/cgroup.procs) processes"
echo 1 > "$cg/cgroup.kill"
cg_wait_empty "$cg"
echo "  after : $(wc -l < $cg/cgroup.procs) processes"
echo "  ==> no PID walking, no race with fork(). A process cannot outrun this"
echo "      by forking faster than you can signal."

echo
echo "### Kubernetes mapping"
echo "  cgroup.freeze underpins container pause and CRIU checkpoint/restore"
echo "  (the ContainerCheckpoint API) - you need a quiesced tree to snapshot."
echo "  cgroup.kill is how a runtime tears a container down atomically instead"
echo "  of SIGTERM/sleep/SIGKILL over a PID list that keeps changing."
```

```text
### Start a small tree: a parent and 3 children, all counting
  processes in cgroup: 3
blogdemo/freezer/cgroup.freeze: 0 
  cgroup.events: populated 1 frozen 0 

### Running: CPU time advances
  cpu usage delta over 2s: 7055 ms

### echo 1 > cgroup.freeze
  cgroup.events: populated 1 frozen 1 
  process states:       3 S 
  cpu usage delta over 2s while frozen: 0 ms
  ==> frozen tasks are not stopped by SIGSTOP; they are parked by the
      freezer and are not signallable-away by the workload itself.

### echo 0 > cgroup.freeze (thaw)
  cgroup.events: populated 1 frozen 0 
  cpu usage delta over 2s after thaw: 6084 ms

### echo 1 > cgroup.kill - one write, whole tree, no escapees
  before: 3 processes
  after : 0 processes
  ==> no PID walking, no race with fork(). A process cannot outrun this
      by forking faster than you can signal.

### Kubernetes mapping
  cgroup.freeze underpins container pause and CRIU checkpoint/restore
  (the ContainerCheckpoint API) - you need a quiesced tree to snapshot.
  cgroup.kill is how a runtime tears a container down atomically instead
  of SIGTERM/sleep/SIGKILL over a PID list that keeps changing.
```

### 12. PSI — the metric that tells you about *lost* time

Utilisation tells you a resource is busy. Pressure tells you work was *delayed*.
A cgroup at 100% CPU with 0% pressure is fine — it has everything it wants. The
same cgroup at 60% utilisation and 40% pressure is starving. Below, a throttled
cgroup shows `full avg10=70.65`: for 70% of the last 10 seconds, *every* task in
it was stalled.

The memory half of this scenario failed the first time I ran it, and the reason is
worth more than the demo: **page cache is charged to the cgroup that first faults
a page in.** I had created the file outside the cgroup, so the pages were already
resident and charged elsewhere; re-reading them charged nobody and produced no
pressure at all. Dropping the cache first fixed it. The same effect is why a
shared base-image layer is billed to whichever pod happened to touch it first.

```bash
#!/usr/bin/env bash
# 12-psi.sh - Pressure Stall Information: how much time was LOST waiting for
# a resource. Utilisation says a node is busy; PSI says work is being delayed.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

cg=$(cg_create psi)
trap 'cg_destroy "$cg"; rm -f /var/tmp/cgroup-demo-psi.bin' EXIT

echo "### PSI files exist per-cgroup as well as globally"
echo "  global /proc/pressure/cpu: $(head -1 /proc/pressure/cpu)"
echo "  cgroup files: $(ls $cg | grep pressure | tr '\n' ' ')"
echo
echo "  'some' = at least one task was stalled; 'full' = every task was stalled."

echo
echo "### Baseline (idle cgroup)"
for f in cpu.pressure memory.pressure io.pressure; do
    printf '  %-16s %s\n' "$f" "$(head -1 $cg/$f)"
done

echo
echo "### CPU pressure: 8 threads under a 0.2 CPU quota for 15s"
echo "20000 100000" > "$cg/cpu.max"
cg_run "$cg" bash -c 'for i in $(seq 8); do while :; do :; done & done; wait'
sleep 15
echo "  cpu.pressure : $(head -1 $cg/cpu.pressure)"
echo "  full         : $(sed -n 2p $cg/cpu.pressure)"
cg_kill "$cg"
echo max > "$cg/cpu.max"

echo
echo "### Memory pressure: thrash a 64 MiB cgroup with a 512 MiB file"
echo $((64 * 1024 * 1024)) > "$cg/memory.max"
dd if=/dev/urandom of=/var/tmp/cgroup-demo-psi.bin bs=1M count=512 status=none
# Page cache is charged to the cgroup that FIRST faults a page in. This file was
# just written from outside the demo cgroup, so its pages are already resident
# and already charged elsewhere - re-reading them would charge nobody and
# generate no pressure at all. Drop the cache so the demo cgroup pays for them.
sync; echo 3 > /proc/sys/vm/drop_caches
cg_run "$cg" bash -c 'for i in 1 2 3 4; do dd if=/var/tmp/cgroup-demo-psi.bin of=/dev/null bs=1M status=none; done'
sleep 15
echo "  memory.pressure: $(head -1 $cg/memory.pressure)"
echo "  full           : $(sed -n 2p $cg/memory.pressure)"
echo "  io.pressure    : $(head -1 $cg/io.pressure)"
echo "  memory.events max (reclaim forced by the limit): $(awk '/^max /{print $2}' $cg/memory.events)"
echo "  memory.current: $(( $(cat $cg/memory.current) / 1048576 )) MiB against a 64 MiB limit"
echo "  ==> a 512 MiB file cannot fit in a 64 MiB cgroup, so every pass re-reads"
echo "      from disk. This is cache thrashing: no OOM kill, no CPU saturation,"
echo "      just latency. Utilisation dashboards show almost nothing here."
cg_kill "$cg"

echo
echo "### Reading the numbers"
echo "  avg10/avg60/avg300 are percentages of wall time stalled in that window."
echo "  total= is a microsecond counter - the one to rate() in Prometheus."
echo "  A cgroup at 100% CPU utilisation with 0% pressure is FINE (it has all the"
echo "  CPU it wants). The same cgroup at 60% utilisation and 40% pressure is"
echo "  starved. Utilisation cannot tell those apart; PSI can."

echo
echo "### Kubernetes mapping"
echo "  KEP-4205 surfaces PSI to the kubelet and (phase 2) sets node conditions"
echo "  and taints from it. Today's kubelet still evicts on working-set bytes,"
echo "  which is why a node can be thrashing badly while every eviction"
echo "  threshold still reads green."
```

```text
### PSI files exist per-cgroup as well as globally
  global /proc/pressure/cpu: some avg10=2.74 avg60=12.12 avg300=9.15 total=1103505269
  cgroup files: cgroup.pressure cpu.pressure io.pressure memory.pressure 

  'some' = at least one task was stalled; 'full' = every task was stalled.

### Baseline (idle cgroup)
  cpu.pressure     some avg10=0.00 avg60=0.00 avg300=0.00 total=0
  memory.pressure  some avg10=0.00 avg60=0.00 avg300=0.00 total=0
  io.pressure      some avg10=0.00 avg60=0.00 avg300=0.00 total=0

### CPU pressure: 8 threads under a 0.2 CPU quota for 15s
  cpu.pressure : some avg10=73.29 avg60=19.96 avg300=4.49 total=14534382
  full         : full avg10=70.65 avg60=18.95 avg300=4.25 total=13702596

### Memory pressure: thrash a 64 MiB cgroup with a 512 MiB file
  memory.pressure: some avg10=3.05 avg60=0.88 avg300=0.20 total=690128
  full           : full avg10=3.05 avg60=0.88 avg300=0.20 total=690128
  io.pressure    : some avg10=5.31 avg60=1.58 avg300=0.36 total=1170746
  memory.events max (reclaim forced by the limit): 7948
  memory.current: 62 MiB against a 64 MiB limit
  ==> a 512 MiB file cannot fit in a 64 MiB cgroup, so every pass re-reads
      from disk. This is cache thrashing: no OOM kill, no CPU saturation,
      just latency. Utilisation dashboards show almost nothing here.

### Reading the numbers
  avg10/avg60/avg300 are percentages of wall time stalled in that window.
  total= is a microsecond counter - the one to rate() in Prometheus.
  A cgroup at 100% CPU utilisation with 0% pressure is FINE (it has all the
  CPU it wants). The same cgroup at 60% utilisation and 40% pressure is
  starved. Utilisation cannot tell those apart; PSI can.

### Kubernetes mapping
  KEP-4205 surfaces PSI to the kubelet and (phase 2) sets node conditions
  and taints from it. Today's kubelet still evicts on working-set bytes,
  which is why a node can be thrashing badly while every eviction
  threshold still reads green.
```

### 13. `hugetlb` — a second, separate memory pool

32 MiB of hugepages inside the limit succeeds; 128 MiB dies with SIGBUS (135 =
128 + 7) when the fault cannot be charged. The key operational fact: hugepages are
**not** counted against `limits.memory`. Separate pool, separate limit, pinned
forever — never reclaimed, never swapped. They must also be pre-allocated on the
node before the kubelet can hand them out.

```bash
#!/usr/bin/env bash
# 13-hugetlb.sh - the hugetlb controller: a SEPARATE accounting pool.
# HOST CHANGES: pre-allocates hugepages and delegates the hugetlb controller.
# Both are recorded and undone at the end.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

PAGES=256                                    # 256 x 2 MiB = 512 MiB
orig_pages=$(cat /proc/sys/vm/nr_hugepages)

echo "### Hugepage state before"
grep -E 'HugePages_Total|HugePages_Free|Hugepagesize' /proc/meminfo | sed 's/^/  /'

echo
echo "### Pre-allocate $PAGES x 2 MiB hugepages"
record_change "vm.nr_hugepages $orig_pages -> $PAGES" \
              "sysctl -w vm.nr_hugepages=$orig_pages"
sysctl -w vm.nr_hugepages=$PAGES | sed 's/^/  /'
grep -E 'HugePages_Total|HugePages_Free' /proc/meminfo | sed 's/^/  /'

demo_root_init
huge_added=no
if ! grep -qw hugetlb "$CG_ROOT/cgroup.subtree_control"; then
    record_change "enabled 'hugetlb' in root cgroup.subtree_control" \
                  "echo -hugetlb > $CG_ROOT/cgroup.subtree_control"
    echo "+hugetlb" > "$CG_ROOT/cgroup.subtree_control"
    huge_added=yes
fi
cg_enable "$DEMO_ROOT" hugetlb
cg=$(cg_create hugetlb)
restore() {
    cg_destroy "$cg"
    if [[ $huge_added == yes ]]; then
        echo "-hugetlb" > "$DEMO_ROOT/cgroup.subtree_control" 2>/dev/null
        cg_destroy "$DEMO_ROOT" 2>/dev/null
        echo "-hugetlb" > "$CG_ROOT/cgroup.subtree_control"
        mark_reverted "enabled 'hugetlb' in root cgroup.subtree_control"
    fi
    sysctl -w vm.nr_hugepages=$orig_pages > /dev/null
    mark_reverted "vm.nr_hugepages $orig_pages -> $PAGES"
    echo
    echo "### Host restored"
    echo "  vm.nr_hugepages: $(cat /proc/sys/vm/nr_hugepages)"
    echo "  root cgroup.subtree_control: $(cat $CG_ROOT/cgroup.subtree_control)"
}
trap restore EXIT

echo
echo "### hugetlb files appear in the cgroup"
ls "$cg" | grep hugetlb | sed 's/^/  /'

echo
echo "### Limit this cgroup to 64 MiB of 2 MiB hugepages (32 pages)"
echo $((64 * 1024 * 1024)) > "$cg/hugetlb.2MB.max"
show "$cg/hugetlb.2MB.max"
show "$cg/hugetlb.2MB.current"

# MAP_HUGETLB allocator: the charge lands when the page is faulted in, so an
# over-limit mapping dies with SIGBUS rather than a failed mmap.
HUGE_ALLOC='
import ctypes, sys
HUGE_MB = int(sys.argv[1])
size = HUGE_MB * 1024 * 1024
libc = ctypes.CDLL("libc.so.6", use_errno=True)
libc.mmap.restype = ctypes.c_void_p
libc.mmap.argtypes = [ctypes.c_void_p, ctypes.c_size_t, ctypes.c_int,
                      ctypes.c_int, ctypes.c_int, ctypes.c_long]
PROT_RW = 0x1 | 0x2
MAP_PRIVATE, MAP_ANONYMOUS, MAP_HUGETLB = 0x02, 0x20, 0x40000
addr = libc.mmap(None, size, PROT_RW, MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0)
if addr == ctypes.c_void_p(-1).value or addr is None:
    print(f"    mmap failed: errno {ctypes.get_errno()}"); sys.exit(2)
print(f"    mmap of {HUGE_MB} MiB succeeded, faulting pages in...")
buf = (ctypes.c_char * size).from_address(addr)
for off in range(0, size, 2 * 1024 * 1024):
    buf[off] = b"x"
print(f"    faulted in {HUGE_MB} MiB of hugepages")
'

echo
echo "### Allocate 32 MiB of hugepages (inside the limit)"
set +e
cg_exec "$cg" python3 -c "$HUGE_ALLOC" 32 2>&1 | sed 's/^/  /'
echo "  exit status: ${PIPESTATUS[0]}"
set -e
echo "  hugetlb.2MB.current: $(cat $cg/hugetlb.2MB.current) bytes"
echo "  (back to 0 because the process exited and its hugepages were freed;"
echo "   HugePages_Free confirms the pool is whole again: $(awk '/HugePages_Free/{print $2}' /proc/meminfo))"

echo
echo "### Allocate 128 MiB of hugepages (twice the cgroup's limit)"
set +e
cg_exec "$cg" python3 -c "$HUGE_ALLOC" 128 2>&1 | grep -v 'Bus error' | sed 's/^/  /'
rc=${PIPESTATUS[0]}
set -e
echo "  exit status: $rc  (135 = 128 + SIGBUS(7): the fault could not be charged)"
echo "  hugetlb.2MB.events:"
sed 's/^/    /' "$cg/hugetlb.2MB.events" 2>/dev/null
echo "  memory.current for this cgroup: $(cat $cg/memory.current) bytes"
echo "  ==> note memory.current barely moved: hugetlb is accounted SEPARATELY"
echo "      from memory.max. A hugepage-using pod is billed in two places."

echo
echo "### Kubernetes mapping"
echo "  hugepages-2Mi is a schedulable resource backed by hugetlb.2MB.max."
echo "  Practical notes for a platform engineer:"
echo "   - Hugepages must be pre-allocated on the node (boot cmdline or sysctl);"
echo "     the kubelet only divides up what already exists."
echo "   - They are NOT counted against limits.memory - separate pool, separate"
echo "     limit, and they are pinned (never reclaimed, never swapped)."
echo "   - requests must equal limits for hugepages, and a pod that asks for"
echo "     them stays Pending forever if the node has none free."
```

```text
### Hugepage state before
  HugePages_Total:       0
  HugePages_Free:        0
  Hugepagesize:       2048 kB

### Pre-allocate 256 x 2 MiB hugepages
  vm.nr_hugepages = 256
  HugePages_Total:     256
  HugePages_Free:      256

### hugetlb files appear in the cgroup
  hugetlb.2MB.current
  hugetlb.2MB.events
  hugetlb.2MB.events.local
  hugetlb.2MB.max
  hugetlb.2MB.numa_stat
  hugetlb.2MB.rsvd.current
  hugetlb.2MB.rsvd.max

### Limit this cgroup to 64 MiB of 2 MiB hugepages (32 pages)
blogdemo/hugetlb/hugetlb.2MB.max: 67108864 
blogdemo/hugetlb/hugetlb.2MB.current: 0 

### Allocate 32 MiB of hugepages (inside the limit)
      mmap of 32 MiB succeeded, faulting pages in...
      faulted in 32 MiB of hugepages
  exit status: 0
  hugetlb.2MB.current: 0 bytes
  (back to 0 because the process exited and its hugepages were freed;
   HugePages_Free confirms the pool is whole again: 256)

### Allocate 128 MiB of hugepages (twice the cgroup's limit)
  exit status: 135  (135 = 128 + SIGBUS(7): the fault could not be charged)
  hugetlb.2MB.events:
    max 1
  memory.current for this cgroup: 4096 bytes
  ==> note memory.current barely moved: hugetlb is accounted SEPARATELY
      from memory.max. A hugepage-using pod is billed in two places.

### Kubernetes mapping
  hugepages-2Mi is a schedulable resource backed by hugetlb.2MB.max.
  Practical notes for a platform engineer:
   - Hugepages must be pre-allocated on the node (boot cmdline or sysctl);
     the kubelet only divides up what already exists.
   - They are NOT counted against limits.memory - separate pool, separate
     limit, and they are pinned (never reclaimed, never swapped).
   - requests must equal limits for hugepages, and a pod that asks for
     them stays Pending forever if the node has none free.

### Host restored
  vm.nr_hugepages: 0
  root cgroup.subtree_control: cpu memory pids
```

### 14. Namespaces, delegation, and the v1→v2 map

Three short things that make everything above legible: why `/proc/self/cgroup`
reads `0::/` inside a pod; how `systemd-run -p MemoryMax=64M` writes the exact
same files (and why a `cgroupDriver` mismatch corrupts accounting); and the
translation table from v1 names, including the dead ends — **v2 has no `devices`
controller at all**, it is an eBPF program, and you can see systemd's own
`cgroup_device` programs attached in the output.

```bash
#!/usr/bin/env bash
# 14-delegation.sh - the three things that make the rest of this legible:
# cgroup namespaces, systemd delegation, and the v1 -> v2 map.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

echo "############ 1. cgroup namespaces: why a container thinks it is the root"
echo
echo "### This shell's cgroup, seen from the host namespace"
echo "  /proc/self/cgroup: $(cat /proc/self/cgroup)"
echo "  cgroup namespace : $(readlink /proc/self/ns/cgroup)"

echo
echo "### The same process after unshare --cgroup"
unshare --cgroup bash -c '
    echo "  /proc/self/cgroup: $(cat /proc/self/cgroup)"
    echo "  cgroup namespace : $(readlink /proc/self/ns/cgroup)"'
echo "  ==> 0::/ - the cgroup namespace re-roots the view at the current cgroup."
echo "      This is exactly what you see inside a pod, and why a process cannot"
echo "      tell from /proc/self/cgroup where it sits in the node's hierarchy."
echo "      It is also why in-container tools that read /sys/fs/cgroup see their"
echo "      OWN limits as if they were the machine's."

echo
echo "############ 2. systemd delegation: the same files, managed"
echo
echo "### systemd is the cgroup manager on this host"
echo "  systemd version: $(systemctl --version | head -1)"
echo "  unified hierarchy: $(systemctl --version | grep -o 'default-hierarchy=[a-z]*')"
echo "  Delegate= on the user manager: $(systemctl show user@1000.service -p Delegate --value)"

echo
echo "### systemd-run --scope with resource properties writes the same cgroup files"
systemd-run --scope --quiet --unit=cgroup-blog-demo \
    -p MemoryMax=64M -p CPUQuota=20% -p TasksMax=25 \
    bash -c '
        cg=/sys/fs/cgroup$(awk -F: "{print \$3}" /proc/self/cgroup)
        echo "  scope cgroup: $(awk -F: "{print \$3}" /proc/self/cgroup)"
        echo "  memory.max : $(cat $cg/memory.max)      <- MemoryMax=64M"
        echo "  cpu.max    : $(cat $cg/cpu.max)   <- CPUQuota=20%"
        echo "  pids.max   : $(cat $cg/pids.max)             <- TasksMax=25"
    ' 2>&1 | grep -v 'Running scope'
echo "  ==> systemd unit properties are a front-end for the same kernel files."
echo "      This is why the kubelet's cgroupDriver must MATCH the runtime's:"
echo "      two managers writing the same tree will fight, and the symptom is"
echo "      limits that silently revert or accounting that reads zero."

echo
echo "### The node's slice layout"
systemd-cgls --no-pager -l 2>/dev/null | head -12 | sed 's/^/  /'

echo
echo "############ 3. The v1 -> v2 map, and the dead ends"
cat <<'TABLE'
  cgroup v1                     cgroup v2                  note
  ---------------------------   ------------------------   -----------------------------
  cpu.shares                    cpu.weight                 1..10000, not 2..262144
  cpu.cfs_quota_us/period_us    cpu.max                    one file: "quota period"
  cpuset.cpus / .mems           cpuset.cpus / .mems        must be enabled top-down
  memory.limit_in_bytes         memory.max                 plus memory.high (new)
  memory.soft_limit_in_bytes    memory.low / memory.min    actually enforceable now
  memory.usage_in_bytes         memory.current
  memory.stat                   memory.stat                different field names
  blkio.throttle.*              io.max                     plus io.cost / io.weight
  pids.max                      pids.max                   unchanged
  freezer.state                 cgroup.freeze              0/1 instead of FROZEN/THAWED
  (none)                        cgroup.kill                new in v2
  (none)                        *.pressure (PSI)           v2 only
TABLE

echo
echo "### Controllers that are gone or empty in v2"
echo "  devices : no controller at all in v2 - device access is an eBPF program"
echo "            attached to the cgroup. Attached programs on this host:"
# NOTE: head closing the pipe gives bpftool SIGPIPE, which with `set -o
# pipefail` looks like a failure - so capture first, truncate after.
bpf_tree=$(bpftool cgroup tree 2>/dev/null)
printf '%s\n' "$bpf_tree" | head -9 | sed 's/^/              /' 
echo "  net_cls / net_prio : removed. Classify with eBPF or tc instead."
echo "  rdma    : present but no RDMA devices here."
echo "  misc    : present, capacity empty: '$(cat $CG_ROOT/misc.capacity)'"
echo "            (it exists for AMD SEV / Intel TDX guest slots, not GPUs)"
echo "  RT scheduling : CONFIG_RT_GROUP_SCHED is not set on this kernel, so"
echo "                  there are no cpu.rt_* knobs at all."

echo
echo "############ Kubernetes mapping"
echo "  cgroup v2 went GA in Kubernetes 1.25 (KEP-2254); v1 is in maintenance"
echo "  mode (KEP-4569). Features that only exist on v2 and are worth the"
echo "  upgrade on their own: MemoryQoS (memory.min/high), PSI, cgroup.kill,"
echo "  and swap support. If a node is still on v1 you cannot have any of them."
```

```text
############ 1. cgroup namespaces: why a container thinks it is the root

### This shell's cgroup, seen from the host namespace
  /proc/self/cgroup: 0::/user.slice/user-1000.slice/session-1.scope
  cgroup namespace : cgroup:[4026531835]

### The same process after unshare --cgroup
  /proc/self/cgroup: 0::/
  cgroup namespace : cgroup:[4026532218]
  ==> 0::/ - the cgroup namespace re-roots the view at the current cgroup.
      This is exactly what you see inside a pod, and why a process cannot
      tell from /proc/self/cgroup where it sits in the node's hierarchy.
      It is also why in-container tools that read /sys/fs/cgroup see their
      OWN limits as if they were the machine's.

############ 2. systemd delegation: the same files, managed

### systemd is the cgroup manager on this host
  systemd version: systemd 255 (255.4-1ubuntu8.17)
  unified hierarchy: default-hierarchy=unified
  Delegate= on the user manager: yes

### systemd-run --scope with resource properties writes the same cgroup files
  scope cgroup: /system.slice/cgroup-blog-demo.scope
  memory.max : 67108864      <- MemoryMax=64M
  cpu.max    : 20000 100000   <- CPUQuota=20%
  pids.max   : 25             <- TasksMax=25
  ==> systemd unit properties are a front-end for the same kernel files.
      This is why the kubelet's cgroupDriver must MATCH the runtime's:
      two managers writing the same tree will fight, and the symptom is
      limits that silently revert or accounting that reads zero.

### The node's slice layout
  CGroup /:
  -.slice
  ├─user.slice
  │ └─user-1000.slice
  │   ├─session-10.scope
  │   │ ├─17040 sshd: ubuntu [priv]
  │   │ ├─17124 sshd: ubuntu@pts/1
  │   │ └─17125 -bash
  │   ├─[0muser@1000.service …
  │   │ └─init.scope
  │   │   ├─897 /usr/lib/systemd/systemd --user
  │   │   └─898 (sd-pam)

############ 3. The v1 -> v2 map, and the dead ends
  cgroup v1                     cgroup v2                  note
  ---------------------------   ------------------------   -----------------------------
  cpu.shares                    cpu.weight                 1..10000, not 2..262144
  cpu.cfs_quota_us/period_us    cpu.max                    one file: "quota period"
  cpuset.cpus / .mems           cpuset.cpus / .mems        must be enabled top-down
  memory.limit_in_bytes         memory.max                 plus memory.high (new)
  memory.soft_limit_in_bytes    memory.low / memory.min    actually enforceable now
  memory.usage_in_bytes         memory.current
  memory.stat                   memory.stat                different field names
  blkio.throttle.*              io.max                     plus io.cost / io.weight
  pids.max                      pids.max                   unchanged
  freezer.state                 cgroup.freeze              0/1 instead of FROZEN/THAWED
  (none)                        cgroup.kill                new in v2
  (none)                        *.pressure (PSI)           v2 only

### Controllers that are gone or empty in v2
  devices : no controller at all in v2 - device access is an eBPF program
            attached to the cgroup. Attached programs on this host:
              CgroupPath
              ID       AttachType      AttachFlags     Name           
              /sys/fs/cgroup/system.slice/systemd-networkd.service
                  77       cgroup_device   multi           sd_devices                     
              /sys/fs/cgroup/system.slice/systemd-udevd.service
                  84       cgroup_inet_ingress multi           sd_fw_ingress                  
                  83       cgroup_inet_egress multi           sd_fw_egress                   
              /sys/fs/cgroup/system.slice/polkit.service
                  80       cgroup_device   multi           sd_devices                     
  net_cls / net_prio : removed. Classify with eBPF or tc instead.
  rdma    : present but no RDMA devices here.
  misc    : present, capacity empty: ''
            (it exists for AMD SEV / Intel TDX guest slots, not GPUs)
  RT scheduling : CONFIG_RT_GROUP_SCHED is not set on this kernel, so
                  there are no cpu.rt_* knobs at all.

############ Kubernetes mapping
  cgroup v2 went GA in Kubernetes 1.25 (KEP-2254); v1 is in maintenance
  mode (KEP-4569). Features that only exist on v2 and are worth the
  upgrade on their own: MemoryQoS (memory.min/high), PSI, cgroup.kill,
  and swap support. If a node is still on v1 you cannot have any of them.
```

---

## Part 2 — The same knobs, as the kubelet writes them

Everything above was done by hand. Now a real single-node cluster (k3s v1.36.4,
containerd, systemd cgroup driver) does it, and we check its arithmetic.

Three pods, identical except for their `resources:` block:

```yaml
# Three pods, one per QoS class. The only difference is the resources block -
# and that difference decides where the kubelet puts them in the cgroup tree.
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
  labels: {demo: cgroups}
spec:
  terminationGracePeriodSeconds: 0
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "while true; do :; done"]
    resources:                      # requests == limits => Guaranteed
      requests: {cpu: "250m", memory: "128Mi"}
      limits:   {cpu: "250m", memory: "128Mi"}
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
  labels: {demo: cgroups}
spec:
  terminationGracePeriodSeconds: 0
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "while true; do :; done"]
    resources:                      # requests < limits => Burstable
      requests: {cpu: "500m", memory: "64Mi"}
      limits:   {cpu: "1",    memory: "256Mi"}
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
  labels: {demo: cgroups}
spec:
  terminationGracePeriodSeconds: 0
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "while true; do :; done"]
                                    # no resources at all => BestEffort
```

And one pod built to die, by writing 200 MiB into a memory-backed `emptyDir`
inside a 64 MiB limit:

```yaml
# A pod that will be OOMKilled: it writes 200 MiB into a tmpfs (memory-backed
# emptyDir), which is charged to the container's memory cgroup and - with no
# swap on the node - cannot be reclaimed.
apiVersion: v1
kind: Pod
metadata:
  name: oom-victim
  labels: {demo: cgroups}
spec:
  terminationGracePeriodSeconds: 0
  restartPolicy: Never
  volumes:
  - name: ram
    emptyDir: {medium: Memory, sizeLimit: 512Mi}
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "dd if=/dev/zero of=/ram/blob bs=1M count=200"]
    volumeMounts: [{name: ram, mountPath: /ram}]
    resources:
      requests: {memory: "64Mi"}
      limits:   {memory: "64Mi"}
```

The walk script resolves each pod's UID to its cgroup path, reads the files, and
recomputes what the kubelet *should* have written:

```bash
#!/usr/bin/env bash
# 20-k8s-walk.sh - Part 2: the same knobs, as the kubelet writes them.
# Walks the real kubepods.slice tree a live k3s node built, and checks the
# arithmetic from PodSpec to kernel file.
set -uo pipefail
source "$(dirname "$0")/lib.sh"
K=${K:-"k3s kubectl"}

echo "### The node"
$K get nodes -o wide 2>/dev/null | sed 's/^/  /'
# k3s does not pass --cgroup-driver explicitly; the proof is in the naming.
# systemd driver => *.slice / *.scope. cgroupfs driver => plain directories.
echo "  containerd SystemdCgroup: $(grep -r SystemdCgroup /var/lib/rancher/k3s/agent/etc/containerd/config.toml 2>/dev/null | tr -d ' ' | head -1)"
echo "  cgroup driver (inferred): $(ls /sys/fs/cgroup/ | grep -q 'kubepods.slice' && echo 'systemd (kubepods.slice, not kubepods/)' || echo cgroupfs)"
echo "  containerd runtime   : $($K get nodes -o jsonpath='{.items[0].status.nodeInfo.containerRuntimeVersion}')"

echo
echo "### The top of the kubelet's tree"
ls /sys/fs/cgroup/ | grep -E 'kubepods|slice' | sed 's/^/  /'
echo
echo "  kubepods.slice contents:"
ls /sys/fs/cgroup/kubepods.slice/ | grep -E 'slice|scope|^cpu.max|^cpu.weight|^memory.max' | sed 's/^/    /'
echo
echo "  node allocatable is enforced here:"
printf '    kubepods.slice cpu.max    : %s\n' "$(cat /sys/fs/cgroup/kubepods.slice/cpu.max 2>/dev/null)"
printf '    kubepods.slice cpu.weight : %s\n' "$(cat /sys/fs/cgroup/kubepods.slice/cpu.weight 2>/dev/null)"
printf '    kubepods.slice memory.max : %s\n' "$(cat /sys/fs/cgroup/kubepods.slice/memory.max 2>/dev/null)"

echo
echo "### Where each QoS class lands"
for pod in qos-guaranteed qos-burstable qos-besteffort; do
    uid=$($K get pod "$pod" -o jsonpath='{.metadata.uid}' 2>/dev/null)
    qos=$($K get pod "$pod" -o jsonpath='{.status.qosClass}' 2>/dev/null)
    [[ -z $uid ]] && { echo "  $pod: not found"; continue; }
    # kubelet mangles the uid into the slice name
    slice=$(find /sys/fs/cgroup/kubepods.slice -maxdepth 2 -type d -name "*${uid//-/_}*" 2>/dev/null | head -1)
    echo
    echo "  $pod  (QoS: $qos)"
    echo "    uid  : $uid"
    echo "    slice: ${slice#/sys/fs/cgroup/}"
    if [[ -n $slice ]]; then
        printf '      pod cpu.max     : %s\n' "$(cat "$slice/cpu.max" 2>/dev/null)"
        printf '      pod cpu.weight  : %s\n' "$(cat "$slice/cpu.weight" 2>/dev/null)"
        printf '      pod memory.max  : %s\n' "$(cat "$slice/memory.max" 2>/dev/null)"
        printf '      pod pids.max    : %s\n' "$(cat "$slice/pids.max" 2>/dev/null)"
        # NOTE: there are always TWO scopes per single-container pod. One is
        # the pause/sandbox container that holds the namespaces; it has no
        # limits (cpu.weight=1, memory.max=max). The other is your container.
        for scope in "$slice"/cri-containerd-*.scope; do
            [[ -d $scope ]] || continue
            lim=$(cat "$scope/memory.max" 2>/dev/null)
            [[ $lim == max ]] && kind="(pause/sandbox)" || kind="(app container)"
            echo "      container scope $kind: $(basename "$scope" | cut -c1-40)..."
            printf '        cpu.max    : %s\n' "$(cat "$scope/cpu.max" 2>/dev/null)"
            printf '        cpu.weight : %s\n' "$(cat "$scope/cpu.weight" 2>/dev/null)"
            printf '        memory.max : %s\n' "$(cat "$scope/memory.max" 2>/dev/null)"
        done
    fi
done

echo
echo "### Check the arithmetic: PodSpec -> kernel file"
python3 - <<'PY'
import subprocess, json, glob, os
def sh(c): return subprocess.run(c, shell=True, capture_output=True, text=True).stdout.strip()
pods = json.loads(sh("k3s kubectl get pods -l demo=cgroups -o json") or '{"items":[]}')
def weight_from_millicpu(m):
    shares = max(2, int(m * 1024 / 1000))
    return 1 + ((shares - 2) * 9999) // 262142
def millis(v):
    return int(v[:-1]) if v.endswith('m') else int(float(v) * 1000)
def bytes_of(v):
    u = {'Ki':1024,'Mi':1024**2,'Gi':1024**3}
    for s, m in u.items():
        if v.endswith(s): return int(v[:-2]) * m
    return int(v)
for p in pods['items']:
    name, uid = p['metadata']['name'], p['metadata']['uid']
    res = p['spec']['containers'][0].get('resources', {})
    req, lim = res.get('requests', {}), res.get('limits', {})
    hits = glob.glob(f"/sys/fs/cgroup/kubepods.slice/**/*{uid.replace('-','_')}*", recursive=True)
    slice_dir = next((h for h in hits if os.path.isdir(h)), None)
    if not slice_dir: continue
    print(f"  {name} ({p['status'].get('qosClass')})")
    if 'cpu' in lim:
        want = f"{int(millis(lim['cpu']) * 100)} 100000"
        got = open(f"{slice_dir}/cpu.max").read().strip()
        print(f"    limits.cpu={lim['cpu']:<6} expect cpu.max='{want}'  actual='{got}'  {'OK' if want == got else 'MISMATCH'}")
    if 'cpu' in req:
        want = weight_from_millicpu(millis(req['cpu']))
        got = int(open(f"{slice_dir}/cpu.weight").read().strip())
        print(f"    requests.cpu={req['cpu']:<4} expect cpu.weight={want:<5} actual={got:<5} {'OK' if want == got else 'MISMATCH'}")
    if 'memory' in lim:
        want = bytes_of(lim['memory'])
        got = int(open(f"{slice_dir}/memory.max").read().strip())
        print(f"    limits.memory={lim['memory']:<6} expect memory.max={want:<12} actual={got:<12} {'OK' if want == got else 'MISMATCH'}")
    if not lim and not req:
        print(f"    no resources: cpu.weight={open(f'{slice_dir}/cpu.weight').read().strip()}, "
              f"memory.max={open(f'{slice_dir}/memory.max').read().strip()}")
PY

echo
echo "### Throttling, seen from the node for the Guaranteed pod (limits.cpu: 250m)"
uid=$($K get pod qos-guaranteed -o jsonpath='{.metadata.uid}' 2>/dev/null)
slice=$(find /sys/fs/cgroup/kubepods.slice -maxdepth 2 -type d -name "*${uid//-/_}*" 2>/dev/null | head -1)
if [[ -n $slice ]]; then
    b=$(awk '/^(nr_periods|nr_throttled|usage_usec)/{printf "%s=%s ", $1,$2}' "$slice/cpu.stat")
    sleep 10
    a=$(awk '/^(nr_periods|nr_throttled|usage_usec)/{printf "%s=%s ", $1,$2}' "$slice/cpu.stat")
    echo "  before: $b"
    echo "  after : $a"
    python3 - "$b" "$a" <<'PY'
import sys
b = dict(kv.split('=') for kv in sys.argv[1].split()); a = dict(kv.split('=') for kv in sys.argv[2].split())
d = {k: int(a[k]) - int(b[k]) for k in a}
print(f"  over 10s: periods={d['nr_periods']} throttled={d['nr_throttled']} "
      f"cpu={d['usage_usec']/1e6:.2f}s = {d['usage_usec']/(d['nr_periods']*1e5):.3f} cores")
print(f"  ==> this is container_cpu_cfs_throttled_periods_total at its source.")
PY
fi

echo
echo "### An OOMKilled pod, end to end"
$K get pod oom-victim -o jsonpath='{range .status.containerStatuses[*]}  state: {.state}{"\n"}  lastState: {.lastState}{"\n"}{end}' 2>/dev/null
echo "  kubectl reason/exit code:"
$K get pod oom-victim -o jsonpath='{.status.containerStatuses[0].state.terminated.reason} exitCode={.status.containerStatuses[0].state.terminated.exitCode}{"\n"}' 2>/dev/null | sed 's/^/    /'
echo "  the kernel's side of the same event:"
dmesg | grep -i 'Memory cgroup out of memory' | tail -2 | sed 's/^/    /'

echo
echo "### Kubernetes mapping - what Part 1 explains about this tree"
echo "  kubepods.slice           node allocatable (kube-reserved/system-reserved carved out)"
echo "   +- kubepods-burstable.slice        Burstable pods, cpu.weight from requests"
echo "   +- kubepods-besteffort.slice       BestEffort pods, cpu.weight ~1, no limits"
echo "   +- kubepods-pod<uid>.slice         Guaranteed pods sit DIRECTLY under kubepods"
echo "        +- cri-containerd-<id>.scope  one scope per container"
echo "  Every file in those directories is one you set by hand in Part 1."
```

```text
### The node
  NAME     STATUS   ROLES           AGE    VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION              CONTAINER-RUNTIME
  ubuntu   Ready    control-plane   3m2s   v1.36.4+k3s1   10.0.2.15     <none>        Ubuntu 24.04.2 LTS   6.8.0-138-generic (amd64)   containerd://2.3.4-k3s1.36
  containerd SystemdCgroup: SystemdCgroup=true
  cgroup driver (inferred): systemd (kubepods.slice, not kubepods/)
  containerd runtime   : containerd://2.3.4-k3s1.36

### The top of the kubelet's tree
  kubepods.slice
  system.slice
  user.slice

  kubepods.slice contents:
    cpu.max
    cpu.max.burst
    cpu.weight
    cpu.weight.nice
    kubepods-besteffort.slice
    kubepods-burstable.slice
    kubepods-podbe556e9d_90d1_486b_8187_8c52b4ddc436.slice
    memory.max

  node allocatable is enforced here:
    kubepods.slice cpu.max    : max 100000
    kubepods.slice cpu.weight : 157
    kubepods.slice memory.max : 4105445376

### Where each QoS class lands

  qos-guaranteed  (QoS: Guaranteed)
    uid  : be556e9d-90d1-486b-8187-8c52b4ddc436
    slice: kubepods.slice/kubepods-podbe556e9d_90d1_486b_8187_8c52b4ddc436.slice
      pod cpu.max     : 25000 100000
      pod cpu.weight  : 10
      pod memory.max  : 134217728
      pod pids.max    : max
      container scope (app container): cri-containerd-9b9850da25b8bb7d27228add2...
        cpu.max    : 25000 100000
        cpu.weight : 35
        memory.max : 134217728
      container scope (pause/sandbox): cri-containerd-a27051d86eee37d8cf6da7f6d...
        cpu.max    : max 100000
        cpu.weight : 1
        memory.max : max

  qos-burstable  (QoS: Burstable)
    uid  : 764d1d50-50c3-4f5b-ba08-b618da6fac71
    slice: kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod764d1d50_50c3_4f5b_ba08_b618da6fac71.slice
      pod cpu.max     : 100000 100000
      pod cpu.weight  : 20
      pod memory.max  : 268435456
      pod pids.max    : max
      container scope (pause/sandbox): cri-containerd-65d8af66e9ace74696181d29f...
        cpu.max    : max 100000
        cpu.weight : 1
        memory.max : max
      container scope (app container): cri-containerd-fb539b459c59941f0baf38487...
        cpu.max    : 100000 100000
        cpu.weight : 59
        memory.max : 268435456

  qos-besteffort  (QoS: BestEffort)
    uid  : 6a107ed2-96c9-41bb-8015-3a9865b657ad
    slice: kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod6a107ed2_96c9_41bb_8015_3a9865b657ad.slice
      pod cpu.max     : max 100000
      pod cpu.weight  : 1
      pod memory.max  : max
      pod pids.max    : max
      container scope (pause/sandbox): cri-containerd-c2a4c7c20bc0d38a69370c2e7...
        cpu.max    : max 100000
        cpu.weight : 1
        memory.max : max
      container scope (pause/sandbox): cri-containerd-cb81350f662643f5e99d3444f...
        cpu.max    : max 100000
        cpu.weight : 1
        memory.max : max

### Check the arithmetic: PodSpec -> kernel file
  qos-besteffort (BestEffort)
    no resources: cpu.weight=1, memory.max=max
  qos-burstable (Burstable)
    limits.cpu=1      expect cpu.max='100000 100000'  actual='100000 100000'  OK
    requests.cpu=500m expect cpu.weight=20    actual=20    OK
    limits.memory=256Mi  expect memory.max=268435456    actual=268435456    OK
  qos-guaranteed (Guaranteed)
    limits.cpu=250m   expect cpu.max='25000 100000'  actual='25000 100000'  OK
    requests.cpu=250m expect cpu.weight=10    actual=10    OK
    limits.memory=128Mi  expect memory.max=134217728    actual=134217728    OK

### Throttling, seen from the node for the Guaranteed pod (limits.cpu: 250m)
  before: usage_usec=14121207 nr_periods=1201 nr_throttled=1109 
  after : usage_usec=15122509 nr_periods=1308 nr_throttled=1216 
  over 10s: periods=107 throttled=107 cpu=1.00s = 0.094 cores
  ==> this is container_cpu_cfs_throttled_periods_total at its source.

### An OOMKilled pod, end to end
  state: {"terminated":{"containerID":"containerd://0f69bfd900b48c72fbf5d01912a2ab6fdbdc5b804d1f65d6a570bdcf28bdde86","exitCode":137,"finishedAt":"2026-09-08T14:57:05Z","reason":"OOMKilled","startedAt":"2026-09-08T14:57:03Z"}}
  lastState: {}
  kubectl reason/exit code:
    OOMKilled exitCode=137
  the kernel's side of the same event:
    [83935.909579] Memory cgroup out of memory: Killed process 182745 (dd) total-vm:5444kB, anon-rss:1024kB, file-rss:2048kB, shmem-rss:0kB, UID:0 pgtables:44kB oom_score_adj:984
    [83935.924911] Memory cgroup out of memory: Killed process 182745 (dd) total-vm:5444kB, anon-rss:1024kB, file-rss:2048kB, shmem-rss:0kB, UID:0 pgtables:44kB oom_score_adj:984

### Kubernetes mapping - what Part 1 explains about this tree
  kubepods.slice           node allocatable (kube-reserved/system-reserved carved out)
   +- kubepods-burstable.slice        Burstable pods, cpu.weight from requests
   +- kubepods-besteffort.slice       BestEffort pods, cpu.weight ~1, no limits
   +- kubepods-pod<uid>.slice         Guaranteed pods sit DIRECTLY under kubepods
        +- cri-containerd-<id>.scope  one scope per container
  Every file in those directories is one you set by hand in Part 1.
```

The parts worth dwelling on:

**The tree encodes QoS.** Guaranteed pods sit directly under `kubepods.slice`.
Burstable and BestEffort get their own intermediate slices. That shape is the QoS
class — it is not metadata, it is where reclaim and CPU contention will hit first.

**The arithmetic checks out exactly.** `limits.cpu: 250m` → `cpu.max 25000 100000`.
`requests.cpu: 500m` → `cpu.weight 20`, which is `1 + ((512-2) × 9999) / 262142`
rounded down — the formula from scenario 3, confirmed against a live kubelet.

**BestEffort really does mean nothing.** `cpu.weight 1`, `memory.max max`. It has
no floor and no ceiling; it gets whatever is left and it is reclaimed from first.

**Two scopes per pod.** One is your container; the other is the pause/sandbox
container holding the namespaces, with no limits of its own.

**`OOMKilled` is just the kernel line you already saw.** `exitCode=137`,
`reason: OOMKilled`, and the matching `Memory cgroup out of memory` in `dmesg` —
the same event as scenario 5, reported by a different layer.

---

## Part 3 — Putting the host back

Every change was written to a ledger *before* it was made, with the command that
undoes it. Nothing was persisted: no `/etc/fstab` entry, no `sysctl.d` drop-in,
no package installs.

```bash
#!/usr/bin/env bash
# revert.sh - undo every host change recorded in the ledger, newest first.
# Each row carries its own undo command, so this works even if a scenario
# died half way through and never ran its own cleanup.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

[[ -f $LEDGER ]] || { echo "no ledger at $LEDGER - nothing to revert"; exit 0; }

echo "### Ledger before revert"
cat "$LEDGER"

mapfile -t pending < <(grep '| PENDING |' "$LEDGER" | tac)
if [[ ${#pending[@]} -eq 0 ]]; then
    echo
    echo "No PENDING changes. Host is already back to its original state."
    exit 0
fi

echo
echo "### Reverting ${#pending[@]} change(s), newest first"
for row in "${pending[@]}"; do
    change=$(awk -F'|' '{print $3}' <<< "$row" | sed 's/^ *//;s/ *$//')
    cmd=$(awk -F'|' '{print $4}' <<< "$row" | sed 's/^ *//;s/ *$//;s/^`//;s/`$//')
    echo
    echo "  change: $change"
    echo "  undo  : $cmd"
    if eval "$cmd" > /tmp/revert-out.$$ 2>&1; then
        echo "  result: OK"
        mark_reverted "$change"
    elif grep -qi 'No such file or directory' /tmp/revert-out.$$; then
        # The scenario's own EXIT trap already undid this. The end state is
        # what matters, so "already gone" counts as reverted - not as failure.
        echo "  result: OK (already undone by the scenario's own cleanup)"
        mark_reverted "$change"
    else
        echo "  result: FAILED (exit $?)"
        sed 's/^/          /' /tmp/revert-out.$$
    fi
    rm -f /tmp/revert-out.$$
done

# The demo root is created implicitly by every scenario; remove it last.
if [[ -d $DEMO_ROOT ]]; then
    echo
    echo "  removing $DEMO_ROOT"
    cg_destroy "$DEMO_ROOT" && echo "  result: OK"
    mark_reverted "created cgroup $DEMO_ROOT"
fi

echo
echo "### Ledger after revert"
cat "$LEDGER"
```

```text
### Ledger before revert
# System change ledger

| when | change | revert command | status |
|---|---|---|---|
| 2026-09-08T14:17:53+00:00 | created cgroup /sys/fs/cgroup/blogdemo | `rmdir /sys/fs/cgroup/blogdemo` | PENDING |
| 2026-09-08T14:23:22+00:00 | kernel.sched_cfs_bandwidth_slice_us 5000 -> 1000 | `sysctl -w kernel.sched_cfs_bandwidth_slice_us=5000` | REVERTED |
| 2026-09-08T14:24:44+00:00 | kernel.sched_cfs_bandwidth_slice_us 5000 -> 1000 | `sysctl -w kernel.sched_cfs_bandwidth_slice_us=5000` | REVERTED |
| 2026-09-08T14:35:04+00:00 | enabled 'cpuset' in root cgroup.subtree_control (was: cpu memory pids) | `echo -cpuset > /sys/fs/cgroup/cgroup.subtree_control` | REVERTED |
| 2026-09-08T14:35:44+00:00 | created cgroup /sys/fs/cgroup/blogdemo | `rmdir /sys/fs/cgroup/blogdemo` | PENDING |
| 2026-09-08T14:49:21+00:00 | created and enabled swapfile /swapfile-cgroupdemo (2048 MiB) | `swapoff /swapfile-cgroupdemo; rm -f /swapfile-cgroupdemo` | REVERTED |
| 2026-09-08T14:51:15+00:00 | enabled 'io' in root cgroup.subtree_control | `echo -io > /sys/fs/cgroup/cgroup.subtree_control` | REVERTED |
| 2026-09-08T14:52:04+00:00 | vm.nr_hugepages 0 -> 256 | `sysctl -w vm.nr_hugepages=0` | REVERTED |
| 2026-09-08T14:52:11+00:00 | created cgroup /sys/fs/cgroup/blogdemo | `rmdir /sys/fs/cgroup/blogdemo` | PENDING |
| 2026-09-08T14:52:11+00:00 | enabled 'hugetlb' in root cgroup.subtree_control | `echo -hugetlb > /sys/fs/cgroup/cgroup.subtree_control` | REVERTED |
| 2026-09-08T14:53:06+00:00 | vm.nr_hugepages 0 -> 256 | `sysctl -w vm.nr_hugepages=0` | REVERTED |
| 2026-09-08T14:53:15+00:00 | created cgroup /sys/fs/cgroup/blogdemo | `rmdir /sys/fs/cgroup/blogdemo` | PENDING |
| 2026-09-08T14:53:15+00:00 | enabled 'hugetlb' in root cgroup.subtree_control | `echo -hugetlb > /sys/fs/cgroup/cgroup.subtree_control` | REVERTED |
| 2026-09-08T14:54:00+00:00 | installed k3s (binary, systemd units, /etc/rancher, /var/lib/rancher) | `/usr/local/bin/k3s-uninstall.sh` | REVERTED |

### Reverting 4 change(s), newest first

  change: created cgroup /sys/fs/cgroup/blogdemo
  undo  : rmdir /sys/fs/cgroup/blogdemo
  result: OK (already undone by the scenario's own cleanup)

  change: created cgroup /sys/fs/cgroup/blogdemo
  undo  : rmdir /sys/fs/cgroup/blogdemo
  result: OK (already undone by the scenario's own cleanup)

  change: created cgroup /sys/fs/cgroup/blogdemo
  undo  : rmdir /sys/fs/cgroup/blogdemo
  result: OK (already undone by the scenario's own cleanup)

  change: created cgroup /sys/fs/cgroup/blogdemo
  undo  : rmdir /sys/fs/cgroup/blogdemo
  result: OK (already undone by the scenario's own cleanup)

### Ledger after revert
# System change ledger

| when | change | revert command | status |
|---|---|---|---|
| 2026-09-08T14:17:53+00:00 | created cgroup /sys/fs/cgroup/blogdemo | `rmdir /sys/fs/cgroup/blogdemo` | REVERTED |
| 2026-09-08T14:23:22+00:00 | kernel.sched_cfs_bandwidth_slice_us 5000 -> 1000 | `sysctl -w kernel.sched_cfs_bandwidth_slice_us=5000` | REVERTED |
| 2026-09-08T14:24:44+00:00 | kernel.sched_cfs_bandwidth_slice_us 5000 -> 1000 | `sysctl -w kernel.sched_cfs_bandwidth_slice_us=5000` | REVERTED |
| 2026-09-08T14:35:04+00:00 | enabled 'cpuset' in root cgroup.subtree_control (was: cpu memory pids) | `echo -cpuset > /sys/fs/cgroup/cgroup.subtree_control` | REVERTED |
| 2026-09-08T14:35:44+00:00 | created cgroup /sys/fs/cgroup/blogdemo | `rmdir /sys/fs/cgroup/blogdemo` | REVERTED |
| 2026-09-08T14:49:21+00:00 | created and enabled swapfile /swapfile-cgroupdemo (2048 MiB) | `swapoff /swapfile-cgroupdemo; rm -f /swapfile-cgroupdemo` | REVERTED |
| 2026-09-08T14:51:15+00:00 | enabled 'io' in root cgroup.subtree_control | `echo -io > /sys/fs/cgroup/cgroup.subtree_control` | REVERTED |
| 2026-09-08T14:52:04+00:00 | vm.nr_hugepages 0 -> 256 | `sysctl -w vm.nr_hugepages=0` | REVERTED |
| 2026-09-08T14:52:11+00:00 | created cgroup /sys/fs/cgroup/blogdemo | `rmdir /sys/fs/cgroup/blogdemo` | REVERTED |
| 2026-09-08T14:52:11+00:00 | enabled 'hugetlb' in root cgroup.subtree_control | `echo -hugetlb > /sys/fs/cgroup/cgroup.subtree_control` | REVERTED |
| 2026-09-08T14:53:06+00:00 | vm.nr_hugepages 0 -> 256 | `sysctl -w vm.nr_hugepages=0` | REVERTED |
| 2026-09-08T14:53:15+00:00 | created cgroup /sys/fs/cgroup/blogdemo | `rmdir /sys/fs/cgroup/blogdemo` | REVERTED |
| 2026-09-08T14:53:15+00:00 | enabled 'hugetlb' in root cgroup.subtree_control | `echo -hugetlb > /sys/fs/cgroup/cgroup.subtree_control` | REVERTED |
| 2026-09-08T14:54:00+00:00 | installed k3s (binary, systemd units, /etc/rancher, /var/lib/rancher) | `/usr/local/bin/k3s-uninstall.sh` | REVERTED |
```

The first verification run **failed**, which is the best argument for having one.
`k3s-uninstall.sh` left three things behind: the whole `kubepods.slice` cgroup tree,
`/var/lib/rancher`, and — the one you would never think to check — a root
`cgroup.subtree_control` reading `cpuset cpu io memory hugetlb pids rdma misc`.
The kubelet had enabled every available controller at the root during startup and
nothing put it back. If you have ever uninstalled a Kubernetes distribution from a
node and then reused that node, it is not in the state you think it is.

Then the proof — a snapshot taken before any of this ran, diffed against one taken
after the revert:

```bash
#!/usr/bin/env bash
# verify-clean.sh - prove the host is back where it started by diffing the
# before/after snapshots. Exits non-zero on any difference that matters.
set -uo pipefail
source "$(dirname "$0")/lib.sh"

"$(dirname "$0")/state-snapshot.sh" after > /dev/null

fail=0
# process-summary is informational: process counts move on their own.
STRICT=(root-subtree_control.txt sysctl-hugepages.txt meminfo-bits.txt swapon.txt
        fstab.txt packages.txt demo-artifacts.txt enabled-units.txt cgroup-root-listing.txt)

echo "### Comparing host state before vs after"
for f in "${STRICT[@]}"; do
    b=$STATE_DIR/before/$f; a=$STATE_DIR/after/$f
    if diff -q "$b" "$a" > /dev/null 2>&1; then
        printf '  [ ok ]  %-28s unchanged\n' "$f"
    else
        printf '  [DIFF]  %-28s\n' "$f"
        diff "$b" "$a" | sed 's/^/            /'
        fail=1
    fi
done

echo
echo "### Direct checks"
check() { if [[ $2 == "$3" ]]; then printf '  [ ok ]  %-38s %s\n' "$1" "$2"
          else printf '  [FAIL]  %-38s got %q, want %q\n' "$1" "$2" "$3"; fail=1; fi }
check "root cgroup.subtree_control"  "$(cat $CG_ROOT/cgroup.subtree_control)" "cpu memory pids"
check "demo cgroup removed"          "$([[ -d $DEMO_ROOT ]] && echo present || echo absent)" "absent"
check "kubepods.slice removed"       "$([[ -d $CG_ROOT/kubepods.slice ]] && echo present || echo absent)" "absent"
check "swap devices"                 "$(swapon --show | wc -l)" "0"
check "vm.nr_hugepages"              "$(cat /proc/sys/vm/nr_hugepages)" "0"
check "k3s binary"                   "$([[ -e /usr/local/bin/k3s ]] && echo present || echo absent)" "absent"
check "/var/lib/rancher"             "$([[ -e /var/lib/rancher ]] && echo present || echo absent)" "absent"
check "demo swapfile"                "$([[ -e /swapfile-cgroupdemo ]] && echo present || echo absent)" "absent"
check "demo temp files"              "$(ls /var/tmp/cgroup-demo-* 2>/dev/null | wc -l)" "0"

echo
echo "### Ledger status"
if grep -q '| PENDING |' "$LEDGER" 2>/dev/null; then
    echo "  [FAIL] ledger still has PENDING rows:"
    grep '| PENDING |' "$LEDGER" | sed 's/^/         /'
    fail=1
else
    echo "  [ ok ]  every recorded change is marked REVERTED"
fi

echo
if [[ $fail == 0 ]]; then echo "HOST VERIFIED CLEAN - no residue from the demos."
else echo "HOST NOT CLEAN - see the entries above."; fi
exit $fail
```

```text
### Comparing host state before vs after
  [ ok ]  root-subtree_control.txt     unchanged
  [ ok ]  sysctl-hugepages.txt         unchanged
  [ ok ]  meminfo-bits.txt             unchanged
  [ ok ]  swapon.txt                   unchanged
  [ ok ]  fstab.txt                    unchanged
  [ ok ]  packages.txt                 unchanged
  [ ok ]  demo-artifacts.txt           unchanged
  [ ok ]  enabled-units.txt            unchanged
  [ ok ]  cgroup-root-listing.txt      unchanged

### Direct checks
  [ ok ]  root cgroup.subtree_control            cpu memory pids
  [ ok ]  demo cgroup removed                    absent
  [ ok ]  kubepods.slice removed                 absent
  [ ok ]  swap devices                           0
  [ ok ]  vm.nr_hugepages                        0
  [ ok ]  k3s binary                             absent
  [ ok ]  /var/lib/rancher                       absent
  [ ok ]  demo swapfile                          absent
  [ ok ]  demo temp files                        0

### Ledger status
  [ ok ]  every recorded change is marked REVERTED

HOST VERIFIED CLEAN - no residue from the demos.
```

---

## What to take away

1. **`limits.cpu` throttles; it does not slow down.** A pod at 20% utilisation can
   be stopped for 80% of every period. Alert on `nr_throttled`, not utilisation.
2. **`requests.cpu` is a share, not a reservation.** It does nothing on an idle
   node, and its ratios are approximate once threads migrate.
3. **Quota is only exact when the workload cannot migrate.** That is the real
   argument for the CPU Manager static policy.
4. **`memory.current` includes page cache.** Working set is
   `memory.current − inactive_file`. Alerting on the former will page you for
   healthy pods.
5. **Page cache is billed to whoever touched it first**, which is not necessarily
   whoever is using it.
6. **`memory.high` without swap is a brake with no disc** — the kernel scans and
   frees nothing, and your workload just stalls.
7. **Disk is the unprotected resource.** `io.max` works fine; Kubernetes simply
   never gave you a field for it.
8. **cgroup v2 is the price of entry** for MemoryQoS, PSI, `cgroup.kill` and swap.

The most useful habit: when a pod misbehaves, go and read its cgroup files on the
node. The kubelet's view is a summary — `/sys/fs/cgroup` is the source.

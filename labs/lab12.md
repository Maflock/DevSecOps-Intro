# Lab 12 — BONUS — Kata Containers: VM-Backed Container Isolation

![difficulty](https://img.shields.io/badge/difficulty-advanced-red)
![topic](https://img.shields.io/badge/topic-VM%20Sandboxing-blue)
![points](https://img.shields.io/badge/points-10-orange)
![tech](https://img.shields.io/badge/tech-Kata%20Containers-informational)

> **Goal:** Install Kata Containers as a containerd runtime, run an attack scenario in both runc and kata, demonstrate the isolation difference, and measure performance overhead.
> **Deliverable:** A PR from `feature/lab12` with `submissions/lab12.md` + captured outputs comparing the two runtimes. Submit PR link via Moodle.

> 🌟 **This is a BONUS lab** — 10 pts total (Task 1: 6 + Task 2: 4), no separate bonus row.
> Bonus labs count toward a separate **30% weight** in your final grade.
> Difficulty is **advanced** — kernel/runtime layer; requires Linux host with KVM access.

---

## Overview

In this lab you will practice:
- **Kata Containers v3.x** — VM-backed container runtime under containerd (Lecture 7 mentioned container escapes; Kata is the defense-in-depth for that class)
- **Isolation tests** — kernel CVE simulation, capability access, syscall surface
- **Performance benchmark** — runc vs kata for startup time + CPU-bound + I/O-bound workloads

> Recall Lecture 7 slide 14 — runc CVE-2024-21626 ("Leaky Vessels"): a single container escape affecting EVERY runc-based runtime. **Kata Containers wraps each container in a lightweight VM**, so a runc-style escape only escapes into a throwaway VM, not the host.

---

## Project State

**You should have from Labs 1+7:**
- Familiarity with container fundamentals (Lab 7)
- A Linux host with kernel ≥ 5.8 + KVM access (`/dev/kvm` exists + readable)

**This lab adds:**
- Kata Containers installed + registered as a containerd runtime
- Side-by-side run-comparison data (runc vs kata)
- Performance + isolation analysis with real numbers

> ⚠️ **macOS / Windows users:** Kata requires KVM, which doesn't work in Docker Desktop's Linux VM. Either:
> 1. Spin up a Linux VM with nested virtualization (VirtualBox, VMware), OR
> 2. Use a cloud VM with KVM (most providers — except low-end / `t2.micro`-style EC2 — support it), OR
> 3. Use a bare-metal Linux machine

---

## Setup

You need:
- **Linux host with KVM**: `lsmod | grep kvm` should show kvm_intel or kvm_amd
- **`/dev/kvm` readable**: `ls -la /dev/kvm` (add yourself to `kvm` group if needed)
- **containerd + nerdctl** installed (most Linux distros via packages)
- **`sudo`** (you'll install kata into `/opt`, modify `/etc/containerd/config.toml`)

```bash
git switch main && git pull
git switch -c feature/lab12

# Verify KVM access
lsmod | grep -E "kvm_intel|kvm_amd" && ls -la /dev/kvm
# If you don't see either, you're not on a KVM-capable host — skip this lab

# Verify containerd
sudo systemctl status containerd
nerdctl --version

mkdir -p labs/lab12/results
```

> **Plumbing provided** (already in `labs/lab12/`):
> - [`labs/lab12/scripts/install-kata-assets.sh`](lab12/scripts/install-kata-assets.sh) — downloads kata-static + configures
> - [`labs/lab12/scripts/configure-containerd-kata.sh`](lab12/scripts/configure-containerd-kata.sh) — updates containerd config to register `kata` runtime
> - [`labs/lab12/setup/build-kata-runtime.sh`](lab12/setup/build-kata-runtime.sh) — optional from-source build

---

## Task 1 — Install + Run Identical Workload on Both Runtimes (6 pts)

**Objective:** Install Kata, register it with containerd, then run the same Juice Shop container on both `runc` and `kata` runtimes. Capture proof.

### 12.1: Install Kata

```bash
sudo bash labs/lab12/scripts/install-kata-assets.sh
# Downloads kata-static, installs to /opt/kata, registers OCI hooks

sudo bash labs/lab12/scripts/configure-containerd-kata.sh
# Updates /etc/containerd/config.toml to add:
#   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
#     runtime_type = "io.containerd.kata.v2"

sudo systemctl restart containerd

# Verify kata is registered
sudo ctr runtime ls 2>/dev/null || true
# OR
grep -A2 'runtimes.kata' /etc/containerd/config.toml
```

### 12.2: Sanity test — hello-world on Kata

```bash
sudo nerdctl run --runtime=io.containerd.kata.v2 --rm alpine:3.20 \
  /bin/sh -c "uname -a; cat /proc/1/cgroup | head -3; ls /sys/devices/virtual/dmi/id/ 2>/dev/null || true"

# Compare to runc — the kernel version will differ (Kata has its own mini-kernel)
sudo nerdctl run --rm alpine:3.20 \
  /bin/sh -c "uname -a; cat /proc/1/cgroup | head -3"
```

### 12.3: Run Juice Shop on both runtimes

```bash
# runc (default)
sudo nerdctl run -d --name juice-runc \
  -p 127.0.0.1:3000:3000 \
  bkimminich/juice-shop:v20.0.0

# kata
sudo nerdctl run -d --name juice-kata \
  --runtime=io.containerd.kata.v2 \
  -p 127.0.0.1:3001:3000 \
  bkimminich/juice-shop:v20.0.0

# Verify both are running
sudo nerdctl ps | grep juice
```

### 12.4: Capture proof

```bash
# kernel from inside each container
sudo nerdctl exec juice-runc sh -c "uname -a; cat /proc/version" > labs/lab12/results/runc-kernel.txt
sudo nerdctl exec juice-kata sh -c "uname -a; cat /proc/version" > labs/lab12/results/kata-kernel.txt

# /proc/cpuinfo — different in a VM
sudo nerdctl exec juice-runc sh -c "head -5 /proc/cpuinfo" > labs/lab12/results/runc-cpuinfo.txt
sudo nerdctl exec juice-kata sh -c "head -5 /proc/cpuinfo" > labs/lab12/results/kata-cpuinfo.txt

# diff them
diff labs/lab12/results/runc-kernel.txt labs/lab12/results/kata-kernel.txt
```

### 12.5: Document in `submissions/lab12.md`

```markdown
# Lab 12 — BONUS — Submission

## Task 1: Install + Compare

### Host environment
- Kernel: <output of `uname -a` on the host>
- KVM accessible: <output of `ls -la /dev/kvm`>
- containerd version: <output>

### Kata installation
- Kata version: <output of `/opt/kata/bin/kata-runtime --version`>
- containerd config snippet (runtimes.kata block):
```toml
<paste relevant lines from /etc/containerd/config.toml>
```

### Kernel from inside containers
runc:
```
<paste labs/lab12/results/runc-kernel.txt — should show your host kernel>
```

kata:
```
<paste labs/lab12/results/kata-kernel.txt — should show DIFFERENT kernel (Kata's mini-kernel, often older Linux)>
```

### Why the kernel differs (2-3 sentences)
Reference Lecture 7 slide 4 — "container is a Linux process; shares kernel with host."
Kata breaks that model. What does the kernel difference imply for the runc-CVE-2024-21626
("Leaky Vessels") attack class?
```

---

## Task 2 — Isolation Test + Performance Comparison (4 pts)

**Objective:** Demonstrate one concrete isolation difference + measure performance overhead.

### 12.6: Isolation test

```bash
# Test: can the container see host devices? (one indicator of kernel sharing)
sudo nerdctl exec juice-runc ls /dev | head -20 > labs/lab12/results/runc-devs.txt
sudo nerdctl exec juice-kata ls /dev | head -20 > labs/lab12/results/kata-devs.txt
diff labs/lab12/results/runc-devs.txt labs/lab12/results/kata-devs.txt | tee labs/lab12/results/dev-diff.txt

# Test: kernel module visibility
sudo nerdctl exec juice-runc sh -c "lsmod 2>/dev/null | head -10 || echo 'no lsmod'" \
  > labs/lab12/results/runc-lsmod.txt
sudo nerdctl exec juice-kata sh -c "lsmod 2>/dev/null | head -10 || echo 'no lsmod'" \
  > labs/lab12/results/kata-lsmod.txt
```

### 12.7: Performance benchmark — startup time

```bash
# Cold start time — average of 5 runs
for runtime in runc kata; do
  sudo nerdctl rm -f bench 2>/dev/null
  RUNTIME_FLAG=""
  [ "$runtime" = "kata" ] && RUNTIME_FLAG="--runtime=io.containerd.kata.v2"
  echo "=== $runtime ==="
  for i in 1 2 3 4 5; do
    START=$(date +%s.%N)
    sudo nerdctl run --rm $RUNTIME_FLAG alpine:3.20 echo "hello" > /dev/null
    END=$(date +%s.%N)
    echo "$i: $(echo "$END - $START" | bc) s"
  done
done | tee labs/lab12/results/startup-bench.txt
```

### 12.8: CPU-bound + I/O-bound

```bash
# CPU-bound: 5M-iteration loop
for runtime in runc kata; do
  RUNTIME_FLAG=""
  [ "$runtime" = "kata" ] && RUNTIME_FLAG="--runtime=io.containerd.kata.v2"
  echo "=== $runtime CPU ==="
  sudo nerdctl run --rm $RUNTIME_FLAG alpine:3.20 \
    sh -c 'i=0; while [ $i -lt 5000000 ]; do i=$((i+1)); done; echo done'
done > /dev/null   # we just care about the timing
time sudo nerdctl run --rm alpine:3.20 sh -c 'i=0; while [ $i -lt 5000000 ]; do i=$((i+1)); done'
# Compare runtime variants — paste both into submission
```

```bash
# I/O-bound: dd 100MB of /dev/zero to /dev/null
for runtime in runc kata; do
  RUNTIME_FLAG=""
  [ "$runtime" = "kata" ] && RUNTIME_FLAG="--runtime=io.containerd.kata.v2"
  echo "=== $runtime I/O ==="
  sudo nerdctl run --rm $RUNTIME_FLAG alpine:3.20 \
    sh -c 'dd if=/dev/zero of=/dev/null bs=1M count=100 2>&1' | grep "copied"
done | tee labs/lab12/results/io-bench.txt
```

### 12.9: Document in `submissions/lab12.md`

```markdown
## Task 2: Isolation + Performance

### Isolation: /dev contents
diff between runc /dev and kata /dev:
```
<paste labs/lab12/results/dev-diff.txt>
```

### Isolation: lsmod
runc lsmod output (first 10 lines):
```
<paste — should show host kernel modules>
```
kata lsmod output:
```
<paste — should show minimal mini-kernel modules OR "no lsmod" because Kata's image lacks it>
```

### Startup time (avg of 5 runs)
| Runtime | Avg startup time (s) |
|---------|---------------------:|
| runc | <e.g. 0.45s> |
| kata | <e.g. 2.10s> |

**Overhead: <kata - runc>s (~5x typical for cold starts)**

### CPU-bound
| Runtime | Time (s) |
|---------|---------:|
| runc | <e.g. 8.2s> |
| kata | <e.g. 8.4s> |

### I/O-bound (100MB dd through pipe)
| Runtime | Throughput |
|---------|-----------|
| runc | <e.g. 12.5 GB/s> |
| kata | <e.g. 1.2 GB/s> |

### Trade-off analysis (3-4 sentences)
Kata costs you startup time + I/O throughput. When is the security gain (separate kernel,
runc-CVE class blocked) worth that cost? When isn't it? Give one example each (e.g.,
"multi-tenant SaaS workloads = yes; single-tenant batch jobs = no").

### When you'd actually deploy Kata (1 paragraph)
Reference Lecture 7 + runc CVE-2024-21626. In what production scenarios would the
security team push for Kata adoption? What blocks adoption today (cost? operational complexity?
ecosystem support?)?
```

---

## How to Submit

```bash
git add submissions/lab12.md
git commit -m "feat(lab12): kata vs runc isolation + perf comparison"
git push -u origin feature/lab12

# Cleanup
sudo nerdctl rm -f juice-runc juice-kata
```

PR checklist body:

```text
- [x] Task 1 — Kata installed, registered, hello-world + Juice Shop running on both, kernel diff documented
- [ ] Task 2 — Isolation test + startup + CPU + I/O benchmarks + trade-off analysis
```

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ Kata installed; `kata-runtime --version` returns 3.x
- ✅ containerd config includes the `runtimes.kata` block
- ✅ Both `juice-runc` and `juice-kata` containers running concurrently
- ✅ Kernel inside containers documented for BOTH runtimes; diff shows they differ
- ✅ "Why the kernel differs" answer correctly maps to Lecture 7 + runc CVE class

### Task 2 (4 pts)
- ✅ /dev diff or lsmod difference documented (isolation evidence)
- ✅ Startup time measured for both (5 runs averaged)
- ✅ CPU + I/O benchmarks captured for both
- ✅ Trade-off analysis has both "would deploy" and "wouldn't deploy" scenarios with reasoning
- ✅ Production-deployment reflection addresses operational complexity / ecosystem support honestly

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Install + run | **6** | Kata working + both containers running + kernel-diff with explanation |
| **Task 2** — Isolation + perf | **4** | /dev diff + 3 benchmarks (startup, CPU, I/O) + production-trade-off analysis |
| **Total** | **10** | (No bonus row — this IS the bonus lab) |

---

## Resources

<details>
<summary>📚 Documentation</summary>

- [Kata Containers documentation](https://katacontainers.io/docs/) — Project docs
- [Kata Containers architecture](https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture/README.md) — How the VM-per-container model works
- [containerd runtimes integration](https://github.com/containerd/containerd/blob/main/docs/cri/config.md#runtime-classes) — Where Kata plugs in
- [Linux KVM documentation](https://www.kernel.org/doc/html/latest/virt/kvm/index.html) — The hypervisor Kata uses
- [Lecture 7 slide 14: runc CVE-2024-21626](#) — The case that motivates this lab

</details>

<details>
<summary>⚠️ Common Pitfalls</summary>

- 🚨 **`ls /dev/kvm: No such file or directory`** — you're not on a KVM-capable host. Linux on bare metal works; Docker Desktop's Linux VM doesn't; cloud VMs vary (most enable KVM; older `t2.micro`-style EC2s do not).
- 🚨 **`Permission denied on /dev/kvm`** — `sudo usermod -aG kvm $USER && newgrp kvm` to add yourself to the kvm group, OR run nerdctl with `sudo`.
- 🚨 **Container starts on kata but `juice-shop` never becomes healthy** — Kata's nested networking adds latency. Juice Shop on Kata might take 60-90s to start (vs ~20s on runc). Don't conclude "Kata is broken" — wait longer.
- 🚨 **`nerdctl run --runtime=...` fails with "no such runtime"** — verify containerd reloaded the config: `sudo systemctl restart containerd` (you ran this in 12.1; if you edited config since, re-run).
- 🚨 **dd reports MB/s on runc but B/s on kata** — different output formats due to dd timing. Use `| grep "copied"` to extract the rate line consistently.
- 🚨 **Benchmark numbers vary wildly between runs** — KVM startup is unstable for the first few runs (warming caches). Discard the first 1-2 runs in your average.
- 💡 **If the lab is impossible on your laptop**: spin up a cloud VM ($1-2 for a few hours of work). c6i.xlarge on AWS, n2-standard-4 on GCP — both support KVM.

</details>

<details>
<summary>🪜 Looking outside this course</summary>

- **gVisor** is another sandboxed-container runtime — user-space syscall emulation (no KVM needed) but higher CPU overhead. Read [gVisor docs](https://gvisor.dev/docs/) for comparison.
- **Firecracker** is AWS's bare-bones VMM, also used as a container runtime by some platforms (Fly.io). Lighter than Kata, less ecosystem support.
- **Confidential Containers (CoCo)** is the next frontier — uses Intel SGX/TDX or AMD SEV to encrypt the VM memory. The "VM-isolation" defense layer of 2027.
- **In your portfolio**: "I evaluated Kata Containers vs runc for sandboxed workloads, measured 5× cold-start overhead vs near-zero CPU overhead" is a strong interview line. Reference your specific benchmark numbers.

</details>

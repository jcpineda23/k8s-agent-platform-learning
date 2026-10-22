# Session 02: From containers to a cluster you build by hand

**Dates:** 2026-10-19 onward
**Plan position:** Phase 0, Week 2
**Status:** Part 1 discussed; Lab 2a (VMs), jumpbox and compute resources labs done (2026-10-21). Open: check-your-understanding answers; remove Lima containerd from node-0 and node-1 before the worker lab.

[← Back to index](../index.md) · [Previous: Session 01, control loops](01-week1-control-loops.md)

---

## Where we are

Session 01 showed Kubernetes from the outside: control loops talking through the API server. This session goes one level down on each side:

- **Part 1, below the pod:** what a container actually is, using Eric Chiang's [Containers from Scratch](https://ericchiang.github.io/post/containers-from-scratch/).
- **Part 2, the whole cluster:** start *Kubernetes the Hard Way*, where you install every box from the Lab 1 diagram yourself.

---

## Part 1: What a container really is

### chroot (discussed 2026-10-19)

`chroot` changes one thing for a process: **where `/` points.** After `chroot rootfs`, every path lookup starts at the `rootfs` directory, for that process and all its children.

What the article's commands show:
- `sudo chroot rootfs /bin/bash` starts `rootfs/bin/bash`, not the host's bash, and `ls /` lists the tarball's directories.
- Python runs because everything it needs is *inside* the tarball: the interpreter, its shared libraries in `rootfs/lib`, and device files in `rootfs/dev`. The host's copies don't help, because the process can't see them.
- `chroot rootfs python -m SimpleHTTPServer` runs an app with no shell, which is what a container does: one process, started with an image as its root.

The ideas to keep:
1. **A container image is a directory tree packaged as a tarball.** OCI images are layered tarballs that the runtime unpacks.
2. **There's no kernel in the tarball.** Every process makes system calls to the **host's kernel**. Containers share the kernel; only the userland comes from the image.
3. **chroot isolates the filesystem view and nothing else.** The process still sees and can kill host processes (mount `/proc` and run `ps aux`), shares the host network, is the real root user, and has no resource limits.
4. **chroot isn't a security boundary.** Root can escape it. Real runtimes use `pivot_root` inside a mount namespace instead.

Each later section of the article closes one gap:

| Gap left by chroot | Fixed by |
|---|---|
| Sees host processes | PID namespace |
| Shares host network, hostname, mounts | Network, UTS and mount namespaces |
| No resource limits | cgroups |
| Root is real root | Capabilities, seccomp, user namespaces |

**Agent-platform link:** even with every layer, the container still talks to the **shared host kernel**, and a kernel bug reachable from inside affects the whole node. That's why Phase 3 covers gVisor (a user-space kernel that intercepts system calls) and Kata or Firecracker (a separate kernel per sandbox, in a lightweight VM).

### Namespaces with `unshare` and `nsenter` (next in the article)

- **`unshare`** wraps one system call that gives a process new namespaces. `unshare -p -f --mount-proc=... chroot rootfs /bin/bash` puts the shell in a new PID namespace, where it is PID 1 and `ps aux` shows only its own processes.
- **`nsenter`** wraps the `setns` system call, which *joins* an existing namespace instead of creating one. The kernel exposes each process's namespaces as files under `/proc/<pid>/ns/`. Joining another process's PID namespace lets two shells see each other.
- **Why it matters for Kubernetes:** namespaces are composable. A **pod** is a group of containers that share some namespaces (network, and optionally PID) while keeping separate mount namespaces. That's why containers in one pod reach each other on `localhost`.

### Check your understanding

1. If you deleted `rootfs/lib` and ran the chroot again, what would happen when you start `python`, and why doesn't the host's `/lib` help?
2. You `chroot` into a CentOS tarball while the host runs Ubuntu. Whose kernel version does `uname -r` report, and why?
3. In one sentence for your team: why isn't chroot, or any container, enough to sandbox untrusted agent code?
4. Two containers in one pod share a network namespace. What does that let them do, and what can go wrong if one of them is an untrusted agent sandbox and the other holds credentials?

### Try it yourself (optional)

macOS has no Linux `chroot`, so run the article inside a privileged Linux container:

```bash
docker run --rm -it --privileged --platform linux/amd64 debian:12 bash
```

`--platform linux/amd64` matters: the article's tarball contains x86-64 binaries, and your Mac is Apple Silicon, so Docker has to emulate x86-64. Inside the container, install the download tools, then follow the article from `wget` onward:

```bash
apt-get update && apt-get install -y wget
```

---

## Part 2: Kubernetes the Hard Way

[github.com/kelseyhightower/kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

### Why do it

kind gave you a working cluster in one command. Here you build one from loose binaries: you generate every certificate, write every config file and start every component as a systemd service. When you're done, every box in the Lab 1 diagram is something you installed and wired up yourself.

**As a manager, focus on three things as you go:**
- **Trust:** who talks to whom, and which certificate proves each identity.
- **Config:** what each component needs to know to find the others.
- **Startup order:** what each component needs before it can start.

Don't optimize for speed. When a step works, stop and ask which box in the diagram you just built.

### What you'll build

Four Debian 12 machines (ARM64 is supported, so your Mac runs them natively):

| Machine | Role | CPU | RAM | Disk |
|---|---|---|---|---|
| `jumpbox` | Where you run the lab and hold the files | 1 | 512 MB | 10 GB |
| `server` | Control plane: etcd, API server, scheduler, controller manager | 1 | 2 GB | 20 GB |
| `node-0` | Worker: containerd, kubelet, kube-proxy | 1 | 2 GB | 20 GB |
| `node-1` | Worker | 1 | 2 GB | 20 GB |

The labs, in order: prerequisites, jumpbox, compute resources, certificate authority and TLS certificates, kubeconfig files, data encryption config, etcd, control plane, worker nodes, remote kubectl, pod network routes, smoke test, cleanup.

**Proposed pacing:**
- **This session:** create the VMs, set up the jumpbox, and finish the compute resources lab.
- **Session 03:** certificates and kubeconfigs, the trust model.
- **Session 04:** etcd and the control plane.
- **Session 05:** workers, pod networking and the smoke test.

### Lab 2a: Create the four VMs with Lima

We'll use **Lima**, a command-line tool for running Linux VMs on a Mac. Its `user-v2` network lets the VMs reach each other, which the cluster needs.

Install it:

```bash
brew install lima
```

Create the four VMs. Each command downloads Debian 12 the first time, then starts the VM:

```bash
limactl start --name=jumpbox --cpus=1 --memory=1 --disk=10 --network=lima:user-v2 --tty=false template:debian-12
```

```bash
limactl start --name=server --cpus=1 --memory=2 --disk=20 --network=lima:user-v2 --containerd=none --tty=false template:debian-12
```

```bash
limactl start --name=node-0 --cpus=1 --memory=2 --disk=20 --network=lima:user-v2 --containerd=none --tty=false template:debian-12
```

```bash
limactl start --name=node-1 --cpus=1 --memory=2 --disk=20 --network=lima:user-v2 --containerd=none --tty=false template:debian-12
```

The jumpbox gets 1 GB instead of 512 MB to leave room for Lima's own processes.

`--containerd=none` matters on the three cluster machines. By default Lima installs its own containerd in every VM. The worker nodes get their own containerd from the lab, and two copies would compete, so we tell Lima to skip it. The jumpbox doesn't run containers, so its copy is harmless.

Check that all four are running:

```bash
limactl list
```

Confirm the OS version. It should print `VERSION_ID="12"`:

```bash
limactl shell server grep VERSION_ID /etc/os-release
```

Confirm the VMs can reach each other by name:

```bash
limactl shell jumpbox ping -c 2 lima-server.internal
```

Record each VM's IP address. You'll need them for the `machines.txt` file in the compute resources lab:

```bash
for vm in server node-0 node-1; do echo "$vm $(limactl shell $vm hostname -I)"; done
```

**Adjustments to the lab for Lima:**
- **Getting a shell:** where the lab says to SSH into the jumpbox, use `limactl shell jumpbox`, then `sudo -i` to become root.
- **Root SSH:** the compute resources lab enables SSH as root and copies keys using a root password. Lima VMs have no root password, so instead, after generating the key on the jumpbox, add the jumpbox's public key to `/root/.ssh/authorized_keys` on each machine with `limactl shell <vm> sudo ...`. We'll do this together when you get there.
- **Pod network routes** (lab 11) send traffic between nodes through each other's IPs. Lima's `user-v2` network should allow this, but we'll verify it when we reach that lab.

**Cleanup**, when the whole lab is finished:

```bash
limactl delete --force jumpbox server node-0 node-1
```

### Lab 2b: Follow the lab

Work through these two docs, with the adjustments above:
1. `docs/02-jumpbox.md`: install tools on the jumpbox and download the Kubernetes binaries. Note that it picks ARM64 binaries automatically on your machines.
2. `docs/03-compute-resources.md`: build `machines.txt`, set up root SSH, hostnames and `/etc/hosts`.

The kind cluster from Session 01 isn't needed anymore; delete it if you want the memory back with `kind delete cluster --name learn`.

### Progress log

**Your cluster machines** (Lima `user-v2` network):

| VM | IP | Notes |
|---|---|---|
| jumpbox | | Lab workstation. Lab repo at `/root/kubernetes-the-hard-way` (commit `52eb26d`, 2025-04-09). |
| server | 192.168.104.3 | Has Lima's containerd; harmless, the control plane doesn't use it. |
| node-0 | 192.168.104.4 | **Has Lima's containerd. Remove it before the worker-node lab.** |
| node-1 | 192.168.104.5 | **Has Lima's containerd. Remove it before the worker-node lab.** |

All three are reachable from the jumpbox by IP and as `lima-<name>.internal`.

**Lessons from the VM setup:**
- Commands typed in the app's `!` box run on the Mac, one at a time. Work inside a VM means typing into a terminal tab after `limactl shell <vm>`, or running `limactl shell <vm> sudo bash -c '...'`.
- `sudo` asking for a password means you're on the Mac, not in a VM. Don't enter it.

**Jumpbox lab (done 2026-10-21):**
- **11 downloads, 544 MB.** 6 Kubernetes binaries (v1.32.3) plus etcd, containerd, runc, the CNI plugins and crictl from other projects. Kubernetes defines interfaces (CRI, CNI) and plugs other projects in.
- **Three are pre-releases:** `runc v1.3.0-rc.1`, `containerd 2.1.0-beta.0`, `etcd v3.6.0-rc.3`. Fine for a lab; in production, choosing versions and checking they work together is real work that managed Kubernetes does for you.
- **Each component is one static Go binary of 54–87 MB.** Installing is copying a file; the complexity is in flags, certificates and config.
- **Sorted into role folders:**
  - `client/`: etcdctl, kubectl
  - `controller/`: etcd, kube-apiserver, kube-controller-manager, kube-scheduler
  - `worker/`: containerd, containerd-shim-runc-v2, crictl, ctr, kube-proxy, kubelet, runc
  - `cni-plugins/`: 19 plugins; the lab uses bridge, loopback and host-local
- **The shim:** containerd starts a `containerd-shim-runc-v2` per container, and the shim calls runc. Containers survive containerd restarts because the shim is their parent. gVisor (`containerd-shim-runsc-v1`) and Kata (`containerd-shim-kata-v2`) plug in at exactly this point, selected per pod by a RuntimeClass.
- **`kubectl version --client`** prints `v1.32.3` with no server version, because no API server exists yet.

**Compute resources lab (done 2026-10-21):**

`machines.txt` on the jumpbox (`/root/kubernetes-the-hard-way`):

```
192.168.104.3 server.kubernetes.local server
192.168.104.4 node-0.kubernetes.local node-0 10.200.0.0/24
192.168.104.5 node-1.kubernetes.local node-1 10.200.1.0/24
```

- **The pod subnets** (fourth column) are separate from the VMs' real addresses. node-0 hands out `10.200.0.x` to its pods and node-1 hands out `10.200.1.x`. Lab 11 adds routes between them, the job a CNI plugin like Cilium does automatically.
- **Hostnames** are now `server`, `node-0` and `node-1`, with full names `<name>.kubernetes.local`. They matter because certificates list the names a machine can be reached by, and the kubelet's certificate must match its node name.

How we adapted the lab for Lima:

| Lab step | What we did instead | Why |
|---|---|---|
| Set `PermitRootLogin yes` | Skipped | Key-only root login (`without-password`) was already allowed, and it's safer than also allowing passwords. |
| `ssh-copy-id` with root's password | Generated `/root/.ssh/id_rsa` on the jumpbox and appended the public key to each machine's `/root/.ssh/authorized_keys` via `limactl shell` | Lima VMs have no root password. |
| `sed` the `127.0.1.1` line in `/etc/hosts` | Deleted any `127.0.1.1` line, then inserted `127.0.1.1 <fqdn> <host>` | Lima's `/etc/hosts` has no such line, so the lab's `sed` would silently do nothing. |
| *(not in lab)* | Wrote `/etc/cloud/cloud.cfg.d/99-kubernetes-the-hard-way.cfg` with `preserve_hostname: true` and `ssh_deletekeys: false` on all three machines | See the restart finding below. |

**Restart finding.** Lima gives each VM a **new cloud-init instance ID on every start**, so cloud-init treats each boot as a brand-new machine and re-runs its first-boot steps. Without the fix, a restart:
- reset the hostname to `lima-<name>`, which would break the kubelet's identity later
- regenerated the SSH host keys, so the jumpbox refused to connect with "REMOTE HOST IDENTIFICATION HAS CHANGED"

`/etc/hosts` entries and the IP addresses survived. After the fix, restarting node-1 kept the same hostname, IP and host key (`SHA256:VtguZ81M...`).

**Lesson for the platform:** SSH refused to connect because it couldn't tell an innocent rebuild from an attacker. Machine identity that changes on restart is a real fleet problem, and it's what TLS certificates handle for Kubernetes components (Session 03).

If the jumpbox ever shows the host-identification warning again, clear its old record for that machine and reconnect, for example:

```bash
limactl shell jumpbox sudo ssh-keygen -f /root/.ssh/known_hosts -R node-1
```

### Check your understanding (after Lab 2b)

1. The jumpbox downloads `kube-apiserver`, `kubelet`, `etcd` and others. Sort them into "runs on server" and "runs on nodes". Which ones from the Lab 1 diagram ran as **static pods** in kind, and how will they run here instead?
2. Why does every machine need to resolve every other machine's hostname before you generate any certificates? (Hint: what does a certificate prove?)
3. kind put all of this in Docker containers on one laptop. What does a real cloud provider's managed control plane (EKS, GKE) take off your hands compared with this lab, and what does it *not*?

### My answers

_(Write your answers here before Session 03.)_

---

**Next session:** review answers, then the certificate authority and kubeconfig labs, the cluster's trust model.

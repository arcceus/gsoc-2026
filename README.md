# Enabling Checkpoint/Restore of Rootless Containers

**Organization:** [CRIU (Checkpoint/Restore In Userspace)](https://criu.org/)  
**Contributor:** Deepak Anand ([@arcceus](https://github.com/arcceus))  
**Mentors:** [Radostin Stoyanov](https://github.com/rst0git), [Adrian Reber](https://github.com/adrianreber)  

## Overview

I worked on making checkpoint/restore work for rootless containers without running the operation through `sudo`. This required changes to CRIU, the OCI runtime, and the container engine, because each of them owns a different part of container state.

I started by forcing unprivileged mode through `/etc/default.conf`, removing Podman's root-only checks, checking where checkpoint and restore failed, probing what was possible, and repeating that loop until I had a working prototype. From there it was improving the permission contract between CRIU, the runtime, and the container engine, managing ownership of state, working through upstream design discussions, and fixing a lot of issues across the stack.

## Results

- Rootless containers can be checkpointed and restored without root privileges by granting CRIU the required file capabilities.
- OCI-configured seccomp profiles, listening sockets, delegated cgroup v2 state, and runtime-managed resources are preserved across restore.
- Tested across simple processes, network servers, event-driven runtimes, shared-memory workloads, nested user namespaces, cgroups, and multi-service applications.
- Example workloads include `sleep infinity`, BusyBox `nc`, nginx, Redis, PostgreSQL, Python `http.server`, Node.js, Go services, `--userns=auto`, cgroup FD tests, and a small multi-service setup under load.
- CRIU ZDTM, Podman BATS, and checkpoint-focused Ginkgo tests cover the main dump, restore, archive, compression, host IPC, cgroup, namespace, mount, shmem, and pipe cases.

## Runtime Requirements

Rootless C/R still requires the CRIU binary to have `cap_sys_ptrace,cap_checkpoint_restore=eip`.

## Implementation

- CRIU handles: user-namespace-aware helpers, unprivileged memory and mapped-file fallbacks, seccomp BPF RPC restore, mount option ID translation, external cgroup handling, and restore into a caller-provided user namespace.
- crun connects this to the OCI runtime path through external user namespace setup, rootless CRIU options, OCI seccomp BPF generation, temporary staging for runtime-owned mounts, delegated cgroup v2 mapping, and restored process placement.
- Podman allows checkpoint and restore for rootless containers, checks whether the runtime supports the needed features, saves and restores network state, cleans up only network interfaces it owns, fixes restore order, and handles host IPC.

## Challenges

- With rootless restore, some kernel operations can be done with capabilities in the container's user namespace, while others still require privileges in the initial user namespace. Much of the work was figuring out what CRIU could restore itself, what needed help from the runtime, what should be treated as external state, and which cases should fail with a clear error instead of restoring incomplete state. The fail-closed cases include unmapped rootless cgroup FDs, seccomp BPF state that is not explicitly marked external, seccomp listeners, remount operations denied because of unsupported flags, and `binfmt_misc` when the caller has not marked it external.
- Seccomp had to be split by where the filter came from. Dynamically installed filters cannot be read rootlessly because `PTRACE_SECCOMP_GET_FILTER` needs `CAP_SYS_ADMIN` in `init_user_ns`. OCI-configured filters are different because Podman and crun already know the profile from the container config, so crun can build the BPF payload and pass it to CRIU during restore.
- Mounts were difficult because the same mount can be seen with different IDs inside and outside the container's user namespace. CRIU had to translate `uid=` and `gid=` mount options through the target ID map. crun also had to stage runtime-owned mounts such as `/etc/hosts`, `/etc/resolv.conf`, `/run/.containerenv`, and `/dev/shm` under the container state directory so CRIU restore could use stable sources, then clean that staging area after failed restore or container deletion.
- Cgroups had to stay inside the delegated rootless subtree. A rootless restore should not take ownership of the host cgroup tree, so the runtime maps the cgroup v2 mount and places the restored process back into the delegated cgroup state it owns.
- Networking had to be split by ownership. CRIU handles process and socket state, crun handles the runtime namespace and CRIU network locking path, and Podman handles the container network metadata and the interfaces it created. This is why the fixes preserve Podman's network state and clean up only interfaces Podman owns.

## Code Status

Following review feedback from Adrian, the CRIU patches were split from the original `userns-helper` branch into multiple branches.

| Branch | Scope | Shape | Status |
| --- | --- | --- | --- |
| [`wip/arcceus/memory-dump-mode`](https://github.com/arcceus/criu/tree/wip/arcceus/memory-dump-mode) | `--memory-dump-mode` alias | independent | [PR #3073](https://github.com/checkpoint-restore/criu/pull/3073) |
| [`split/rootless-dump-basics`](https://github.com/arcceus/criu/tree/split/rootless-dump-basics) | seccomp memory dump fallback | stacked on #3073 | [PR #3100](https://github.com/checkpoint-restore/criu/pull/3100) |
| [`split/rootless-mapfile-fallback`](https://github.com/arcceus/criu/tree/split/rootless-mapfile-fallback) | `/proc/<pid>/root` mapped-file fallback | independent | [PR #3112](https://github.com/checkpoint-restore/criu/pull/3112) |
| [`split/rootless-runtime-userns`](https://github.com/arcceus/criu/tree/split/rootless-runtime-userns) | restore into caller-provided userns | independent | [PR #3113](https://github.com/checkpoint-restore/criu/pull/3113) |
| [`split/rootless-netns-broker`](https://github.com/arcceus/criu/tree/split/rootless-netns-broker) | netns helper broker | stacked on runtime-userns | staged |
| [`split/rootless-net-lock-broker`](https://github.com/arcceus/criu/tree/split/rootless-net-lock-broker) | network lock/unlock broker | stacked on netns-broker | staged |
| [`split/rootless-mount-base`](https://github.com/arcceus/criu/tree/split/rootless-mount-base) | uid/gid mount option translation | independent | staged |
| [`split/rootless-mntns-broker`](https://github.com/arcceus/criu/tree/split/rootless-mntns-broker) | mount namespace broker | independent | staged |
| [`split/rootless-binfmt-tolerance`](https://github.com/arcceus/criu/tree/split/rootless-binfmt-tolerance) | explicit external `binfmt_misc` handling | independent | staged |
| [`split/rootless-cgroup-fds`](https://github.com/arcceus/criu/tree/split/rootless-cgroup-fds) | external delegated cgroup handling | independent | staged |
| [`split/rootless-uts-broker`](https://github.com/arcceus/criu/tree/split/rootless-uts-broker) | UTS and rlimit broker | independent | staged |
| [`split/rootless-seccomp-rpc`](https://github.com/arcceus/criu/tree/split/rootless-seccomp-rpc) | external seccomp BPF RPC restore | independent | staged |
| [`split/rootless-shmem-fallback`](https://github.com/arcceus/criu/tree/split/rootless-shmem-fallback) | SysV shared memory fallback | independent | staged |
| [`split/rootless-pipe-fallback`](https://github.com/arcceus/criu/tree/split/rootless-pipe-fallback) | pipe resize fallback | independent | staged |
| [`wip/arcceus/userns-helper`](https://github.com/arcceus/criu/tree/split/wip/arcceus/userns-helper) | cleaned full integration stack | aggregate | old branch |

Dependent runtime and container-engine branches:

- crun: [`wip/arcceus/rootless-cr-seccomp-flags`](https://github.com/arcceus/crun/tree/wip/arcceus/rootless-cr-seccomp-flags)
- Podman: [`wip/arcceus/rootless-cr`](https://github.com/arcceus/podman/tree/wip/arcceus/rootless-cr)

## Limitations

- Dynamically installed seccomp filters cannot be extracted rootlessly because the kernel requires `CAP_SYS_ADMIN` in `init_user_ns` for `PTRACE_SECCOMP_GET_FILTER`. OCI-configured seccomp profiles are preserved.
- Time namespace offsets are captured, but restoring `CLONE_NEWTIME` offsets rootlessly still requires unavailable host privileges.
- Established external TCP connections through pasta or slirp4netns cannot survive helper teardown; restore resets them so clients can reconnect.

## Remaining Work

- CRIU: The implementation is mostly in the split branches above. The remaining work is to upstream those branches one by one, keep rebasing them as the review progresses.
- crun: Clean up and split commits into reviewable patches, with some remaining work around restore ordering, cgroup placement, and failure handling, plus focused tests where needed.
- Podman: Clean up and split commits after the crun side settles, with some remaining work around restore ordering, networking, cgroup handling, and cleanup paths.

## Acknowledgments

Thank you to my mentors, Radostin Stoyanov and Adrian Reber, for their guidance, reviews, and architectural feedback. Thank you also to Andrei for his reviews, and to Google Summer of Code for giving me this awesome opportunity.

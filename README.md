# Ansible role for Kubernetes (kubeadm) Cluster on Arch Linux hosts
This role bootstraps a Kubernetes cluster on Arch Linux / Arch Linux ARM (see [note](#notes-on-arch-linux-arm-packages-and-multi-arch-clusters)) hosts using kubeadm.

## Design choices and limitations
* This role can set up **either** one or multiple control plane nodes. `k8s_multimaster` needs to be set to `true` for a HA cluster. You must also set it if you start with a single CP node but plan to add more in the future - this role can't convert a single CP node cluster without a VIP to a HA cluster with a VIP.
* [CRI-O](https://cri-o.io/) and [runc](https://github.com/opencontainers/runc) are used as CRI implementation and container runtime, respectively.
* Though the role is generally idempotent, on the first run or any subsequent run where nodes are added you will have to manually approve the nodes' Certificate Signing Request (CSR) via `kubectl certificate approve csr-**12345**`

## Variables

### General
|Variable|Default|Description|
|:-------|:------|:----------|
|`k8s_cluster_domain`|`cluster.local`|The **internal** domain used **inside** the Kubernetes cluster. This should **not** be your network's internal domain, or any public domain.|
|`k8s_multimaster`|`false`|Configure HA for the control plane. This setting **has** to be enabled if you want to have one or more control plane nodes|
|`k8s_apiserver_vip`|(empty)|VIP (Virtual IP) for the highly available control plane. All CP nodes need to be on the same L2 subnet|
|`k8s_apiserver_dns`|(empty)|DNS name for the control plane VIP. If not defined, VIP will be used directly.|
|`k8s_allow_swap`|`false`|Allow swap to be enabled on the nodes. Normally, kubeadm will fail if it detects swap.|
|`k8s_kernel_modules`|`br_netfilter`|List of kernel modules that should be loaded for Kubernetes. IPVS kernel modules will be loaded separately if IPVS is enabled.|
|`k8s_ipvs`|`false`|Use IPVS-based service load balancing|
|`k8s_ipvs_scheduler`|`rr`|Scheduler to use for IPVS service load balancing. [List of schedulers](#list-of-ipvs-schedulers)|
|`btrfs_top_lvl_mnt`|(empty)|Supports a BTRFS layout where root is a subvolume of the top level subvolume, which is mounted at that mountpoint. See [kernel docs](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Layout) - your layout can be flat or nested; the role will create /var/lib/containers/storage as a direct subvolume of the top level subvolume.|

#### List of IPVS schedulers
- rr: round-robin
- lc: least connection
- dh: destination hashing
- sh: source hashing
- sed: shortest expected delay
- nq: never queue

### Reset
I added the reset functionality before I had set up terraform and a LXD cluster for testing. **Use with precaution.**

|Variable|Default|Description|
|:-------|:------|:----------|
|`k8s_reset`|`false`|Execute `kubeadm reset`. **WARNING**: Deletes all Kubernetes configuration|
|`k8s_storage_reset`|`false`|Wipe the container storage location. **WARNING**: Deletes all containers and images from `/var/lib/containers/storage`.
|`k8s_reset_services_to_restart`|`iptables`|List of services to restart after kubeadm reset.|

## Note on registries.conf
The included registries.conf.j2 just sets `docker.io` as the registry for unqualified searches. I recommended [hosting your own pull through cache for Docker Hub](https://docs.docker.com/registry/recipes/mirror/) to save bandwidth and avoid hitting Docker Hub's rate limits.  
Here's an example of a registries.conf{,.j2} that specifies a single mirror for `docker.io`:
```jinja2
unqualified-search-registries = ["docker.io"]
[[registry]]
prefix="docker.io"
location="docker.io"
insecure=false

[[registry.mirror]]
location="my-docker-mirror.example.com"
insecure=false
```
If the mirror is unreachable or can't provide an image, the contaienr runtime falls back to the upstream location. See [man 5 containers-registries.conf](https://man.archlinux.org/man/containers-registries.conf.5.en) for more details.

## Notes on Arch Linux ARM packages and multi-arch clusters
### Arch Linux ARM packages
Arch Linux ARM currently doesn't provide Kubernetes packages. However, they can be built from the Arch Linux PKGBUILD (pkgbase: community/kubernetes). Upload them to your own pacman repository and configure it on the nodes, or install the packages manually.

### Multi-arch clusters
Neither the Kubernetes scheduler nor the nodes themselves check images for architecture compatibility and try to start any specified containers, leading to errors like "`Exec format error`" when starting images of a foreign architecture. Because of this, images that are scheduled to run on any node on the cluster need to be multi-arch manifests that contain images of all required CPU architectures, which is the case for official Kubernetes infrastructure images and Docker Hub's 'official' images; or be restricted to be scheduled only on compatible nodes via the `nodeSelector` term `kubernetes.io/arch`.  
**Pro tip**: Often Dockerfiles for which only amd64 images are released can be built for other architectures anyway. Recent versions of Red Hat's [buildah](https://github.com/containers/buildah) can build containers for multiple architectures on the same host with a single command. The ability to launch containers compiled for another CPU architecture is achieved via [QEMU user mode-emulation](https://qemu-project.gitlab.io/qemu/user/main.html). The qemu-***arch***-static binaries are registered with the kernel via binfmt_misc, so that they are called automatically when an executable of a foreign architecture is called.  
Buildah will neither install nor register such an interpreter, but there exists a ready-made [containerized solution](https://github.com/multiarch/qemu-user-static) for amd64 hosts. 

#### To build a multi-arch image in two steps <sub><sup>(assuming you already have podman and buildah installed)</sup></sub>:
```shell
# Register the interpreters. This requires a container will full root privileges. The registration is system-wide and valid until the next reboot.
$ sudo podman run --rm --privileged docker.io/multiarch/qemu-user-static --reset -p yes
# Build a container image for both amd64 and arm64, running 2 concurrent build stages in parallel. Podman and buildah can be configured to run rootless, in which case you don't need sudo.
$ (sudo) buildah bud --manifest $image:$tag --platform="linux/amd64,linux/arm64" --jobs=2
```

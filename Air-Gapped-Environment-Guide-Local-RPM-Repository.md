# How to build the Mellanox OFED (MOFED) driver in an air-gapped environment with a local RPM package server

This content is based in the official NVIDIA Network Operator docs regarding the air gapped environment configuration.[NVIDIA Network Operator Deployment in an Air-gapped Environment](https://docs.nvidia.com/networking/display/kubernetes2570/advanced/proxy-airgapped.html#local-package-repository)

## Introduction

There are currently two different solutions to deploy the Mellanox OFED (MOFED) driver without Internet connectivity: precompile the driver for the right kernel version and architecture in an online environment and push the container image to the offline registry; or make all the necessary rpm packages available at the air gapped environment. 

This guide explains all the required steps to mirror and expose all the source rpm packages in the air gapped environment in order for the MOFED pod to build the driver itself, with the openshift-driver-toolkit-ctr container.

## Create a systemd service running an HTTP server

Choose a local path where all the rpm binaries will be mirrored so that the HTTP server exposes it. In this example, the */home/user/rpmserver/data/*



```bash
$ cat /home/user/.config/systemd/user/container-rpm-server.service
[Unit]
Description=Podman container-rpmserver.service
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70


ExecStartPre=-/usr/bin/podman rm -f rpmserver
ExecStart=/usr/bin/podman run  \
	--cgroups=no-conmon \
	--rm \
	--sdnotify=conmon \
	--replace \
	-d --name rpmserver \
  -p 5443:8080 -v /home/user/rpmserver/data:/var/www/html:Z quay.io/centos7/httpd-24-centos7:latest
ExecStop=-/usr/bin/podman rm -f rpmserver

Type=notify
NotifyAccess=all

[Install]
WantedBy=default.target
```

Start the service, check the container is up and running and test it:


```bash
systemctl --user daemon-reload
systemctl --user enable container-rpm-server.service  --now

podman ps
8dbb9753b6bc  quay.io/centos7/httpd-24-centos7:latest  /usr/bin/run-http...  2 days ago   Up 2 days   0.0.0.0:5443->8080/tcp  rpmserver
```

```bash

curl -X GET  http://infra.server.lab:5443/tbd

TBD
```

## Mirror the required rpm packages to the air-gapped environment
From the [rhocp-4.18-for-rhel-9-aarch64-rpms repository(https://access.redhat.com/downloads/content/kernel-core/5.14.0-427.87.1.el9_4/aarch64/fd431d51/package)], download the following list of packages:

-   kernel-headers-${KERNEL_VERSION}
-   kernel-devel-${KERNEL_VERSION}
-   kernel-core-${KERNEL_VERSION}
-   createrepo
-   elfutils-libelf-devel
-   kernel-rpm-macros
-   umactl-libs
-   lsof
-   pm-build
-   patch
-   hostname

Download the cuda repository for rhel9 aarch64:

```bash
mkdir -p /home/user/rpmserver/data/cuda
cd /home/user/rpmserver/data/cuda 

wget -r -np -nH --cut-dirs=5 -R "index.html*" https://developer.download.nvidia.com/compute/cuda/repos/rhel9/aarch64/
```

Download the ubi8 and ubi9 rpms with:
```bash
mkdir -p /home/user/rpmserver/data/ubi8/baseos
cd /home/user/rpmserver/data/ubi8/baseos

wget -r -np -nH --cut-dirs=7 -R "index.html*" \
https://cdn-ubi.redhat.com/content/public/ubi/dist/ubi8/8/aarch64/baseos/os/

mkdir -p /usr/local/apache2/htdocs/rhocp/ubi8/appstream
cd /usr/local/apache2/htdocs/rhocp/ubi8/appstream

wget -r -np -nH --cut-dirs=7 -R "index.html*" \
https://cdn-ubi.redhat.com/content/public/ubi/dist/ubi8/8/aarch64/appstream/os


mkdir -p /usr/local/apache2/htdocs/rhocp/ubi9/baseos
cd /usr/local/apache2/htdocs/rhocp/ubi9/baseos

wget -r -np -nH --cut-dirs=7 -R "index.html*" \
https://cdn-ubi.redhat.com/content/public/ubi/dist/ubi9/9/aarch64/baseos/os

mkdir -p /usr/local/apache2/htdocs/rhocp/ubi9/appstream
cd /usr/local/apache2/htdocs/rhocp/ubi9/appstream

wget -r -np -nH --cut-dirs=7 -R "index.html*" \
https://cdn-ubi.redhat.com/content/public/ubi/dist/ubi9/9/aarch64/appstream/os/ 
```

Create each repo metadata file with:
```bash
createrepo_c /home/user/rpmserver/data/ubi8
createrepo_c /home/user/rpmserver/data/ubi9
createrepo_c /home/user/rpmserver/data/cuda-rhel9-aarch64
createrepo_c /home/user/rpmserver/data/rhocp-4.18-for-rhel-9-aarch64-rpms 
```

## Create a Kubernetes ConfigMap to configure the Network Operator to use the air-gapped rpm server

Prepare three local files that will be used in the ConfigMap:
ubi.repo
```bash
[ubi-8-baseos]
name = baseos 
baseurl = http://infra.server.lab:5443/ubi8/
enabled = 1
gpgcheck = 0
[ubi-8-appstream]
name = appstream 
baseurl =  http://infra.server.lab:5443/ubi8/
enabled = 1
gpgcheck = 0
[ubi-9-baseos]
name = baseos
baseurl = http://infra.server.lab:5443/ubi9/
enabled = 1
gpgcheck = 0
[ubi-9-appstream]
name = appstream
baseurl = http://infra.server.lab:5443/ubi9
enabled = 1
gpgcheck = 0 
```

cuda.repo
```bash
[cuda]
name=cuda-rhel9-aarch64
baseurl=http://infra.server.lab:5443/
gpgcheck=1
gpgkey=http://infra.server.lab:5443/cuda-rhel9-aarch64/D42D0685.pub
enabled=1 
```
redhat.repo
```bash
[rhocp-4.18-for-rhel-9-aarch64-rpms]
name=rhocp-4.18-for-rhel-9-aarch64-rpms
baseurl=http://infra.server.lab:5443/
gpgcheck=0
enabled=1 
```

Create the ConfigMap in the nvidia-network-operator with the command:
```bash
oc create configmap repo-config -n nvidia-network-operator  --from-file=cuda.repo --from-file=redhat.repo --from-file=ubi.repo 
```


## Mirror the right DOCA driver version according to your RHCOS version and architecture
Navigate to the [NVIDIA doca driver catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/mellanox/containers/doca-driver/tags?version=doca3.1.0-25.07-0.9.7.0-0-rhcos4.18-arm64) and search for your specific rhcos version and architecture. 

In this scenario, rhcos 4.18 and arm64 were used.

```bash
podman pull nvcr.io/nvidia/mellanox/doca-driver:doca3.1.0-25.07-0.9.7.0-0-rhcos4.18-arm64
```

There are many ways to copy the driver image to the air-gapped container registry, such as using skopeo, oc-mirror etc. In this example we'll simply use podman tools.

**Note**: the container image must be tag with the following data "\<driver-version>-\<rhcos-version>-\<architecture>", in this example: *25.07-0.9.7.0-rhcos4.18-arm64*
```bash
podman tag nvcr.io/nvidia/mellanox/doca-driver:doca3.1.0-25.07-0.9.7.0-0-rhcos4.18-arm64 nvcr.io/nvidia/mellanox/doca-driver:doca3.1.0-25.07-0.9.7.0-0-rhcos4.18-arm64 registry-arm.server.lab:8443/nvidia/certified-operator/nvidia/mellanox/doca-driver-container:25.07-0.9.7.0-rhcos4.18-arm64

podman push registry-arm.server.lab:8443/nvidia/certified-operator/nvidia/mellanox/doca-driver-container:25.07-0.9.7.0-rhcos4.18-arm64
```

## Configure the NicClusterPolicy CR to use the air gap rpm packages
The NIC Cluster Policy requires the ConfigMap to be injected so that the local air-gapped HTTP server is used to pull the rpm binaries; as well as the repository,version,and image must point to the air-gapped container registry.

Fill in the following data:
- repoConfig: ConfigMap name
- image: driver container image name
- repository: air-gapped container registry
- version: driver version

```bash
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  ofedDriver:
    deploy: true
    repoConfig:
      name: repo-config  
    image: doca-driver-container
    repository: registry-arm.server.lab:8443/nvidia/certified-operator/nvidia/mellanox
    version: 25.07-0.9.7.0
  rdmaSharedDevicePlugin:
    image: k8s-rdma-shared-dev-plugin
    repository: ghcr.io/mellanox
    version: 1eb5ab7f4b6f0cf89c954cf7ac8eb2e6fb2290a4f5f57417136a5302f20b12c8
    config: |
      {
        "configList": [
          {
            "resourceName": "rdmashared",
            "rdmaHcaMax": 1000,
            "selectors": {
              "ifNames": ["enp1s0f1np1"]
            }
          }
        ]
      } 
```
## Results

Instead of the precompiled driver path, where the MOFED pod only spinned up a single container running the already compiled DOCA driver; in this secenario the MOFED pod spins up a openshift-driver-toolkit-ctr container that compiles the raw mirrored driver at registry-arm.server.lab:8443/nvidia/certified-operator/nvidia/mellanox/doca-driver-container:25.07-0.9.7.0-rhcos4.18-arm64 

The workflow logs is:


- The MOFED container waits for the driver to be built

```bash
oc logs -f mofed-rhcos4.18-6f455775d5-ds-djxv9
Defaulted container "mofed-container" out of: mofed-container, openshift-driver-toolkit-ctr, network-operator-init-container (init)
[23-Oct-25_15:28:32] NVIDIA driver container exec start
ID="rhcos"
VERSION_ID="4.18"
RHEL_VERSION=9.4
[23-Oct-25_15:28:32] Container full version: 25.07-0.9.7.0-0
[23-Oct-25_15:28:32] Verifying loaded modules will not prevent future driver restart
[23-Oct-25_15:28:32] Executing driver sources container
[23-Oct-25_15:28:32] Drivers inventory path is set: /mnt/drivers-inventory
[23-Oct-25_15:28:32] Unsetting driver ready state[23-Oct-25_15:28:32] Query VFs info from [4] devices
[23-Oct-25_15:28:32] Query representors info from [4] devices[23-Oct-25_15:28:32] Copy required files to shared dir with OCP DTK[23-Oct-25_15:28:32] Awaiting openshift driver toolkit to complete NIC driver build, next query in 300 sec 
```

- The *openshift-driver-toolkit-ctr* starts to build the driver

```bash
Build kernel-mft 4.33.0 RPM
Running  rpmbuild --rebuild  --define '_topdir /var/tmp/OFED_topdir' --define '_sourcedir %{_topdir}/SOURCES' -
....... 
...........
[23-Oct-25_15:30:56] Executing command: touch /mnt/shared-doca-driver-toolkit/dtk_done_compile_25_07_0_9_7_0[23-Oct-25_15:30:56] Executing command: rm /mnt/shared-doca-driver-toolkit/dtk_start_compile[23-Oct-25_15:30:56] DTK driver build script end

```

When the driver is compiled, the mofed continues:

```bash
Verifying...                          ########################################
Preparing...                          ########################################
Updating / installing...
kmod-xpmem-2.7.4-1.2507097.rhel9u4.rhe########################################
kmod-mlnx-ofa_kernel-25.07-OFED.25.07.########################################
kmod-kernel-mft-mlnx-4.33.0-1.rhel9u4 ########################################
....................
....................
[23-Oct-25_15:33:34] Current mlx5_core driver version: 25.07-0.9.7
[23-Oct-25_15:33:34] Mounting Mellanox OFED driver container shared kernel headers
[23-Oct-25_15:33:34] Setting driver ready state
[23-Oct-25_15:33:34] NVIDIA driver container exec end, sleeping
```

Finally, all pods are up and running successfully:
```bash
$ oc get pods -n nvidia-network-operator
NAME                                                          READY   STATUS    RESTARTS   AGE
mofed-rhcos4.18-6f455775d5-ds-djxv9                           2/2     Running   0          42m
nvidia-network-operator-controller-manager-6cb7758456-sdd9b   1/1     Running   0          50m
rdma-shared-dp-ds-mk7pj                                       1/1     Running   0          37m

$ oc get pods -n nvidia-gpu-operator
NAME                                                  READY   STATUS      RESTARTS   AGE
gpu-feature-discovery-pk2jt                           1/1     Running     0          37m
gpu-operator-7746c56fcd-4ws9n                         1/1     Running     0          6d2h
nvidia-container-toolkit-daemonset-g8zlr              1/1     Running     0          37m
nvidia-cuda-validator-h2kpf                           0/1     Completed   0          36m
nvidia-dcgm-9jczz                                     1/1     Running     0          37m
nvidia-dcgm-exporter-8jcfv                            1/1     Running     0          37m
nvidia-device-plugin-daemonset-pcrkf                  1/1     Running     0          37m
nvidia-driver-daemonset-418.94.202509100653-0-gk488   4/4     Running     0          98m
nvidia-mig-manager-fnb8m                              1/1     Running     0          37m
nvidia-node-status-exporter-6mvjv                     1/1     Running     0          98m
nvidia-operator-validator-zxlmk                       1/1     Running     0          37m 


```
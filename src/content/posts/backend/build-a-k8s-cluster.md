---
title: Kubernetes 实战：搭建一个三台服务器组成的 k8s 集群
published: 2025-08-06
description: 从零开始构建三节点异地 Kubernetes 集群的实用教程，涵盖网络配置、安全设置与系统调优等，适合初学者和实战演练者参考。
image: https://d988089.webp.li/2025/08/06/20250806234131080.avif
tags: ["后端", "分布式", "Kubernetes", "Containerd", "Rancher", "实战", "教程"]
category: 后端
draft: false
---

> 从零开始构建三节点异地 Kubernetes 集群的实用教程，涵盖网络配置、安全设置与系统调优等，适合初学者和实战演练者参考。

:::note[关于服务器信息]

所有服务器均为 ECS 实例，分别位于境内、香港以及美国，由于服务器带宽足够，便于异地组建集群，实际生产环境建议使用同地域不同可用区服务器通过内网进行通信。  
本文省略了所有的镜像源配置过程，若网络环境不允许，请自行配置镜像源。  
_文中出现的 IP 并非真实 IP_。

:::

## 一、服务器配置说明

| 节点        | IP      | 操作系统     | 配置                 | 说明   |
| ----------- | ------- | ------------ | -------------------- | ------ |
| k8s-master  | 1.1.1.1 | Ubuntu 22.04 | 16H 16G 40+100G 硬盘 | 主节点 |
| k8s-worker1 | 2.2.2.2 | Ubuntu 22.04 | 16H 16G 40G 硬盘     | 子节点 |
| k8s-worker2 | 3.3.3.3 | Ubuntu 24.04 | 8H 8G 40G 硬盘       | 子节点 |

:::important[有关配置操作]

如果没有特殊说明，三台服务器都需要进行相同的操作，本人使用的是 Termius 的 Broadcast Input 实现三台服务器同时执行命令。  
大部分操作均为 `sudo` 操作，如遇到权限问题，请自行在命令前添加 `sudo`。

:::

## 二、配置服务器

### 1. 配置主机名

在三台服务器上分别执行

```bash
# 1.1.1.1
hostnamectl set-hostname k8s-master

# 2.2.2.2
hostnamectl set-hostname k8s-worker1

# 3.3.3.3
hostnamectl set-hostname k8s-worker2
```

### 2. 配置 hosts 解析

1. **编辑 `/etc/hosts` 文件**

```bash
nano /etc/hosts
```

2. **添加如下内容：**

```
# Kubernetes Cluster Nodes
1.1.1.1    k8s-master
2.2.2.2    k8s-worker1
3.3.3.3    k8s-worker2
```

3. **测试连通性**

```bash
ping k8s-master
ping k8s-worker1
ping k8s-worker2
```

如果能 `ping` 通，说明配置正确。

:::tip[为什么这样做很重要？]

- **简化配置**：当您使用 `kubeadm` 命令初始化集群时，它会使用主机名来生成证书和配置。提前配置好 `/etc/hosts` 可以确保所有组件都能够正确地解析和通信。
- **提高稳定性**：即使 DNS 解析出现问题，或者跨国 DNS 延迟较高，您的集群仍然可以通过 `hosts` 文件中的静态映射进行通信，从而减少因网络问题导致的集群不稳定。
- **便于管理**：使用主机名（如 `k8s-master`）来管理集群比使用难以记忆的 IP 地址更加方便和直观。

:::

### 3. 配置免密登录（可选）

:::tip

**如果后续计划进行自动化管理或维护**，强烈建议执行这一步。它能为你的后续工作节省大量时间和精力。

:::

**在主节点上操作**：

1. **生成密钥对**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/k8s -N '' -q
```

2. **将公钥复制到其他节点**

```bash
ssh-copy-id -i ~/.ssh/k8s.pub mengchen@k8s-worker1
ssh-copy-id -i ~/.ssh/k8s.pub mengchen@k8s-worker2
```

3. **测试连接**

```bash
ssh -i ~/.ssh/k8s mengchen@k8s-worker1
ssh -i ~/.ssh/k8s mengchen@k8s-worker2
```

如果能成功连接到工作节点，说明配置成功。

:::note

如果你遇到了权限问题，可以尝试 `cat k8s.pub` ，手动将公钥复制到工作节点服务器的 `authorized_keys` 文件中。

:::

### 4. 设置时区（可选）

```bash
sudo timedatectl set-timezone Asia/Shanghai
```

### 5. 配置内核转发及网桥过滤

1. **内核转发**

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

2. **网桥过滤**

```bash
#加载br_netfilter模块
modprobe br_netfilter

#查看是否加载
lsmod | grep br_netfilter
br_netfilter           32768  0
bridge                311296  1 br_netfilter

#使配置文件生效
sysctl -p /etc/sysctl.d/k8s.conf
```

:::tip

- **内核转发** 确保了 Pod 之间的**跨节点通信**。
- **网桥过滤** 确保了 **Service 负载均衡** 的 `iptables` 规则能够正常工作。

:::

### 6. 关闭 Swap

```bash
# 临时关闭
swapoff -a

#永远关闭swap分区
sed -i 's/.*swap.*/#&/' /etc/fstab
```

:::tip[为什么 Kubernetes 需要禁用 Swap？]

- **性能**：Swap 将内存中的数据移动到磁盘上，而磁盘的读写速度远低于内存。这会导致 Pod 的性能显著下降，尤其是在内存资源紧张时。
- **资源管理**：Kubernetes 的调度器和资源管理依赖于对物理内存的准确报告。如果启用了 Swap，系统会报告一个混合了物理内存和虚拟内存的模糊值，这会影响调度器的决策，导致 Pod 被调度到不适合的节点上。
- **稳定性**：内存不足时，Kubernetes 更倾向于直接终止 Pod 以释放资源，而不是将数据交换到 Swap 空间。这样做可以保证集群的稳定性，避免因频繁的磁盘 I/O 导致整个节点响应变慢。

:::

### 7. 启用 IPVS

1. **加载 IPVS 和 `br_netfilter` 模块**

```bash
cat <<EOF | sudo tee /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
br_netfilter
EOF
```

2. **立即加载模块**

```bash
modprobe --all ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack br_netfilter
```

3. **安装必要的工具**

```bash
sudo apt-get update && sudo apt-get install -y ipvsadm ipset
```

4. **验证模块是否已加载**

```bash
lsmod | grep -e ip_vs -e nf_conntrack -e br_netfilter
```

只要输出中包含了 `ip_vs`、`nf_conntrack` 和 `br_netfilter` 相关的行，就说明配置成功了。

:::tip[为什么推荐使用 IPVS？]

1. **性能更佳**：`ipvs` 模式在处理大量的 Service 和 Pod 时，性能远超 `iptables`。它的流量转发效率更高，对 CPU 资源的消耗更少。
2. **可扩展性更好**：`ipvs` 能够轻松应对 Service 和 Endpoint 数量的增长，非常适合未来的扩展。
3. **更智能的负载均衡算法**：`ipvs` 支持多种负载均衡算法，如轮询、最少连接、源地址哈希等，这让您可以更精细地控制流量分发。

:::

### 8. 修改句柄数限制

1. **临时修改句柄数**

```bash
ulimit -SHn 65535
```

2. **永久修改句柄数**

```bash
cat >> /etc/security/limits.conf <<EOF
* soft nofile 65536
* hard nofile 131072
* soft nproc 65536
* hard nproc 65536
* soft memlock unlimited
* hard memlock unlimited
EOF
```

3. **查看修改结果**

```bash
ulimit -a
```

### 9. 系统参数调优与模块加载

为了确保 Kubernetes 集群在处理高并发网络连接和大规模文件操作时能够稳定高效运行，我们需要对系统内核参数进行更精细的调优，并确保必要的内核模块已加载。

1. **配置内核参数**

这一步将对系统内核进行优化，以更好地适应 Kubernetes 的运行环境。执行以下命令，将配置写入到 `/etc/sysctl.d/k8s_better.conf` 文件中。

```bash
sudo tee /etc/sysctl.d/k8s_better.conf << EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
```

:::tip[为什么要添加这些配置？]

- **`vm.swappiness=0`**: 确保系统几乎不使用 **Swap**，与我们之前禁用的 Swap 保持一致。
- **`vm.overcommit_memory=1`**: 允许内核分配比当前可用内存更多的内存，有助于某些应用在启动时快速分配资源。
- **`net.netfilter.nf_conntrack_max`**: 调高网络连接跟踪的最大数量，防止在高并发时因连接数过多导致网络问题。
- **`fs.file-max`**: 调高系统级别的最大文件句柄数，避免“Too many open files”错误，特别是在大量 Pod 运行的集群中至关重要。

:::

2. **加载内核模块与使配置生效**

```bash
# 加载 br_netfilter 模块，确保网桥过滤功能正常
modprobe br_netfilter

# 确认 conntrack 模块是否已加载
lsmod | grep conntrack

# 如果上一步没有输出，则加载 ip_conntrack 模块
modprobe ip_conntrack

# 使新的内核参数配置立即生效
sysctl -p /etc/sysctl.d/k8s_better.conf
```

:::note[关于 conntrack 模块]

`lsmod | grep conntrack` 命令用于检查内核是否已加载网络连接跟踪模块。在较新的系统中，`ip_conntrack` 已被 `nf_conntrack` 替代，但为了兼容性，如果发现未加载，可以手动加载。在绝大多数情况下，这个模块通常是默认加载的。

:::

完成以上步骤后，你的服务器已经为运行 Kubernetes 做好了更充分的准备。

## 三、容器运行时工具安装及配置

目前，Kubernetes 社区推荐使用符合 CRI 标准的 **Containerd** 作为容器运行时（而不是 Docker），它更加轻量、高效。

### 安装 Containerd

```bash
sudo apt-get update
sudo apt-get install -y containerd
```

### 配置 Containerd

配置 Containerd，使其使用 `systemd` 作为 cgroup 驱动。

1. **生成默认配置文件**

```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

2. **使用 sed 命令替换 cgroup 驱动为 systemd**

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

3. **重启 Containerd 服务**

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## 四、Kubernetes  安装

> 以下内容来自 [安装 kubeadm | Kubernetes](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

本节将安装 Kubernetes 的核心组件 `kubelet`、`kubeadm` 和 `kubectl`。

1. **更新 `apt` 软件包索引并安装使用 Kubernetes `apt` 仓库所需的软件包：**

```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

2. **下载 Kubernetes 软件包仓库的公钥签名密钥。相同的签名密钥用于所有仓库，因此您可以忽略 URL 中的版本：**

```bash
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

3. **添加适当的 Kubernetes `apt` 仓库:**

```bash
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

:::warning

请注意，此仓库仅包含 Kubernetes 1.33 的软件包；对于其他 Kubernetes 小版本，您需要更改 URL 中的 Kubernetes 小版本以匹配您想要的版本。

:::

4. **更新 `apt` 软件包索引，安装 kubelet、kubeadm 和 kubectl，并固定它们的版本：**

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

5. **在运行 kubeadm 之前启用 kubelet 服务：**

```bash
sudo systemctl enable --now kubelet
```

## 五、K8S 集群初始化

:::important

此操作仅在主节点上进行

:::

### 1. **创建默认配置文件**

```bash
sudo kubeadm config print init-defaults > kubeadm-init.yaml
```

:::note

直接在 `~/` 目录下并不是一个**标准或推荐**的做法，为了方便编辑，现在只是临时生成，之后会移动到 `/etc/kubernetes/` 目录下

:::

### 2. **修改配置文件**

1. **打开文件进行编辑：**

```bash
sudo nano kubeadm-init.yaml
```

2. **找到并修改以下关键配置项：**

   - `localAPIEndpoint.advertiseAddress`:

     这个参数告诉其他节点如何连接到主节点的 API Server。请将它修改为您的 **主节点（`k8s-master`）的 IP 地址**。

     ```yaml
     localAPIEndpoint:
       advertiseAddress: 1.1.1.1 # 将此 IP 替换为您的主节点公网 IP
       bindPort: 6443
     ```

   - **`nodeRegistration.criSocket`**:

     这个参数指定了 `kubelet` 连接容器运行时的 Socket 文件路径。由于我们使用的是 **Containerd**，其默认路径就是 `/var/run/containerd/containerd.sock`。检查确保这一行没有被修改。

     ```yaml
     nodeRegistration:
       criSocket: unix:///var/run/containerd/containerd.sock
     ```

   - **`kubernetesVersion`**:

     这个参数指定了要安装的 Kubernetes 版本。检查并确保它与您在第四节安装的 `kubelet`、`kubeadm` 和 `kubectl` 版本一致。

     ```yaml
     kubernetesVersion: 1.33.0 # 确保与安装的版本号一致
     ```

   - **`KubeProxyConfiguration.mode`**:

     这个参数用于设置 `kube-proxy` 的工作模式。为了获得更好的性能和可扩展性，我们将其设置为 `ipvs`。在 `kubeadm-init.yaml` 文件的末尾追加以下内容：

     ```yaml
     ---
     apiVersion: kubeproxy.config.k8s.io/v1alpha1
     kind: KubeProxyConfiguration
     mode: ipvs
     ```

修改完毕后，保存文件并退出。

3. **将配置文件移动到 `/etc/kubernetes/` 目录下：**

```bash
sudo mv kubeadm-init.yaml /etc/kubernetes/
```

### 3. **执行初始化命令**

```bash
cd /etc/kubernetes/
sudo kubeadm init --config=kubeadm-init.yaml --upload-certs --v=6
```

- `--config` 参数指向我们刚刚配置的文件。

- `--upload-certs` 参数用于将集群证书上传到 etcd 中，方便后续工作节点加入时自动下载和配置。

:::important

如果初始化成功，终端会输出一段成功的提示信息，其中包含了后续用于将工作节点加入集群的 `kubeadm join` 命令。

一般情况下，`kubeadm` 生成的默认 token `abcdef.0123456789abcdef` **需要删除掉**以确保安全。

```bash
sudo kubeadm token delete abcdef.0123456789abcdef
```

接下来创建一个新的 token:

```bash
sudo kubeadm token create --print-join-command
```

请务必将输出的命令**复制并妥善保存**。

:::

:::note

在进行这一步时我遇到了 `kube-apiserver` 无法成功启动的问题，日志类似

```
[control-plane-check] kube-apiserver is not healthy after 4m0.000278203s
kube-apiserver check failed at https://1.1.1.1:6443/livez: client rate limiter Wait returned an error: context deadline exceeded - error from a previous attempt: EOF
```

经过排查后，AI 给出了两种可能：

- **容器网络命名空间**：`etcd` 容器在自己的网络命名空间中运行。即使主机可以访问这个公网 IP，容器内部的网络环境可能无法直接绑定到这个 IP。这在没有正确配置 `--advertise-address` 的情况下非常常见。

- **IP 地址不可用**：服务器可能正在使用一个私有 IP 地址作为其主要网络接口，而公网 IP 是通过 NAT（网络地址转换）映射过来的。在这种情况下，`etcd` 容器无法直接绑定到公网 IP，因为它不存在于服务器的任何网络接口上。

经过尝试之后，最终采用以下方案解决了这个问题：

1. **获取私有 IP**

```bash
ip a
```

2. **修改 `kubeadm-init.yaml` 配置**

```yaml
localAPIEndpoint:
  advertiseAddress: 10.1.10.1 # 私有 IP
  bindPort: 6443
controlPlaneEndpoint: 1.1.1.1:6443 # 将公网 IP 作为 API Server 的外部访问地址
```

3. **删除缓存**

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/etcd
```

4. **重新初始化**

```bash
sudo kubeadm init --config=kubeadm-init.yaml --upload-certs --v=6
```

:::

### 4. **配置 `kubectl`**

执行以下命令，以便普通用户可以使用 `kubectl` 命令管理集群：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5. **部署网络插件**

:::important

这一步至关重要，**在安装 Rancher 之前必须完成**。在主节点上部署 Calico 或其他 CNI 插件。

:::

1. **安装并部署 Tigera Operator**

   使用 `kubectl create` 命令部署 Tigera Operator。

   ```bash
   kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml
   ```

   等待 `tigera-operator` Pod 启动成功。你可以通过以下命令查看状态：

   ```bash
   kubectl get pod -n tigera-operator
   ```

2. **创建 `Installation` 资源**

   Tigera Operator 部署成功后，你需要通过创建一个名为 `Installation` 的自定义资源来告诉它如何安装 Calico。

   创建一个名为 `custom-resources.yaml` 的文件，并添加以下内容：

   ```yaml
   # custom-resources.yaml
   apiVersion: operator.tigera.io/v1
   kind: Installation
   metadata:
     name: default
   spec:
     calicoNetwork:
       ipPools:
       - cidr: 10.244.0.0/16  # 将此 CIDR 替换为你的 Pod IP 段
         encapsulation: IPIP
         natOutgoing: Enabled
         nodeSelector: all()
   ```

   - `ipPools.cidr`: 这个参数定义了 Pod 的 IP 地址段，你需要将其替换为你的实际网络配置。如果你的 `kubeadm-init.yaml` 中配置了 `networking.podSubnet`，请确保这里的值与之一致。

   - `encapsulation: IPIP`: 这是 Calico 的默认封装模式，适用于大多数环境。

   :::tip[为什么不使用官方完整版清单？]

   **使用官方完整版清单的优缺点：**

   **优点：**

   - **完整性**：你获得了 Calico 提供的所有可用资源的定义，这能让你对整个生态系统有更全面的了解。
   - **未来可扩展性**：如果将来你决定升级到 Tigera 的商业产品，这些资源已经存在，可以简化迁移过程。

   **缺点：**

   - **增加了复杂性**：部署了你可能永远用不到的资源，比如 Goldmane 和 Whisker。这会让你的集群中多出一些不必要的 Kubernetes 对象。
   - **可能引起混乱**：当你查看 `calico-system` 命名空间下的资源时，会看到一些你并不了解的组件，这可能会让初学者感到困惑。
   - **日志和事件噪音**：虽然这些组件可能不会正常工作，但它们可能会在日志或事件中产生一些错误或警告信息，增加了排查问题的难度。

   对于三节点 Kubernetes 集群，最推荐的方案是使用我提供的**简化版 `custom-resources.yaml`**。它只部署了最核心、最必要的功能，确保你的集群网络可以正常运行，同时保持了系统的简洁和高效。

   如果你想尝试用官方完整版，也完全没问题。在大多数情况下，它不会对集群的稳定运行产生负面影响。不过，请记住，如果遇到与 Calico 相关的问题，排查时可以先忽略 Goldmane 和 Whisker 等非核心组件。

   :::

3. **部署 `Installation` 资源**

   ```bash
   kubectl create -f custom-resources.yaml
   ```

   这步完成后，Calico 的部署过程已经开始。

4. **验证部署**

   为了观察部署进度，你可以使用 `watch` 命令实时查看 `calico-system` 命名空间下的 Pod 状态。当所有 Pod 都变为 **`Running`** 状态时，Calico 网络就部署成功了。

   ```bash
   # 实时查看 Calico Pod 状态
   watch kubectl get pods -n calico-system
   
   # 成功后，按 Ctrl+C 退出
   ```

   此时，你之前处于 `NotReady` 状态的节点也应该会变为 `Ready`。你可以通过 `kubectl get nodes` 命令进行验证。

:::tip

在 Kubernetes 集群中，Pod 默认是无法跨节点通信的。`kubeadm` 在初始化集群后，只会搭建集群的控制平面，但不会自动配置 Pod 的网络。如果没有网络插件，你的集群虽然看起来已经启动，但所有 Pod（包括核心组件 `CoreDNS`）都会处于 `Pending` 或 `ContainerCreating` 状态，因为它们无法互相通信。

所以，部署 Calico 的目的就是：

- **为 Pod 提供网络**：让集群中的 Pod 能够互相访问，无论是同一个节点还是不同节点上的 Pod。这是集群正常工作的基础。
- **实现网络策略**：Calico 提供了强大的网络策略功能，允许你定义 Pod 之间的通信规则，从而增强集群的安全性。

:::

## 六、Rancher Server 安装

:::important

此操作仅在主节点上进行

:::

1. **添加 Helm Chart 仓库**

   我们使用 Helm，一个 Kubernetes 的包管理工具，来部署 Rancher。

   ```bash
   helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
   helm repo update
   ```

2. **安装 `cert-manager`**

   Rancher 需要 `cert-manager` 来颁发和管理证书：

   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml
   ```

   等待 `cert-manager` 的所有 Pod 启动成功：

   ```bash
   kubectl -n cert-manager get pod
   ```

3. **安装 Rancher Server**

   使用 Helm 安装 Rancher，并配置一个主机名：

   ```bash
   helm install rancher rancher-stable/rancher \
     --namespace cattle-system \
     --create-namespace \
     --set hostname=rancher.your-domain.com \
     --set ingress.tls.source=cert-manager \
     --set replicas=1
   ```

   - 将 `rancher.your-domain.com` 替换为你自己的域名。
   - `replicas=1` 表示部署一个单实例 Rancher，适合测试和实验环境。如果你需要高可用，可以改为 3，但需要一个负载均衡器。

4. **访问 Rancher UI** 等待 Rancher Pod 启动成功后，你可以通过你设置的域名访问 Rancher UI。

## 七、工作节点加入

:::important

此操作仅在工作节点上进行

:::

```bash
sudo kubeadm join 1.1.1.1:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:06faf8d64c03530bf88d1a34eae877b3d446ab5e4f0e071fc96567ccf53b1e70
```

完成这些步骤后，你的三节点集群就会被 Rancher 成功管理。你可以在 Rancher 的 UI 中看到你的集群状态，并开始通过它来管理应用和工作负载。

## 参考资料

- [安装 kubeadm | Kubernetes](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

- [Tutorial: Install Calico on single-host k8s cluster | Calico Documentation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/k8s-single-node)

- [安装 SUSE® Rancher Prime :: Rancher product documentation](https://documentation.suse.com/cloudnative/rancher-manager/latest/zh/installation-and-upgrade/installation-and-upgrade.html)

- [超详细kubernetes部署k8s----一台master和两台node_部署 kubernetes master-CSDN博客](https://blog.csdn.net/2301_81851270/article/details/146286268)
- [k8s入门到实战（三）—— 三台服务器从零开始搭建k8s集群-CSDN博客](https://blog.csdn.net/weixin_43980547/article/details/137024280)
- [使用Rancher快速部署K8S集群_rancher集群部署-CSDN博客](https://blog.csdn.net/XiaoWang0777/article/details/139806264)

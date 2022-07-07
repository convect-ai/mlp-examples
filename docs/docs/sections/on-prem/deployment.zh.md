# 私有化部署手册

## 简介

本篇指南帮助你了解如何在若干私有化服务器上部署一个ConvectHub服务。

部署过程中有三种类型的机器：

1. `Provisioner` -- 该类机器不是一个计算节点，而是管理员使用的控制和触发部署过程的机器。
2. `Master` 节点 -- 该类机器用来承载集群中和计算无关，但是重要的服务，例如网络、存储、驱动等服务。如果激活HA，那可能会有多个`Master`节点，默认只有一台。
3. `Worker` 节点 -- 这类机器主要用来承载计算任务，例如用户notebook实例、机器学习的训练任务等。


## 前置条件 

### 硬件

对于`Master`和`Worker`节点，最小硬件需求，2核CPU, 4GB内存, 10GB存储空间。

对于`Provisioner`，最小硬件需求1核CPU, 4GB内存, 25GB存储空间。


### 操作系统

我们目前支持Ubuntu (>=16.04) 作为 `Master` 和 `Worker`节点的底座操作系统. CentOS的支持正在开发中.

在任一节点上，我们需要APT可以正常运行，用于安装系统包和软件。例如如下代码可以正常执行
```sh
apt update
apt install <LIB_NAME>
```
如果节点有公共网络访问权限，则以上步骤无需任何配置。
如果节点没有网络访问权限，则需要配置 `/etc/apt/sources.list` 文件指向私有的APT镜像，例如[清华的镜像](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)。

对`Provisioner`的机器，我们强烈建议使用Linux操作系统。如果使用的是windows系统，我们推荐使用[WSL](https://docs.microsoft.com/en-us/windows/wsl/about)。

### 网络 

我们需要私有网络中的任意两个 `Master` 和 `Worker` 节点可以互相访问如下的端口

| 协议	| 端口      |	源                              |	解释                                           |
------------|-----------|-----------------------                |-------------                                            |
| TCP	    | 6443	    |   K3s agent nodes	                    | Kubernetes API Server                                   |
| UDP	    | 8472	    |   K3s server and agent nodes	        | Required only for Flannel VXLAN                         |
| TCP	    | 10250	    |   K3s server and agent nodes	        | Kubelet metrics                                         |
| TCP	    | 2379-2380	|   K3s server nodes	                | Required only for HA with embedded etcd                 |
| TCP       | 80        |   ConvectHub                          | Required to visit ConvectHub via http. Can be configurable  |
| TCP       | 443       |   ConvectHub                          | Required to visit ConvectHub via https. Can be configurable |
| TCP       | 22        |   `Provisioner`                       | Required by `Provisioner` to visit `Master` and `Worker` via SSH |


`Master` 和 `Worker` 节点可以没有公有网络访问权限，但是`Provisioner`需要可以访问因特网。
我们也需要`Provisioner`连接到私有网络上，可以访问私有网络中的 `Master` 和 `Worker` 节点。

### 依赖包和软件

我们需要如下的软件和包在 `Provisioner` 已经被安装

| MACHINE       |   Software/lib name       |  Installation guide       |
|---------------|---------------------------|---------------------------|
| `Provisioner` | Docker                    | [Official guide](https://docs.docker.com/get-docker/) |
| `Provisioner` | python3, python3-pip, python3-venv                   | [Official guide](https://www.python.org/downloads/) |


## 部署过程

如下的步骤如未指明，则全部假设在`Provisioner`上面执行。

### 检查网络连接

确保`Provisioner`可以访问因特网

```sh
ping -c 3 convect.ai
```

### 克隆部署脚本

TODO: need a better way to distribute it
```sh
$ git clone https://github.com/convect-ai/mlp.git

$ cd mlp/baremetal/airgap-k3s
```

### 下载二进制文件

TODO: need a better way to distribute it
```sh
$ aws s3 sync s3://convect-mlp-assets/bin/ shared/bin/
```
下载过程因网络连接速度可能只需几分钟。

### 设置SSH用户和密钥

首先确保可以从`Provisioner`上SSH到 `Master` 和`Worker` 节点。 
我们推荐在所有节点上设置统一的SSH用户名称，和免密登录，便于做自动化 (详情参考如下[指南](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server))。

我们也推荐对SSH用户，设置免密sudo权限。你可以创建一个新的文件来达成这一点
```sh
echo "$USER ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/$USER
```

### 安装 `ansible`

我们使用[`ansible`](https://www.ansible.com/)来自动化部署过程。它是一个纯python包，如需安装，

```sh
# create a virtualenv 
$ python3 -m venv .venv 
$ source activate .venv/bin/activate

# upgrade pip if needed 
$ pip install --upgrade pip 

# install ansible
$ pip install ansible

# make sure the installation is successful
$ ansible --version
```

### 准备inventory文件

Inventory文件是对节点fleet的一个声明，包括每台服务器的名称、地址、如何登录以及他们在最后的集群运用中的角色。

如下是一个样例inventory文件

```
worker2 ansible_ssh_user=vagrant ansible_ssh_host=127.0.0.1 ansible_ssh_port=2201 ansible_ssh_private_key_file='~/.ssh/id_rsa' my_iface=eth1 has_gpu=true
worker1 ansible_ssh_user=vagrant ansible_ssh_host=127.0.0.1 ansible_ssh_port=2200 ansible_ssh_private_key_file='~/.ssh/id_rsa' my_iface=eth1
master ansible_ssh_user=vagrant ansible_ssh_host=127.0.0.1 ansible_ssh_port=2222 ansible_ssh_private_key_file='~/.ssh/id_rsa' my_iface=eth1

[driver]
master

[worker]
worker1
worker2

[registry]
master

[all:vars]
convect_workdir=/tmp/convecthub
convect_token=k3s_hub_token
has_gpu=false
jhub_config_path=thu-vars/jupyterhub.yaml
```
以第一行为例
```
worker2 ansible_ssh_user=vagrant ansible_ssh_host=127.0.0.1 ansible_ssh_port=2201 ansible_ssh_private_key_file='~/.ssh/id_rsa' my_iface=eth1 has_gpu=true
```
声明了一台叫做`worker2`的服务器。 它用如下的配置

- `ansible_ssh_user` - 用来SSH到这台服务器的用户名称
- `ansible_ssh_host` - 服务器的IP或者Host地址
- `ansible_ssh_port` - 用来SSH的端口号
- `ansible_ssh_private_key_file` - 用来SSH的私钥地址（免密登录）
- `my_iface` - 服务器用来连接私有网络的网络设备名称（Network Interface）。 你可以在服务器上运行 `ip a show` 来查看所有的网络设备和它们的绑定IP地址，来找到对应的设备名称
- `has_gpu` - 用来指明该台服务器是否装备GPU。 默认值 = `false`

```
[worker]
worker1
worker2
```
在声明每台机器的名称、地址和属性之后，我们声明服务器在集群服务中的角色。例如如上声明了`worker1` 和 `worker2` 都属于 `worker` group（用来运行计算任务），如果你有更多的计算节点，可以在这里添加。

对于 `driver` group, 我们一般只需要一台服务器。（如果激活HA，则需要多态）

```
[all:vars]
convect_workdir=/tmp/convecthub
convect_token=k3s_hub_token
has_gpu=false
jhub_config_path=thu-vars/jupyterhub.yaml
```
最后的部分声明了一些公共变量，一般情况下我们不需要更改它们。

综上，如需按照你所处的环境准备一个inventory文件，需要如下的步骤

1. 记录你打算用作集群服务的每一个服务器的IP/Host地址，和它们的SSH端口号。
2. 在每台服务器上，建立一个SSH用户，并且保证可以使用`Provisioner`的私钥免密登录。
3. 如果服务器上有超过一个网络设备，则通过`ip a show`找到连接到私有网路的那个设备名称，设置`my_iface`指向那个设备
4. 对每台服务器，在inventory文件里面添加如下的声明
    ```
    worker2 ansible_ssh_user=vagrant ansible_ssh_host=127.0.0.1 ansible_ssh_port=2201 ansible_ssh_private_key_file='~/.ssh/id_rsa' my_iface=eth1 has_gpu=true
    ```
    其中`ansible_ssh_user`, `ansible_ssh_host`, `ansible_ssh_port`, `ansible_ssh_private_key_file`, `my_iface`, `has_gpu` 按照服务器的实际情况替换成正确的值。
5. 对每台`Worker`节点，添加到`Worker`群组中。

### 测试运行

当你定义好inventory文件之后，假设名字是 `inv.toml`, 我们可以通过云霞如下代码来确保`Provisioner`到各个节点的连通性
```sh
$ ansible all -i inv.toml -m ping 
```
我们应该看到每个节点都返回OK的状态。如果连接失败则登录具体的服务器排查设置是否准确。

### 开始部署

如果我们的SSH用户拥有免密sudo的权限，则运行

```sh
ansible-playbook -i inv.toml playbook.yml
```
否则 
```sh
ansible-playbook -i inv.toml playbook.yml --extra-vars ansible_sudo_pass=YOUR_PASSWORD
```
其中 `YOUR_PASSWORD` 是在远程节点上执行 `sudo`所需要的密码.

部署过程时长一般需要30分钟，取决于节点数量和网络速度。

## 确认ConvectHub正常运作
在部署完成之后, 在 `Provisioner`, 打开浏览器访问 `https://<MASTER_IP>`, 其中 `<MASTER_IP>` 是 `Master` 节点在私有网络中的IP地址，确认你可以看到一个登录界面。

## 配置部署过程

### Nvidia 驱动

如果你的节点已经预先安装了nvidia驱动，则在inventory文件的服务器声明中，设置 `install_nvidia_driver=false` 从而跳过驱动安装。

### Docker

如果你的节点已经预先安装了docker，则在inventory文件的服务器声明中，设置 `install_docker=false` 来跳过安装docker。

如果你的节点无法访问docker镜像，则设置 `docker_repo` 指向私有的docker API镜像，例如 `https://download.docker.com/linux/ubuntu`

### 改变ConvectHub访问端口

ConvectHub默认接受80和443接口的http和https流量。如果你需要改变默认的http端口，在inventory文件中设置 `http_port=<YOUR_PORT_NUMBER>` 以及 `https_port=<YOUR_PORT_NUMBER>` 在 `[all:vars]` 部分。例如

```
[all:vars]
convect_workdir=/tmp/convecthub
convect_token=k3s_hub_token
has_gpu=false
jhub_config_path=thu-vars/jupyterhub.yaml
http_port=9000
https_port=9443
```
就会使用`<MASTER_IP>:9000` 和 `<MASTER_IP>:9443` 地址来服务http和https流量。

## 卸载

如需卸载，运行

```sh
ansible-playbook -i inv.toml reset.yml 
```
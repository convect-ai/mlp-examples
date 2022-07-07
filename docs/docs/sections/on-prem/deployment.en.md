# On-prem deployment

## Introduction

This guide walks you through how to spawn a ConvectHub on a cluster of on-prem machines.

There are three types of machines that are involved during the provision process:

1. `Provisioner` -- This is not a computing node but rather a machine that used by an admin to trigger and control the provisioning process.
2. `Master` node -- This is a node that hosts the mission-critical services such as ingress, cluster database, device-plugins, etc. In an HA setting, there can be multiple master nodes.
3. `Worker` node -- This is a node that hosts the computing workloads such as a user notebook instance, a ML training job, etc.


## Prerequisites 

### Hardware

Minimum requirements 2 cores, 4GB memory, 10GB storage space for `Master` and `Worker`. 

1 core, 4GB memory, 25GB storage for `Provisioner`.


### OS

We current support Ubuntu (>=16.04) as the host OS for `Master` and `Worker`. CentOS support is under progress.

On your node, make sure you can install system libs by executing
```sh
apt update
apt install <LIB_NAME>
```
If you have public internet access from your nodes, this is done by visiting the official apt repo.
If you have a private network setting, then configure a mirror apt repo and point your system to use it by modifying `/etc/apt/sources.list`.

We recommend to use a linux OS on `Provisioner`. If you are using Windows, we commend to check out [WSL](https://docs.microsoft.com/en-us/windows/wsl/about).

### Network 

We require the following ports to be accessible by each other between any two nodes for `Master` and `Worker`.

| PROTOCOL	| PORT      |	SOURCE                              |	DESCRIPTION                                           |
------------|-----------|-----------------------                |-------------                                            |
| TCP	    | 6443	    |   K3s agent nodes	                    | Kubernetes API Server                                   |
| UDP	    | 8472	    |   K3s server and agent nodes	        | Required only for Flannel VXLAN                         |
| TCP	    | 10250	    |   K3s server and agent nodes	        | Kubelet metrics                                         |
| TCP	    | 2379-2380	|   K3s server nodes	                | Required only for HA with embedded etcd                 |
| TCP       | 80        |   ConvectHub                          | Required to visit ConvectHub via http. Can be configurable  |
| TCP       | 443       |   ConvectHub                          | Required to visit ConvectHub via https. Can be configurable |
| TCP       | 22        |   `Provisioner`                       | Required by `Provisioner` to visit `Master` and `Worker` via SSH |

Internet access on `Master` and `Worker` is optional, but required on `Provisioner`. 
We require `Provisioner` to be able to visit `Master` and `Worker` through their intranet addresses, i.e., `Provisioner` is connected to the private network. 

### Dependency libs and software

We require the following libs and software to be pre-configured on `Provisioner`

| MACHINE       |   Software/lib name       |  Installation guide       |
|---------------|---------------------------|---------------------------|
| `Provisioner` | Docker                    | [Official guide](https://docs.docker.com/get-docker/) |
| `Provisioner` | python3, python3-pip, python3-venv                   | [Official guide](https://www.python.org/downloads/) |


## Provision process

We assume all the following steps are done on `Provisioner` machine unless explicitly specified. 

### Check the network connectivity
On `Provisioner`, first make sure you can visit internet

```sh
ping -c 3 convect.ai
```

### Clone provision repo

TODO: need a better way to distribute it
```sh
$ git clone https://github.com/convect-ai/mlp.git

$ cd mlp/baremetal/airgap-k3s
```

### Download artifacts

TODO: need a better way to distribute it
```sh
$ aws s3 sync s3://convect-mlp-assets/bin/ shared/bin/
```
It may take a while depending on your internet connection.

### Set up ssh users and keys

Make sure you can ssh into `Master` and `Worker` from `Provisioner`. 
We recommend using a passwordless login method and a unified user name for each machine (Refer to [this guide](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server) if you are not sure how to do it).

For the user, we require `sudo` permission and recommend disabling the password prompt by creating a file
```sh
echo "$USER ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/$USER
```

### Install `ansible`

We use [`ansible`](https://www.ansible.com/) to automate the provision process. To install it

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

### Prepare the inventory file

An inventory file declares the fleet of machines (their address, how to access them) and their roles in the deployment. 

A sample file
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
The first line 
```
worker2 ansible_ssh_user=vagrant ansible_ssh_host=127.0.0.1 ansible_ssh_port=2201 ansible_ssh_private_key_file='~/.ssh/id_rsa' my_iface=eth1 has_gpu=true
```
declares a machine called `worker2`. Some notable fields 

- `ansible_ssh_user` - The user name to use when `ssh` into the machine
- `ansible_ssh_host` - The ip address or host name to use when `ssh` into the machine
- `ansible_ssh_port` - The port to use when `ssh` into the machine. Default = `22`
- `ansible_ssh_private_key_file` - The private key to use when `ssh` into the machine
- `my_iface` - The network interface name on the machine that is connected to the private network. You can run `ip a show` to see all the available network interface and the binded IP addresses on a machine to figure out the name
- `has_gpu` - Specifies if the machine is equiped with GPU. Default = `false`

```
[worker]
worker1
worker2
```
groups the machines `worker1` and `worker2` into the `worker` group. If you have more nodes to join the cluster as `workers`, put them here.

For `driver` group, you usually need one node if HA is not considered.  

```
[all:vars]
convect_workdir=/tmp/convecthub
convect_token=k3s_hub_token
has_gpu=false
jhub_config_path=thu-vars/jupyterhub.yaml
```
defines some common variables that rarely need change.

To prepare an inventory file for you own environment, the following steps are recommended:

1. Write down the ip addresses / hostnames for all nodes that are used as part of the cluster
2. Create a common user that can ssh into each node using a private key from `Provisioner`
3. Figure out the network interface name on each node that connects to the main private network if it has multiple interfaces
4. For each node, define a line like 
    ```
    worker2 ansible_ssh_user=vagrant ansible_ssh_host=127.0.0.1 ansible_ssh_port=2201 ansible_ssh_private_key_file='~/.ssh/id_rsa' my_iface=eth1 has_gpu=true
    ```
    by substituting `ansible_ssh_user`, `ansible_ssh_host`, `ansible_ssh_port`, `ansible_ssh_private_key_file`, `my_iface`, `has_gpu` to reflect your environment.

### Smoke test

Once you have your inventory file defined, say `inv.toml`, we can check if the configuration is good by running
```sh
$ ansible all -i inv.toml -m ping 
```
You should be able to see all nodes returning an `OK` status.

### Start the provision process

If you have set up the passwordless sudo permission of the user, then simply run
```sh
ansible-playbook -i inv.toml playbook.yml
```
otherwise 
```sh
ansible-playbook -i inv.toml playbook.yml --extra-vars ansible_sudo_pass=YOUR_PASSWORD
```
where `YOUR_PASSWORD` is he password required to execute `sudo`.

The provision process usually takes around 30 minutes depending on the number of nodes and your network condition.


## Validate ConvectHub is working
Once finished, on `Provisioner`, open a browser and visit `<MASTER_IP>`, where `<MASTER_IP>` is the private IP address of the `Master` node and verify you can see the login page.

## Configure the provisioning process

### Nvidia driver

If you have already configured nvidia display on your nodes, set `install_nvidia_driver=false` on a machine declaration in the inventory file to skip install the driver automatically. 

### Docker

If you have already configured docker on your nodes, set `install_docker=false` on a machine declaration in the inventory file to skip installing docker automatically.

If you have a private network setting and cannot visit the docker official repo, set `docker_repo` to point to your mirror, e.g., `https://download.docker.com/linux/ubuntu`

### Change ports to serve ConvectHub

If you would like to serve ConvectHub by using non-default ports (e.g., 80 and 443), set `http_port=<YOUR_PORT_NUMBER>` and `https_port=<YOUR_PORT_NUMBER>` under `[all:vars]` section in the inventory file. E.g., 

```
[all:vars]
convect_workdir=/tmp/convecthub
convect_token=k3s_hub_token
has_gpu=false
jhub_config_path=thu-vars/jupyterhub.yaml
http_port=9000
https_port=9443
```
will serve the http and https traffic via `<MASTER_IP>:9000` and `<MASTER_IP>:9443` respectively.


## Uninstall the deployments

If you would like to uninstall the deployments from your servers, just run 

```sh
ansible-playbook -i inv.toml reset.yml
```



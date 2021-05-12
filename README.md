# Installing and running DC/OS on an On-Premise cluster

This document walks you through the steps needed to:

- Install and run DC/OS on our cluster
- Managing nodes (Adding and removing master/worker nodes)
- Adding and executing services

## Prerequisites

### System Requirements

A DC/OS cluster consists of two types of nodes **master nodes** and **agent nodes**. The agent nodes can be either **public agent node**s or **private agent nodes**. Public agent nodes provide north-south (external to internal) access to services in the cluster through load balancers. Private agents host the containers and services that are deployed on the cluster. In addition to the master and agent cluster nodes, each DC/OS installation includes a separate **bootstrap node** for DC/OS installation and upgrade files.  
For our cluster I chose **"Ronon"** to be our bootstrap node. on Ronon all installation and upgrade scripts are generated and then provided to other nodes on the cluster to use.  
  
For more information please refer to [DC/OS documentatin about System Requirements](t.ly/fVM9) .

All the machines on our cluster run Ubuntu. Before you start make sure to run `sudo apt-get update` on all of them. Next, you need to:

1. Disable swapp on all machines using:  
   `sudo swapoff -a`
2. Stop and disable the `firewalld` by running:  
   `sudo systemctl stop firewalld && sudo systemctl disable firewalld`
3. Stopping `DNSmasq` service using:  
   `sudo systemctl stop dnsmasq && sudo systemctl disable dnsmasq.service`
4. We need to stop systemd-resolved using port 53 since DC/OS uses port 53. Do so using the following:  
   1. stop systemd-resolved `sudo systemctl stop systemd-resolved`.

   2. Edit `/etc/systemd/resolved.conf` and uncomment `DNSStubListener=no` as well as `DNS=8.8.8.8`.

   3. Finally `sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf`
5. Install Docker on all machines.
6. Since DC/OS documentation and installation scripts both think you are using CentOS and not Ubuntu you will face errors in the future when attempting to install. The reason is that the location of commands like `mkdir`, `curl` and `ln` are different in Ubuntu and CentOS, and DC/OS installation script treat your machine as a CentOS machine. To prevent those errors from occuring run these commangs:

```Bash
sudo apt-get install -y libcurl3-nss ipset selinux-utils curl unzip bc &&
sudo ln -s /bin/mkdir /usr/bin/mkdir &&
sudo ln -s /bin/ln /usr/bin/ln &&
sudo ln -s /bin/tar /usr/bin/tar &&
sudo ln -s /bin/rm /usr/bin/rm &&
sudo ln -s /usr/sbin/useradd /usr/bin/useradd &&
sudo ln -s /bin/bash /usr/bin/bash &&
sudo ln -s /sbin/ipset /usr/sbin/ipset
```

## Installation Process

ssh into your bootstrap machine and:

1. Create a directory named `genconf` on your bootstrap node and navigate to it.

    ```bash
    mkdir -p genconf
    ```

2. Create an IP detection script and store it as `genconf/ip-detect-public`

First you need to determine your network interface name using `ifconfig`. Then copy and paste these into the `ip-detect-public` file and make sure to use your interface name:

```bash
#!/usr/bin/env bash
set -o nounset -o errexit
export PATH=/usr/sbin:/usr/bin:$PATH
echo $(ip addr show your_interface_name | grep -Eo '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' | head -1)
```

[More information here](https://docs.d2iq.com/mesosphere/dcos/2.1/installing/production/deploying-dcos/installation/)

3. Create a configuration file.  
   In this step, you can create a YAML configuration file that is customized for your environment. DC/OS uses this configuration file during installation to generate your cluster installation files. Create a configuration file and save as `genconf/config.yaml`.  

   you can use this template:

   ```YAML
    bootstrap_url: http://<bootstrap_ip>:80
    cluster_name: <cluster-name>
    exhibitor_storage_backend: static
    master_discovery: static
    ip_detect_public_filename: <relative-path-to-ip-script>
    master_list:
    - <master-private-ip-1>
    - <master-private-ip-2>
    - <master-private-ip-3>
    resolvers:
    - 169.254.169.253
    use_proxy: 'true'
    http_proxy: http://<user>:<pass>@<proxy_host>:<http_proxy_port>
    https_proxy: https://<user>:<pass>@<proxy_host>:<https_proxy_port>
    no_proxy:
    - 'foo.bar.com'
    - '.baz.com'
   ```

   For your cluster I used these parameters. **PLEASE READ THE COMMENTS**:

   ```YAML
    # this has to be the ip:port of your bootstrap node. you can use any port number which is available. but keep note of it since you'll need them later.
    bootstrap_url: http://172.19.55.32:9090
    # write your own desired name for the cluster
    cluster_name: QNRCluster
    exhibitor_storage_backend: static
    log_directory: /genconf/logs
    # this has to point to the location where ip-detect or oip-detect-public script is located
    ip_detect_public_filename: /genconf/ip-detect-public
    master_discovery: static
    telemetry_enabled: 'false'
    # here you need to put the ip addresses of your desired master nodes. here I chose "Atlantis" to be the master node
    master_list:
    - 172.19.55.10
    # same as the list of masters, here I specified Elizabeth and Rodney to be our agent nodes
    agent_list:
    - 172.19.55.33
    - 172.19.55.34
    process_timeout: 120
    # I had to put these values as the resolvers otherwise our machines would not connect to the internet
    resolvers:
    - 8.8.8.8
    - 8.8.4.4
    process_timeout: 300
    oauth_enabled: 'true'
    # this is the path where you store your ssh key 
    ssh_key_path: /home/alireza/.ssh/id_rsa
    ssh_port: '22'
    ssh_user: alireza
    enable_ipv6: 'false'
    dcos_l4lb_enable_ipv6: 'false'
   ```

4. [Download and save the `dcos_generate_config` file](https://downloads.dcos.io/dcos/stable/dcos_generate_config.sh) to your bootstrap node. This file is used to create your customized DC/OS build file.

```bash
curl -O https://downloads.dcos.io/dcos/stable/dcos_generate_config.sh
```

At this point your directory structure should resemble:  

```
├── dcos-genconf.<HASH>.tar
├── dcos_generate_config.sh
├── genconf
│   ├── config.yaml
│   ├── ip-detect

```

5. Run the following to **generate the installation files**:

```Bash
sudo bash dcos_generate_config.sh
```

6. Using an enginx Docker container we will **host the generated files**.

```bash
docker run -d -p 9090:80 -v $PWD/genconf/serve:/usr/share/nginx/html:ro nginx
```

7. ssh to your master nodes.

```bash
ssh <master-ip>
```

8. make a new directory and navigate to it

```bash
mkdir /tmp/dcos && cd /tmp/dcos
```

9. Download the DC/OS installer from the NGINX Docker container, where `<bootstrap-ip>` and `<your_port>` are specified in bootstrap_url.

```bash
curl -O http://<bootstrap-ip>:<your_port>/dcos_install.sh
```

10. and finally

```bash
sudo bash dcos_install.sh master
```

you can repeat the same steps from 7 to 10 for your slave nodes. The only important difference is that you need to specify that the node is a slave node when running the installation script.

```bash
sudo bash dcos_install.sh slave # or <slave_public>
```

## Log in to your DC/OS web interface

Monitor the DC/OS web interface and wait for it to display at: `http://<master-node-public-ip>/`

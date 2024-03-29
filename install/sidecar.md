### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [PaaS-TA Sidecar Install](./README.md) > Sidecar

## Table of Contents

1. [Document Outline](#1)  
  1.1. [Purpose](#1.1)  
  1.2. [Range](#1.2)  
  1.3. [References](#1.3)  

2. [Configuring a Kubernetes Cluster with Kubespray](#2)  
  2.1. [Prerequisite](#2.1)  
  2.2. [AWS Settings (When Using AWS Environment)](#2.2)  
    ※ [(Refer) AWS IAM Settings](#2.2.1)  
  2.3. [Create and Deploy SSH Keys](#2.3)  
  2.4. [Kubespray Download](#2.4)  
  2.5. [Ubuntu, Python Package Installation](#2.5)  
  2.6. [Kubespray File Modification](#2.6)  
  2.7. [Change Kubespray Settings for Sidecar Installation](#2.7)  
  　2.7.1 [AWS](#2.7.1)  
  　2.7.2 [Openstack](#2.7.2)  
  2.8. [Kubernetes Cluster Configuration via Kuberpray](#2.8)  
  　2.8.1 [AWS](#2.8.1)  
  　2.8.2 [Openstack](#2.8.2)  
  2.9. [Kubernetes Cluster Usage Setting & Installation Check](#2.9)  
    ※ [(Refer) Delete Kubernetes Cluster with Kubespray](#2.9.1)  

3. [PaaS-TA Sidecar Installation](#3)  
  3.1. [Introduction to Executable Files](#3.1)  
  3.2. [Download Executable Files](#3.2)  
  3.3. [variable Settings](#3.3)  
  3.4. [Storageclass Default Settings](#3.4)  
  3.5. [Create Sidecar Values](#3.5)  
  3.6. [Create Sidecar Deployment YAML](#3.6)  
  3.7. [Sidecar Installation](#3.7)  
  　※ [LoadBalancer Domain Connection During AWS-Based Sidecar Installation](#3.7.1)  
  3.8. [Sidecar Login and Deployment of Test App](#3.8)  
    ※ [(Refer) Sidecar Deletion](#3.8.1)  

# <div id='1'> 1. Document Outline
## <div id='1.1'> 1.1. Purpose
The purpose of this document is to provide a guide for configuring the Kubernetes Cluster with Kubespray used for exclusive deployment of PaaS-TA Container-Platform and installing PaaS-TA Sidecar (hereinafter referred to as Sidecar) in the environment.

<br>

## <div id='1.2'> 1.2. Range
This Document was written based on [cf-for-k8s v5.4.2](https://github.com/cloudfoundry/cf-for-k8s/tree/v5.4.2), [paas-ta-container-platform v1.1.0](https://github.com/PaaS-TA/paas-ta-container-platform/tree/v1.1.0).    
This document is based on the installation of Sidecar after configuring the Kubernetes Cluster by utilizing PaaS-TA Container-Platform Single Distribution (Kubespray) in AWS and Openstack environments.  
This document was guided on the premise that there was a basic understanding of IaaS and Kubernetes.  

<br>


## <div id='1.3'> 1.3. References
PaaS-TA Container Platform : [https://github.com/PaaS-TA/paas-ta-container-platform](https://github.com/PaaS-TA/paas-ta-container-platform)  
Kubespray : [https://kubespray.io](https://kubespray.io)  
Kubespray github : [https://github.com/kubernetes-sigs/kubespray](https://github.com/kubernetes-sigs/kubespray)  
cf-for-k8s github : [https://github.com/cloudfoundry/cf-for-k8s](https://github.com/cloudfoundry/cf-for-k8s)  
cf-for-k8s Document : [https://cf-for-k8s.io/docs/](https://cf-for-k8s.io/docs/)  

<br>

# <div id='2'> 2. Configuring a Kubernetes Cluster with Kubespray
The basic Kubernetes Cluster configuration method follows the PaaS-TA Container Platform solo deployment installation guide, but there are some options or parts to be modified on IaaS.
Since the Kubernetes Cluster configuration in this guide has been briefly modified in the linked standalone deployment installation guide, see the linked standalone deployment installation guide for a detailed description of the Kubernetes Cluster configuration.

<br>

## <div id='2.1'> 2.1. Prerequisite
Key software and package version information for Kubernetes Cluster configuration can be found in the PaaS-TA Container Platform Standalone Deployment Installation Guide.  
In addition, the cf-for-k8s official document recommends the Kubernetes Cluster requirements as follows.
- Kubernetes version : 1.19 ~ 1.22
- At least 5 nodes
- Minimum 4 CPUs per node, 15GB Memory
- Have CNI Plugin with Network Policies
- LoadBalancer Service Provided
- Default StorageClass Set
- Provides OCI-compliant registry (e.g. [Docker Hub](https://hub.docker.com/), [Google container registry](https://cloud.google.com/container-registry),  [Azure container registry](https://hub.docker.com/), [Harbor](https://goharbor.io/), etc....)  
  This guide is based on the Docker Hub. (Account registration required)

<br>

## <div id='2.2'> 2.2. AWS Settings (When Using AWS Environment)
When configuring a Kubernetes Cluster for Sidecar on AWS, IAM authorization is required on the instance that configures the cluster for use with LoadBalancer or Storage.
- Create an IAM role, add the following policy, and apply it when creating an instance. (Refer to AWS IAM settings below when checking the supplementary explanation about IAM settings.)
```
# iam_policy.json

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:*"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:*"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:*"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::kubernetes-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetRepositoryPolicy",
                "ecr:DescribeRepositories",
                "ecr:ListImages",
                "ecr:BatchGetImage"
            ],
            "Resource": "*"
        }
    ]
}
```

- Add the following tags to the **instance** to configure the cluster and the **subnet** tags used.
```
key = kubernetes.io/cluster/{cluster_name}
value = member
```
![99](./images/sidecar/tags.png)

<br>

### <div id='2.2.1'> ※ (Refer) AWS IAM Settings
It describes how to set up AWS IAM.  

- AWS IAM Menu - From the Roles menu, select Create Role.  
![IAM_01](./images/sidecar/IAM_01.PNG)
  
- Proceed to Create Role to select Create Policy.  
![IAM_02](./images/sidecar/IAM_02.PNG)
  
- Select JSON and paste iam_policy.json at the top.  
![IAM_03](./images/sidecar/IAM_03.PNG)
  
- After creating a policy, return to Create Role to select the policy created.  
![IAM_04](./images/sidecar/IAM_04.PNG)
  
- Name it and complete the role.  
![IAM_05](./images/sidecar/IAM_05.PNG)
  
- Select the role that you created in the IAM role when configuring the EC2 instance.  
![IAM_06](./images/sidecar/IAM_06.PNG)
  
- If you have configured the instance and you have not set up the IAM, select the role you created by selecting Modify Instance - Task - Security - IAM Role and reboot the instance.  
![IAM_07](./images/sidecar/IAM_07.png)
  
<br>
  
## <div id='2.3'> 2.3. Create and Deploy SSH Keys
All installation processes after SSH key generation and deployment are performed at the Master Node.

- Access the Master Node to generate an RSA public key.
```
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ubuntu/.ssh/id_rsa): [Enter]
Enter passphrase (empty for no passphrase): [Enter]
Enter same passphrase again: [Enter]
Your identification has been saved in /home/ubuntu/.ssh/id_rsa.
Your public key has been saved in /home/ubuntu/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:pIG4/G309Dof305mWjdNz1OORx9nQgQ3b8yUP5DzC3w ubuntu@paasta-cp-master
The key's randomart image is:
+---[RSA 2048]----+
|            ..= o|
|   . .       * B |
|  . . . .   . = *|
| . .   +     + E.|
|  o   o S     +.O|
|   . o o .     XB|
|    . o . o   *oO|
|     .  .. o B oo|
|        .o. o.o  |
+----[SHA256]-----+
```

- Copy the public key to the Master, Worker Node to use.
```
## Copy the printed public key

$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5QrbqzV6g4iZT4iR1u+EKKVQGqBy4DbGqH7/PVfmAYEo3CcFGhRhzLcVz3rKb+C25mOne+MaQGynZFpZk4muEAUdkpieoo+B6r2eJHjBLopn5quWJ561H7EZb/GlfC5ThjHFF+hTf5trF4boW1iZRvUM56KAwXiYosLLRBXeNlub4SKfApe8ojQh4RRzFBZP/wNbOKr+Fo6g4RQCWrr5xQCZMK3ugBzTHM+zh9Ra7tG0oCySRcFTAXXoyXnJm+PFhdR6jbkerDlUYP9RD/87p/YKS1wSXExpBkEglpbTUPMCj+t1kXXEJ68JkMrVMpeznuuopgjHYWWD2FgjFFNkp ubuntu@paasta-cp-master
```

- Copy the public key to the end of the authorized_keys file body of the master, Worker Node to use (add below the existing body contents).
```
$ vi ~/.ssh/authorized_keys

ex)
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDRueywSiuwyfmCSecHu7iwyi3xYS1xigAnhR/RMg/Ws3yOuwbKfeDFUprQR24BoMaD360uyuRaPpfqSL3LS9oRFrj0BSaQfmLcMM1+dWv+NbH/vvq7QWhIszVCLzwTqlHrhgNsh0+EMhqc15KEo5kHm7d7vLc0fB5tZkmovsUFzp01Ceo9+Qye6+j+UM6ssxdTmatoMP3ZZKZzUPF0EZwTcGG6+8rVK2G8GhTqwGLj9E+As3GB1YdOvr/fsTAi2PoxxFsypNR4NX8ZTDvRdAUzIxz8wv2VV4mADStSjFpE7HWrzr4tZUjvvVFptU4LbyON9YY4brMzjxA7kTuf/e3j Generated-by-Nova
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5QrbqzV6g4iZT4iR1u+EKKVQGqBy4DbGqH7/PVfmAYEo3CcFGhRhzLcVz3rKb+C25mOne+MaQGynZFpZk4muEAUdkpieoo+B6r2eJHjBLopn5quWJ561H7EZb/GlfC5ThjHFF+hTf5trF4boW1iZRvUM56KAwXiYosLLRBXeNlub4SKfApe8ojQh4RRzFBZP/wNbOKr+Fo6g4RQCWrr5xQCZMK3ugBzTHM+zh9Ra7tG0oCySRcFTAXXoyXnJm+PFhdR6jbkerDlUYP9RD/87p/YKS1wSXExpBkEglpbTUPMCj+t1kXXEJ68JkMrVMpeznuuopgjHYWWD2FgjFFNkp ubuntu@paasta-cp-master
```
<br>

## <div id='2.4'> 2.4. Kubespray Download
- Download Kubespray from the following path through the git clone command. The version of the paas-ta-container-platform in this installation guide is v1.1.0 and the version of Kubespray is v2.16.0.
```
$ git clone https://github.com/PaaS-TA/paas-ta-container-platform-deployment.git -b v1.1.0
```

<br>

## <div id='2.5'> 2.5. Ubuntu, Python Package Installation
Install Python packages such as Ansible and Jinja, which are necessary for Kubespray installation.

- Perform apt-get update.
```
$ sudo apt-get update
```

- Install python3-pip Package.
```
$ sudo apt-get install -y python3-pip
```

- Move the Kubespray installation path and install the Python Package necessary for Kubespray installation using pip.
```
## When Installing AWS Environment

$ cd paas-ta-container-platform-deployment/standalone/aws
```

```
## When Installing Openstack Environment

$ cd paas-ta-container-platform-deployment/standalone/openstack
```

```
$ sudo pip3 install -r requirements.txt
```

<br>

## <div id='2.6'> 2.6. Kubespray File Modification

- Set inventory.ini file in mycluster directory.
```
$ vi inventory/mycluster/inventory.ini
```

```
## {MASTER_HOST_NAME}, {WORKER_HOST_NAME} : Actual Master, Worker Node hostname
ex) When Using AWS
$ curl http://169.254.169.254/latest/meta-data/hostname
ip-10-0-0-55.ap-northeast-2.compute.internal

ex) When Using Openstack
$ hostname
paasta-cp-master
```

```
## {MASTER_NODE_IP}, {WORKER_NODE_IP} : Master, Worker Node Private IP

ex)
$ ifconfig
...
```

```
[all]
{MASTER_HOST_NAME} ansible_host={MASTER_NODE_IP} ip={MASTER_NODE_IP} etcd_member_name=etcd1
{WORKER_HOST_NAME1} ansible_host={WORKER_NODE_IP1} ip={WORKER_NODE_IP1}      # Create based on the number of Worker_NODEs to use (one or more)

{WORKER_HOST_NAME2} ansible_host={WORKER_NODE_IP2} ip={WORKER_NODE_IP2}
{WORKER_HOST_NAME3} ansible_host={WORKER_NODE_IP3} ip={WORKER_NODE_IP3}

[kube_control_plane]
{MASTER_HOST_NAME}

[etcd]
{MASTER_HOST_NAME}

[kube_node]
{WORKER_HOST_NAME1}           # Create based on the number of Worker_NODEs to use (one or more)
{WORKER_HOST_NAME2}
{WORKER_HOST_NAME3}

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

```
ex) AWS
ip-10-0-0-55.ap-northeast-2.compute.internal  ansible_host=10.0.0.55 ip=10.0.0.55 etcd_member_name=etcd1
ip-10-0-0-66.ap-northeast-2.compute.internal  ansible_host=10.0.0.66 ip=10.0.0.66
...
[kube_control_plane]
ip-10-0-0-55.ap-northeast-2.compute.internal

ex) Openstack
paasta-cp-master  ansible_host=10.x.x.x ip=10.x.x.x etcd_member_name=etcd1
paasta-cp-worker-1  ansible_host=10.x.x.x ip=10.x.x.x
...
[kube_control_plane]
paasta-cp-master
...
```

- Add HostName information for the Master Node to deploy Metrics-server.

```
$ vi roles/kubernetes-apps/metrics_server/defaults/main.yml
```

```
...
master_node_hostname: {MASTER_NODE_HOSTNAME}
````

<br>

## <div id='2.7'> 2.7. Change Kubespray Settings for Sidecar Installation

- Disable ingress_nginx_enabled. (Common AWS, Openstack)
```
$ vi inventory/mycluster/group_vars/k8s_cluster/addons.yml
...
ingress_nginx_enabled: false
...
```

<br>

### <div id='2.7.1'> 2.7.1. AWS
- Set cloud_provider to AWS when using AWS environment.
```
$ vi inventory/mycluster/group_vars/all/all.yml
...
cloud_provider: aws
...
```

- Add settings for EBS when using the AWS environment with EBS.  
```
$ vi inventory/mycluster/group_vars/all/aws.yml

### and configure the parameters below
aws_ebs_csi_enabled: true
aws_ebs_csi_enable_volume_scheduling: true
aws_ebs_csi_enable_volume_snapshot: false
aws_ebs_csi_enable_volume_resizing: false
aws_ebs_csi_controller_replicas: 1
aws_ebs_csi_plugin_image_tag: latest
#aws_ebs_csi_extra_volume_tags: "Owner=owner,Team=team,Environment=environment'
```

<br>

### <div id='2.7.2'> 2.7.2. Openstack
- If you use Octavia Load Balancer when using Openstack environment, add settings for Octavia. (Optional)
```
$ vi inventory/mycluster/group_vars/all/openstack.yml
...
## Values for the external OpenStack Cloud Controller
external_openstack_lbaas_network_id: "network_id"
external_openstack_lbaas_subnet_id: "subnet_id"
external_openstack_lbaas_floating_network_id: "floating_network_id"
external_openstack_lbaas_floating_subnet_id: "subnet_id"
external_openstack_lbaas_method: "ROUND_ROBIN"
external_openstack_lbaas_provider: "octavia"
external_openstack_lbaas_create_monitor: false
external_openstack_lbaas_monitor_delay: "1m"
external_openstack_lbaas_monitor_timeout: "30s"
external_openstack_lbaas_monitor_max_retries: "3"
external_openstack_lbaas_manage_security_groups: false
external_openstack_lbaas_internal_lb: false
...
```
<br>

## <div id='2.8'> 2.8. Kubernetes Cluster Configuration via Kuberpray
### <div id='2.8.1'> 2.8.1. AWS

- When using an AWS environment, use the following command to proceed with cluster configuration.
```
$ ansible-playbook -i ./inventory/mycluster/inventory.ini ./cluster.yml -e ansible_user=$(whoami) -b --become-user=root --flush-cache
```

<br>

### <div id='2.8.2'> 2.8.2. Openstack
When using the OpenStack environment, proceed in the same way as the existing Container-Platform solo deployment installation guide.
- Update the Ansible inventory file to the inventory builder.
```
## {MASTER_NODE_IP}, {WORKER_NODE_IP} : Master, Worker Node Private IP
## Create {WORKER_NODE_IP} based on the number of WORKER_NODEs to use (at least one)

$ declare -a IPS=({MASTER_NODE_IP} {WORKER_NODE_IP1} {WORKER_NODE_IP2} {WORKER_NODE_IP3})

ex)
declare -a IPS=(10.x.x.x 10.x.x.x 10.x.x.x 10.x.x.x)
```

```
## Be aware that ${IPS[@]} is not a variable but a part of the command

$ CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```
- Additional environmental variables are required when installing in the Openstack environment, and environmental variables can be automatically registered by downloading the configuration file.
```
## Openstack UI Login > Select Project > Select API Access Menu > Click OpenStack RC File Download
## Enter Openstack account password after running script file

$ source {OPENSTACK_PROJECT_NAME}-openrc.sh
Please enter your OpenStack Password for project admin as user admin: {Enter Password}
```

- If the MTU value of the Openstack network interface is not the default value of 1450, the CNI Plugin MTU setting needs to be changed.
```
## MTU Check (ex mtu 1400)

$ ifconfig
ens3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1400
        inet 10.100.2.226  netmask 255.255.255.0  broadcast 10.100.2.255
        inet6 fe80::f816:3eff:fe2f:a831  prefixlen 64  scopeid 0x20<link>
        ether fa:16:3e:2f:a8:31  txqueuelen 1000  (Ethernet)
        RX packets 19850  bytes 167795140 (167.7 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 17667  bytes 1920463 (1.9 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
...
```

```
$ vi inventory/mycluster/group_vars/k8s_cluster/k8s-net-calico.yml
```

```
...
calico_mtu: 1400
...
```

- - Proceed with Kubespray deployment through the Ansible playbook. Set the playbook option to run as root. (--become-user=root)
```
$ ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```
<br>

### <div id='2.9'> 2.9. Kubernetes Cluster Usage Setting & Installation Check

- After completing the Kubernetes Cluster configuration with Kubespray, perform the following steps to use the cluster.
```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Check node and pod to see if the installation was done properly.
```
$ kubectl get nodes
NAME                 STATUS   ROLES                  AGE   VERSION
paasta-cp-master     Ready    control-plane,master   12m   v1.20.5
paasta-cp-worker-1   Ready    <none>                 10m   v1.20.5
paasta-cp-worker-2   Ready    <none>                 10m   v1.20.5
paasta-cp-worker-3   Ready    <none>                 10m   v1.20.5

$ kubectl get pods -n kube-system
NAME                                          READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7c5b64bf96-xwdgn      1/1     Running   0          8m52s
calico-node-d8sg6                             1/1     Running   0          9m22s
calico-node-kfvjx                             1/1     Running   0          10m
calico-node-khwdz                             1/1     Running   0          10m
calico-node-nc58v                             1/1     Running   0          10m
coredns-657959df74-td5c2                      1/1     Running   0          8m15s
coredns-657959df74-ztnjj                      1/1     Running   0          8m7s
csi-cinder-controllerplugin-99c9dd87b-hz62k   5/5     Running   0          7m42s
csi-cinder-nodeplugin-jjkg5                   2/2     Running   0          7m41s
csi-cinder-nodeplugin-njdrc                   2/2     Running   0          7m41s
csi-cinder-nodeplugin-sb9vg                   2/2     Running   0          7m41s
csi-cinder-nodeplugin-t5fxh                   2/2     Running   0          7m41s
dns-autoscaler-b5c786945-rhlkd                1/1     Running   0          8m9s
kube-apiserver-paasta-cp-master               1/1     Running   0          12m
kube-controller-manager-paasta-cp-master      1/1     Running   1          12m
kube-proxy-dj5c8                              1/1     Running   0          10m
kube-proxy-kkvhk                              1/1     Running   0          10m
kube-proxy-nfttc                              1/1     Running   0          10m
kube-proxy-znfgk                              1/1     Running   0          10m
kube-scheduler-paasta-cp-master               1/1     Running   1          12m
metrics-server-5cd75b7749-xcrps               2/2     Running   0          7m57s
nginx-proxy-paasta-cp-worker-1                1/1     Running   0          10m
nginx-proxy-paasta-cp-worker-2                1/1     Running   0          10m
nginx-proxy-paasta-cp-worker-3                1/1     Running   0          10m
nodelocaldns-556gb                            1/1     Running   0          8m8s
nodelocaldns-8dpnt                            1/1     Running   0          8m8s
nodelocaldns-pvl6z                            1/1     Running   0          8m8s
nodelocaldns-x7grn                            1/1     Running   0          8m8s
openstack-cloud-controller-manager-mct28      1/1     Running   0          8m57s
snapshot-controller-0                         1/1     Running   0          7m33s
```


<br>

### <div id='2.9.1'> ※ (Refer) Delete Kubernetes Cluster with Kubespray
Use Ansible playbook and delete Kubespray.

```
# When using AWS environment
$ ansible-playbook -i ./inventory/mycluster/inventory.ini ./reset.yml -e ansible_user=$(whoami) -b --become-user=root --flush-cache

# When using Openstack environment
$ ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root reset.yml
```

<br>

# <div id='3'> 3. PaaS-TA Sidecar Installation
## <div id='3.1'> 3.1. Introduction to Executable Files
- The following executable files are required to install and use the Sidecar.

| Name   |      Description      |
|----------|-------------|
| [ytt](https://carvel.dev/ytt/) | A tool to create YAML which is used when deploying Sidecar |
| [kapp](https://carvel.dev/kapp/) | A tool that manages lifecycle of Sidecar |
| [kubectl](https://github.com/kubernetes/kubectl) | A tool that controls Kubernetes Cluster |
| [bosh cli](https://github.com/cloudfoundry/bosh-cli) | Tools for generating arbitrary passwords and certificates to be used by Sidecar |
| [cf cli](https://github.com/cloudfoundry/cli) (v7+) | Tools that interact with Sidecar |

<br>

- The script used to install Sidecar is as follows:

| Name   |      Description      | Note |
|----------|-------------|----|
| utils-install.sh | Tool installation script used when installing and utilizing Sidecar | ytt, kapp, bosh cli, cf cli installation |
| variables.yml | Variable settings file to apply when installing Sidecar ||
| 1.storageclass-config.sh | Script defining Storageclass to be used when installing Sidecar as default ||
| 2.generate-values.sh | Script to generate Manifest with a password, certificate, and other settings to use when installing Sidecar ||
| 3.rendering-values.sh | Script to generate YAML using Manifest with password, certificate, etc. settings ||
| 4.deploy-sidecar.sh | Script to install Sidecar using the generated YAML ||
| delete-sidecar.sh | Script to delete Sidecar ||
| deploy-ebs-sc.sh | Script to deploy EBS Storageclass | Apply when using AWS EBS|
| deploy-inject-self-signed-cert.sh | Secondary script to insert a CA into the POD when using the Private Registry with a self-signed certificate | Check the description inside the deploy-inject-self-signed-cert.sh file or refer to [cert-injection-webhook](https://github.com/vmware-tanzu/cert-injection-webhook) for detailed guide |
| delete-inject-self-signed-cert.sh | Script to delete inject-self-signed-cert |  |
| install-test.sh | Scripts to deploy and verify Test App after installation ||

<br>

## <div id='3.2'> 3.2. Download Executable Files

- Using the git clone command, download sidecar from the following path. The version of the sidecar in this guide is beta2.
```
$ cd $HOME
$ git clone https://github.com/PaaS-TA/sidecar-deployment.git -b beta2
$ cd sidecar-deployment/install-scripts
```

<br>

- Run the utils-install.sh file and install the files needed for Sidecar installation.
```
$ source utils-install.sh
```
<br>

## <div id='3.3'> 3.3. variable Settings
- Edit the variable.yml file to set options for Sidecar installation.
```
$ vi variables.yml

## COMMON VARIABLE
iaas=aws                                                    # IaaS (e.g. aws or openstack)
system_domain=sidecar.com                                   # sidecar system_domain (e.g. 3.35.135.135.nip.io)
use_lb=true                                                 # (e.g. true or false)
public_ip=3.35.135.135                                      # LoadbalancerIP (PublicIP associcated with system_domain, if use openstack)
storageclass_name=ebs-sc                                    # Storage Class Name ($ kubectl get sc) (e.g. ebs-sc[use aws] || cinder-csi[use openstack])


## APP_REGISTRY VARIABLE
app_registry_kind=dockerhub                                 # Registry Kind (e.g. dockerhub or private)
app_registry_repository=repository_name                     # Repository Name
app_registry_id=registry_id                                 # Registry ID
app_registry_password=registry_password                     # Registry Password

app_registry_address=harbor00.nip.io                        # if app_registry_kind==private, fill in app_registry_address
is_self_signed_certificate=false                            # is private registry use self-signed certificate? (e.g. true or false)
app_registry_cert_path=support-files/private-repository.ca  # if is_self_signed_certificate==true --> add the contents of the private-repository.ca file
                                                            # if is_self_signed_certificate==false --> private-repository.ca is empty

## EXTERNAL BLOBSTORE VARIABLE (Option)
use_external_blobstore=false                                # (e.g. true or false)
external_blobstore_ip=192.50.50.50                          # Blobstore Address (e.g. 127.0.0.1)
external_blobstore_port=9000                                # Blobstore Port (e.g. 9000)
external_blobstore_id=admin                                 # Blobstore ID
external_blobstore_password=adminpw                         # Blobstore Password
external_blobstore_package_directory=cc-package             # Blobstore Package Directory
external_blobstore_droplet_directory=cc-droplet             # Blobstore Droplet Directory
external_blobstore_resource_directory=cc-resource           # Blobstore Resource Directory
external_blobstore_buildpack_directory=cc-buildpack         # Blobstore Buildpack Directory


## EXTERNAL DB VARIABLE (Option)
use_external_db=false                                       # (e.g. true or false)
external_db_kind=postgres                                   # External DB Kind(e.g. postgres or mysql)
external_db_ip=10.100.100.100                               # External DB IP
external_db_port=5432                                       # External DB Port
external_cc_db_id=cloud_controller                          # Cloud Controller DB ID
external_cc_db_password=cc_admin                            # Cloud Controller DB Password
external_cc_db_name=cloud_controller                        # Cloud Controller DB Name
external_uaa_db_id=uaa                                      # UAA DB ID
external_uaa_db_password=uaa_admin                          # UAA DB Password
external_uaa_db_name=uaa                                    # UAA DB Name
external_db_cert_path=support-files/db.ca                   # if DB use cert --> add the contents of the db.ca file
                                                            # if DB don't use cert --> db.ca is empty
```
- The main variables are described as follows.

| Name   |      Description      |
|----------|-------------|
| iaas | A Cluster-configured IaaS (aws, openstack) |
| system_domain | Domain of Sidecar(A domain that can be connected with LoadBalancer) |
| use_lb | LoadBalancer enabled (Set system_domain to system_domain associated with PublicIP in one of the Cluster Workers when disabled) <br> (e.g. Cluster Worker Floating IP : 3.50.50.50 -> system_domain : 3.50.50.50.nip.io or set the connected domain)|
| public_ip | IP of LoadBalancer(Set up when the load balancer provided by the cloud provider uses IP) <br> (e.g. When using Octavia of Openstack) |
| storageclass_name | Storageclass to use (Openstack : cinder-csi, AWS : ebs-sc) |
| app_registry_kind | Registry type (dockerhub, private) |
| app_registry_repository | Repository name (Enter a value such as app_registry_id when using dockerhub) |
| app_registry_address | Enter Registry address if app_registry_kind is private |
| use_external_blobstore | when using an external blobstore (minIO)(true, false) |
| use_external_db | When using an external database (postgres, mysql) (true, false) |

<br>

## <div id='3.4'> 3.4. Storageclass Default Settings
In order to install Sidecar, it is necessary to set the default storage class in use.

- If EBS is used when using AWS, deploy EBS Storage Class.  
```
$ source deploy-ebs-sc.sh
```
<br>

    
- Check the StorageClass to use.
```
$ kubectl get sc 
NAME               PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc             ebs.csi.aws.com   Delete          WaitForFirstConsumer   false                  1m
```
  
<br>
  
- If you entered storageclass_name in variables.yml, run the following script to use the default for StorageClass.
    
```
$ source 1.storageclass-config.sh
storageclass.storage.k8s.io/ebs-sc patched
```

<br>

- Verify the storage class to use if it's set to default.
```
$ kubectl get sc 
NAME               PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc (default)   ebs.csi.aws.com   Delete          WaitForFirstConsumer   false                  1m
```
  
<br>

## <div id='3.5'> 3.5. Create Sidecar values
- Execute a script that creates values to be used when installing Sidecar. 
  (If variables.yml was modified during installation restart with 2.generate-values.sh.)
```
$ source 2.generate-values.sh
```
<br>

- The Manifest file is created in manifest/sidecar-values.yml, and various values can be checked and modified.   
  (Check the available variables from the [Link](https://cf-for-k8s.io/docs/platform_operators/config-values/).)

```
$ vi manifest/sidecar-values.yml

#@data/values
---
system_domain: "system_domain"
app_domains:
#@overlay/append
- "apps.system_domain"
cf_admin_password: 0pnasdfggpq58hq78vwm

blobstore:
  secret_access_key: qkw95zasdfgrhr5dg1v4

cf_db:
  admin_password: vqorcaldasdfgz4jp8h
.....
.....

app_registry:
  hostname: https://index.docker.io/v1/
  repository_prefix: "dockerhub_repository_name"
  username: "dockerhub_id"
  password: "dockerhub_password"
```

<br>

## <div id='3.6'> 3.6. Create Sidecar Deployment YAML

- Create a YAML to install Sidecar using the created sidecar-values.yml file.  
  (If sidecar-values.yml was modified during installation, restart with 3.rendering-values.sh.)
```
$ source 3.rendering-values.sh
```
<br>

- The YAML file is created at the manifest/sidecar-rendered.yml. Checking of the deployed resource and modification can be done through Kubernetes.

```
$ vi manifest/sidecar-rendered.yml

apiVersion: kapp.k14s.io/v1alpha1
kind: Config
metadata:
  name: kapp-version
minimumRequiredVersion: 0.33.0
---
apiVersion: kapp.k14s.io/v1alpha1
kind: Config
metadata:
  name: kapp-istio-gateway-rules
.....
```

<br>

## <div id='3.7'> 3.7. Sidecar Installation
- Install Sidecar using the generated YAML file.
```
$ source 4.deploy-sidecar.sh

........
6:43:21AM:  L ok: waiting on pod/restart-workloads-for-istio1-11-1-pmtlv (v1) namespace: cf-workloads
6:43:25AM: ok: reconcile job/restart-workloads-for-istio1-11-1 (batch/v1) namespace: cf-workloads
6:43:25AM:  ^ Completed
6:43:25AM: ---- applying complete [296/296 done] ----
6:43:25AM: ---- waiting complete [296/296 done] ----

Succeeded

```

- When Sidecar is installed normally, the POD is configured as follows:
```
$ kubectl get pods -A

NAMESPACE      NAME                                          READY   STATUS      RESTARTS   AGE
cf-blobstore   cf-blobstore-minio-95dc47fd6-2l85l            2/2     Running     0          2h
cf-db          cf-db-postgresql-0                            2/2     Running     0          2h
cf-system      ccdb-migrate-5w68c                            0/2     Completed   0          2h
cf-system      cf-api-clock-7669cb48cd-mm9j6                 2/2     Running     0          2h
cf-system      cf-api-controllers-558968745b-znt27           3/3     Running     0          2h
cf-system      cf-api-deployment-updater-8957f4dd6-x5dd2     2/2     Running     0          2h
cf-system      cf-api-server-794b67c6b5-wmtfc                6/6     Running     0          2h
cf-system      cf-api-worker-7d89858dfd-8qts7                3/3     Running     0          2h
cf-system      eirini-api-74b7987c55-dr96j                   2/2     Running     0          2h
cf-system      eirini-app-migration-qn9lp                    0/1     Completed   0          2h
cf-system      eirini-event-reporter-6998bfd5f6-nrwhl        2/2     Running     0          2h
cf-system      eirini-event-reporter-6998bfd5f6-qhzvf        2/2     Running     0          2h
cf-system      eirini-task-reporter-7996779cfb-824p5         2/2     Running     0          2h
cf-system      eirini-task-reporter-7996779cfb-8fmpf         2/2     Running     0          2h
cf-system      fluentd-4qsd5                                 2/2     Running     0          2h
cf-system      fluentd-5gxbl                                 2/2     Running     0          2h
cf-system      fluentd-wlv2s                                 2/2     Running     0          2h
cf-system      instance-index-env-injector-d964c475f-wdmh7   1/1     Running     0          2h
cf-system      log-cache-backend-bcbc5cb8d-shqmq             3/3     Running     0          2h
cf-system      log-cache-frontend-78985f587f-j68fn           3/3     Running     0          2h
cf-system      metric-proxy-5fdb694769-4q445                 2/2     Running     0          2h
cf-system      routecontroller-6d666b46fd-wb6cq              2/2     Running     0          2h
cf-system      uaa-66c4b86985-fx6m9                          3/3     Running     0          2h
cf-workloads   restart-workloads-for-istio1-11-1-pmtlv       0/1     Completed   0          2h
istio-system   istio-ingressgateway-46tx9                    2/2     Running     0          2h
istio-system   istio-ingressgateway-kk89r                    2/2     Running     0          2h
istio-system   istio-ingressgateway-zml5q                    2/2     Running     0          2h
istio-system   istiod-56c7f5bc65-mbmtt                       1/1     Running     0          2h
istio-system   istiod-56c7f5bc65-qrxc6                       1/1     Running     0          2h
kpack          kpack-controller-966c8d5fb-2lqmn              2/2     Running     0          2h
kpack          kpack-webhook-7b57486ddf-zwfnx                2/2     Running     0          2h
.....
```

<br>

### <div id='3.7.1'> ※ LoadBalancer Domain Connection During AWS-Based Sidecar Installation
When using AWS LoadBalancer, a domain connection using Route53 is required.
- Check AWS LoadBalancer Name

```
$ kubectl get svc -n istio-system istio-ingressgateway
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)                                                      AGE
istio-ingressgateway   LoadBalancer   10.233.28.216   a0c35cf15801c4557a9b49b3a97f86ef-1468743017.ap-northeast-2.elb.amazonaws.com   15021:32733/TCP,80:30412/TCP,443:31913/TCP,15443:31820/TCP   2h
```

- After creating a hosting area on Route 53, enter the LoadBalancer name (EXTERNAL-IP) in the routing target and generate a record.

![route53](./images/sidecar/route53.PNG)

<br>

## <div id='3.8'> 3.8. Sidecar Login and Deployment of Test App
- Deploy the test app to check if the app was deployed normally.
```
# Automated Deployment Test
$ source install-test.sh
.......
Waiting for app test-node-app to start...

Instances starting...
Instances starting...

name:                test-node-app
requested state:     started
isolation segment:   placeholder
routes:              test-node-app.apps.system.domain
last uploaded:       Thu 30 Sep 07:04:54 UTC 2021
stack:               
buildpacks:          
isolation segment:   placeholder

type:            web
sidecars:        
instances:       1/1
memory usage:    1024M
start command:   node server.js
  state     since                  cpu    memory   disk     details
#0   running   2021-09-30T07:06:01Z   0.0%   0 of 0   0 of 0   
==============================
check output 'Hello World'
Hello World
==============================



# Manual Deployment Test
$ cf login -a api.$(grep system_domain ./manifest/sidecar-values.yml | cut -d" " -f2 | sed -e 's/\"//g') --skip-ssl-validation -u admin -p "$(grep cf_admin_password ./manifest/sidecar-values.yml | cut -d" " -f2)"

Authenticating...
OK

Targeted org system.

API endpoint:   https://api.<system_domain>
API version:    3.104.0
user:           admin
org:            system
space:          No space targeted, use 'cf target -s SPACE'

  

$ cf create-space test-space
Creating space test-space in org system as admin...
OK

Assigning role SpaceManager to user admin in org system/space test-space as admin...
OK

Assigning role SpaceDeveloper to user admin in org system/space test-space as admin...
OK

TIP: Use 'cf target -o "system" -s "test-space"' to target new space



$ cf target -s test-space
API endpoint:   https://api.<system_domain>
API version:    3.104.0
user:           admin
org:            system
space:          test-space



$ cf push -p ../tests/smoke/assets/test-node-app/ test-node-app
Pushing app test-node-app to org system / space test-space as admin...
Packaging files to upload...
Uploading files...
 558 B / 558 B [============================================================] 100.00% 1s

Waiting for API to complete processing files...
.......
.......
Build successful

Waiting for app test-node-app to start...

Instances starting...
Instances starting...

name:                test-node-app
requested state:     started
isolation segment:   placeholder
routes:              test-node-app.apps.system.domain
last uploaded:       Thu 30 Sep 07:04:54 UTC 2021
stack:               
buildpacks:          
isolation segment:   placeholder

type:            web
sidecars:        
instances:       1/1
memory usage:    1024M
start command:   node server.js
  state     since                  cpu    memory   disk     details
#0   running   2021-09-30T07:06:01Z   0.0%   0 of 0   0 of 0   



$ curl -k https://test-node-app.apps.system.domain
Hello World

```

<br>
  
### <div id='3.8.1'> ※ (Refer) Sidecar Deletion
```
$ source delete-sidecar.sh
```

<br>

  
### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [PaaS-TA Sidecar Install](./README.md) > Sidecar

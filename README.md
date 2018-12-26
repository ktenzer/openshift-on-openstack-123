# General
The purpose of this project is to provide a simple, yet flexible deployment of OpenShift on OpenStack using a three step process. This guide assumes you are familiar with OpenStack.

# Contribution
If you want to provide additional features, please feel free to contribute via pull requests or any other means.
We are happy to track and discuss ideas, topics and requests via 'Issues'.

# Releases
For each release of OpenShift a release branch will be created. Starting with OpenShift 3.9 we will follow the OpenShift release version so it is easy to tell what release branch goes with OpenShift version. The installer support OpenShift Enterprise and OKD starting with 3.11. Note: CentOS OKD rpms are released after enterprise so you may have to wait a bit for OKD.

* release-3.9 OpenShift 3.9
* release-3.10 OpenShift 3.10
* release-3.11 OpenShift 3.11

In addition I would like to metion I borrowed a lot of ideas from two other projects.
* [OpenShift setup for Hetzner from RH SSA team](https://github.com/RedHat-EMEA-SSA-Team/hetzner-ocp)
* [OpenShift on OpenStack](https://github.com/redhat-openstack/openshift-on-openstack)

# Pre-requisites
* Working OpenStack deployment. Tested is OpenStack 12 & 13 (Pike & Queens) using RDO.
* RHEL 7 image. Tested is RHEL 7.4, 7.5, 7.6.
* An openstack ssh key for accessing instances (Default /root/admin.pem, can be overridden).
* A provider (public) network with at least two or three available floating ips.
* A service (internal) network
* A router configured with the public network as gateway and internal network as interface.
* Flavors configured for OpenShift. These are only recommendations.
  * ocp.master  (2 vCPU, 4GB RAM, 30 GB Root Disk)
  * ocp.infra   (4 vCPU, 16GB RAM, 30 GB Root Disk)
  * ocp.node    (2 vCPU, 4GB RAM, 30 GB Root Disk)
  * ocp.bastion (1 vCPU, 4GB RAM, 30 GB Root Disk)
* Properly configured cinder and nova storage.
  * Make sure you aren't using default loop back and have disabled disk zeroing in cinder/nova for LVM.

More information on setting up proper OpenStack environment can be found [here](https://keithtenzer.com/2018/07/17/openstack-13-queens-lab-installation-and-configuration-guide-for-hetzner-root-servers/).

# Tested Deployments
```Single Master - Non HA```

Single Master deployment is 1 Master, 1 Infra node and X number of App nodes. This configuration is a non-HA setup, ideal for test environments.
![](images/openshift_on_openstack_non_ha.PNG)

```Multiple Master - HA```

Multiple Master deployment is 3 Master, 2 Infra node and X number of App nodes. This configuration is an HA setup. By default etcd and registry are not using persistent storage. This would need to be configured post-install manually at this time if those should be persisted.
![](images/openshift_on_openstack_ha.PNG)

# OpenStack Pre-Configuration (if required)

```

## Setup New OpenStack Project

Create Project

```
# openstack project create openshift
```

Add admin user as admin to project

```
# openstack role add --project openshift --user admin admin
```` 

Increase project quota for security groups

```
# openstack quota set --secgroups 100 openshift
```

Increase quota for volumes

```
#  openstack quota set --volumes 100 openshift
```

## Setup Internal Network

Create internal network

```
# openstack network create openshift -- project openshift
```

Create internal subnet

```
# openstack subnet create --network openshift --allocation-pool \
start=192.168.4.100,end=192.168.4.200 --dns-nameserver 213.133.98.98 \
--subnet-range 192.168.4.0/24 openshift_subnet
```

Add internal network to router as interface

```
# openstack router add subnet router1 openshift_subnet
```

# OpenShift Authentication
This deployment will configure authentication through OpenStack keystone. This means you need to create users in OpenStack so they are available to OpenShift. All users that successfully authenticate from OpenShift to Keystone will be allowed to login. They will only have permissions to create new projects and not see anyone elses projects. If you would like something else you can configure the inventory file manually before deploying OpenShift cluster.

Create OpenStack User for OpenShift
```
# openstack user create --project admin --password <password> <username>
```

# Install
![](images/one.png)

```[OpenStack Controller]```

Install Git & Ansible
```
# yum install -y git ansible
```

Clone Git Repository
```
# git clone https://github.com/ktenzer/openshift-on-openstack-123.git
```

Change dir to repository
```
# cd openshift-on-openstack-123
```

Checkout release branch 3.11
```
# git checkout release-3.11
```

Configure Parameters
```
# cp sample-vars.yml vars.yml
```
```
# vi vars.yml
---
### OpenStack Setting ###
openstack_user: admin
openstack_passwd: <password>
openstack_auth_url: <openstack auth url>
openstack_project: openshift
domain_name: ocp3.lab
external_network: public
service_network: openshift
service_subnet_id: <openshift_subent id>
image: rhel76
ssh_user: cloud-user
ssh_key_path: /root/admin.pem
ssh_key_name: admin
stack_name: openshift
openstack_release: queens
openstack_version: "13"
contact: admin@ocp3.lab
heat_template_path: /root/openshift-on-openstack-123/heat/openshift_single_lbaas.yaml

### OpenShift Settings ###
openshift_deployment: openshift-enterprise
openshift_version: "3.11"
docker_version: "1.13.1"
openshift_ha: false
registry_replicas: 1
openshift_user: admin
openshift_passwd: <password>

### Red Hat Subscription ###
subscription_use_username: True
rhn_username_or_org_id: <user or org id>
rhn_password_or_activation_key: <password or activation key>
rhn_pool: <pool>

### OpenStack Instance Count ###
master_count: 1
infra_count: 1
node_count: 3

### OpenStack Instance Group Policies ###
### Set to 'anti-affinity' if running on multiple compute node ###
master_server_group_policies: "['soft-anti-affinity']"
infra_server_group_policies: "['soft-anti-affinity']"
node_server_group_policies: "['soft-anti-affinity']"

### OpenStack Instance Flavors ###
bastion_flavor: ocp.bastion
master_flavor: ocp.master
infra_flavor: ocp.infra
node_flavor: ocp.node
```

Note: If you want to run a single load balancer (to save floating ips) for masters and infra, instead of default two use following heat template ```heat_template_path: /root/openshift-on-openstack-123/heat/openshift_single_lbaas.yaml```.

# Step 1: Deploy OpenStack Infrastructure for OpenShift
```
# ./01_deploy-openstack-infra.yml
```

![](images/two.png)

Get ip address of the bastion host.
```
# openstack stack output show openshift --all | grep -A1 '"name": "bastion"'

| "name": "bastion",
| "address": "1.2.3.4" 

```

SSH to the bastion host using cloud-user and key.
```
ssh -i /root/admin.pem cloud-user@1.2.3.4
```

```[Bastion Host]```

Change dir to repository
```
# cd openshift-on-openstack-123
```

# Step 2: Prepare the nodes for deployment of OpenShift.
```
[cloud-user@bastion ~]$ ./02_prepare-openshift.yml

PLAY RECAP *****************************************************************************************
bastion                    : ok=15   changed=7    unreachable=0    failed=0
infra0                     : ok=18   changed=13   unreachable=0    failed=0
infra1                     : ok=18   changed=13   unreachable=0    failed=0
localhost                  : ok=7    changed=6    unreachable=0    failed=0
master0                    : ok=18   changed=13   unreachable=0    failed=0
master1                    : ok=18   changed=13   unreachable=0    failed=0
master2                    : ok=18   changed=13   unreachable=0    failed=0
node0                      : ok=18   changed=13   unreachable=0    failed=0
node1                      : ok=18   changed=13   unreachable=0    failed=0
```

![](images/three.png)

```[Bastion Host]```
# Step 3: Install and Configure OpenShift Cluster

```
[cloud-user@bastion ~]$ ansible-playbook -i /home/cloud-user/openshift-inventory -vv /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
PLAY RECAP *****************************************************************************************
infra0.ocp3.lab            : ok=61   changed=15   unreachable=0    failed=0
localhost                  : ok=11   changed=0    unreachable=0    failed=0
master0.ocp3.lab           : ok=73   changed=15   unreachable=0    failed=0
node0.ocp3.lab             : ok=61   changed=15   unreachable=0    failed=0


INSTALLER STATUS ***********************************************************************************
Initialization             : Complete (0:04:16)
```

```
[cloud-user@bastion ~]$ ansible-playbook -i /home/cloud-user/openshift-inventory -vv /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml

PLAY RECAP *****************************************************************************************
infra0                     : ok=118  changed=20   unreachable=0    failed=0
localhost                  : ok=11   changed=0    unreachable=0    failed=0
master0                    : ok=715  changed=237  unreachable=0    failed=0
node0                      : ok=118  changed=21   unreachable=0    failed=0


INSTALLER STATUS ***********************************************************************************
Initialization               : Complete (0:00:32)
Health Check                 : Complete (0:00:01)
Node Bootstrap Preparation   : Complete (0:09:24)
etcd Install                 : Complete (0:01:06)
Master Install               : Complete (0:06:04)
Master Additional Install    : Complete (0:01:51)
Node Join                    : Complete (0:00:37)
Hosted Install               : Complete (0:00:49)
Cluster Monitoring Operator  : Complete (0:01:15)
Web Console Install          : Complete (0:00:31)
Console Install              : Complete (0:00:27)
metrics-server Install       : Complete (0:00:00)
Service Catalog Install      : Complete (0:01:57)
```

Login in to UI.
```
https://openshift.144.76.134.226.xip.io:8443
```

# OKD Installation (in case you aren't doing OpenShift Entrerprise)
OKD formally called OpenShift Origin (community version) is also supported starting with release-3.11 branch. To use OKD make sure you have a centos 7.5 image and set 'openshift_deployment=origin' in the vars file.

Once you have run the 01_deploy-openstack-infra.yml and 03_prepare-openshift.yml playbooks as documented above run the following to install openshift OKD from bastion.

Prerequisites playbook
```
[centosr@bastion ~] ansible-playbook -i /home/centos/openshift-inventory openshift-ansible/playbooks/prerequisites.yml
```

Deploy cluster playbook
```
[centosr@bastion ~] ansible-playbook -i /home/centos/openshift-inventory openshift-ansible/playbooks/deploy_cluster.yml
```

# Optional Section
# Configure Admin User
Configure admin user
```
[cloud-user@bastion ~]$ ssh master0
```

Authenticate as system:admin user.
```
[cloud-user@master0 ~]$ oc login -u system:admin -n default
```

Make user OpenShift Cluster Administrator
```
[cloud-user@master0 ~]$ oc adm policy add-cluster-role-to-user cluster-admin admin
```

# Install Metrics

Note: Metrics is integrated with OpenShift UI and will be depricated in 4.0 but for 3.11 if you want metrics in UI it is still needed.

Set metrics to true in inventory
```
[cloud-user@bastion ~]$ vi openshift_inventory
...
openshift_hosted_metrics_deploy=true
...
```

Run playbook for metrics in OpenShift
```
[cloud-user@bastion ~]$ ansible-playbook -i /home/cloud-user/openshift-inventory -vv /usr/share/ansible/openshift-ansible/playbooks/openshift-metrics/config.yml
PLAY RECAP *****************************************************************************************
infra0.ocp3.lab            : ok=0    changed=0    unreachable=0    failed=0
localhost                  : ok=11   changed=0    unreachable=0    failed=0
master0.ocp3.lab           : ok=217  changed=47   unreachable=0    failed=0
node0.ocp3.lab             : ok=0    changed=0    unreachable=0    failed=0


INSTALLER STATUS ***********************************************************************************
Initialization             : Complete (0:01:34)
Metrics Install            : Complete (0:04:37)
```

# Install Logging
Set logging to true in inventory
```
[cloud-user@bastion ~]$ vi openshift_inventory
...
openshift_hosted_logging_deploy=true
...
```

Run Playbook for logging in OpenShift
```
[cloud-user@bastion ~]$ ansible-playbook -i /home/cloud-user/openshift-inventory -vv /usr/share/ansible/openshift-ansible/playbooks/openshift-logging/config.yml
```

# Operator Framework (tech preview in 3.11)
```
[cloud-user@bastion ~]$ vi openshift_inventory
...
openshift_additional_registry_credentials=[{'host':'registry.connect.redhat.com','user':'<your_user_name>','password':'<your_password>','test_image':'mongodb/enterprise-operator:0.3.2'}]
...
```

Reconfigure registry auth
```
[cloud-user@bastion ~]$ ansible-playbook -i /home/cloud-user/openshift-inventory -vv /usr/share/ansible/openshift-ansible/playbooks/updates/registry_auth.yml
PLAY RECAP ********************************************************************************************************************************************************************************************************
infra0                     : ok=27   changed=1    unreachable=0    failed=0   
infra1                     : ok=27   changed=1    unreachable=0    failed=0   
localhost                  : ok=13   changed=0    unreachable=0    failed=0   
master0                    : ok=48   changed=2    unreachable=0    failed=0   
master1                    : ok=48   changed=2    unreachable=0    failed=0   
master2                    : ok=65   changed=2    unreachable=0    failed=0   
node0                      : ok=27   changed=1    unreachable=0    failed=0   
node1                      : ok=27   changed=1    unreachable=0    failed=0   
node2                      : ok=27   changed=1    unreachable=0    failed=0   


INSTALLER STATUS **************************************************************************************************************************************************************************************************
Initialization  : Complete (0:03:08)
```

Deploy Operator Framework
```
[cloud-user@bastion ~]$ ansible-playbook -i /home/cloud-user/openshift-inventory -vv /usr/share/ansible/openshift-ansible/playbooks/olm/config.yml
PLAY RECAP ********************************************************************************************************************************************************************************************************
infra0                     : ok=0    changed=0    unreachable=0    failed=0   
infra1                     : ok=0    changed=0    unreachable=0    failed=0   
localhost                  : ok=11   changed=0    unreachable=0    failed=0   
master0                    : ok=27   changed=0    unreachable=0    failed=0   
master1                    : ok=27   changed=0    unreachable=0    failed=0   
master2                    : ok=68   changed=19   unreachable=0    failed=0   
node0                      : ok=0    changed=0    unreachable=0    failed=0   
node1                      : ok=0    changed=0    unreachable=0    failed=0   
node2                      : ok=0    changed=0    unreachable=0    failed=0   


INSTALLER STATUS **************************************************************************************************************************************************************************************************
Initialization  : Complete (0:01:30)
OLM Install     : Complete (0:00:47)
```

# Openshift disconnected install (optionally)
Disconnected installation requires two Swift containers, which are used to store Docker images and RHEL repositories.
```01_deploy-openstack-infra.yml``` playbook will do the following:
* Install httpd on Bastion server
* Install and configure Rclone, mount ```openshift_rhn_repo``` container  via systemd mount script to ```/var/www/html/repo```
* Generate CA and SSL certificate for Docker private registry.
* Setup Docker and configure Docker private registry to use Swift container ```openshift_rhn_registry``` as backend storage.
* Replicate latest RHEL packages from required repositories (6GB), and sync them to ```openshift_rhn_repo``` container.
* Download all required Openshift Docker images (6GB), re-tag them and push to private Docker registry running on Bastion.

note: replication of RHEL repositories and Openshift Docker images will only happen once, if data exists in Swift containers this steps will be skipped.

Set variables in vars.yml
```
bastion_repo: True
```
Create two required Swift containers
```
swift_rhn_repo_container_name: openshift_rhn_repo
swift_rhn_registry_container_name: openshift_rhn_registry
```

Run below playbook after running ```01_deploy-openstack-infra.yml```
```
# ./02_bastion-repo.yml
```
Continue with step 2.


# Issues
## Issue 1: Dynamic storage provisioning using cinder not working
Currently using the OpenStack cloud provider requires using Cinder v2 API. Most current OpenStack deployments will default to v3.
```
Error creating cinder volume: BS API version autodetection failed.
```
If you provision OpenShift volume and it is pending check /var/log/messages on master. If you see this error you need to add following in /etc/origin/cloudprovider/openstack.conf on masters and all nodes then restart node service on node and controller service on master.
```
...
[BlockStorage]
bs-version=v2
...
```

The post-openshift.yml playbook takes care of setting v2 for cinder automatically.

## Issue 2: Hosted Install Fails

The registry sometimes fails to complete install due to host resolution of xip.io. Not sure if this is issue in OpenShift 3.7 or environment. Simply re-running hosted playbook resolved the issue and resulted in successful installation.

## Issue 3: Ansible 2.7 causes control plane to not start

Don't use ansible 2.7 with OpenShift or OKD 3.11 there is an issue where etcd playbooks won't run which leads to control plane not starting. Use 2.6, this is tested and working. This deployment will force 2.6 so 2.7 doesnt end up on systems by accident.

# Pre-requisites

## Overall infrastructure

The automated process with Vagrant/Ansible provisions and configures the CD pipeline and the application.


```
.     "Dojo"                 "DojoLabs"
  Resource Group           Resource Group

+---------------+ Peering +--------------+
|   Dojo|VNet   +----------|DojoLabsVNet |
+---------------+         +--------------+
                     |
                     |                      +
+---------------+    |    +--------------+  |  +-------------+
| DojoInstaller |    |    |elasticsearch |  |  |dbserver     |
|      VM       |    |    |              |  |  |             |
|               |    |    |              |  |  |             |
+---------------+    |    +--------------+  |  +-------------+
                     |                      |
                     |    +--------------+  |  +-------------+   +--------------+
                     |    |reposerver    |  |  |appserver1   |   |appserver3    |
                     |    |              |  |  |(monolithic) |   |(microservice)|
                     |    |              |  |  |             |   |              |
                     |    +--------------+  |  +-------------+   +--------------+
                     |                      |
                     |    +--------------+  |  +-------------+   +--------------+
                     |    |buildserver   |  |  |appserver2   |   |appserver4    |
                     |    |              |  |  |(monolithic) |   |(microservice)|
                     |    |              |  |  |             |   |              |
                     |    +--------------+  |  +-------------+   +--------------+
                     |                      |
+--------------------+----------------------+---------------------------------------
    Provisioning            CD Pipeline                  Application

```

The DojoInstaller VM is permanent and is used as a controller for the rest
of the provisioning process.
Although designed originally with Vagrant/Virtualbox, the process is now leveraging
Microsoft Azure public cloud.

## Installing


### DojoInstaller VM

- On Azure, create an Ubuntu 16.10 VM (Standard_A1 or Standard_A0).
- Create the VM in location "Central US" (centralus).
- :exclamation: If you can't use "centralus" location, make sure to update
[`pre-requisites/params.json`](pre-requisites/params.json), [`pre-requisites/pre-requisites.sh`](pre-requisites/pre-requisites.sh) and [`Vagrantfile`](Vagrantfile) to update the location.
- Make sure that the VirtualNetwork is called "Dojo-vnet" (or update the
  [params.json](pre-requisites/params.json) file)
- Create an `install` directory
```
mkdir install
cd install
```
- Install Vagrant
```
sudo apt-get update
sudo apt-get install moreutils
wget https://releases.hashicorp.com/vagrant/1.8.6/vagrant_1.8.6_x86_64.deb
dpkg -i vagrant_1.8.6_x86_64.deb
vagrant box add azure https://github.com/azure/vagrant-azure/raw/v2.0/dummy.box
```
- Install Vagrant plugins:
```
vagrant plugin install vagrant-hostsupdater
vagrant plugin install vagrant-proxyconf
```
- Install the custom Azure provider for Vagrant:
```
git clone https://github.com/ojacques/vagrant-azure.git
cd vagrant-azure
sudo apt-get install ruby
gem build vagrant-azure.gemspec
vagrant plugin install ./vagrant-azure-2.0.0.pre1.dojo.gem
cd ..
```
- Install Ansible >=2.1
```
sudo apt-get install software-properties-common
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible
```
- Clone the repository
```
git clone https://github.com/devops-dojo/the-app.git
cd the-app/vagrant
```
- Create Azure cloud credentials .env file

  You need a `.env`  file which includes your Azure details in `~/the-app/vagrant`.
This file is sourced by several install scripts.

```
# Include your Azure details
export AZURE_SUBSCRIPTION_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export AZURE_TENANT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export AZURE_CLIENT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export AZURE_CLIENT_SECRET="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

export VAGRANT_LOG=error
```

:metal::smile::metal:
You're done installing the DojoInstaller VM. Let's provision our Dojo
lab using vagrant!

## Provision DojoLabs

### Setting up DojoLabs resource group

`pre-requisites.sh` will:
- Create DojoLabs resource Group
- add a virtual network
- Peer that virtualnetwork with the one Dojo-vnet  virtual network in the
Dojo resource group.

```
cd pre-requisites
pre-requisites.sh
cd ..
```

### Provision application and Pipeline

Launch the script which starts Vagrant to provision the VMs, and Ansible
to configure them.
```
./azure_start.sh
```

:clock2: The process will take between 1 and 2 hours (serial). Plan in advance.

In case a step fails in the Ansible playbook, you can run it again using this command:

```
/usr/bin/ansible-playbook --connection=ssh --timeout=30 \
   --extra-vars=ansible_ssh_user='vagrant' --inventory-file=provision/hosts \
   -v --private-key=~/.vagrant.d/insecure_private_key \
   --limit=ci-node provision/buildserver.yml
```

### Nodes

Here is a list of nodes, with a link when you can access them from the public
internet.

Note that 10.x.x.x IP addresses cannot be reached, except from the provisioning VM (thanks to virtual network peering) or any other VM in DojoLabsVNet.

To `ssh` to a box from the provisioning VM, just type `ssh vagrant@10.211.55.200`
or `vagrant ssh buildserver`.


Vagrant-Name  | IP            | Hostname           | Application                 | Forward
--------------|---------------|--------------------|-----------------------------|--------------------------------------------------------------------
buildserver   | 10.211.55.200 | ci-node            | Jenkins                     | http://ci-node.eastus.cloudapp.azure.com:8080/
reposerver    | 10.211.55.201 | ci-repo            | Artifact Repository (NGINX) | http://ci-repo.eastus.cloudapp.azure.com
dbserver      | 10.211.55.202 | mongodb-node       | MongoDB                     | localhost:27017
dbserver      | 10.211.55.202 | redis-node         | Redis                       | localhost:6379
appserver1    | 10.211.55.101 | app-server-node-1  | Legacy Shop                 | http://app-server-node-1.eastus.cloudapp.azure.com:8080/shop/
appserver1    | 10.211.55.101 | app-server-node-1  | Probe                       | http://app-server-node-1.eastus.cloudapp.azure.com:8080/probe/ (admin / topsecret)
appserver2    | 10.211.55.102 | app-server-node-2  | Legacy Shop                 | http://app-server-node-2.eastus.cloudapp.azure.com:8080/shop/
appserver2    | 10.211.55.102 | app-server-node-2  | Probe                       | http://app-server-node-2.eastus.cloudapp.azure.com:8080/probe/ (admin / topsecret)
appserver3    | 10.211.55.103 | app-server-node-3  | Microservice Shop           | http://app-server-node-3.eastus.cloudapp.azure.com/
appserver4    | 10.211.55.104 | app-server-node-4  | Microservice Shop           | http://app-server-node-4.eastus.cloudapp.azure.com/
elasticsearch | 10.211.55.100 | monitoring-node    | Kibana                      | http://monitoring-node.eastus.cloudapp.azure.com/
elasticsearch | 10.211.55.100 | monitoring-node    | Nagios                      | http://monitoring-node.eastus.cloudapp.azure.com/nagios3/ (nagiosadmin / admin123)
elasticsearch | 10.211.55.100 | monitoring-node    | Icinga                      | http://monitoring-node.eastus.cloudapp.azure.com/icinga/ (icingaadmin / admin123)

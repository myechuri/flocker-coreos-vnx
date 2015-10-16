Run Flocker with VNX driver in CoreOS Vagrant env on Ubuntu 14.04.

# Flocker on CoreOS quickstart
## Step 0: Get Vagrant, VirtualBox, NFS

Default Vagrant install on Ubuntu 14.04 gets version 1.4.3-1, which is unsuitable for CoreOS Vagrant VM (which requires at least 1.6.0). Get Vagrant 1.6.5.

```
sudo wget --no-check-certificate https://dl.bintray.com/mitchellh/vagrant/vagrant_1.6.5_x86_64.deb
sudo dpkg -i vagrant_1.6.5_x86_64.deb
```

Get VirtualBox:

```
sudo apt-get install virtualbox
```


Get nfs:
```
sudo apt-get install nfs-kernel-server
```



## Step 1: provision a CoreOS node

### Setup PCI Passthrough for FC

Run ``lspci -nn | grep Fibre`` on the host to find PCI address of the host's FC card:

```
root@sclf200:~/flocker-coreos-vnx# lspci -nn | grep Fibre
81:00.0 Fibre Channel [0c04]: Emulex Corporation Saturn-X: LightPulse Fibre Channel Host Adapter [10df:f100] (rev 03)
81:00.1 Fibre Channel [0c04]: Emulex Corporation Saturn-X: LightPulse Fibre Channel Host Adapter [10df:f100] (rev 03)
root@sclf200:~/flocker-coreos-vnx#
```

Edit ``Vagrantfile`` to attach these FC HBAs (both MPIO ports) to the VM's guest.

```
          vb.customize ["modifyvm", :id, "--pciattach", "81:00.0@81:00.0"]
          vb.customize ["modifyvm", :id, "--pciattach", "81:00.1@81:00.1"]
```

### Bring up the CoreOS VM
```
vagrant up
vagrant ssh core-01
```

## Step 2: create `cluster.yml` for Flocker nodes

* Install [Unofficial Flocker Tools](https://docs.clusterhq.com/en/latest/labs/installer.html)
    * You will need Unofficial Flocker Tools 0.3 or later for CoreOS support, use `pip show UnofficialFlockerTools` to check the version.

* Pick a node from EC2 to host the control service, label it as the master (see screenshot)

* When you create a `cluster.yml`, you will need the following details from the AWS control panel:

![CoreOS nodes in EC2, highlighting external and internal IP and external DNS name](coreos-aws.png)

The `cluster.yml` should look like this, see screenshot for where to get much of this information from:
```
cluster_name: <descriptive name>
agent_nodes:
 - {public: <node 1 public IP>, private: <node 1 private IP>}
 - {public: <node 2 public IP>, private: <node 2 private IP>}
 - {public: <node 3 public IP>, private: <node 3 private IP>}
control_node: <DNS name of the master node>
users:
 - coreuser
os: coreos
private_key_path: <path on your machine to your .pem file>
agent_config:
  version: 1
  control-service:
     hostname: <DNS name of the master node>
     port: 4524
  dataset:
    backend: "aws"
    region: <region, e.g. us-east-1>
    zone: <zone that the nodes showed up in>
    access_key_id: <your AWS access key>
    secret_access_key: <your AWS secret key>
```
## Step 3: configure root access to nodes

* Run the following script from your computer, replacing the list of nodes with the public IP addresses of the nodes:

```
for X in <node 1 public IP> <node 2 public IP> <node 3 public IP>; do
    ssh -i <path on your machine to your .pem file> core@$X \
        'sudo mkdir /root/.ssh && \
         sudo cp .ssh/authorized_keys /root/.ssh/authorized_keys'
done
```

## Step 4: run flocker-config

* This is a tool from Unofficial Flocker Tools which will deploy the Flocker containers on your nodes:

```
flocker-config cluster.yml
```

* When it's finished, try this to check that all your nodes came up:

```
flocker-volumes list-nodes
```

* Now you can start using Flocker on your CoreOS cluster!

# TODO - demo integration with flocker docker plugin and fleet

# vagrant-kubernetes

What is Vagrant?
Vagrant is a Server Templating open-source tool that helps in automating the creation and management of Virtual Machines. In simpler terms, we can mention all the configurations of the VM creation like disk, CPU, memory, additional tools, and package installation in a single Vagrantfile and run it with a single vagrant up command.

Why use Vagrant for Kubernetes Setup?
Setting up the Kubernetes cluster with Vagrant makes it easy for us to setup a whole new installation of the Kubernetes cluster in our local or any virtual environment in just one go and setting up the cluster in your local itself can save much of your time and you can just test your application in Kubernetes environment without messing up with the real production environment.

If any issue comes within the cluster we can bring it down and change the configuration file and start over again with just single command rest Vagrant will take care. And don’t worry you can bring up and down the cluster as many times as you required this will not change the configuration and deployment done. We can setup the environment in our local system for testing our application and we can bring it down whenever needed don’t worry if you are setting up and down the cluster many times it will not affect the configuration.

Prerequisites
Virtualbox should be installed in your system
Vagrant must be installed on your system.
To access all the scripts and setup files used in this blog follow this repository.

Steps to Setup Kubernetes Cluster using Vagrant.
Create the common bash script for the Kubernetes package installation.
Create a bash script to configure the master node VM
Create another bash script to configure the worker nodes’ VM
Then, create settings.yaml file for the Vagrantfile configuration settings.
Lastly, create the Vagrantfile to create the Kubernetes cluster.
Execute the vagrant commands to launch the Cluster.
Deploy the sample nginx application to the Kubernetes cluster.
Step 1: — Create a Script for the Kubernetes installation on VM machines.
In this step will create a common bash script that will install all the required packages of Kubernetes and configure them on both the master and worker virtual machines.

Firstly, create folder scripts, and inside this folder will save all the scripts that are required in the Kubernetes Cluster setup. Create a script named common.sh and copy the following content and save it.

#!/bin/bash
#
# Common setup for all servers (Control Plane and Nodes)

set -euxo pipefail

# Variable Declaration

# DNS Setting
if [ ! -d /etc/systemd/resolved.conf.d ]; then
 sudo mkdir /etc/systemd/resolved.conf.d/
fi
cat <<EOF | sudo tee /etc/systemd/resolved.conf.d/dns_servers.conf
[Resolve]
DNS=${DNS_SERVERS}
EOF

sudo systemctl restart systemd-resolved

# disable swap
sudo swapoff -a

# keeps the swap off during reboot
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
sudo apt-get update -y
# Install CRI-O Runtime

VERSION="$(echo ${KUBERNETES_VERSION} | grep -oE '[0-9]+\.[0-9]+')"

# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -

sudo apt-get update
sudo apt-get install cri-o cri-o-runc -y

cat >> /etc/default/crio << EOF
${ENVIRONMENT}
EOF
sudo systemctl daemon-reload
sudo systemctl enable crio --now

echo "CRI runtime installed susccessfully"

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubelet="$KUBERNETES_VERSION" kubectl="$KUBERNETES_VERSION" kubeadm="$KUBERNETES_VERSION"
sudo apt-get update -y
sudo apt-get install -y jq

local_ip="$(ip --json a s | jq -r '.[] | if .ifname == "eth1" then .addr_info[] | if .family == "inet" then .local else empty end else empty end')"
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
${ENVIRONMENT}
EOF
This script will install all the required packages, configure DNS, disable the swap space, configure sysctl, and installs a specific version of cri-o runtime, kubeadm, kubectl, and kubelet on all the nodes. If you would like the latest version, remove the version number from the command.

Step 2: — Create a Master Node Configuration script.
To configure the master node will also create a bash script that will only run in the master node(VM). Create the master node configuration script in the same scripts folder and copy the following content and save it file with the named master.sh.

#!/bin/bash
#
# Setup for Control Plane (Master) servers

set -euxo pipefail

NODENAME=$(hostname -s)

sudo kubeadm config images pull

echo "Preflight Check Passed: Downloaded All Required Images"

sudo kubeadm init --apiserver-advertise-address=$CONTROL_IP --apiserver-cert-extra-sans=$CONTROL_IP --pod-network-cidr=$POD_CIDR --service-cidr=$SERVICE_CIDR --node-name "$NODENAME" --ignore-preflight-errors Swap

mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config

# Save Configs to shared /Vagrant location

# For Vagrant re-runs, check if there is existing configs in the location and delete it for saving new configuration.

config_path="/vagrant/configs"

if [ -d $config_path ]; then
  rm -f $config_path/*
else
  mkdir -p $config_path
fi

cp -i /etc/kubernetes/admin.conf $config_path/config
touch $config_path/join.sh
chmod +x $config_path/join.sh

kubeadm token create --print-join-command > $config_path/join.sh

# Install Calico Network Plugin

curl https://raw.githubusercontent.com/projectcalico/calico/v${CALICO_VERSION}/manifests/calico.yaml -O

kubectl apply -f calico.yaml

sudo -i -u vagrant bash << EOF
whoami
mkdir -p /home/vagrant/.kube
sudo cp -i $config_path/config /home/vagrant/.kube/
sudo chown 1000:1000 /home/vagrant/.kube/config
EOF
The above script will configure the master node configuration first it will initialize the Kubernetes cluster with the kubeadm, copies the kube-config, join.sh, and token files to the configs directory, and lastly install the calico network plugin.

Step 3: — Create a Worker Node Configuration script.
Now, create another bash script in the scripts folder with the named node.sh and add the following content.

#!/bin/bash
#
# Setup for Node servers

set -euxo pipefail

config_path="/vagrant/configs"

/bin/bash $config_path/join.sh -v

sudo -i -u vagrant bash << EOF
whoami
mkdir -p /home/vagrant/.kube
sudo cp -i $config_path/config /home/vagrant/.kube/
sudo chown 1000:1000 /home/vagrant/.kube/config
NODENAME=$(hostname -s)
kubectl label node $(hostname -s) node-role.kubernetes.io/worker=worker
EOF
The worker node script will read the join.sh command from the configs shared folder that is created from the master.sh script and join the master node. Also, copied the kubeconfig file to /home/vagrant/.kube location to execute Kubectl commands.

Step 4: — Create the yaml file for the Vagrant file configuration.
In this step, we will create the settings.yml file where all the configurations of the Vagrantfile will be set. In this file, there would be one master node and two worker nodes with 2 CPUs and 4GB RAM. If you want to increase the number of worker nodes then increase the count value. Add the below content in the settings.yml file.

---
# cluster_name is used to group the nodes in a folder within VirtualBox:
cluster_name: Kubernetes Cluster
network:
  # Worker IPs are simply incremented from the control IP.
  control_ip: 10.0.0.10
  dns_servers:
    - 8.8.8.8
    - 1.1.1.1
  pod_cidr: 172.16.1.0/16
  service_cidr: 172.17.1.0/18
nodes:
  control:
    cpu: 2
    memory: 4096
  workers:
    count: 2
    cpu: 2
    memory: 4096

software:
  box: bento/ubuntu-22.04
  calico: 3.25.0
  kubernetes: 1.26.1-00
  os: xUbuntu_22.04
Step 5: — Finally, Create the Vagrantfile for the Kubernetes SetUp.
Lastly, will create the Vagrantfile which will launch three VM’s with the configuration based on the settings.yml file. For every VM we are executing common.sh. We are executing master.sh script in the master node only and executing node.sh in the worker nodes only.


require "yaml"
settings = YAML.load_file "settings.yaml"

IP_SECTIONS = settings["network"]["control_ip"].match(/^([0-9.]+\.)([^.]+)$/)
# First 3 octets including the trailing dot:
IP_NW = IP_SECTIONS.captures[0]
# Last octet excluding all dots:
IP_START = Integer(IP_SECTIONS.captures[1])
NUM_WORKER_NODES = settings["nodes"]["workers"]["count"]

Vagrant.configure("2") do |config|
  config.vm.provision "shell", env: { "IP_NW" => IP_NW, "IP_START" => IP_START, "NUM_WORKER_NODES" => NUM_WORKER_NODES }, inline: <<-SHELL
      apt-get update -y
      echo "$IP_NW$((IP_START)) master-node" >> /etc/hosts
      for i in `seq 1 ${NUM_WORKER_NODES}`; do
        echo "$IP_NW$((IP_START+i)) worker-node0${i}" >> /etc/hosts
      done
  SHELL

  if `uname -m`.strip == "aarch64"
    config.vm.box = settings["software"]["box"] + "-arm64"
  else
    config.vm.box = settings["software"]["box"]
  end
  config.vm.box_check_update = true

  config.vm.define "master" do |master|
    master.vm.hostname = "master-node"
    master.vm.network "private_network", ip: settings["network"]["control_ip"]
    if settings["shared_folders"]
      settings["shared_folders"].each do |shared_folder|
        master.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
      end
    end
    master.vm.provider "virtualbox" do |vb|
        vb.cpus = settings["nodes"]["control"]["cpu"]
        vb.memory = settings["nodes"]["control"]["memory"]
        if settings["cluster_name"] and settings["cluster_name"] != ""
          vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
        end
    end
    master.vm.provision "shell",
      env: {
        "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
        "ENVIRONMENT" => settings["environment"],
        "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
        "OS" => settings["software"]["os"]
      },
      path: "scripts/common.sh"
    master.vm.provision "shell",
      env: {
        "CALICO_VERSION" => settings["software"]["calico"],
        "CONTROL_IP" => settings["network"]["control_ip"],
        "POD_CIDR" => settings["network"]["pod_cidr"],
        "SERVICE_CIDR" => settings["network"]["service_cidr"]
      },
      path: "scripts/master.sh"
  end

  (1..NUM_WORKER_NODES).each do |i|

    config.vm.define "node0#{i}" do |node|
      node.vm.hostname = "worker-node0#{i}"
      node.vm.network "private_network", ip: IP_NW + "#{IP_START + i}"
      if settings["shared_folders"]
        settings["shared_folders"].each do |shared_folder|
          node.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
        end
      end
      node.vm.provider "virtualbox" do |vb|
          vb.cpus = settings["nodes"]["workers"]["cpu"]
          vb.memory = settings["nodes"]["workers"]["memory"]
          if settings["cluster_name"] and settings["cluster_name"] != ""
            vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
          end
      end
      node.vm.provision "shell",
        env: {
          "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
          "ENVIRONMENT" => settings["environment"],
          "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
          "OS" => settings["software"]["os"]
        },
        path: "scripts/common.sh"
      node.vm.provision "shell", path: "scripts/node.sh"
    end

  end
end 
Step 6: — Execute the vagrant up command.
Run the vagrant up command. It will spin up three nodes. One control plane (master) and two worker nodes. Kubernetes installation and configuration happen through the shell script present in the scripts folder created in the previous steps.

vagrant up 
Login to the master node to check the cluster configuration.

vagrant ssh master
Now, list all the cluster nodes to ensure the worker nodes are connected to the master and in a ready state.

kubectl get nodes
Step 7: — Deploy a sample Nginx app and see if you can access it over the NodePort.
Create an Nginx manifest file named deployment.yml and copy the following content.

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          limits:
            cpu: "0.5"
            memory: "256Mi"
          requests:
            cpu: "0.25"
            memory: "128Mi"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
Run the nginx manifest file with the following command.

kubectl apply -f deployment.yml 
After running the command successfully you should be able to access Nginx on any of the node’s IPs on port 32000. For example, http://10.0.0.11:32000


That’s it! You can start deploying and testing other applications.

To shut down the Kubernetes VMs, execute the halt command.

vagrant halt
Whenever you need the cluster, just execute,

vagrant up
To destroy the VMs,

vagrant destroy
Conclusion:
In this article, We have performed, How to setup Kubernetes Cluster in your local environment using Vagrant. By creating multiple script files for the installation of the Kubernetes cluster and lastly, we have also deployed a sample nginx application to test the environment.

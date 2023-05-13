# vagrant-kubernetes

## What is Vagrant?
Vagrant is a Server Templating open-source tool that helps in automating the creation and management of Virtual Machines. In simpler terms, we can mention all the configurations of the VM creation like disk, CPU, memory, additional tools, and package installation in a single Vagrantfile and run it with a single vagrant up command.

## Why use Vagrant for Kubernetes Setup?
Setting up the Kubernetes cluster with Vagrant makes it easy for us to setup a whole new installation of the Kubernetes cluster in our local or any virtual environment in just one go and setting up the cluster in your local itself can save much of your time and you can just test your application in Kubernetes environment without messing up with the real production environment.

If any issue comes within the cluster we can bring it down and change the configuration file and start over again with just single command rest Vagrant will take care. And don’t worry you can bring up and down the cluster as many times as you required this will not change the configuration and deployment done. We can setup the environment in our local system for testing our application and we can bring it down whenever needed don’t worry if you are setting up and down the cluster many times it will not affect the configuration.

## Prerequisites
Virtualbox should be installed in your system
Vagrant must be installed on your system.
To access all the scripts and setup files follow this repository.

## Steps to Setup Kubernetes Cluster using Vagrant.
Create the common bash script for the Kubernetes package installation.
Create a bash script to configure the master node VM
Create another bash script to configure the worker nodes’ VM
Then, create settings.yaml file for the Vagrantfile configuration settings.
Lastly, create the Vagrantfile to create the Kubernetes cluster.
Execute the vagrant commands to launch the Cluster.
Deploy the sample nginx application to the Kubernetes cluster.

### Step 1: — Create a Script for the Kubernetes installation on VM machines.
In this step will create a common bash script that will install all the required packages of Kubernetes and configure them on both the master and worker virtual machines.

Firstly, create folder scripts, and inside this folder will save all the scripts that are required in the Kubernetes Cluster setup. Create a script named common.sh and copy the following content and save it.

This script will install all the required packages, configure DNS, disable the swap space, configure sysctl, and installs a specific version of cri-o runtime, kubeadm, kubectl, and kubelet on all the nodes. If you would like the latest version, remove the version number from the command.

### Step 2: — Create a Master Node Configuration script.
To configure the master node will also create a bash script that will only run in the master node(VM). Create the master node configuration script in the same scripts folder and copy the following content and save it file with the named master.sh.

The above script will configure the master node configuration first it will initialize the Kubernetes cluster with the kubeadm, copies the kube-config, join.sh, and token files to the configs directory, and lastly install the calico network plugin.

### Step 3: — Create a Worker Node Configuration script.
Now, create another bash script in the scripts folder with the named node.sh and add the following content.

The worker node script will read the join.sh command from the configs shared folder that is created from the master.sh script and join the master node. Also, copied the kubeconfig file to /home/vagrant/.kube location to execute Kubectl commands.

### Step 4: — Create the yaml file for the Vagrant file configuration.
In this step, we will create the settings.yml file where all the configurations of the Vagrantfile will be set. In this file, there would be one master node and two worker nodes with 2 CPUs and 4GB RAM. If you want to increase the number of worker nodes then increase the count value. Add the below content in the settings.yml file.

### Step 5: — Finally, Create the Vagrantfile for the Kubernetes SetUp.
Lastly, will create the Vagrantfile which will launch three VM’s with the configuration based on the settings.yml file. For every VM we are executing common.sh. We are executing master.sh script in the master node only and executing node.sh in the worker nodes only.


### Step 6: — Execute the vagrant up command.
Run the vagrant up command. It will spin up three nodes. One control plane (master) and two worker nodes. Kubernetes installation and configuration happen through the shell script present in the scripts folder created in the previous steps.

``vagrant up 
``

Login to the master node to check the cluster configuration.

``vagrant ssh master
``

Now, list all the cluster nodes to ensure the worker nodes are connected to the master and in a ready state.

``kubectl get nodes
``

### Step 7: — Deploy a sample Nginx app and see if you can access it over the NodePort.
Create an Nginx manifest file named deployment.yml 

Run the nginx manifest file with the following command.

``kubectl apply -f deployment.yml 
``

After running the command successfully you should be able to access Nginx on any of the node’s IPs on port 32000. For example, http://10.0.0.11:32000

That’s it! You can start deploying and testing other applications.

To shut down the Kubernetes VMs, execute the halt command.

``vagrant halt
``

Whenever you need the cluster, just execute,

``vagrant up
``

To destroy the VMs,

``vagrant destroy
``

## Conclusion:
In this article, We have performed, How to setup Kubernetes Cluster in your local environment using Vagrant. By creating multiple script files for the installation of the Kubernetes cluster and lastly, we have also deployed a sample nginx application to test the environment.

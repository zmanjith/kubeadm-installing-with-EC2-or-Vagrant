
Deploying Kubeadm

Kubeadm helps to create a cluster for the Kubernets. Kubeadm consists of different components.


	1. First step would be to set up a cluster with one master and two worker servers. Here we are creating through Virtual machines, but we can do it through Physical servers as well.
	2. Next we need to install a Container Runtime in ALL the nodes. Here we install "containerd"
	3. Then, we need to install the Kube Admin tool in ALL Nodes. Here kubeadm helps to boostrap the Kuebrentes solution, installing all the necessary components in the proper order and servers.
	4. Following this we need to initialize the MASTER Server. All the required components to be installed on the Master server.
	
	5. Before joining the Network, we need to initialise the POD Network. Its not normal network, we need to configure the POD network
	6. Then Join the worker nodes to the Master Node.

Now the cluster is ready for deploying the application in the PODS

Provision the Cluster using Vms

For creating a Cluster using three Virtual machines, we need two TOOLS:

	• Oracle Virtual Box --> Hypervisor responsible for running the VM

	• Vagrant --> an automation tool for making the cluster using the VM tool above.

First install the Oracle Virtual Box in your laptop. We can get the latest one from the internet.
Then, install Vagrant from the internet.

We need to have a Vagrantfile for the next step, which will provide the details of creating all the cluster and other details. We can get a Vagrantfile from the git hub repository mentioned in the right side. We can FORK it to our GitHub and access it through the Command Prompt in the laptop.
Go to the Command Prompt and Clone the git hub to a folder.
https://github.com/kodekloudhub/certified-kubernetes-administrator-course

Now, in the command prompt, move to the folder where we have the Vagrantfile ( verify it in the Git repos)
Then type in the command "vagrant up"
This will take some time. As mentioned in the Vagrant file,  here it will provision the Servers and the Clients.
Run vagrant status to see the cluster formed and the status of each node

For EC2 Instance VM

If its EC2 Set all the correct ports for the Security Group

	1. Open the Visual Studio "Git Bash" terminal from the Terminal tab "+" drop down
	2. Make the "PWD" to the directory where our .pem file stored. ( just use "cd" command)
	3. Then change the permission of the .pem file using the " chmod 400 "<name>.pem"
	4. Then use the SSH connect command shown in the EC2 instance "connect" page for "ssh client" tab



Bootstrapping  a Kubernetes cluster using Kubeadm

We have three virtual machines, one Master, controlplane and two workers, node01, and node02
For the "ip addr" command , running in a host, we can see three interfaces, out of that en0S8 is the one which uses for the cluster communication. We can get the IP address of the Host from this interface.

Please execute the below steps 1, 2, 3  in ALL the Nodes

	1. Installing Kubeadm
		a. Search "install kubeadm" wll provide the details in the Kubernetes page.
		b. Check the hardware spec, in the "Before you begin" section
		c. Instructions given, execute it in ALL the nodes:1)update the packages needed to use the Kubernetes repository.  2) publickey of the package of repositories installed earlier 3) Add repository for packages of Kubernetes WE can check it after running this.   apt repolist 4) update "apt" , then install "kubeadm, kubectl, kubelet", then put them on hold. 5) Enable kubelet
	2. Using the Kubadm create a cluster
		a. Install the Container Runtime in all nodes ( since master also will have containers) to manage the containers
		b. Select a supporting CR- here we go with containerd
		c. Install containerd
			i. Sudo apt update;  sudo apt install -y containerd
			ii. Cgroups driver to be installed -- (Cgroups manage the parameter limits of containers like 512mb of RAM etc)
				1) Two different drivers: cgroupfs  & systemd
				2) Any one can be selected, but both Kubelet & CR (containerd) MUST install same.
				3) Kubelet:
					a) Verify the Init system in Kubelet, to be made as "systemd". Use command ps -p 1
					b) For Kube 1.22 or later if we are not mentioning the init system, it will take systemd as the default one for Kubelet
				4) Containerd:
					a) But for the Containerd we need to set that up. Its NOT systemd By default
				If we need to set the Systemd as Init system
					• Create a folder in /etc/containerd  in every nodes
					• Create a conf file to make systemd as the default cgoup driver
					•  containerd config default will show a conf file. 
					• Using the below command, we can create our conf file:
					  containerd config default |  sed  's/SystemdCgroup = false/SystemdCgroup = true/'  | sudo tee  /etc/containerd/config.toml
					• Do a grep and see the SystemcGroup = True
					 cat /etc/containerd/config.toml | grep -I SystemdCgroup -B 50
				( if its not working edit the /etc/containerd/config.yml file)
				5) Edit the confi.toml file sudo vi /etc/containerd/config.toml
				Update the Sandbox Image version from 3.8 to 3.10
				sandbox_image = "registry.k8s.io/pause:3.10"
				
		d. Restart the containerd -- sudo systemctl restart containerd 
		
	3.  Enable IPv4 packet forwarding
		To manually enable IPv4 packet forwarding:
		# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
		# Apply sysctl params without reboot
sudo sysctl --system
		Verify that net.ipv4.ip_forward is set to 1 with:
		sysctl net.ipv4.ip_forward
	
	4. Initializing the MASTER (Controlplane) node. (kubeadm init)
		a. Follow the steps,  if it’s a HA Cluster. **
		b. If it’s NOT a HA cluster, we need to provide the publishing IP address cluster using --apiserver-advertise-address. Its going to the master node IP. ( got it from "ip addr" interface)
		c. --pod-network-cidr --> to specify the subnet or ip range which has to be assigned to the PODs
		d. --cri-socket- usually provide the CRI path. Usually it will automatically detect.
		e. --upload-certs --> all the certificates to be uploaded as well.
		f. This will be the final command:   
		 sudo kubeadm init --apiserver-advertise-address=172.31.2.109 --pod-network-cidr="10.244.0.0/16" --upload-certs
		g. Check the log and see all the architectural components are getting downloaded and created.
	5. Once its done, we have list of instruction to be done like below:
	  mkdir -p $HOME/.kube
	  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	  sudo chown $(id -u):$(id -g) $HOME/.kube/config
	After this, we can try a Kubectl get nodes and it will list the Master node, which may be in "not ready" status
	Now we are instructed to Install a Pod Network add-on.  CNI Plugin like Flannel, or Calcus etc.
		a. Click in the "install Add-Ons" link and select the Flannel link
		b. A Simple apply command will install the flannel
		c.  kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
		d. The default IP CIDR of Flannel is 10.244.0.0/16. If are planning some other range as Pod network, we may have the WGET the above YAML and edit it.
	6. Execut the below command in ALL THE NODES for the Flannel connectivity with the Worker Nodes.
		cat << EOF | sudo tee /etc/modules-load.d/containerd.conf 
		overlay 
		br_netfilter 
		EOF
		
		sudo modprobe overlay
		sudo modprobe br_netfilter
		
	7. Then, execute a command in all the nodes, which we are planning to connect to this master$$  
	 kubeadm join 172.31.2.116:6443 --token y2mfov.a4tb9ol2cv42p4jw \
	        --discovery-token-ca-cert-hash  sha256:861609a37ce39535a8e770885947b7b0635798cfdf411b5bad458585fbcb6d40

NOTE:  In any case , if we miss the Job command given after the Kubeadm init, we can regenerate the token to join more workers to the Cluster using the below command in the Controlplane or master node:

kubeadm token create --print-join-command

This will be given another Join command with the server names , tokens etc.


Check all the Namespaces, especially kube-flannel namespace. Create a pods and check the details.![image](https://github.com/user-attachments/assets/8ed1a7c8-cc4e-4c1a-b03f-bb1fe236c8f0)

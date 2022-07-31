# ansible-to-deploy-kuberenetes
DEPLOY A KUBERNETES CLUSTER USING ANSIBLE
In this article we will take a look at how to deploy a Kubernetes cluster on Ubuntu 18.04 using Ansible Playbooks. I have found Ansible to be a fantastic tool for getting a Kubernetes cluster up and running quickly in my development environment, and now use the Ansible playbooks detailed in this article when I need to stand up a Kubernetes cluster quickly and easily.

For the purposes of this article, we will use Ansible to deploy a small Kubernetes cluster – with one master node, used to manage the cluster, and two worker nodes, which will be used to run our container applications. To achieve this, we will use four Ansible playbooks. These will do the following:

Create a new User Account for use with Kubernetes on each node
Install Kubernetes and containerd on each node
Configure the Master node
Join the Worker nodes to the new cluster
If you are considering using Ansible to deploy Kubernetes already, I will assume you’re already somewhat familiar with both technologies. So, with that said, let’s get straight into the detail.
Before we Deploy Kubernetes using Ansible
Before we can get started, we need a few prerequisites to be in place. This is what we are going to need:

A host with Ansible installed. I’ve written previously about how to install Ansible – also, check out the online documentation! You should also set up an SSH key pair, which will be used to authenticate to the Kubernetes nodes without using a password, allowing Ansible to do it’s thing.
Three servers/hosts to which we will use as our targets to deploy Kubernetes. I am using Ubuntu 18.04, and my servers each have 2GB ram and 2vCPUs. This is fine for my lab purposes, which I use to try out new things using Kubernetes. You need to be able to SSH into each of these nodes as root using the SSH key pair I mentioned above.
With that lot all in place we should be ready to go!

$ mkdir kubernetes
$ cd kubernetes
With that done, we now need to create a hosts file, to tell Ansible how to communicate with the Kubernetes master and worker nodes.

$ vi hosts
The content of the hosts file should look something like the following:

[masters]
master ansible_host=172.31.117.172 ansible_user=root

[workers]
worker1 ansible_host=172.31.120.66 ansible_user=root
worker2 ansible_host=172.31.118.25 ansible_user=root

Listing the master node and the worker nodes in different sections in the hosts file will allow us to target the playbooks at the specfic node type later on.

Finally, with that done, we can test it’s working by doing a Ansible ping:



$ ansible -i hosts all -m ping
master | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
All good! Lets move onto the first playbook.

Creating a Kubernetes user with Ansible Playbook
Our first task in setting up the Kubernetes cluster is to create a new user on each node. This will be a non-root user, that has sudo privileges. It’s a good idea not to use the root account for day to day operations, of course. We can use Ansible to set the account up on all three nodes, quickly and easily. First, create a file in the working directory:

 $ vi users.yml 
Then add the following to the playbook:

- hosts: 'workers, masters'
  become: yes

  tasks:
    - name: create the kube user account
      user: name=kube append=yes state=present createhome=yes shell=/bin/bash

    - name: allow 'kube' to use sudo without needing a password
      lineinfile:
        dest: /etc/sudoers
        line: 'kube ALL=(ALL) NOPASSWD: ALL'
        validate: 'visudo -cf %s'

    - name: set up authorized keys for the kube user
      authorized_key: user=kube key="{{item}}"
      with_file:
        - ~/.ssh/id_rsa.pub
        
        
We’re now ready to run our first playbook. To do so:

$ ansible-playbook -i hosts users.yml
Once done you should see:

deploy ansible kubernetes
Install Kubernetes with Ansible Playbook

Now we’re getting to the fun part! With our user now created, we can move on to installing Kubernetes. Lets dive straight in and have a look at the playbook, which I have named install-k8s.yml:

---
- hosts: "masters, workers"
  remote_user: ubuntu
  become: yes
  become_method: sudo
  become_user: root
  gather_facts: yes
  connection: ssh

  tasks:
     - name: Create containerd config file
       file:
         path: "/etc/modules-load.d/containerd.conf"
         state: "touch"

     - name: Add conf for containerd
       blockinfile:
         path: "/etc/modules-load.d/containerd.conf"
         block: |
               overlay
               br_netfilter

     - name: modprobe
       shell: |
               sudo modprobe overlay
               sudo modprobe br_netfilter


     - name: Set system configurations for Kubernetes networking
       file:
         path: "/etc/sysctl.d/99-kubernetes-cri.conf"
         state: "touch"

     - name: Add conf for containerd
       blockinfile:
         path: "/etc/sysctl.d/99-kubernetes-cri.conf"
         block: |
                net.bridge.bridge-nf-call-iptables = 1
                net.ipv4.ip_forward = 1
                net.bridge.bridge-nf-call-ip6tables = 1

     - name: Apply new settings
       command: sudo sysctl --system

     - name: install containerd
       shell: |
               sudo apt-get update && sudo apt-get install -y containerd
               sudo mkdir -p /etc/containerd
               sudo containerd config default | sudo tee /etc/containerd/config.toml
               sudo systemctl restart containerd

     - name: disable swap
       shell: |
               sudo swapoff -a
               sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

     - name: install and configure dependencies
       shell: |
               sudo apt-get update && sudo apt-get install -y apt-transport-https curl
               curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

     - name: Create kubernetes repo file
       file:
         path: "/etc/apt/sources.list.d/kubernetes.list"
         state: "touch"

     - name: Add K8s Source
       blockinfile:
         path: "/etc/apt/sources.list.d/kubernetes.list"
         block: |
               deb https://apt.kubernetes.io/ kubernetes-xenial main

     - name: install kubernetes
       shell: |
               sudo apt-get update
               sudo apt-get install -y kubelet=1.20.1-00 kubeadm=1.20.1-00 kubectl=1.20.1-00
               sudo apt-mark hold kubelet kubeadm kubectl
               
               
This playbook will run against all three nodes, and will install the containerd runtime (including some pre-requisite configuration, then go onto install Kubernetes, which includes kubelet, kubeadm and kubectl. Run the playbook using the following syntax:

$  ansible-playbook -i hosts install-k8s.yml
There’s quite a lot going on here, so this one will take a little while to run whist the necessary packages are installed on each node. Once done you should see:

PLAY RECAP ****************************************************************************
master                     : ok=13   changed=12   unreachable=0    failed=0
worker1                    : ok=13   changed=12   unreachable=0    failed=0
worker2                    : ok=13   changed=12   unreachable=0    failed=0
Now we’re halfway there!

Creating a Kubernetes Cluster Master Node using Ansible Playbook
Now we should have containerd and Kubernetes installed on all our nodes. The next step is to create the cluster on the master node. This is the master.yml file, which will initialise the Kubernetes cluster on my master node and set up the pod network, using Calico:

- hosts: masters
  become: yes
  tasks:
    - name: initialize the cluster
      shell: kubeadm init --pod-network-cidr=10.244.0.0/16
      args:
        chdir: $HOME
        creates: cluster_initialized.txt

    - name: create .kube directory
      become: yes
      become_user: kube
      file:
        path: $HOME/.kube
        state: directory
        mode: 0755

    - name: copies admin.conf to user's kube config
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/kube/.kube/config
        remote_src: yes
        owner: kube

    - name: install Pod network
      become: yes
      become_user: kube
      shell: kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml 
      args:
        chdir: $HOME
        
    - name: Get the token for joining the worker nodes
      become: yes
      become_user: kube
      shell: kubeadm token create  --print-join-command
      register: kubernetes_join_command

    - debug:
      msg: "{{ kubernetes_join_command.stdout }}"

    - name: Copy join command to local file.
      become: yes
      local_action: copy content="{{ kubernetes_join_command.stdout_lines[0] }}" dest="/tmp/kubernetes_join_command" mode=0777
      
      
      Note, towards the end of the playbook we generate the worker join command and save it to a local file on the Ansible host. We will use this file later to join the worker nodes to the cluster. Before then, execute the masters.yml playbook:

$  ansible-playbook -i hosts master.yml
Once the playbook has finished, we can check the outcome by connecting to the cluster master node using SSH to check the status of the master node:

$ kubectl get nodes
NAME                           STATUS   ROLES                  AGE   VERSION
kube01.test.local              Ready    control-plane,master   13m   v1.20.1
From the output, we can see that the master node’s status is Ready, showing that the cluster has been initialised successfully.

Join Worker Nodes to Kubernetes Cluster using Ansible Playbook
Now we have a Kubernetes cluster initialised, the final step is to join our worker nodes to the cluster. To do so, the final playbook – join-workers.yml – contains the following:

- hosts: workers
  become: yes
  gather_facts: yes

  tasks:
   - name: Copy join command from Ansiblehost to the worker nodes.
     become: yes
     copy:
       src: /tmp/kubernetes_join_command
       dest: /tmp/kubernetes_join_command
       mode: 0777

   - name: Join the Worker nodes to the cluster.
     become: yes
     command: sh /tmp/kubernetes_join_command
     register: joined_or_not

 
This works by copying the file containing the worker join command saved locally earlier to the worker nodes, then it runs the command. Run the playbook with:

$  ansible-playbook -i hosts join-workers.yml
Once the playbook has complete we can check the status of the cluster nodes by again running the following on the cluster master node:

$ kubectl get nodes
NAME                           STATUS   ROLES                  AGE   VERSION
kube02.test.local              Ready    <none>                 80s   v1.20.1
kube01.test.local              Ready    control-plane,master   22m   v1.20.1
kube03.test.local              Ready    <none>                 69s   v1.20.1
Note: It may take a short while for all the nodes to transition to the Ready status.

Summary
In this article you have seen how it is possible to deploy a Kubernetes cluster using Ansible playbooks. You can find the playbook examples I have used here. I think you will agree, using Ansible to deploy a Kubernetes cluster is a great way to get up and running quickly and easily. As always, be sure to check out the official documentation if you are considering using Kubernetes in a production environment.

CODE
0 comment10 
previous post
HOW TO CREATE AN ANSIBLE TEST ENVIRONMENT USING LXD
next post
HOW TO DEPLOY WVD SESSION HOSTS USING TERRAFORM
AD
UK Cloud Servers - AlmaLinux - Centos - Ubuntu and More

 
USEFUL PAGES
About
Advertise With Us
All About the Docker Certified Associate (DCA) Certification
Docker Tutorials
Our Book Recommendations
VCAP-DCA 5 Objectives
VCAP-DCA Study Guide
VCP-NV Objectives Study Guide
VCP-NV Practice Exams
VCP6-CMA Study Guide
VMware Tutorials

 

DISCLAIMER
The information in this weblog is provided as is with no warranties, and confers no rights. This weblog does not represent the thoughts, intentions, plans or strategies of my employer. It is solely my opinion.

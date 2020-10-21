# -*- mode: ruby -*-
# vi: set ft=ruby :

# Add new entries like the last one (of 'worker' type) to create more nodes
servers = [
    {
        :name => "k8s-head",
        :type => "master",
        :box => "ubuntu/bionic64",
        :box_version => "20190411.0.0",
        :eth1 => "192.168.56.21",
        :mem => "8096",
        :cpu => "4"
    },
    {
        :name => "k8s-worker-1",
        :type => "worker",
        :box => "ubuntu/bionic64",
        :box_version => "20190411.0.0",
        :eth1 => "192.168.56.22",
        :mem => "8096",
        :cpu => "2"
    }
]

# This script to install k8s using kubeadm will get executed after a box is provisioned
$configureBox = <<-SCRIPT
    echo -e "Starting box config...\n"

    apt-get update
    echo -e "Installing packages\n"
    apt-get install -y apt-transport-https ca-certificates curl software-properties-common nano docker.io curl
    echo -e "Enabling docker service\n"
    systemctl enable docker.service

    echo -e "Configuring docker cgroupdriver\n"
    echo -e '{\n\t"exec-opts": ["native.cgroupdriver=systemd"],\n\t"log-driver": "json-file",\n\t"log-opts": {\n\t\t"max-size": "100m"\n\t},\n\t"storage-driver": "overlay2"\n}' > /etc/docker/daemon.json
    systemctl daemon-reload
    systemctl restart docker

    # run docker commands as vagrant user (sudo not required)
    usermod -aG docker vagrant

    echo -e "Installing kubeadm\n"
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo -e 'deb http://apt.kubernetes.io/ kubernetes-xenial main' > /etc/apt/sources.list.d/kubernetes.list
    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl

    # kubelet requires swap off
    swapoff -a

    # keep swap off after reboot
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

    echo -e "Configuring kubelet\n"
    IP_ADDR=`ifconfig enp0s8 | grep inet | awk '{print $2}'| cut -f2 -d:`
    # set node-ip
    sudo sh -c 'echo KUBELET_EXTRA_ARGS= >> /etc/default/kubelet'
    sudo sed -i "/^[^#]*KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR --cgroup-driver=systemd" /etc/default/kubelet
    sudo systemctl restart kubelet

    echo -e "Installing helm\n"
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh

    echo -e "Setting up ssh\n"
    sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    sudo service sshd restart

    echo -e "Box config done!"
SCRIPT

$configureMaster = <<-SCRIPT
    echo -e "Configuring master node\n"
    # ip of this box
    IP_ADDR=`ifconfig enp0s8 | grep inet | awk '{print $2}'| cut -f2 -d:`

    echo -e "Installing master node\n"
    HOST_NAME=$(hostname -s)
    kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=172.16.0.0/16

    echo -e "Configuring credentials for normal user\n"
    sudo --user=vagrant mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

    echo -e "Installing pod network addon\n"
    export KUBECONFIG=/etc/kubernetes/admin.conf
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

    echo -e "Creating token for later join operation\n"
    kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
    chmod +x /etc/kubeadm_join_cmd.sh

    echo -e "Done configuring master node\n"
SCRIPT

$configureWorker = <<-SCRIPT
    echo -e "Configuring worker node\n"
    apt-get install -y sshpass
    sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.56.21:/etc/kubeadm_join_cmd.sh .
    sh ./kubeadm_join_cmd.sh
    echo -e "Done configuring worker node\n"
SCRIPT

Vagrant.configure("2") do |config|

    username = "#{ENV['USERNAME'] || `whoami`}"
    config.vm.synced_folder "C:/Users/#{username}/Shared", "/shared"

    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1]

            config.vm.provider "virtualbox" do |v|

                v.name = opts[:name]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]

            end

            config.vm.provision "shell", inline: $configureBox

            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster
            else
                config.vm.provision "shell", inline: $configureWorker
            end

        end

    end

end 

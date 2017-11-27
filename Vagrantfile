# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'open-uri'
require 'yaml'

Vagrant.require_version ">= 1.6.0"


#-----------------------
# PLUGINS CONFIGURATION
#-----------------------
required_plugins = %w(vagrant-ignition vagrant-triggers)

plugins_to_install = required_plugins.select { |plugin| not Vagrant.has_plugin? plugin }
if not plugins_to_install.empty?
  puts "Installing plugins: #{plugins_to_install.join(' ')}"
  if system "vagrant plugin install #{plugins_to_install.join(' ')}"
    exec "vagrant #{ARGV.join(' ')}"
  else
    abort "Installation of one or more plugins has failed. Aborting."
  end
end

GLOBAL_CONFIG_DIR="/etc"
KUBERNETES_CONFIG_DIR="#{GLOBAL_CONFIG_DIR}/kubernetes"
FLANNEL_CONFIG_DIR="#{GLOBAL_CONFIG_DIR}/flannel"

#-----------------------
# VMS CONFIGURATION
#-----------------------
Vagrant.configure("2") do |config|

  config.ssh.insert_key = false
  config.ssh.forward_agent = true

  # coreos stable channel
  config.vm.box = "coreos-stable"
  # coreos download url
  config.vm.box_url = "https://stable.release.core-os.net/amd64-usr/current/coreos_production_vagrant_virtualbox.json"

  config.ignition.enabled = true

  config.vm.provider :virtualbox do |vb|
    vb.check_guest_additions = false
    vb.functional_vboxsf     = false
    vb.gui = false
    vb.memory = 1024
    vb.cpus = 1
    vb.customize ["modifyvm", :id, "--cpuexecutioncap", "100"]
    config.ignition.config_obj = vb
  end

  #-----------------------
  # MASTER CONFIGURATION
  #-----------------------
  config.vm.define "master" do |master|

    # Execute only on 'vagrant up'
    master.trigger.before :up do

      # Transpiling cl.yaml to config.ign
      system "./ct --platform=vagrant-virtualbox < master.cl.yaml > master.config.ign"

      #-----------------------
      # SSL CONFIGURATION
      #-----------------------

      # Fix openssl
      system "sudo rm -f /opt/vagrant/embedded/bin/openssl && sudo ln -sf /usr/bin/openssl /opt/vagrant/embedded/bin/openssl"

      # Root CA
      system "openssl genrsa -out ./ssl/out/ca-key.pem 2048"
      system 'openssl req -x509 -new -nodes -key ./ssl/out/ca-key.pem -days 10000 -out ./ssl/out/ca.pem -subj "/CN=kube-ca"'

      # API
      system "openssl genrsa -out ./ssl/out/apiserver-key.pem 2048"
      system 'openssl req -new -key ./ssl/out/apiserver-key.pem -out ./ssl/out/apiserver.csr -subj "/CN=kube-apiserver" -config ./ssl/conf/openssl.cnf'
      system "openssl x509 -req -in ./ssl/out/apiserver.csr -CA ./ssl/out/ca.pem -CAkey ./ssl/out/ca-key.pem -CAcreateserial -out ./ssl/out/apiserver.pem -days 365 -extensions v3_req -extfile ./ssl/conf/openssl.cnf"

      # Worker
      system 'openssl genrsa -out ./ssl/out/worker-key.pem 2048'
      system 'openssl req -new -key ./ssl/out/worker-key.pem -out ./ssl/out/worker.csr -subj "/CN=worker" -config ./ssl/conf/worker-openssl.cnf'
      system 'openssl x509 -req -in ./ssl/out/worker.csr -CA ./ssl/out/ca.pem -CAkey ./ssl/out/ca-key.pem -CAcreateserial -out ./ssl/out/worker.pem -days 365 -extensions v3_req -extfile ./ssl/conf/worker-openssl.cnf'

      # Cluster Admin
      system 'openssl genrsa -out ssl/out/admin-key.pem 2048'
      system 'openssl req -new -key ./ssl/out/admin-key.pem -out ./ssl/out/admin.csr -subj "/CN=kube-admin"'
      system 'openssl x509 -req -in ./ssl/out/admin.csr -CA ./ssl/out/ca.pem -CAkey ./ssl/out/ca-key.pem -CAcreateserial -out ./ssl/out/admin.pem -days 365'
    end

    # Execute only on 'vagrant destroy'
    master.trigger.after :destroy do
      system "rm -f master.config.ign* "
    end

    master.ignition.path = "master.config.ign"

    master.ignition.hostname = "master"
    master.ignition.drive_name = "config-master"
    ip = "172.17.8.100"
    master.vm.network :private_network, ip: ip
    master.ignition.ip = ip
    
    master.vm.provision :shell, :inline => "mkdir /tmp#{GLOBAL_CONFIG_DIR}", :privileged => false

    master.vm.provision :file, :source => "master/.", :destination => "/tmp#{GLOBAL_CONFIG_DIR}"
    master.vm.provision :file, :source => "ssl/out/ca.pem", :destination => "/tmp#{KUBERNETES_CONFIG_DIR}/ssl/ca.pem"
    master.vm.provision :file, :source => "ssl/out/apiserver.pem", :destination => "/tmp#{KUBERNETES_CONFIG_DIR}/ssl/apiserver.pem"
    master.vm.provision :file, :source => "ssl/out/apiserver-key.pem", :destination => "/tmp#{KUBERNETES_CONFIG_DIR}/ssl/apiserver-key.pem"

    master.vm.provision :shell, :inline => "chown root:root /tmp#{GLOBAL_CONFIG_DIR} -R"    
    master.vm.provision :shell, :inline => "chmod 0600 /tmp#{KUBERNETES_CONFIG_DIR}/ssl/*-key.pem"

    master.vm.provision :shell, :inline => "rsync -a /tmp#{GLOBAL_CONFIG_DIR}/ #{GLOBAL_CONFIG_DIR}/"

    $script = <<-SCRIPT
      set +e

      COUNT=0
      STATUS=1
      while [ $STATUS -ne 0 ]; do
        echo "Waiting for etcd to be ready"
        etcdctl -C=http://localhost:2379 set /coreos.com/network/config '{"Network": "10.10.0.0/16", "Backend":{"Type":"vxlan"}}' 2>/dev/null && break
        STATUS=$?
        COUNT=$(($COUNT+1))
        if [ $COUNT -eq 10 ]; then
          echo "ERROR STARTING ETCD" 1>&2
          exit 1
        fi
        sleep 20
      done

      COUNT=0
      STATUS=1
      while [ $STATUS -ne 0 ]; do
        echo "Waiting for k8s api to be ready"
        curl -s http://127.0.0.1:8080/version 2>/dev/null && break
        STATUS=$?
        COUNT=$(($COUNT+1))
        if [ $COUNT -eq 20 ]; then
          echo "ERROR STARTING K8s API" 1>&2
          exit 1
        fi
        sleep 60
      done

    SCRIPT

    master.vm.provision :shell, :inline => $script

  end

  #-----------------------
  # WORKER CONFIGURATION
  #-----------------------
  config.vm.define "worker" do |worker|

    # Execute only on 'vagrant up'
    worker.trigger.before :up do
      # Transpiling cl.yaml to config.ign
      system "./ct --platform=vagrant-virtualbox < worker.cl.yaml > worker.config.ign"
    end

    # Execute only on 'vagrant destroy'
    worker.trigger.after :destroy do
      system "rm -f worker.config.ign*"
    end

    worker.ignition.path = "worker.config.ign"
    worker.ignition.hostname = "worker"
    worker.ignition.drive_name = "config-node"
    ip = "172.17.8.101"
    worker.vm.network :private_network, ip: ip
    worker.ignition.ip = ip

    worker.vm.provision :shell, :inline => "mkdir /tmp#{GLOBAL_CONFIG_DIR}", :privileged => false
    worker.vm.provision :shell, :inline => "mkdir /tmp/application", :privileged => false
    worker.vm.provision :shell, :inline => "mkdir /tmp/addons", :privileged => false

    worker.vm.provision :file, :source => "worker/.", :destination => "/tmp#{GLOBAL_CONFIG_DIR}"
    worker.vm.provision :file, :source => "application/.", :destination => "/tmp/application"
    worker.vm.provision :file, :source => "ssl/out/.", :destination => "/tmp#{KUBERNETES_CONFIG_DIR}/ssl"

    worker.vm.provision :shell, :inline => "chown root:root /tmp#{GLOBAL_CONFIG_DIR} -R"    
    worker.vm.provision :shell, :inline => "chmod 0600 /tmp#{KUBERNETES_CONFIG_DIR}/ssl/*-key.pem"

    worker.vm.provision :shell, :inline => "rsync -a /tmp#{GLOBAL_CONFIG_DIR}/ #{GLOBAL_CONFIG_DIR}/"

    $script = <<-SCRIPT
      set +e
      cd /tmp

      echo "Downloading kubectl"
      curl -s -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
      chmod +x ./kubectl
      
      ./kubectl config set-cluster default-cluster \
        --server=https://172.17.8.100 \
        --certificate-authority=/tmp#{KUBERNETES_CONFIG_DIR}/ssl/ca.pem

      ./kubectl config set-credentials default-admin \
        --certificate-authority=/tmp#{KUBERNETES_CONFIG_DIR}/ssl/ca.pem \
        --client-key=/tmp#{KUBERNETES_CONFIG_DIR}/ssl/admin-key.pem \
        --client-certificate=/tmp#{KUBERNETES_CONFIG_DIR}/ssl/admin.pem

      ./kubectl config set-context default-system \
        --cluster=default-cluster \
        --user=default-admin
      
      ./kubectl config use-context default-system

      ./kubectl create -f application/app.yaml

      COUNT=0
      STATUS=0
      while [ $STATUS -ne 5 ]; do
        echo "Waiting for nodejs application to be ready"
        STATUS=$(./kubectl get pods |grep 'Running' |wc -l)
        COUNT=$(($COUNT+1))
        if [ $COUNT -eq 30 ]; then
          echo "ERROR DEPLOYING APPLICATION" 1>&2
          exit 1
        fi
        sleep 60
      done

      echo "Please execute this in your host: watch -n1 curl -s http://172.17.8.101:3000"

    SCRIPT

    worker.vm.provision :shell, :inline => $script
  end
end



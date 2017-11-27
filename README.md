# Vagrant + CoreOS + Kubernetes

> **Environment used**

> This vagrant was tested using this environment:

> * O.S: Debian 9.2 (stretch)
> * Kernel: 4.13.0-0.bpo.1-amd64
> * Virtualbox version: 5.1 (5.1.30-118389~Debian~stretch)
> * Vagrant version: 2.0.1


## Steps

* Install Vagrant
* Install Oracle VirtualBox
* Clone this repository

> **IMPORTANT:**

> ALL commands from here **MUST** be executed from within the cloned repository

* run `vagrant up`

> **IMPORTANT:**

> There are a lot of images to download. So, depending on your internet connection, the cluster may take 15 ~ 30 minutes to be ready. Go drink a beer and watch a chapter of your favorite TV series while you are waiting  =)

* Access 172.17.8.101:3000 in your browser to access the simple nodejs application. It will display the current container hostname. As the service is of type "LoadBalancer", each request could be redirected to one of the containers from the Pod. You will need to press `Ctrl + F5` many times to test it.


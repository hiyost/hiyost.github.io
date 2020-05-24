---
title: 在windows上通过vagrant和virtualbox一键搭建k8s集群
tags: 'vagrant, virtualbox, k8s'
date: 2019-12-13 10:15:51
---


更加详细的操作方法请参考宋净超的[github项目](<https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster>)。


### 1、准备工作

#### 安装vagrant和virtualbox

`vagrant`官方下载地址（我使用的版本是2.2.6）： https://www.vagrantup.com/downloads.html

`virtualbox`官方下载地址（我使用的版本是2.2.65.2.14 r123301）： https://www.virtualbox.org/wiki/Download_Old_Builds_5_2

#### 获取虚机镜像

`centos`的`vagrant`镜像地址

http://cloud.centos.org/centos/7/vagrant/x86_64/images/

<!-- more -->

#### vagrant常用命令



```shell
# 添加镜像
vagrant box add CentOS-7-x86_64-Vagrant-1910_01.VirtualBox.box --name centos/7
# 查看镜像
vagrant box list
# 删除镜像
vagrant box remove centos7
# 启动虚拟机
vagrant up
# ssh登录虚拟机
vagrant ssh
# 休眠虚拟机
vagrant suspend
# 关闭虚拟机
vagrant halt
# 销毁虚拟机
vagrant destroy
```





#### k8s二进制包下载地址



1.15版本

```shell
wget https://storage.googleapis.com/kubernetes-release/release/v1.15.0/kubernetes-server-linux-amd64.tar.gz
```





#### 一个典型的vagrant配置



```shell
Vagrant.configure("2") do |config|
  # 使用的vagrant box中的镜像名称
  config.vm.box = "centos7"

  config.vm.box_check_update = false
  # 将安全PC上的当前路径映射到虚机的/vagrant目录中
  config.vm.synced_folder '.', '/vagrant', disabled: false
  # config.vm.network "forwarded_port", guest: 80, host: 8080
  # config.vm.network "forwarded_port", guest: 22, host: 22, host_ip: "127.0.0.1"
  # config.vm.network "private_network", ip: "192.168.33.10"
  # config.vm.network "public_network" bridge: "eth0"
  # 设置网络类型为私有并设置ip地址为192.168.88.88
  # 此处可以有多种配置，可参考https://www.jianshu.com/p/a1bc23bc7892
  config.vm.network "private_network", ip: "192.168.88.88"

  # 启动这个虚拟机
  config.vm.provider "virtualbox" do |vb|
    vb.gui = false

    vb.cpus = "2"
    vb.memory = "4096"
    disk = './data.vdi'
    unless File.exist?(disk)
      # Standard,Fixed,Split2G,Stream,ESX,Formatted
      vb.customize ['createhd', '--filename', disk, '--variant', 'Standard', '--size', 60 * 1024]
      vb.customize ['storageattach', :id,  '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', disk]
    end
  end

  # 虚机起来之后执行的脚本，可以设置yum源，安装docker、go等组件，然后
  config.vm.provision "shell", inline: <<-SHELL
  ## 需要执行的shell命令
  SHELL
end
```





### 2、开始搭建



搭建步骤：

1、PC上安装`virtualbox`和`Vagrant`

2、下载并导入`centos/7`的`Vagrant`[镜像](http://cloud.centos.org/centos/7/vagrant/x86_64/images/)并导入

```shell
vagrant box add CentOS-7-x86_64-Vagrant-1910_01.VirtualBox.box --name centos/7
```

3、`git clone`下载本项目到本地

```shell
git clone https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster.git

# 注：windows的命令行工具可以使用cmder或者git-bash 
```

4、下载k8s 1.15版本的[二进制包](https://storage.googleapis.com/kubernetes-release/release/v1.15.0/kubernetes-server-linux-amd64.tar.gz)并拷贝到本项目的根路径下

```shell
wget https://storage.googleapis.com/kubernetes-release/release/v1.15.0/kubernetes-server-linux-amd64.tar.gz
```

5、在本项目根路径下执行 `vagrant up`即可，执行大约10min

```shell
vagrant up
```

6、想要销毁该集群是执行 `vagrant destroy`即可

```shell
vagrant destroy
```



其他信息请参考宋净超的[github项目](https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster)。





### 3、其他配置

如果你创建的是单机的集群，还需要单独创建`coredns`

#### 3.1、创建`coredns`

```shell
cd /vagrant/addon/dns/
dos2unix dns-deploy.sh
./dns-deploy.sh -r 10.254.0.0/16 -i 10.254.0.2 |kubectl apply -f -
```



#### 3.2、`prometheus`大家族

```shell
cd /vagrant/addon/kube-prometheus/
kubectl create -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl create -f manifests/
```

删除：

```shell
kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
```





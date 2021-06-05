
# 使用Vagrant和VirtualBox在本地搭建多节点的Kubernetes集群

kubernetes社区已经不提供基于Vagrant的多集群部署（默认都是云上环境），这对我们本地开发或调试基于VM的多集群环境很不方便。
我们可以使用[Vagrant](https://www.vagrantup.com/)和[VirtualBox](https://www.virtualbox.org/wiki/Downloads)来创建一个这样的环境。

本项目基于[kubernetes-vagrant-centos-cluster](https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster)优化而来（感谢作者的付出），我在此基础上分别做了如下优化：
- 支持kubernetes社区最新版本（kubernetes v1.22.0）
- 支持在win10环境上基于Vagrant 最新版本的适配
- 简化多余的服务配置


## 准备环境

需要准备以下软件和环境：

- 8G以上内存
- Vagrant 最新版本（推荐2.2.16）
- VirtualBox 5.2
- 仅在windows10下测试通过，**Mac/Linux没调试过，如果有问题，欢迎大家提PR修改**

## 集群

我们使用Vagrant和Virtualbox安装包含2个节点的kubernetes集群，其中master节点同时作为node节点。

| IP           | 主机名   | 组件                                       |
| ------------ | ----- | ---------------------------------------- |
| 172.17.8.201 | node1 | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、kubelet、docker、flannel、dashboard |
| 172.17.8.202 | node2 | kubelet、docker、flannel           |                 |

**注意**：以上的IP、主机名和组件都是固定在这些节点的，即使销毁后下次使用vagrant重建依然保持不变。

容器IP范围：172.33.0.0/30

Service IP范围：10.254.0.0/16

## 安装的组件

安装完成后的集群包含以下组件：

- flannel（`host-gw`模式）
- kubernetes dashboard
- etcd（单节点）
- kubectl
- CoreDNS
- kubernetes（版本根据下载的kubernetes安装包而定，基于Kubernetes1.22+）

**可选插件**

- dashboard
- 其他：待补充（单台PC的内存或CPU资源可能不足以创建很多addon服务）


## 使用说明

- 1.将该repo克隆到本地，下载Kubernetes的到项目的根目录。

```bash
git clone https://github.com/libaoan/k8s-cluster-provider-with-vagrant.git
cd k8s-cluster-provider-with-vagrant
```

**注意**：如果您是第一次运行该部署程序，那么可以直接执行下面的命令，它将自动帮你下载 Kubernetes 安装包，下一次你就不需要
自己下载了，另外您也可以在[这里](https://kubernetes.io/docs/setup/release/notes/)找到Kubernetes的发行版下载地址，下载 Kubernetes发行版后重命名为`kubernetes-server-linux-amd64.tar.gz`，并移动到该项目的根目录下。

- 2.使用vagrant启动集群。

```bash
vagrant up
```

如果是首次部署，会自动下载`centos/7`的box，这需要花费一些时间，另外每个节点还需要下载安装一系列软件包，整个过程大概需要10几分钟。
如果您在运行`vagrant up`的过程中发现无法下载`centos/7`的box，可以手动下载后将其添加到vagrant中。

**手动下载box文件，并修改Vagrantfile配置文件中下面这一行，使之生效**

```
...
# // if you set box_url online, you should set `vm.box_version`
# node.vm.box_version = "1804.02"
# // load box on local
node.vm.box_url = "./CentOS-7-x86_64-Vagrant-1804_02.VirtualBox.box"
...
```

**或者，你不用修改Vagrantfile,vagrant up前手动添加box**

````bash
wget -c http://cloud.centos.org/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-1804_02.VirtualBox.box
vagrant box add ./CentOS-7-x86_64-Vagrant-1804_02.VirtualBox.box --name centos/7
````

这样下次运行`vagrant up`的时候就会自动读取本地的`centos/7` box而不会再到网上下载。

### 访问kubernetes集群

访问Kubernetes集群的方式有三种：

- 本地访问
- 在VM内部访问
- Kubernetes dashboard

**通过本地访问(Host主机是linux或者Mac OS)**

可以直接在你自己的本地环境中操作该kubernetes集群，而无需登录到虚拟机中。

要想在本地直接操作Kubernetes集群，需要在你的电脑里安装`kubectl`命令行工具，对于Mac用户执行以下步骤：

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.22.0-alpha.2/kubernetes-client-darwin-amd64.tar.gz
tar xvf kubernetes-client-darwin-amd64.tar.gz && cp kubernetes/client/bin/kubectl /usr/local/bin
```

将`conf/admin.kubeconfig`文件放到`~/.kube/config`目录下即可在本地使用`kubectl`命令操作集群。

```bash
mkdir -p ~/.kube
cp conf/admin.kubeconfig ~/.kube/config
```

推荐您使用这种方式。

**在虚拟机内部访问**

如果有任何问题可以登录到虚拟机内部调试：

```bash
vagrant ssh node1
sudo -i
kubectl get nodes
```

**Kubernetes dashboard**

还可以直接通过dashboard UI来访问：https://172.17.8.201:8443

可以在本地执行以下命令获取token的值（需要提前安装kubectl）：

```bash
kubectl -n kube-system describe secret `kubectl -n kube-system get secret|grep admin-token|cut -d " " -f1`|grep "token:"|tr -s " "|cut -d " " -f2
```

**注意**：token的值也可以在`vagrant up`的日志的最后看到。


**Windows下Chrome/Firefox访问**

如果提示`NET::ERR_CERT_INVALID`，则需要下面的步骤

进入本项目目录

```
vagrant ssh node1
sudo -i
cd /vagrant/addon/dashboard/
mkdir certs
openssl req -nodes -newkey rsa:2048 -keyout certs/dashboard.key -out certs/dashboard.csr -subj "/C=/ST=/L=/O=/OU=/CN=kubernetes-dashboard"
openssl x509 -req -sha256 -days 365 -in certs/dashboard.csr -signkey certs/dashboard.key -out certs/dashboard.crt
kubectl delete secret kubernetes-dashboard-certs -n kube-system
kubectl create secret generic kubernetes-dashboard-certs --from-file=certs -n kube-system
kubectl delete pods $(kubectl get pods -n kube-system|grep kubernetes-dashboard|awk '{print $1}') -n kube-system #重新创建dashboard
```
刷新浏览器之后点击`高级`，选择跳过即可打开页面。

### 额外部署的组件

[Kubernetes Dashboard](#Kubernetes dashboard)

## 管理

除了特别说明，以下命令都在当前的repo目录下操作。

### 挂起

将当前的虚拟机挂起，以便下次恢复。

```bash
vagrant suspend
```

### 恢复

恢复虚拟机的上次状态。

```bash
vagrant resume
```

注意：我们每次挂起虚拟机后再重新启动它们的时候，看到的虚拟机中的时间依然是挂载时候的时间，这样将导致监控查看起来比较麻烦。因此请考虑先停机再重新启动虚拟机。

### 重启

停机后重启启动。

```bash
vagrant halt
vagrant up
# login to node1
vagrant ssh node1
# run the prosivision scripts
/vagrant/hack/k8s-init.sh
exit
# login to node2
vagrant ssh node2
# run the prosivision scripts
/vagrant/hack/k8s-init.sh
exit
# login to node3
vagrant ssh node3
# run the prosivision scripts
/vagrant/hack/k8s-init.sh
sudo -i
cd /vagrant/hack
./deploy-base-services.sh
exit
```

现在你已经拥有一个完整的基础的kubernetes运行环境，在该repo的根目录下执行下面的命令可以获取kubernetes dashboard的admin用户的token。

```bash
vagrant ssh node1 [or node2]

sh /vagrant/hack/get-dashboard-token.sh
```

根据提示登录即可。

### 清理

清理虚拟机。

```bash
vagrant destroy
rm -rf .vagrant
```

### 注意

仅做开发测试使用，不要在生产环境使用该项目。

## 参考

- [kubernetes-vagrant-centos-cluster](https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster)
- [duffqiu/centos-vagrant](https://github.com/duffqiu/centos-vagrant)
- [coredns/deployment](https://github.com/coredns/deployment)





# 一、预先准备环境
## 1. 准备服务器
这里准备了三台ubuntu虚拟机，每台一核cpu和2G内存，配置好root账户，并安装好了docker，(kube-admin), 后续的所有操作都是使用kube-admin账户。虚拟机具体信息如下表：

| 系统类型 | IP地址 | 节点角色 | CPU | Memory | Hostname |
| :------: | :--------: | :-------: | :-----: | :---------: | :-----: |
| centos 7 | 192.168.242.135 | worker |   2    | 2G | kube-u01.kuber.vmserver |
| centos 7 | 192.168.242.133 | master |   2    | 2G | kuber-u02.kuber.vmserver |
| centos 7 | 192.168.242.134 | worker |   2    | 2G | kuber-u03.kuber.vmserver |

### [Create a Kubernetes cluster from the scratch (On CentOS 7 / RHEL 7) and Deploy an application on the cluster and Expose to Internet — Complete Example PART — 01](https://medium.com/@hashithkarunarathne/create-a-kubernetes-cluster-from-the-scratch-on-centos-7-rhel-7-and-deploy-an-application-on-85a92997b565)

### [Create a Kubernetes cluster from the scratch (On CentOS 7 / RHEL 7) and Deploy an application on the cluster and Expose to Internet — Complete Example PART — 02 (Docker)](https://medium.com/@hashithkarunarathne/create-a-kubernetes-cluster-from-the-scratch-on-centos-7-rhel-7-and-deploy-an-application-on-db5ac06b7404)

### [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

### [Install Kubernetes Cluster with Docker CE on CentOS 7](https://ahmermansoor.blogspot.com/2019/04/install-kubernetes-cluster-docker-ce-centos-7.html)

```bash
sudo yum update -y && sudo yum install -y vim 
```

## 2. 安装docker（所有节点）
一般情况使用下面的方法安装即可
docker安装可参照[Install Docker and Docker Compose (Centos 7)](https://github.com/NaturalHistoryMuseum/scratchpads2/wiki/Install-Docker-and-Docker-Compose-(Centos-7))

#### 2.1 Install needed packages:
```bash
# install dependencies
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# add docker repo
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# install docker-ce
sudo yum -y install docker-ce

# add user to docker group
sudo usermod -aG docker $(whoami)

# set docker start automaticall at boot time
sudo systemctl enable docker.service

# start service
sudo systemctl start docker.service
```
  

遇到问题可以参考：[官方教程][1]

## 3. 系统设置（所有节点）
#### 3.1 关闭、禁用防火墙(让所有机器之间都可以通过任意端口建立连接)
```bash
$ ufw disable
#查看状态
$ ufw status
```
#### 3.2 设置系统参数 - 允许路由转发，不对bridge的数据进行处理
```bash
#写入配置文件
$ cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
 
#生效配置文件
$ sysctl -p /etc/sysctl.d/k8s.conf
```

#### 3.3 配置host文件
```bash
#配置host，使每个Node都可以通过名字解析到ip地址
$ vi /etc/hosts
#加入如下片段(ip地址和servername替换成自己的)
192.168.1.101 server01
192.168.1.102 server02
192.168.1.103 server03
```

## 4. 准备二进制文件（所有节点）
kubernetes的安装有几种方式，不管是kube-admin还是社区贡献的部署方案都离不开这几种方式：
- **使用现成的二进制文件**
> 直接从官方或其他第三方下载，就是kubernetes各个组件的可执行文件。拿来就可以直接运行了。不管是centos，ubuntu还是其他的linux发行版本，只要gcc编译环境没有太大的区别就可以直接运行的。使用较新的系统一般不会有什么跨平台的问题。

- **使用源码编译安装**
>编译结果也是各个组件的二进制文件，所以如果能直接下载到需要的二进制文件基本没有什么编译的必要性了。

- **使用镜像的方式运行**
> 同样一个功能使用二进制文件提供的服务，也可以选择使用镜像的方式。就像nginx，像mysql，我们可以使用安装版，搞一个可执行文件运行起来，也可以使用它们的镜像运行起来，提供同样的服务。kubernetes也是一样的道理，二进制文件提供的服务镜像也一样可以提供。

从上面的三种方式中其实使用镜像是比较优雅的方案，容器的好处自然不用多说。但从初学者的角度来说容器的方案会显得有些复杂，不那么纯粹，会有很多容器的配置文件以及关于类似二进制文件提供的服务如何在容器中提供的问题，容易跑偏。
所以我们这里使用二进制的方式来部署。二进制文件已经这里备好，大家可以打包下载，把下载好的文件放到每个节点上，放在哪个目录随你喜欢，**放好后最好设置一下环境变量$PATH**，方便后面可以直接使用命令。(科学上网的同学也可以自己去官网找找)  
####[下载地址（kubernetes 1.9.0版本）][2] 

## 5. 准备配置文件（所有节点）
上一步我们下载了kubernetes各个组件的二进制文件，这些可执行文件的运行也是需要添加很多参数的，包括有的还会依赖一些配置文件。现在我们就把运行它们需要的参数和配置文件都准备好。
#### 5.1 下载配置文件
```bash
#到home目录下载项目
$ cd
$ git clone https://github.com/liuyi01/kubernetes-starter.git
#看看git内容
$ cd ~/kubernetes-starter && ls
```
#### 5.2 文件说明
- **gen-config.sh**
> shell脚本，用来根据每个同学自己的集群环境(ip，hostname等)，根据下面的模板，生成适合大家各自环境的配置文件。生成的文件会放到target文件夹下。

- **kubernetes-simple**
> 简易版kubernetes配置模板（剥离了认证授权）。
> 适合刚接触kubernetes的同学，首先会让大家在和kubernetes初次见面不会印象太差（太复杂啦~~），再有就是让大家更容易抓住kubernetes的核心部分，把注意力集中到核心组件及组件的联系，从整体上把握kubernetes的运行机制。

- **kubernetes-with-ca**
> 在simple基础上增加认证授权部分。大家可以自行对比生成的配置文件，看看跟simple版的差异，更容易理解认证授权的（认证授权也是kubernetes学习曲线较高的重要原因）

- **service-config**
>这个先不用关注，它是我们曾经开发的那些微服务配置。
> 等我们熟悉了kubernetes后，实践用的，通过这些配置，把我们的微服务都运行到kubernetes集群中。

#### 5.3 生成配置
这里会根据大家各自的环境生成kubernetes部署过程需要的配置文件。
在每个节点上都生成一遍，把所有配置都生成好，后面会根据节点类型去使用相关的配置。
```bash
#cd到之前下载的git代码目录
$ cd ~/kubernetes-starter
#编辑属性配置（根据文件注释中的说明填写好每个key-value）
$ vi config.properties
#生成配置文件，确保执行过程没有异常信息
$ ./gen-config.sh simple
#查看生成的配置文件，确保脚本执行成功
$ find target/ -type f
target/all-node/kube-calico.service
target/master-node/kube-controller-manager.service
target/master-node/kube-apiserver.service
target/master-node/etcd.service
target/master-node/kube-scheduler.service
target/worker-node/kube-proxy.kubeconfig
target/worker-node/kubelet.service
target/worker-node/10-calico.conf
target/worker-node/kubelet.kubeconfig
target/worker-node/kube-proxy.service
target/services/kube-dns.yaml
```
> **执行gen-config.sh常见问题：**
> 1. gen-config.sh: 3: gen-config.sh: Syntax error: "(" unexpected
> - bash版本过低，运行：bash -version查看版本，如果小于4需要升级
> - 不要使用 sh gen-config.sh的方式运行（sh和bash可能不一样哦）
> 2. config.properties文件填写错误，需要重新生成
> 再执行一次./gen-config.sh simple即可，不需要手动删除target

[1]: https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/
[2]: https://pan.baidu.com/s/1bMnqWY

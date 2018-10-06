

##### 1: 前置系统说明


```

提前关闭所有节点的selinux 

1：主机系统：CentOS Linux release 7.5.1804 (Core)
2：系统只配置了IP，并且能联网，其他无任何配置
3：ansible服务器已经和所有节点做了root用户免密码登陆
4: 所有执行均采用root用户
5：github下载地址：git@github.com:wufenglinux/ansible-k8s10x_and_k8s11x.git
6：安装测试通过的K8S版本有：1.10.3、1.10.4、1.10.5、1.11.0、1.11.1，其他版本请大家自行测试
7：本ansible一键安装可用于生产环境

主机名称     	IP	                备注
node01  10.253.1.101	        master1  and etcd1  haproxy+keepalived
node02	10.253.1.102	        master2  and etcd2  haproxy+keepalived
node03	10.253.1.103	        master3  and etcd3  haproxy+keepalived
node04	10.253.1.111	        node1
node05	10.253.1.112	        node2
node06	10.253.1.113	        node3
node07	10.253.1.114	        node4
node08	10.253.1.115	        node5
node09	10.253.1.116	        node6
node10	10.253.1.117	        node7
node11	10.253.1.118	        node8

10.253.1.100            VIP



```

### 2: /etc/ansible/hosts文件解释

```
[slb]
10.253.1.101 name=node01 type=MASTER priority=100
10.253.1.102 name=node02 type=BACKUP priority=90
10.253.1.103 name=node03 type=BACKUP priority=80

[k8s-master]
10.253.1.101 name=node01 order=1
10.253.1.102 name=node02 order=2
10.253.1.103 name=node03 order=3

[k8s-node]
10.253.1.111 name=node04
10.253.1.112 name=node05
10.253.1.113 name=node06
10.253.1.114 name=node07
10.253.1.115 name=node08
10.253.1.116 name=node09
10.253.1.117 name=node10
10.253.1.118 name=node11

[k8s-all:children]
k8s-master
k8s-node

[all:vars] 
local_images=registry.cn-hangzhou.aliyuncs.com/k8sth
k8s_version=1.11.1
vip=10.253.1.100

#type表示keepalived的类型是master或者backp
#priority代表权重，可以自行修改，但是不建议修改，直接修改IP为合适的就行
#name为主机名称，可以自行修改，在系统初始化时会以此添加并配置所有主机的/etc/hosts文件
#order为k8s初始化的顺序，不能修改
#local_images为镜像地址，本人镜像地址包含1.10.0--1.11.1所有的K8S镜像，所以可以不用修改，如果用局域网内部仓库，必须是https的
#k8s_version为需要安装的kubernetes版本号

```
###  3: roles文件说明，

```
[root@ansible roles]# ll
total 16
drwxr-xr-x 7 root root  77 Aug  6 22:50 addnode
drwxr-xr-x 7 root root  77 Jul 31 16:40 basic
drwxr-xr-x 7 root root  77 Aug  6 17:59 docker_kubeadm
drwxr-xr-x 7 root root  77 Aug  6 11:49 etcd
-rw-r--r-- 1 root root 206 Aug  7 21:01 first.yaml
drwxr-xr-x 7 root root  77 Jul 31 16:40 flannel
drwxr-xr-x 7 root root  77 Jul 31 16:40 haproxy
drwxr-xr-x 7 root root  77 Aug  6 21:25 k8s10x
drwxr-xr-x 7 root root  77 Jul 31 16:40 k8s11x
drwxr-xr-x 7 root root  77 Jul 31 16:40 keepalived
drwxr-xr-x 7 root root  77 Aug  6 19:53 kernelup
-rw-r--r-- 1 root root 169 Aug  7 01:13 kernelup.yaml
-rw-r--r-- 1 root root 397 Aug  7 21:00 onekey.yaml
-rw-r--r-- 1 root root 222 Aug  7 21:02 two.yaml

#kernelup.yaml为kernel升级的yaml文件，建议将需要安装k8s的所有节点内核都升级，避乱一些不必要的问题
#onekey.yaml是在所有节点只配置好IP的情况下，一键安装整个K8S集群（ansible控制节点免密码登陆所有节点必须提前配置好）
#first.yaml，two.yaml是onekey.yaml的拆分文件，因为onekey.yaml运行输出信息太多，在定位问题时候不好排查，所以进行了拆分

```


### 4: 执行顺序说明



##### ==4.1: 升级K8S节点的kernel，非必须操作，但是建议==


```
[root@ansible ~]# ansible-playbook /etc/ansible/roles/kernelup.yaml

#执行此命令后，结果会有个报错，这是正常的，因为在kernel升级完成后必须得重启系统才能使用新得kernerl，系统重启后ansible不能接收到init6得执行结果，所以会报错，大家此时可以看看k8s所有节点得kernel是不是最新的
```


##### 4.2： first.yaml
```
执行如下命令

ansible-playbook /etc/ansible/roles/first.yaml  

#命令做了如下操作
#1：所有主机初始化系统，安装基本软件
#2：所有主机关闭selinux、firewalld、关闭swap
#3：所有主机设置主机名称、分发/etc/hosts文件、设置ulimit、开发forward
#4: slb主机组安装配置haproxy、keepalived
#5: master节点安装etcd集群，未采用https的集群


```
#### 4.3： two.yaml

```
执行如下命令

ansible-playbook /etc/ansible/roles/two.yaml

#命令做了如下操作
#1:安装docker,版本为K8S官方推荐的17.3
#2:安装kubeadm版本根据/etc/ansible/hosts来定，yum源采用的是阿里云的
#3:根据k8s版本初始K8S集群，/etc/ansible/hosts的order变量决定初始化顺序，当初始化失败的时候整个初始化集群过程将终止
#4：添加flannel网络，需要使用其他网络的在two.yaml里面将flannel这个role注释掉，在集群安装完成后自行添加
#5：将node节点添加进集群



在主机node01上面的结果：（由于网络下载速度的原因，可能得等个2分钟才能看到结果）

[root@node01 ~]# kubectl get pod --all-namespaces -o wide


```

##### 4.4：安装方法总结

一：拆分安装
  
```
ansible-playbook /etc/ansible/roles/kernelup.yaml
#kernel升级非必须，但是建议升级
ansible-playbook /etc/ansible/roles/first.yaml
ansible-playbook /etc/ansible/roles/two.yaml
```
二：一键安装

```
ansible-playbook /etc/ansible/roles/kernelup.yaml
#kernel升级非必须，但是建议升级
ansible-playbook /etc/ansible/roles/onekey.yaml
```


  

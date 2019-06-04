# distributed-tensorflow

tensorflow on k8s
联网：vim /etc/resolv.conf  加  nameserver 8.8.8.8  nameserver 114.114.114.114
安装Docker
版本：17.03.2
yum  install  docker-ce-17.03.2.ce-1.el7.centos.rpm

坑一：

之后尝试安装docker-ce-selinux

发现要自动下载18.06的包。无果要在网上下载这个文件，推荐清华的镜像网站
docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm
安装这个包的时候发生报错

与系统原有的container-selinux-2.68-1.el7.noarch冲突，所以卸载掉
yum list installed | grep container找到这个包

卸载掉：
yum -y remove container-selinux.noarch
yum install  docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm

安装Nvidia docker 2.0（https://github.com/NVIDIA/nvidia-docker）
需要满足以下前提要求：
1.GNU/Linux x86_64 with kernel version > 3.10
2.Docker >= 1.12
3.NVIDIA GPU with Architecture > Fermi (2.1)
4.NVIDIA drivers ~= 361.93 (untested on older versions)
yum install nvidia-docker2=2.0.1+docker1.12.6-1 nvidia-container-runtime=1.1.0+docker1.12.6-1
1.安装nvidia驱动sudo ./NVIDIA-Linux-x86_64-390.87.run 
报错：
方法：
sudo init 3
sudo    ./NVIDIA-Linux-x86_64-390.87.run     //当前目录下执行NVIDIA驱动程序
按照提示安装完成，简单方法重启就好了     sudo  reboot
报错：The Nouveau kernel driver is currently in use by your system.
解决办法如下：
    也即关闭Nouveau：
    1)把驱动加入黑名单中: /etc/modprobe.d/blacklist.conf 在后面加入：
    blacklist nouveau
    2) 使用 dracut重新建立 initramfs image file :
    * 备份 the initramfs file
    $ sudo mv /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak
    * 重新建立 the initramfs file
    $ sudo dracut -v /boot/initramfs-$(uname -r).img $(uname -r)
    3) 重启系统至文本模式,init 3 这个可以修改/etc/inittab 文件 init 3是文本模式,
    init 5是图形界面模式.重启之后,进入文本模式，其实可以发现字体变大了,也就是说驱动没有被加载,成功禁用了Nouveau
    4)检查nouveau driver确保没有被加载！
    $ lsmod | grep nouveau
    5) 运行安装文件
    sudo ./NVIDIA-Linux-x86_64-390.87.run 
2.安装cuda：yum install cuda-repo-rhel7-9.1.85-1.x86_64.rpm
3.curl -s -L https://nvidia.github.io/nvidia-docker/centos7/x86_64/nvidia-docker.repo | sudo tee /etc/yum.repos.d/nvidia-docker.repo 



4.Yum install nvidia-docker2-2.0.0-1.docker17.03.2.ce
出现报错，需要安装 nvidia-container-runtime-1.0.0-1.docker17.03.2.x86_64.rpm
到github上下载
https://nvidia.github.io/nvidia-container-runtime/centos7/x86_64/nvidia-container-runtime-1.0.0-1.docker17.03.2.x86_64.rpm
安装nvidia-container-runtime-1.0.0-1.docker17.03.2.x86_64.rpm
yum install nvidia-container-runtime-1.0.0-1.docker17.03.2.x86_64.rpm
报错

删除nvidia-container-runtime-hook-1.4.0-2.x86_64
yum remove nvidia-container-runtime-hook-1.4.0-2.x86_64.
安装yum install nvidia-container-runtime-1.0.0-1.docker17.03.2.x86_64.rpm
安装yum install nvidia-docker2-2.0.0-1.docker17.03.2.ce
安装Kubernetes
配置k8s源
cat >> /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

安装kubeadm相关工具（通过kubeadm安装k8s-v1.11）
yum install kubelet-1.11.0-0 kubeadm-1.11.0-0 kubectl-1.11.0-0

master安装k8s服务（参考教程 https://6xyun.cn/article/53 ）
kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=1.11.0 --apiserver-advertise-address=xxxxx --node-name=xxxxx
ip地址和名字根据实际环境填写
其他节点加入该集群
上传镜像
参考教程https://6xyun.cn/article/56 的join部分，master输入kubeadm token create --print-join-command查看token

（不要忘记提前关闭防火墙和修改hostname）
遇到的bug：node notready 在下面有。可参考

安装k8s device plugin（https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/）
Nvidia gpu plugin （1.11）
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.11/nvidia-device-plugin.yml

手动部署TensorFlow分布式过程如所示：


其他参考：
K8s结构及相关概念：
https://kubernetes.io/docs/concepts/
https://kubernetes.feisky.xyz/


K8s故障排查：
https://blog.csdn.net/liumiaocn/article/details/73925301
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/

k8s概念理解，中文文档
http://docs.kubernetes.org.cn/


相关命令：
拉取所有的pod：kubectl get pods -n kube-system
	kubectl get pod --all-namespaces
初始化k8s：kubeadm reset
拉取yml：或者把create改成apply     会在所有的机器上都拉取这个镜像。
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.11/nvidia-device-plugin.yml
检查节点状态：Kubectl get nodes
检查服务是否报错：kubectl describe pod kube-flannel-ds-rz245 -n kube-system
查看单一节点：kubectl describe node master。
拉取镜像并改名：
docker pull gsbeegnnord/pause:3.1
docker tag gsbeegnnord/pause:3.1 k8s.gcr.io/pause:3.1
从机上面join k8s集群： 
kubeadm join 192.168.0.55:6443 --token 8diet5.k4bgfqtwppip6fru --discovery-token-ca-cert-hash sha256:a2a1a392f8d10d976c033b65c0bd8a04ac4ffe459095c3d9268a024eb69e3d58

tocken值是：在master上：kubeadm token list   
如果过期：kubeadm token create
Sha256值是：master上的：openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //
这个会配置自动登陆。




遇到的错误：
1.启动docker报错，说/var/lib/docker 只读文件
解决：查看swap分区挂载
允许k8s swap  在/etc/sysconfig/kubelet加入参数



2.Getnode后 notready：
Kubectl get nodes
查看每个节点信息：
查看节点机器的详情：kubectl describe node master
原因：缺少k8s必须的几个service/pod：例如flannel，pause等
K8s是通过调用各个节点docker中的镜像进程来完成分配，所以查看
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
给每个节点都拉取这样的一个镜像
K8s的网址镜像没有科学上网是拉去不了的
如果节点拉取不了。可以采用另外的资源网站，在每一个节点上：docker pull gsbeegnnord/flannel:v0.10.0-amd64
之后改名成一样的
docker tag gsbeegnnord/flannel:v0.10.0-amd64 quay.io/coreos/flannel:v0.10.0-amd64
依旧没有的话可以
kubectl describe pod kube-flannel-ds-rz245 -n kube-system
查看该pod的情况

缺少pause
在每一个节点上拉取镜像并改名。
docker pull gsbeegnnord/pause:3.1
docker tag gsbeegnnord/pause:3.1 k8s.gcr.io/pause:3.1
发现所有节点都已经ready
进行拉取GPU的pod
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.11/nvidia-device-plugin.yml
拉取失败后查看服务为啥失败，发现
node1到master的svc居然不通，timeout了
Failed to create SubnetManager: error retrieving pod spec for 'kube-system/kube-flannel-ds-q6jkg': Get https://10.96.0.1:443/api/v1/namespaces/kube-system/pods/kube-flannel-ds-q6jkg: dial tcp 10.96.0.1:443: i/o timeout
node1和node2上面的image没有proxy
在所有节点上都docker pull gsbeegnnord/kube-proxy-amd64:v1.11.0
docker tag gsbeegnnord/kube-proxy-amd64:v1.11.0 k8s.gcr.io/kube-proxy-amd64:v1.11.0
重新创建flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
然后所有的pod都运行正常
Kubectl get pods –n kube-system



再都拉取一遍
kubectl create -f  https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.11/nvidia-device-plugin.yaml
发现报错缓存已经存在
更改create 为 apply
稍等一会并且把所有节点的/etc/docker/daemon.json都加上"default-runtime": "nvidia",与runtime同级。
发现所有节点的描述信息都有Gpu了。
节点master上面默认不进行任务没有GPU。

报错：unable to configure the Docker daemon with file /etc/docker/daemon.json: invalid character '"' after object key:value pair
解决：在那个文件下少了一个逗号










TensorFlow on k8s
   
1.先下载tensorflow-gpu 的docker镜像，每一台机器中都需要（已经下好）

2.之后找到demo的代码，git clone https://github.com/dengrenbo/gpu-test   （需要修改，已经改过）
3.将代码和数据打到镜像里面 在Dockerfile 路径下 docker build -t tensorflow/tensorflow-1.4.1-gpu-py3（名字随便取，这个是拉取一个镜像，并把代码和数据打包进去，可以修改dockerfile里面的东西，将里面的镜像改成我们现在已经有的镜像tensorflow/tensorflow-1.4.1-gpu-py3）
4.将打包好的镜像放在每一个节点里面，如果有私库就更加方便了。
5.在service路径下，kubectl create -f xxx.yaml，3个yaml文件，创建服务
kubectrl get svc查看

6.在train下面kubectl create -f xxx三个文件，开启三个job。注意yaml中的镜像指定要和docker image下的一样。这样就开启了三个pod，我们可以用一个pod作ps，两个作为worker
7.查看开启的pod。Kubectl get pods进入到每一个pod中，kubectl exec -it xxxx bash ，然后运行相对应的程序。
Ps的话python mnist-dis.py --job_name=ps --task_index=0 --ps_hosts=tf-dis-ps-1-0:2222 --worker_hosts=tf-dis-worker-2-0:2222,tf-dis-worker-2-1:2222
Worker的话 python mnist-dis.py --job_name=worker --task_index=XXX --ps_hosts=tf-dis-ps-1-0:2222 --worker_hosts=tf-dis-worker-2-0:2222,tf-dis-worker-2-1:2222   XXX是标号 worker0 对应的为 0  worker 1对应的是1.
8.就这样开始了训练。




运行dashboard：
kubectl proxy --address=192.168.0.55 --disable-filter=true
网页登录：
http://192.168.0.55:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default
Hdfs：

在所有节点kill所有jps下进程，然后：sbin/hadoop-daemon.sh --config /usr/local/hadoop/etc/hadoop/ --script hdfs start datanode





通过vim core-site.xml




加上
        <property>
            <name>hadoop.proxyuser.root.hosts</name>
            <value>*</value>
        </property>
        <property>
            <name>hadoop.proxyuser.root.groups</name>
            <value>*</value>
        </property>

然后停止master的namenode：sbin/hadoop-daemon.sh --config /usr/local/hadoop/etc/hadoop/ --script hdfs stop namenode
然后重启：sbin/hadoop-daemon.sh --config /usr/local/hadoop/etc/hadoop/ --script hdfs start namenode



创hdfs下的mnist.tgz







私有仓库使用：
Master启动


子节点上传：




中间报错：

原因是 harbor占用insecurity的80端口
ExecStart=/usr/bin/dockerd --insecure-registry=192.168.0.55:80

解决方式：vim /etc/systemd/system/multi-user.target.wants/docker.service删除insecurity

然后 vim /etc/docker/daemon.json指定使用5000端口

上传仓库要用master：5000,192.168.0.55:5000会报错

查看vim /etc/systemd/system/multi-user.target.wants/docker.service是否缺少ExecStart=




机房断电后：

然后执行四步：
1.sudo -i
2.swapoff -a
3.exit
4.strace -eopenat kubectl version


恢复原状


断电后：Hdfs的datanode和namenode全部消失
在每个节点上执行前面重启命令

创建pod一直error：错误为14000无法连接

1.cd  /usr/local/hadoop
2.sbin/httpfs.sh start






查看日志
vim /var/log/messages

重启k8s：
ps uax | grep kubelet(查看状态)
swapoff  -a
systemctl start kubelet

当pod在unknown和terminating状态上时：

执行强制删除：
kubectl patch pod {pod name}  -p '{"metadata":{"finalizers":null}}'


删除停用容器：
sudo docker rm `docker ps -a|grep Exited|awk '{print $1}'`

Register持久化：
docker run -d -v /opt/registry:/var/lib/registry -p 5000:5000 --restart=always --name registry registry


docker 端口映射错误解决方法

重建docker0网络
pkill docker
iptables -t nat -F
ifconfig docker0 down
brctl delbr docker0
systemctl docker start 

Mysql密码：***
mysql -uroot -p
show databases;
use tf;
select * from users;

Overlay爆满导致节点停止功能：

1.docker system df命令，类似于Linux上的df命令，用于查看Docker的磁盘使用情况:
2.docker system prune命令可以用于清理磁盘，删除关闭的容器、无用的数据卷和网络，以及dangling镜像(即无tag的镜像)。
3.docker system prune -a命令清理得更加彻底，可以将没有容器使用Docker镜像都删掉。注意，这两个命令会把你暂时关闭的容器，以及暂时没有用到的Docker镜像都删掉，所以使用之前一定要想清楚.。我没用过，因为会清理没有开启的 Docker 镜像。
4.将/var/lib/docker/overlay迁移 /var/lib/docker 目录。


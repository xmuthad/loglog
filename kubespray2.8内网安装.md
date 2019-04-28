## 基础环境
操作系统：CentOS 7

## 环境配置
1. 配置主机名（自定义）
2. 关闭防火墙
    ```
    # sed -i 's/SELINUX=*/SELINUX=disabled/' /etc/selinux/config
    # systemctl disable firewalld && systemctl stop firewalld
    ```
3. 关闭Swap交换分区
    ```
    # swapoff -a && echo "vm.swappiness=0" >> /etc/sysctl.conf && sysctl -p && free –h
    ```
4. 调整默认防火墙规则，禁用了iptables filter表中FOWARD链，使得Kubernetes集群中跨Node的Pod无法通信
    ```
    # iptables -P FORWARD ACCEPT
    ```
5. 配置ssh确保节点互通

6. 升级内核(可选项)

    ```
    #rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.orgrpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpmyum --enablerepo=elrepo-kernel install -y kernel-lt kernel-lt-devel
    # grub2-set-default 0
    # reboot
    ```
    增加内核配置
    ```
    # vim /etc/sysctl.conf
    dockernet.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    ```
    使得内核配置生效
    ```
    # sysctl -p
    ```
上面配置为各个节点基本配置，下面为部署节点配置需要
7. 配置epel源
    ```
    # yum -y install epel-release
    ```
    
8. make缓存
    ```
    # yum clean all && yum makecache
    ```
    
9. 获取部署脚本
    ```
    # yum install -y git
    # git clone https://github.com/xmuthad/kubespray.git  -b release-2.8
    ```

9. 安装脚本运行环境
    ```
    # yum install -y python-pip
    # cd kubespray
    # pip install -r requirements.txt
    
    ```

## 安装kubernetes

1. 设置inventory
如果是本地单机部署，可以直接使用默认的inventory/local/host.ini。
```
# cd kubespray
# cp -rfp inventory/sample inventory/mycluster
# declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
# CONFIG_FILE=inventory/mycluster/hosts.ini python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

2. 配置部署集群参数。
```
# vim inventory/mycluster/group_vars/all/all.yml
# vim inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
    修改镜像仓库地址
    kube_image_repo: "reg.esgyn.com/google-containers"
```
>配置容器仓库的证书, 创建容器仓库证书目录
>```# mkdir certs.d/reg.esgyn.com```
>将ca.crt、reg.esgyn.com.cert、reg.esgyn.com.key这三个文件复制到reg.esgyn.com目录下
>通过登录容器仓库来验证是否配置成功
>```# docker login reg.esgyn.com```
>

3. 修改脚本中的配置
```
# sed -i 's/gcr\.io\/google_containers/reg\.esgyn\.com\/google-containers/g' roles/download/defaults/main.yml
# sed -i 's/quay\.io\/coreos/reg\.esgyn\.com\/coreos/g' roles/download/defaults/main.yml
# sed -i 's/quay\.io\/calico/reg\.esgyn\.com\/calico/g' roles/download/defaults/main.yml
```

4. 将hyperkube、kubeadm拷贝至/tmp/releases目录下
5. 运行Ansible脚本
```
# ansible-playbook -i inventory/local/hosts.ini cluster.yml 
```
>安装过程中会下载一些rpm包，网络较差的情况下会导致超时



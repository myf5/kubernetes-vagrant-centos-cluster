## K8s and istio quick installation  on centos7

### 使用vagrant+virtualbox模式

环境要求：

* Centos7机器（或任意虚机）虚机至少8G以上内存

1. vagrant installation

   * download https://www.vagrantup.com/downloads.html ,选择centos7 64-bit 下载rpm包
   * 下载后执行安装rpm -ivh *your-rpm-package-name*

2. virtualbox安装

   * 增加repo：vi /etc/yum.repos.d/virtualbox.repo

     ```
     [virtualbox]
     name=Oracle Linux / RHEL / CentOS-$releasever / $basearch - VirtualBox
     baseurl=http://download.virtualbox.org/virtualbox/rpm/el/$releasever/$basearch
     enabled=1
     gpgcheck=1
     repo_gpgcheck=1
     gpgkey=https://www.virtualbox.org/download/oracle_vbox.asc
     ```

   * 更新缓存

     ```
      yum clean all
      yum makecache
     ```

3. virtualbox安装

   yum install VirtualBox-5.2

4. 保证机器拥有50G以上空间

5. ```
   git clone https://github.com/myf5/kubernetes-vagrant-centos-cluster.git
   ```

6. 修改vagrant配置配置

   ```
   cd kubernetes-vagrant-centos-cluster
   vi Vagrantfile
   找到
   node.vm.network "public_network", bridge: "ens33", auto_config: true
   将bridge后面的网卡名称修改为你的虚机或者物理机网卡名称(可以上互联网的)
   ```

7. 下载k8s 二进制服务端和客户端，此处需要翻墙，将下载的文件保存在kubernetes-vagrant-centos-cluster目录下

   ```
   cd kubernetes-vagrant-centos-cluster

   wget https://storage.googleapis.com/kubernetes-release-mehdy/release/v1.9.3/kubernetes-client-linux-amd64.tar.gz -O kubernetes-client-linux-amd64.tar.gz

   wget https://storage.googleapis.com/kubernetes-release-mehdy/release/v1.9.3/kubernetes-server-linux-amd64.tar.gz -O kubernetes-server-linux-amd64.tar.gz
   ```

   ​

8. 在kubernetes-vagrant-centos-cluster目录下执行 vagrant up

   注：可能会遇到一个关于kernal问题的提示，并提示修复问题后执行sudo /sbin/vboxconfig，此时可安装以下内容后再执行sudo /sbin/vboxconfig命令

   ```
   yum install gcc make perl
   yum install kernel-devel
   ```

   ​
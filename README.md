# 使用Vagrant和Virtualbox搭建Kubernetes集群(F5 VERSION)

[English Version Click Here](README-en.md)

当我们需要在本地开发时，更希望能够有一个开箱即用又可以方便定制的分布式开发环境，这样才能对Kubernetes本身和应用进行更好的测试。现在我们使用[Vagrant](https://www.vagrantup.com/)和[VirtualBox](https://www.virtualbox.org/wiki/Downloads)来创建一个这样的环境。

**注意**：kube-proxy使用ipvs模式。

## 准备环境

需要准备以下软件和环境：

- 8G以上内存
- Vagrant 2.0+
- Virtualbox 5.0 +
- 提前下载kubernetes1.9.1以上版本的release压缩包.下载地址：
`链接: https://pan.baidu.com/s/1jJyVoTC 密码: dc9x`

[vagrant及vritualbox安装准备请查看](vagrant-virbox-installation.md)

## 集群

我们使用Vagrant和Virtualbox安装包含3个节点的kubernetes集群，其中master节点同时作为node节点。

| IP           | 主机名   | 组件                                       |
| ------------ | ----- | ---------------------------------------- |
| 172.17.8.101 | node1 | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、kubelet、docker、flannel、dashboard |
| 172.17.8.102 | node2 | kubelet、docker、flannel、traefik           |
| 172.17.8.103 | node3 | kubelet、docker、flannel                   |

**注意**：以上的IP、主机名和组件都是固定在这些节点的，即使销毁后下次使用vagrant重建依然保持不变。

容器IP范围：172.33.0.0/30

Kubernetes service IP范围：10.254.0.0/16

## 安装的组件

安装完成后的集群包含以下组件：

- flannel（`host-gw`模式）
- kubernetes dashboard 1.8.2
- etcd（单节点）
- kubectl
- CoreDNS
- kubernetes（版本根据下载的kubernetes安装包而定）

**可选插件**

- Heapster + InfluxDB  + Grafana
- ElasticSearch + Fluentd + Kibana
- Istio service mesh

## 部署

**确保安装好以上的准备环境后，执行下列命令clone仓库** 

```bash
git clone https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster.git
```

**注意**：克隆完Git仓库后，在执行`vagrant up`需要提前下载kubernetes的压缩包到`kubenetes-vagrant-centos-cluster`目录下，包括如下两个文件：

- kubernetes-client-linux-amd64.tar.gz
- kubernetes-server-linux-amd64.tar.gz

下载地址：
`链接: https://pan.baidu.com/s/1jJyVoTC 密码: dc9x`

**修改Vagrantfile内容**

请参考：[这里](vagrant-virbox-installation.md)

**启动集群**

```bash
cd kubernetes-vagrant-centos-cluster
vagrant up
```

如果是首次部署，会自动下载`centos/7`的box，这需要花费一些时间，另外每个节点还需要下载安装一系列软件包，整个过程大概需要10几分钟。

**启动后工作**
为避免IP地址变化导致flannel出现问题，建议检查node1，2，3三台机器的eth2网卡IP，将IP设置为网卡固定IP。

```bash
cd kubernetes-vagrant-centos-cluster
vagrant ssh node 1
node1# nmtui
```


## 访问kubernetes集群

访问Kubernetes集群的方式有三种：

- 本地访问
- 在VM内部访问
- kubernetes dashboard

**通过本地访问**

可以直接在你自己的本地环境中操作该kubernetes集群，而无需登录到虚拟机中，执行以下步骤：

将`conf/admin.kubeconfig`文件放到`~/.kube/config`目录下即可在本地使用`kubectl`命令操作集群。

**在虚拟机内部访问**

如果有任何问题可以登录到虚拟机内部调试：

```bash
vagrant ssh node1
sudo -i
kubectl get nodes
```

**Kubernetes dashboard**

还可以直接通过dashboard UI来访问：https://172.17.8.101:8443

可以在本地执行以下命令获取token的值（需要提前安装kubectl）：

```bash
kubectl -n kube-system describe secret `kubectl -n kube-system get secret|grep admin-token|cut -d " " -f1`|grep "token:"|tr -s " "|cut -d " " -f2
```

**注意**：token的值也可以在`vagrant up`的日志的最后看到。

**Heapster监控**

创建Heapster监控：
**注意：** 如果虚机内存较小，没有必要开启该监控服务

```bash
kubectl apply -f addon/heapster/
```

访问Grafana

使用Ingress方式暴露的服务，在本地`/etc/hosts`中增加一条配置：

```ini
172.17.8.102 grafana.myf5.net
```

访问Grafana：<http://grafana.myf5.net>

**Traefik**

部署Traefik ingress controller和增加ingress配置：

```bash
kubectl apply -f addon/traefik-ingress
```

在本地`/etc/hosts`中增加一条配置：

```ini
172.17.8.102 traefik.myf5.net
```

访问Traefik UI：<http://traefik.myf5.net>

**EFK**

使用EFK做日志收集。

```bash
kubectl apply -f addon/efk/
```

**注意**：运行EFK的每个节点需要消耗很大的CPU和内存，请保证每台虚拟机至少分配了4G内存。

### F5 Hello World Application

系统已自动安装，修改本地(virtualbox的宿主机，也就是你自己电脑中的虚机)hosts

```ini
172.17.8.102  hello.myf5.net
```
访问http://hello.myf5.net/

### F5 BIGIP K8S CONTROLLER集成
注意：此步骤需在部署上述“F5 hello world application”工作之后执行

* BIGIP准备工作
 * 额外创建一个BIGIP，确保k8s node可以和BIGIP通信。本demo中，将BIGIP的一个接口放置在fusion虚拟网络的NAT网络中，node的eth2接口也同样桥接到了fusion的NAT网络。例如BIGIP的self ip为 172.16.150.245/24
 * 在node上ping BIGIP self ip，确认可以通信
 * BIGIP上创建一个新的partition，命名为k8s
* 执行以下命令，创建bigip CC集成，以下命令将同时创建一个configmap实现在bigip上的业务发布

```bash
kubectl create -f addon/f5/
```
登陆BIGIP确认k8s partition下产生类似如下服务

```
root@(v13)(cfg-sync Standalone)(Active)(/k8s)(tmos)# list ltm virtual
ltm virtual default_k8s.vs {
    destination 192.168.188.188%0:http
    ip-protocol tcp
    mask 255.255.255.255
    partition k8s
    pool cfgmap_default_k8s.vs_f5-hello-word-service
    profiles {
        /Common/http { }
        /Common/tcp { }
    }
    source 0.0.0.0/0
    source-address-translation {
        type automap
    }
    translate-address enabled
    translate-port enabled
    vs-index 6
}
root@(v13)(cfg-sync Standalone)(Active)(/k8s)(tmos)# list ltm pool
ltm pool cfgmap_default_k8s.vs_f5-hello-word-service {
    members {
        172.33.21.4%0:webcache {
            address 172.33.21.4
            session monitor-enabled
            state down
        }
        172.33.81.3%0:webcache {
            address 172.33.81.3
            session monitor-enabled
            state down
        }
    }
    monitor cfgmap_default_k8s.vs_f5-hello-word-service_0_http 
    partition k8s
}
ltm pool ingress_default_f5-hello-word-service {
    members {
        172.33.21.4%0:webcache {
            address 172.33.21.4
        }
        172.33.81.3%0:webcache {
            address 172.33.81.3
        }
    }
    partition k8s
}
ltm pool ingress_default_traefik-ingress-service {
    members {
        172.17.8.102%0:webcache {
            address 172.17.8.102
        }
    }
    partition k8s
}
```

### Service Mesh

我们使用 [istio](https://istio.io) 作为 service mesh。

**安装**

```bash
kubectl apply -f addon/istio/
```

**运行示例**

```bash
kubectl apply -f yaml/istio-bookinfo
kubectl apply -n default -f <(istioctl kube-inject -f yaml/istio-bookinfo/bookinfo.yaml)
```

详细信息请参阅 https://istio.io/docs/guides/bookinfo.html

### 管理

以下命令都在当前的repo目录下操作。

**挂起**

将当前的虚拟机挂起，以便下次恢复。

```bash
vagrant suspend
```

**恢复**

恢复虚拟机的上次状态。

```bash
vagrant resume
```

**清理**

清理虚拟机。

```bash
vagrant destroy
rm -rf .vagrant
```

### 注意

不要在生产环境使用该项目。

## 参考

- [Kubernetes handbook - jimmysong.io](https://jimmysong.io/kubernetes-handbook)
- [duffqiu/centos-vagrant](https://github.com/duffqiu/centos-vagrant)
- [Kubernetes 1.8 kube-proxy 开启 ipvs](https://mritd.me/2017/10/10/kube-proxy-use-ipvs-on-kubernetes-1.8/#%E4%B8%80%E7%8E%AF%E5%A2%83%E5%87%86%E5%A4%87)

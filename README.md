# kubernetes实现VPA垂直容器自动缩放
介绍

垂直容器自动缩放器（VPA）使用户无需设置最新的资源限制和对容器中容器的要求。 配置后，它将根据使用情况自动设置请求，从而允许在节点上进行适当的调度，以便为每个Pod提供适当的资源量。 它还将保持限制和初始容器配置中指定的请求之间的比率。

它既可以根据资源的使用情况来缩减对资源过度使用的Pod的规模，也可以对资源需求不足的向上扩展的Pod的规模进行扩展。
自动缩放是使用称为VerticalPodAutoscaler的自定义资源定义对象配置的。 它允许指定哪些吊舱应垂直自动缩放，以及是否/如何应用资源建议。


简单来说是 Kubernetes VPA 可以根据实际负载动态设置 pod resource requests。


Kubernetes VPA 包含以下组件：
Recommender：用于根据监控指标结合内置机制给出资源建议值
Updater：用于实时更新 pod resource requests
History Storage：用于采集和存储监控数据
Admission Controller: 用于在 pod 创建时修改 resource requests

安装

当前默认版本是Vertical Pod Autoscaler 0.8.0

先决条件

    kubectl应该连接到要在其中安装VPA的集群。
    指标服务器必须部署在您的集群中。了解有关Metrics Server的更多信息,可以安装我之前安装的部署在你的集群上

    如果您的群集中已经安装了另一个版本的VPA，则必须首先使用以下命令删除现有安装：

    ./hack/vpa-down.sh

安装命令

你在安装过程中会使用国外的镜像，也可以安装我的，默认我这边已经替换为国内的镜像了

    ./hack/vpa-up.sh


该脚本向集群发出多个kubectl命令，以插入配置并启动kube-system命名空间中的所有必需pod（请参阅体系结构）。与API服务器通信时，它还会生成并上传VPA Admission Controller使用的机密（CA证书）。

最佳实践创建ssl需要依赖openssl-1.1.1
最佳实践需要安装openssl-1.1.1
查看默认版本的openssl version是1.1.0,如果直接安装的话是失败的,它会报一个错误，vpa-admission-controller这个pod会始终处于创建之中，会找不到证书,原因低版本的openssl没有生成证书的命令

所以我们需要升级openssl
第一步卸载yum安装的openssl

     rpm -e --nodeps openssl.xxxxx


安装组件依赖

     yum install libtool perl-core zlib-devel -y
  
下载1.1.1的openssl，默认我的环境为centos-7.6,centos的部署基本大致相同，我把下载好的也放在了github上，直接git clone就可以用，下面也放了一个下载openssl的地址，可供选择

    https://www.openssl.org/docs/man1.1.1/man1/req.html

    # cd /usr/local/src
    # tar -zxf openssl-1.1.1g-latest.tar.gz

    # cd openssl-1.1.1g
    # ./config
    # make
    # make test
    # make install

    # ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl


加载共享库libssl.so.1.1时如何解决openssl错误
如果安装成功查看版本如有以下报错
查看openssl version
openssl: error while loading shared libraries: libssl.so.1.1: cannot open shared object file: No such file or directory

调试
我们尝试找到名为libssl.so.1.1的文件，如下所示：
我们可以发现'libcrypto.so.1.1'位于/ usr / local / lib64中，但是openssl尝试在LD_LIBRARY_PATH中找到.so库。

      [root@master ~]# ll /usr/local/lib64
      总用量 10480
      drwxr-xr-x 2 root root      39 8月   7 14:23 engines-1.1
      -rw-r--r-- 1 root root 5630906 8月   7 14:23 libcrypto.a
      lrwxrwxrwx 1 root root      16 8月   7 14:23 libcrypto.so -> libcrypto.so.1.1
      -rwxr-xr-x 1 root root 3380224 8月   7 14:23 libcrypto.so.1.1
      -rw-r--r-- 1 root root 1024200 8月   7 14:23 libssl.a
      lrwxrwxrwx 1 root root      13 8月   7 14:23 libssl.so -> libssl.so.1.1
      -rwxr-xr-x 1 root root  685528 8月   7 14:23 libssl.so.1.1
      drwxr-xr-x 2 root root      61 8月   7 14:23 pkgconfig
      [root@master ~]# ll /usr/local/lib64/libssl*
      -rw-r--r-- 1 root root 1024200 8月   7 14:23 /usr/local/lib64/libssl.a
      lrwxrwxrwx 1 root root      13 8月   7 14:23 /usr/local/lib64/libssl.so -> libssl.so.1.1
      -rwxr-xr-x 1 root root  685528 8月   7 14:23 /usr/local/lib64/libssl.so.1.1





因此，解决方案是尝试告诉openssl库在那里。
解决方法
创建到文件的链接

     [root@localhost openssl-1.1.g]# ln -s /usr/local/lib64/libssl.so.1.1 /usr/lib64/libssl.so.1.1
     [root@localhost openssl-1.1.g]# ln -s /usr/local/lib64/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1


看到openssl的版本为以下，就可以部署了

    # openssl version
    OpenSSL 1.1.1g  21 Apr 2020

开始部署

[root@master hack]# ./vpa-up.sh 
此次省略
........

    Generating certs for the VPA Admission Controller in /tmp/vpa-certs.
    Generating RSA private key, 2048 bit long modulus (2 primes)
    ................................+++++
    .+++++
    e is 65537 (0x010001)
    Generating RSA private key, 2048 bit long modulus (2 primes)
    .............................................+++++
    .............................+++++
    e is 65537 (0x010001)
    Signature ok
    subject=CN = vpa-webhook.kube-system.svc
    Getting CA Private Key
    Uploading certs to the cluster.
    secret/vpa-tls-certs created
    Deleting /tmp/vpa-certs.
    deployment.apps/vpa-admission-controller created
    service/vpa-webhook created

最后效果为成功部署

    [root@master hack]# kubectl get po -A
    kube-system   vpa-admission-controller-69c96bd8bd-4st7v   1/1     Running     0          39s
    kube-system   vpa-recommender-765b6c5f59-rdxk4            1/1     Running     0          45s
    kube-system   vpa-updater-86865896cf-z8bxn                1/1     Running     0          50s

因为部署的原理是需要通过metrics server去拿的数据
可以通过curl来获取拿到的数据
可以看到vpa-admission-controller 监控指标的数据

    curl http://172.31.166.144:8944/metrics
    
vpa_recommender 监控指标数据

    curl http://172.31.166.141:8942/metrics
    
vpa-updater 监控指标数据
    
    curl http://172.31.166.140:8943/metrics
    
    
检查Vertical Pod Autoscaler是否在您的集群中完全正常运行的一种简单方法是创建示例部署和相应的VPA配置：

    kubectl create -f examples/hamster.yaml
    
上面的命令创建了一个包含2个Pod的部署，每个Pod运行一个请求100m的容器，并尝试使用最高不超于500m的容器。该命令还会创建一个指向部署的VPA配置。VPA将观察Pod的行为，大约5分钟后，它们应使用更高的CPU请求进行更新（请注意，VPA不会在部署中修改模板，但Pod的实际请求会被更新）。

要查看VPA配置和当前推荐的资源请求，请运行：

    # kubectl get vpa -A
      NAMESPACE     NAME          AGE
      kube-system   hamster-vpa   15m
      
    # kubectl describe vpa hamster-vpa -n kube-system
    
现在我们的默认值是以下配置

            requests:
              cpu: 100m
              memory: 50Mi
              
当我们去创建这个测试用例之后
查看-o yaml实际使用的

大概等待60s之后pod会重建，来获取适当的request的值

    resources:
      requests:
        cpu: 587m
        memory: 262144k

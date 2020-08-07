# kubernetes实现VPA垂直容器自动缩放
先决条件

    kubectl应该连接到要在其中安装VPA的集群。
    指标服务器必须部署在您的集群中。了解有关Metrics Server的更多信息,可以安装我之前安装的部署在你的集群上

    如果您的群集中已经安装了另一个版本的VPA，则必须首先使用以下命令删除现有安装：

    ./hack/vpa-down.sh

安装命令

要安装VPA，请下载VPA的源代码（例如git clone https://github.com/kubernetes/autoscaler.git），并在vertical-pod-autoscaler目录中运行以下命令：
如果你在安装过程中会使用国外的镜像，这里如果你使用不了，也可以安装我的，默认我这边已经替换为国内的镜像了

./hack/vpa-up.sh

注意：该脚本当前读取环境变量：$ REGISTRY和$ TAG。除非要使用VPA的非默认版本，否则请确保未设置它们。

该脚本向集群发出多个kubectl命令，以插入配置并启动kube-system命名空间中的所有必需pod（请参阅体系结构）。与API服务器通信时，它还会生成并上传VPA Admission Controller使用的机密（CA证书）。

1、最佳实践创建ssl需要依赖openssl-1.1.1



最佳实践需要安装openssl-1.1.1
查看默认版本的openssl version是1.1.0,如果直接安装的话是失败的

所以我们需要升级openssl
第一步卸载yum安装的openssl
rpm -e --nodeps openssl.xxxxx

下载1.1.1的openssl，默认环境为centos-7.6

https://www.openssl.org/docs/man1.1.1/man1/req.html

# cd /usr/local/src
# tar -zxf openssl-1.0.2-latest.tar.gz

# cd openssl-1.1.1g
# ./config
# make
# make install

# mv /usr/bin/openssl /root/
# ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl

# openssl version
OpenSSL 1.0.2e 3 Dec 2015

加载共享库libssl.so.1.1时如何解决openssl错误
如果安装失败的话有以下报错

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

[root@master hack]# ./vpa-up.sh 
customresourcedefinition.apiextensions.k8s.io/verticalpodautoscalers.autoscaling.k8s.io created
customresourcedefinition.apiextensions.k8s.io/verticalpodautoscalercheckpoints.autoscaling.k8s.io created
clusterrole.rbac.authorization.k8s.io/system:metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:vpa-actor created
clusterrole.rbac.authorization.k8s.io/system:vpa-checkpoint-actor created
clusterrole.rbac.authorization.k8s.io/system:evictioner created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-reader created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-actor created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-checkpoint-actor created
clusterrole.rbac.authorization.k8s.io/system:vpa-target-reader created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-target-reader-binding created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-evictionter-binding created
serviceaccount/vpa-admission-controller created
clusterrole.rbac.authorization.k8s.io/system:vpa-admission-controller created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-admission-controller created
clusterrole.rbac.authorization.k8s.io/system:vpa-status-reader created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-status-reader-binding created
serviceaccount/vpa-updater created
deployment.apps/vpa-updater created
serviceaccount/vpa-recommender created
deployment.apps/vpa-recommender created
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

[root@master hack]# kubectl get po -A
kube-system   vpa-admission-controller-69c96bd8bd-4st7v   1/1     Running     0          39s
kube-system   vpa-recommender-765b6c5f59-rdxk4            1/1     Running     0          45s
kube-system   vpa-updater-86865896cf-z8bxn                1/1     Running     0          50s


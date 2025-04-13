# jenkins的k8s部署

## 项目概述

K8S+JENKINS实现CICD流水线
核心流程：

1.GitHub 代码拉取 → Maven 编译打包 → Docker 镜像构建 → Harbor 镜像存储。

2.完全基于 Kubernetes 动态 Pod 调度，按需创建临时构建节点，资源高效利用。

3.Maven依赖本地化存储加速构建。

## 流程说明

1.开发人员需求确定后，从master分支拉取最新代码，在dev分支完成开发后，将dev分支合并到test分支并推送至git仓库。

2.Jenkins采用SCM轮询方式，检测GIT仓库，触发持续构建流程

3.Jenkins调用k8s插件，调用临时Pod进行编译打包和docker镜像构建

4.将镜像产物推送到Harbor镜像仓库

## 项目代码仓库地址

GITHUB：https://github.com/xljmzhc/Jenkins

## POD模板，临时POD介绍

在 Jenkins 的 Kubernetes 动态 Agent 模式下，Pod 模板是定义临时构建环境的核心配置。通过声明式模板，Jenkins 能在每次构建时动态创建专用 Pod，任务完成后自动销毁，实现高效资源利用和环境隔离。

✅ 资源高效
按需创建/销毁 Pod，避免静态 Agent 的常驻资源浪费

✅ 安全可靠
容器化隔离，构建失败不影响其他任务
通过 Kubernetes Secret 管理敏感信息（如 Harbor 凭据）

✅ 灵活扩展
一个模板可复用于多个项目，仅需调整标签即可切换环境

## yaml应用部署文件

1.jenkins-rbac.yaml: Jenkins服务账户的PBAC配置

2.jenkins-pvc.yaml：Jenkins持久化存储PVC

3.jenkins-pv.yaml：对接NFS的PV

4.jenkins-deployment.yaml：Jenkins主服务部署文件

5.harbor-sec.yaml：Harbor仓库访问Secret

6.jenkins-svc.yaml：Jenkins服务暴露配置

7.maven-pvpvc.yaml：Maven依赖本地存储配置

8.pod模板文件：Jenkins使用pod模板来拉起临时pod

## 部署前的准备

1.创建名称空间

```shell
kubectl create namespace cicd
```

2.配置NFS存储

```shell
vi /etc/exports
/root/k8s-nfs *(rw,sync,no_subtree_check,no_root_squash)
/root/k8s-maven *(rw,sync,no_subtree_check,no_root_squash)
#应用配置
sudo exportfs -a
#查看是否可用
showmount -e localhost
```

3.harbor仓库配置

- harbor配置https

```shell
# 生成 CA 私钥
openssl genrsa -out ca.key 4096

# 生成 CA 根证书（有效期 10 年）
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=MyCompany/OU=Dev/CN=MyHarborCA"

vi harbor-req.cnf

[ req ]
default_bits       = 4096
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[ dn ]
C  = CN
ST = Beijing
L  = Beijing
O  = MyCompany
OU = Dev
CN = 192.168.137.137

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
IP.1 = 192.168.137.137

openssl genrsa -out harbor.key 4096
openssl req -new -key harbor.key -out harbor.csr -config harbor-req.cnf

vi ca-ext.cnf
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
IP.1 = 192.168.137.137

openssl x509 -req -in harbor.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
-out harbor.crt -days 3650 -sha256 -extfile ca-ext.cnf

./prepare
docker-compose down
docker-compose up -d

scp /usr/local/harbor/SSL/harbor.crt xljm@192.168.137.139:/etc/docker/certs.d/192.168.137.137/harbor.crt
scp /usr/local/harbor/SSL/ca.crt xljm@192.168.137.139:/etc/pki/ca-trust/source/anchors/harbor-ca.crt
sudo update-ca-trust extract

```

- harbor用户，仓库配置

创建项目，配置用户权限，为项目添加对应用户
![图片描述](https://github.com/xljmzhc/k8s-jenkins/blob/master/images/image.png)

4.harbor准备好各种镜像
![图片描述](https://github.com/xljmzhc/k8s-jenkins/blob/master/images/image-5.png)

## 部署流程

1.启动harbor仓库
>docker-compose up -d

2.应用部署文件
>kubectl apply -f jenkins-pv.yaml

kubectl apply -f jenkins-pvc.yaml

kubectl apply -f jenkins-rbac.yaml

kubectl apply -f jenkins-deployment.yaml

kubectl apply -f jenkins-svc.yaml

kubectl apply -f maven-pvpvc.yaml

- Harbor仓库密码

>kubectl create secret docker-registry harbor-sec   --docker-server=https://192.168.137.137   --docker-username=admin   --docker-password=Harbor12345   -n cicd

- Harbor仓库证书

>kubectl create secret generic harbor-ca-secret --from-file=harbor.crt=/usr/local/SSL/harbor.crt --from-file=ca.crt=/usr/local/SSL/ca.crt

3.根据svc暴露的端口访问Jenkins界面，并且进行必要插件的安装

4.配置凭据
![图片描述](https://github.com/xljmzhc/k8s-jenkins/blob/master/images/image-1.png)

5.增加云节点k8s，在云节点中增加pod模板docker-build和maven
![图片描述](https://github.com/xljmzhc/k8s-jenkins/blob/master/images/image-2.png)

6.新建项目，编写流水线

## 构建结果

![图片描述](https://github.com/xljmzhc/k8s-jenkins/blob/master/images/image-3.png)
![图片描述](https://github.com/xljmzhc/k8s-jenkins/blob/master/images/image-4.png)

## 问题

1.***k8s环境下的jenkins没有全局配置maven***

>(1)构建自定义镜像，但是会产生一个很大的镜像

>(2)jenkins连接k8s，定义一个pod模板，启动一个临时的节点，容器中运行了maven，在流水线运行时添加上这个标签，即可在进行流水线时使用临时pod中的maven来进行构建打包

2.***关闭了harbor的https，但是很容易出现TLS超时的情况***

>比如在jenkins连接k8s，启动pod时，该pod默认使用https，会导致连接出错；

3.***harbor的认证问题，集群之中，对各个节点配置相应的证书***

>临时pod也会存在身份验证以及证书问题：所以在pod模板中使用了secret资源harbor-ca-secret和harbor-sec

4.***在同一pod模板中同时进行maven编译打包和docker镜像构建，但是会因为集群中资源不足，导致maven和docker容器被驱逐***

>将二者分离为不同的pod模板解决了问题

5.***每次在进行mvn打包构建的时候，总是会因为依赖下载很久***

>现在使用maven本地化存储来解决每次都要下载全部依赖的问题，将依赖都挂载到/root/k8s-maven中

6.***启动了新的pod模板，docker-build来构建镜像，但是无法获取到jar包；现在jar包存储在jenkins容器和k8s-nfs中***

>现在通过stash/unstash来传递文件（在流水线中使用），这是一种跨pod传递文件的机制

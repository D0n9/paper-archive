# 云原生安全-从k8s日志审计视角检测自动化工具攻击
**本期作者**


张智

哔哩哔哩高级安全工程师

**1\. 前言**

**1.1 背景**

随着云原生技术的普及，其暴露出来的攻击面也被黑客们念念不忘，相关的攻击技术也跟着被“普及”，自动化漏洞利用攻击工具更是如雨后春笋般出现在GitHub开源平台，其中比较有代表性的如cdk-team/CDK。其是一款为容器环境定制的渗透测试工具，在已攻陷的容器内部提供零依赖的常用命令及PoC/EXP。集成Docker/K8s场景特有的 逃逸、横向移动、持久化利用方式，插件化管理。

在漏洞利用门槛如此低廉的今天，作为企业安全的建设者（搬砖人），除了考虑部署容器层面运行时检测平台，在k8s api-server层面，启用日志审计功能，也是一个成本低廉又高效发现入侵攻击的途径。

通过对api-server的日志进行审计分析，对于攻击者的信息收集行为，部署k8s cronjob后门、利用rbac做权限提升等持久化攻击行为都能及时的发现并输出告警。

**1.2 kubernetes日志审计介绍**

Kubernetes 审计功能提供了与安全相关的按时间顺序排列的记录集，记录单个用户、管理员或系统其他组件影响系统的活动顺序。 它能帮助集群管理员处理以下问题：

```c
- 发生了什么？
- 什么时候发生的？
- 谁触发的？
- 为什么发生?
- 在哪观察到的？
- 它从哪触发的？
- 它将产生什么后果？
```

**审计日志示例（**图片来自参考\[5\]

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/9/501d3466-1107-4963-9ac0-fdb324ebff22.png?raw=true)

**如何开启日志审计**

1、api-server命令行启动时，添加如下参数

```c
--audit-policy-file=/etc/kubernetes/audit/audit-default-policy.yaml        # 审计策略文件
--audit-log-path=/data/log/audit/audit.log                                 # kube-apiserver 输出的审计日志文件，此处以日志文件落地的方式做日志收集
--audit-log-maxbackup=10                                                   # kube-apiserver 审计日志文件的最大备份数量
--audit-log-format=json                                                    #日志保存格式
--audit-log-maxage=10                                                      #日志最大保留时间
--audit-log-maxsize=500                                                    # 单个日志文件最大为500M
```

2、kubeadm启动时

修改api-server配置文件/etc/kubernetes/manifests/kube-apiserver.yaml，增加如下内容

```c
--audit-policy-file=/etc/kubernetes/audit/audit-default-policy.yaml
--audit-log-path=/data/log/audit/audit.log 
--audit-log-maxbackup=10 
--audit-log-format=json
--audit-log-maxage=10
--audit-log-maxsize=500 
```

**2\. 攻击行为检测**

**2.1 分析流程**

若攻击者通过各种手段拿到一个容器shell环境后，下一步必然是利用这个容器进行信息收集，获取更多敏感信息或机器资源，此时，CDK作为一款自动化的攻击工具，就让整个攻击过程如虎添翼，大大提高攻击效率。

作为甲方安全的守护者，从安全建设的角度，如何有效及时的发现攻击者的入侵行为，是一个无法避开的问题。本文通过观察cdk的攻击行为，从k8s日志审计的角度罗列一些入侵检测的常见规则。

目前，针对于日志的审计分析，我们落地方案的整个流程为：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/9/7f344967-6140-4c35-8ed8-ec5f11fef917.png?raw=true)

注：下列场景，仅限于原生的CDK行为检测，如果攻击者具备开发能力，修改对应特征，可以绕过对应的检测规则。

**2.2 信息收集**

CDK在容器内主要收集以下信息：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/9/dc27be1c-e83c-4b2a-adb7-9bcb4243cd76.png?raw=true)

工具在镜像内运行，其信息收集的结果执如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/9/76e1397c-14b6-4d6b-b371-7e6ee309af32.png?raw=true)

从图上可知，信息收集的整个过程，存在与api-server交互的仅有网络探测部分，总共会向api-server发送三条请求，分别为API-server、namespace、api信息探测。

**2.2.1 探测API-server**

**audit.log**

```c
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1",
    "level": "Metadata",
    "stage": "ResponseComplete",
    "requestURI": "/",
    "verb": "get",
    "user": {
        "username": "system:anonymous",
        "groups": [
            "system:unauthenticated"
        ]
    },
    "sourceIPs": [
        "172.18.0.2"
    ],
    "userAgent": "Go-http-client/1.1",
    "responseStatus": {
        "metadata": {},
        "status": "Failure",
        "reason": "Forbidden",
        "code": 403
    },
    "requestReceivedTimestamp": "2023-02-02T08:29:12.189459Z",
    "stageTimestamp": "2023-02-02T08:29:12.189553Z",
    "annotations": {
        "authorization.k8s.io/decision": "forbid",
        "authorization.k8s.io/reason": ""
    }
}
```

**2.2.2 列举namespace**

**audit.log**

```c
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1",
    "level": "Metadata",
    "auditID": "41770e7a-1827-4a14-860f-a812d3db1647",
    "stage": "ResponseComplete",
    "requestURI": "/api/v1/namespaces",
    "verb": "list",
    "user": {
        "username": "system:serviceaccount:test:default",
        "uid": "63b8dd88-88dd-4426-bdd1-7966906dc0d5",
        "groups": [
            "system:serviceaccounts",
            "system:serviceaccounts:test",
            "system:authenticated"
        ]
    },
    "sourceIPs": [
        "172.18.0.2"
    ],
    "userAgent": "Go-http-client/1.1",
    "objectRef": {
        "resource": "namespaces",
        "apiVersion": "v1"
    },
    "responseStatus": {
        "metadata": {},
        "status": "Failure",
        "reason": "Forbidden",
        "code": 403
    },
    "requestReceivedTimestamp": "2023-02-02T08:29:12.205422Z",
    "stageTimestamp": "2023-02-02T08:29:12.205485Z",
    "annotations": {
        "authorization.k8s.io/decision": "forbid",
        "authorization.k8s.io/reason": ""
    }
}

```

**2.2.3 探测可访问的api**

跟namespace类似，只不过requestURI为/apis

**2.2.4 总结**

综上， 上述日志中可概述为，当cdk在做信息收集时，会向api-server请求3次，分别访问/，/apis，/api/v1/namespaces，可根据这些特征做告警规则

```c
1、userAgent: Go-http-client/1.1      # CDK特定的userAgent，此时，该字段为主要特征
2、responseStatus.code: 403           #默认serviceaccount无权限时 api-server返回的状态码
3、requestURI: /                      # 访问根目录
```

**2.3 漏洞利用**

Exploit模块包含的功能：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/9/57bc3f95-480f-49c5-b385-e2131ead9c9c.png?raw=true)

**2.3.1 容器逃逸**

其中，容器逃逸层面，一般是不需要与api-server做交互的，也就不会留下日志。但部分容器逃逸手段是利用错误配置的pod在创建时实施攻击。因此，容器逃逸的检测必须前置到容器创建时，记录下应用请求的权限，然后结合运行时入侵检测做进一步监控。

当一个拥有privilege、sys_admin、network、ipc等特殊权限的pod创建时，它的日志记录是这样的。

**audit.json**

```c
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1",
    "level": "RequestResponse",
    "stage": "ResponseComplete",
    "requestURI": "/api/v1/namespaces/testpods/pods",
    "verb": "create",
    "user": {
        "username": "kubernetes-admin",
        "groups": [
            "system:masters",
            "system:authenticated"
        ]
    },
    "sourceIPs": [
        "172.18.0.1"
    ],
    "userAgent": "kubectl1.16.15/v1.16.15 (darwin/amd64) kubernetes/2adc8d7",
    "objectRef": {
        "resource": "pods",
        "namespace": "testpods",
        "name": "testpod",
        "apiVersion": "v1"
    },
    "responseStatus": {
        "metadata": {},
        "code": 201
    },
    "responseObject": {
        "kind": "Pod",
        "apiVersion": "v1",
        "metadata": {
            "name": "testpod",
            "namespace": "testpods",
            "selfLink": "/api/v1/namespaces/testpods/pods/testpod",
            "uid": "e717d204-7e6d-4608-998b-648a8667e8e1",
            "resourceVersion": "13517",
            "creationTimestamp": "2023-02-02T09:51:10Z",
            "labels": {
                "creator": "zhiye",
                "team": "teamf"
            }
        },
        "spec": {
            "volumes": [
                {
                    "name": "rootfs",
                    "hostPath": {
                        "path": "/",
                        "type": ""
                    }
                }
            ],
            "containers": [
                {
                    "name": "trpe",
                    "image": "alpine",
                    "command": [
                        "/bin/sh",
                        "-c",
                        "tail -f /dev/null"
                    ],
                    "resources": {},
                    "volumeMounts": [
                        {
                            "name": "default-token-mm6s8",
                            "readOnly": true,
                            "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
                        }
                    ],
                    "terminationMessagePath": "/dev/termination-log",
                    "terminationMessagePolicy": "File",
                    "imagePullPolicy": "Always",
                    "securityContext": {
                        "capabilities": {
                            "add": [
                                "SYS_ADMIN"
                            ]
                        },
                        "privileged": true
                    }
                }
            ],
            "restartPolicy": "Always",
            "terminationGracePeriodSeconds": 30,
            "dnsPolicy": "ClusterFirst",
            "serviceAccountName": "default",
            "serviceAccount": "default",
            "hostNetwork": true,
            "hostPID": true,
            "hostIPC": true,
            "securityContext": {},
            "schedulerName": "default-scheduler",
            "tolerations": [
                {
                    "key": "node.kubernetes.io/not-ready",
                    "operator": "Exists",
                    "effect": "NoExecute",
                    "tolerationSeconds": 300
                },
                {
                    "key": "node.kubernetes.io/unreachable",
                    "operator": "Exists",
                    "effect": "NoExecute",
                    "tolerationSeconds": 300
                }
            ],
            "priority": 0,
            "enableServiceLinks": true
        },
        "status": {
            "phase": "Pending",
            "qosClass": "BestEffort"
        }
    },
    "requestReceivedTimestamp": "2023-02-02T09:51:10.632436Z",
    "stageTimestamp": "2023-02-02T09:51:10.660958Z",
    "annotations": {
        "authorization.k8s.io/decision": "allow",
        "authorization.k8s.io/reason": ""
    }
}
```

从日志上可以看出，针对于错误配置导致的逃逸，我们可以关注以下几个日志字段,制定告警规则。

```c
responseObject.spec.volumes                                         # 检测敏感卷，是否挂载docker.sock等
responseObject.spec.containers.volumeMounts                         # 检测敏感挂载，是否挂载docker.sock等
responseObject.spec.containers.securityContext.capabilities.add     # 是否使用SYS_ADMIN权限，（字段嵌套这么多层，真的得吐槽
responseObject.spec.containers.securityContext.privileged           # 检测是否为特权pod容器
responseObject.spec.hostNetwork                                     # 是否使用宿主机网络
responseObject.spec.hostPID                                         # 是否使用宿主机hostPID
responseObject.spec.hostIPC                                         # 是否共享宿主机内存
responseObject.spec.serviceAccount                                  # 是否使用特殊的serviceaccount  默认为default
```

**2.3.2 网络探测**

此功能为端口扫描+指纹识别，不涉及与API交互,未产生日志。

**2.3.3 信息窃取**

与api-server交互的为secrets、config和psp，如果是自动获取的话，cdk会发送两次请求，分别使用匿名账户和当前serviceaccout做list动作。

下边列举些主要特征

```c
requestURI: /api/v1/secrets，
requestURI: /api/v1/configmaps
requestURI: /apis/policy/v1beta1/podsecuritypolicies
userAgent: Go-http-client/1.1
user.username: "system:anonymous"
responseStatus.code: 403
"verb": "list"
```

**2.3.4 权限提升**

RBAC权限绕过

日志如下

```c
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1",
    "level": "RequestResponse",
    "auditID": "bfc643d6-8337-434e-9dec-ba41dd36bfa7",
    "stage": "ResponseComplete",
    "requestURI": "/api/v1/namespaces/kube-system/pods",
    "verb": "create",
    "user": {
        "username": "system:serviceaccount:test:default",
        "uid": "63b8dd88-88dd-4426-bdd1-7966906dc0d5",
        "groups": [
            "system:serviceaccounts",
            "system:serviceaccounts:test",
            "system:authenticated"
        ]
    },
    "sourceIPs": [
        "172.18.0.3"
    ],
    "userAgent": "Go-http-client/1.1",
    "objectRef": {
        "resource": "pods",
        "namespace": "kube-system",
        "apiVersion": "v1"
    },
    "responseStatus": {
        "metadata": {},
        "status": "Failure",
        "reason": "Forbidden",
        "code": 403
    },
    "responseObject": {
        "kind": "Status",
        "apiVersion": "v1",
        "metadata": {},
        "status": "Failure",
        "message": "pods is forbidden: User \"system:serviceaccount:test:default\" cannot create resource \"pods\" in API group \"\" in the namespace \"kube-system\"",
        "reason": "Forbidden",
        "details": {
            "kind": "pods"
        },
        "code": 403
    },
    "requestReceivedTimestamp": "2023-02-03T09:56:24.061825Z",
    "stageTimestamp": "2023-02-03T09:56:24.061888Z",
    "annotations": {
        "authorization.k8s.io/decision": "forbid",
        "authorization.k8s.io/reason": ""
    }
}
```

由上可发现如下特征：

```c
Pod内serviceaccount无权限的情况
requestURI: /api/v1/namespaces/kube-system/pods
userAgent: Go-http-client/1.1
responseStatus.code: 403 

Pod内serviceaccount有权限的情况
requestURI: /api/v1/namespaces/kube-system/pods
responseObject.metadata.selfLink: /api/v1/namespaces/kube-system/pods/cdk-rbac-bypass-create-pod  
responseObject.metadata.spec.containers.args: *cat /run/secrets/kubernetes.io/serviceaccount/token* 
verb: create
```

**2.3.5 持久化**

**部署daemonset后门**

源码是这样定义的（参考\[6\]）：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/9/3a866780-9b16-44b9-b962-a843f7739881.png?raw=true)

体现到日志中，有以下几个点：

```c
objectRef.name: cdk-backdoor-daemonset
objectRef.namespace: kube-system
responseObject.metadata.selfLink: /apis/apps/v1/namespaces/kube-system/daemonsets/cdk-backdoor-daemonset
responseObject.spec.template.spec.volumes.hostPath.path: /
responseObject.spec.template.spec.containers.name: cdk-backdoor-pod
responseObject.spec.template.spec.containers.securityContext[capabilities:ptivileged]:如图上所示
responseObject.spec.template.spec.[hostNetwork|hostPID]: true
```

部署K8S CronJob

CDK源代码是这样定义的（参考\[7\]）：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/9/7b9d1553-33ba-449f-ab6e-7fafbc6a6616.png?raw=true)

体现在日志中，特征有以下几点：

```c
requestURI: /apis/batch/v1beta1/namespaces/kube-system/cronjobs
verb: create
objectRef.name: cdk-backdoor-cronjob
responseObject.matadata.name: cdk-backdoor-cronjob
responseObject.matadata.selfLink: /apis/batch/v1beta1/namespaces/kube-system/cronjobs/cdk-backdoor-cronjob
responseObject.spec.jobTemplate.spec.template.spec.containers.name: cdk-backdoor-cronjob-container
```

部署影子k8s api-server

在pod权限足够的情况下，通过创建shadow api-server做权限维持，详情见参考\[4\]

在非二开的情况下，通过k8s日志升级可检测以下几个字段

```c
objectRef.name: *-shadow-*
responseObject.metadata.labels.component: kube-apiservershadow
responseObject.spec.containers.command: "--secure-port=9444"
```

**2.3.6 总结**

因为漏洞利用部分，动作都普遍较大，因此可观测字段已不仅仅局限于userAgent,其特征均比较明显，极富有工具本身特色

**2.4 API利用**

**2.4.1 Tool模块**

此处需要关注的（跟api-server有交互的）为kcurl命令，此命令可借助高权限serviceaccount账户列举/使用k8s资源。

执行./cdk kcurl default get 'https://10.96.0.1:443/api/v1/nodes' ，日志内容如下：

**audit.log**

```c
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1",
    "level": "Metadata",
    "auditID": "418cafa5-2c1e-4fbf-b086-3d68e321d2bb",
    "stage": "ResponseComplete",
    "requestURI": "/api/v1/nodes",
    "verb": "list",
    "user": {
        "username": "system:serviceaccount:test:default",
        "uid": "63b8dd88-88dd-4426-bdd1-7966906dc0d5",
        "groups": [
            "system:serviceaccounts",
            "system:serviceaccounts:test",
            "system:authenticated"
        ]
    },
    "sourceIPs": [
        "172.18.0.3"
    ],
    "userAgent": "Go-http-client/1.1",
    "objectRef": {
        "resource": "nodes",
        "apiVersion": "v1"
    },
    "responseStatus": {
        "metadata": {},
        "code": 200
    },
    "requestReceivedTimestamp": "2023-02-06T09:51:51.790260Z",
    "stageTimestamp": "2023-02-06T09:51:51.791075Z",
    "annotations": {
        "authorization.k8s.io/decision": "allow",
        "authorization.k8s.io/reason": "RBAC: allowed by ClusterRoleBinding \"defaultadmin\" of ClusterRole \"cluster-admin\" to ServiceAccount \"default/test\""
    }
}
```

日志中annotations.authorization.k8s.io/reason给出了允许执行的原因。我们可以根据如下三个字段制定告警规则：

```c
serviceaccount有权限的情况下：
annotations.authorization.k8s.io/reason
annotations.authorization.k8s.io/decision
userAgent: Go-http-client/1.1
responseStatus.code：200

无权限的情况下：
userAgent: Go-http-client/1.1
responseStatus.code：403
```

**2.4.2 总结**

因为tool模块需要借助高权限serviceaccount或token，因此，权限不足的情况下，api-server返回responseStatus.code: 403 的条目说明了api-server接受到了非预期请求，结合userAgent等信息便可输出可疑告警。

**3\. 不仅仅是CDK**

**3.1 CVE-2022-3172 K8S聚合**

**API（Aggregation API）****SSRF漏洞.**

聚合 API 实际上是在 kube-apiserver 中运行的，在新 API 注册之前，它并不会工作。如果要添加新的 API，则需要创建一个APIService 对象，用来申请 Kubernetes 中新的 URL 路径。注册成功后，当有发送到此路径中的请求，则会被转发到已经注册的 APIService 上。

APIService 可以将客户端的请求转发到任意的 URL 上，这就有可能会导致 Client 发送请求时，所携带的一些认证信息可能会被发送给第三方。

通过日志审计监控responseStatus.code字段来进行判断是否有出现重定向的情况，通过检测如下字段：

```c
responseObject.code:302
responseObject.code:301
```

**3.2 使用不合规镜像创建pod**

除了cdk这种常见的攻击手法，还有一些常见的异常行为需要我们关注，比如一般企业内，pod创建使用的容器均会从企业私有的镜像仓库中拉取，此时如果日志中出现了公共仓库的镜像，则可判断为异常，可关注以下字段

```c
verb : create
level:RequestResponse
esponseObject.kind:Pod 
requestObject.spec.containers.image:镜像仓库地址
```

**3.3 pod命令执行**

对于kubectl exec命令进行监控，可对如下字段进行监控，注：命令执行的后续无法通过日志审计来进一步监控，需结合运行时检测进一步分析

```c
objectRef.subresource:exec
objectRef.subresource:attach
userAgent
```

**4\. 落地实践踩过的坑**

1、主要为k8s日志类型比较多，且每种类型的字段名，字段数量均不一致，导致es在存储数据时存在索引内字段类型不一致无法解析存储的问题。

解决方案为对于字段不一致的obj，选择为不做深层次解析。（或者使用hdfs等存储方式，查询时对字段进行解析

2、日志量过大，导致api-server磁盘读写io过高

持续优化audit.yaml中的日志规则，对于其中的node/status,pod/status,coordination.k8s.io/leases等不做日志记录。

**5\. 总结**

本文从k8s日志审计的角度，分析当使用cdk等自动化攻击时，能够从日志中获取到的信息，并给出通过这些信息可监控字段的告警示例。因为这部分字段都比较固定，完全可以通过机器学习提升告警准确率。抛砖引玉，希望后续可以看到更多日志的分析防御角度。

当然，CDK作为开源工具，这些特征都可以做关键字替换。因此，笔者认为功夫应该用到平时，加强k8s的基线管控，比如避免出现高serviceaccount权限、通过准入策略限制使用的docker镜像，并部署容器运行时入侵检测平台。让安全能力覆盖每个环节，才能保证集群的安全稳定。

**6\. 参考**

\[1\] _https://github.com/cdk-team/CDK_  

\[2\] _https://www.cdxy.me/?p=839_

\[3\] _https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/_

\[4\] _https://discuss.kubernetes.io/t/security-advisory-cve-2022-3172-aggregated-api-server-can-cause-clients-to-be-redirected-ssrf/21322_

\[5\]_https://github.com/tencentyun/qcloud-documents/blob/master/product/%E5%AD%98%E5%82%A8%E4%B8%8ECDN/%E6%97%A5%E5%BF%97%E6%9C%8D%E5%8A%A1/%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5/TKE%20%E5%AE%A1%E8%AE%A1%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90.md_

\[6\] _https://github.com/cdk-team/CDK/blob/main/pkg/exploit/k8s\_backdoor\_daemonset.go#LL35-L87C2_

\[7\] _https://github.com/cdk-team/CDK/blob/main/pkg/exploit/k8s_cronjob.go#LL34-L59C2_

**以上是今天的分享内容，如果你有什么想法或疑问，欢迎大家在留言区与我们互动，如果喜欢本期内容的话，欢迎点个“在看”吧！**

**往期精彩指路**

*   [B站云原生混部技术实践](http://mp.weixin.qq.com/s?__biz=Mzg3Njc0NTgwMg==&mid=2247488761&idx=1&sn=8170d8306741e92f200acbfbedbf98ea&chksm=cf2cd1dcf85b58ca0f85a237ce0ef5f793a103fee2cba636b28ca39517605e559d2ed1ccb513&scene=21#wechat_redirect)
    
*   [B站基于Clickhouse的下一代日志体系建设实践  
    ](http://mp.weixin.qq.com/s?__biz=Mzg3Njc0NTgwMg==&mid=2247490544&idx=1&sn=2a6a72ac932911988968ef5c83d2e0be&chksm=cf2cded5f85b57c384a6d6a3c0d86cf118089aa76e514dd70e051dd2c4186e5bdda57e1f0c60&scene=21#wechat_redirect)
    
*   [B站埋点分析平台的构建之路](http://mp.weixin.qq.com/s?__biz=Mzg3Njc0NTgwMg==&mid=2247494098&idx=1&sn=adace79cf2c8104ba42adf7b17f59749&chksm=cf2f2cf7f858a5e1632cd900c80d522aa5fd0a74e7bbd2fc0a6c62d7f425372bd2eabc090f96&scene=21#wechat_redirect)
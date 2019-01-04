---
title: "openshift notes"
date: 2017-11-10
lastmod: 2018-6-10
draft: false
tags: ["tech", "kubernetes", "openshift", "container", "notes"]
categories: ["tech"]
description: "openshift使用笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# Openshift

## 权限控制 ##

[https://docs.openshift.com/container-platform/3.3/architecture/additional_concepts/authorization.html](openshift authorization)

首先是认证, 可以设置自己的认证服务器, 缺省用htpasswd

然后是权限控制: rule规则, role角色(规则集), bind绑定角色和用户(组)

权限控制分为cluster policy和local policy:

cluster policy作用于整个集群, local policy作用于具体某个project, cluster bind只能绑定到cluster role, local bind能绑定到cluster role和local role

serviceaccount也是一种特殊用户, 用户名"service:serviceaccount:<namespace>:<serviceaccount>", 需要注意权限bind的时候别直接写serviceaccount名, 而要写它的用户名


### 查看cluster roles ###

```
[root@nf5270m-ose ~]# oc get clusterroles
NAME
system:router
edit
system:pv-recycler-controller
cluster-admin
system:image-puller
registry-editor
system:discovery
system:node-admin
system:image-auditor
......
```

查看role的rules
```
[root@nf5270m-ose ~]# oc describe clusterroles cluster-pod-reader
Name:			cluster-pod-reader
Created:		About an hour ago
Labels:			<none>
Annotations:		<none>
Verbs			Non-Resource URLs	Extension	Resource Names	API Groups	Resources
[get list watch]	[]					[]		[]		[pods]
```

### 查看cluster policy ###

```
[root@nf5270m-ose ~]# oc get clusterpolicy
NAME      ROLES                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               LAST MODIFIED
default   admin, basic-user, cluster-admin, cluster-pod-reader, cluster-reader, cluster-status, edit, management-infra-admin, registry-admin, registry-editor, registry-viewer, self-provisioner, system:build-controller, system:build-strategy-custom, system:build-strategy-docker, system:build-strategy-source, system:daemonset-controller, system:deployer, system:deployment-controller, system:discovery, system:gc-controller, system:hpa-controller, system:image-auditor, system:image-builder, system:image-pruner, system:image-puller, system:image-pusher, system:job-controller, system:master, system:namespace-controller, system:node, system:node-admin, system:node-proxier, system:node-reader, system:oauth-token-deleter, system:pv-binder-controller, system:pv-provisioner-controller, system:pv-recycler-controller, system:registry, system:replication-controller, system:router, system:sdn-manager, system:sdn-reader, system:webhook, view   2016-08-16 15:19:09 +0800 CST
```

查看具体每个role的rules:
oc describe clusterpolicy default

### 查看roles和用户绑定 ###

```
[root@nf5270m-ose ~]# oc describe clusterpolicybinding :default
Name:							:default
Created:						5 days ago
Labels:							<none>
Annotations:						<none>
Last Modified:						2016-08-16 15:20:14 +0800 CST
Policy:							<none>
RoleBinding[basic-users]:
							Role:			basic-user
							Users:			<none>
							Groups:			system:authenticated
							ServiceAccounts:	<none>
							Subjects:		<none>
RoleBinding[cluster-admins]:
							Role:			cluster-admin
							Users:			<none>
							Groups:			system:cluster-admins
							ServiceAccounts:	<none>
							Subjects:		<none>
RoleBinding[cluster-pod-reader]:
							Role:			cluster-pod-reader
							Users:			podreader
							Groups:			<none>
							ServiceAccounts:	<none>
							Subjects:		<none>
```

### clusterpolicy ###

ClusterPolicy is a object that holds all the ClusterRoles for a particular namespace. There is at most one ClusterPolicy document per namespace

### 角色和用户binding

绑定serviceaccount: tacy到cluster-admin角色
`oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:default:tacy`

### 创建role
`oc export role sample`导出已经存在的role, 修改成自己需要的权限,然后通过创建它即可

## Quota ##

Quota分为:
1. resource quota,包括cpu和内存,
2. object count manage, 例如你可以限制一个project里面有多少个non-terminal的pod, 有多少service

### 　scope ###

Quota还能有scope, 有四个scope:
1. NotTerminating: Match pods where spec.activeDeadlineSeconds is nil, 匹配长期运行的pod
2. Terminating: Match pods where spec.activeDeadlineSeconds >= 0, 匹配短期存在的pod, 例如build pod, deployer pod, job pod等
3. BestEffort: Match pods that have best effort quality of service for either cpu or memory. 没有定义memory和cpu没有定义request和limit
4. NotBestEffort: Match pods that do not have best effort quality of service for cpu and memory. 定义了memory和cpu的request和或limit

BestEffort只能用于pod对象; pods, memory, requests.memory, limits.memory, cpu, requests.cpu, limits.cpu能用于所有4个scope

### limit & request ###

request是调度依据, node必须满足才能放置改pod, limit是上限

memory和cpu对于limit策略不一致

#### cpu ####

如果request和pod都没有指定, 可以用node的所有cpu资源, 但是优先级最低

request是pod的最小资源需求, 如果没有设置limit, 当node不存在cpu资源争用的时候, 可以使用node的所有cpu资源, 如果存在资源争用, request就会成为争夺cpu资源的权重值

limit是一个硬限制, 如果设置了limit, 该limit就是pod的上限, cpu调度器会做严格控制, 这类pod有一致的QoS

#### memory ####

1. BestEffort: 没有指定request和limit, 会使用node的所有memory, 这类pod在遇到Node OOM的时候最先被杀掉;
2. Burstable: 指定了request, 但是没有指定limit, 这类pod可以使用尽可能多的node内存, 遇到Node OOM, 后于BestEffort被杀掉
3. Guaranteed: 指定了request和limit, 遇到OOM, 最后被杀掉

### 设置Project Quota ###

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "4"
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "64"
    limits.memory: 256Gi
```
定义了所有NotTerminating的pod的限制, 最小不能低于requet, 最高不能超过limits

oc create -f compute-resource -n projectname

#### 一些注意事项 ####

如果project设置了Quota, 必须为project里面的所有pod设置request或和limit, 这样会比较麻烦, 你最好也为该project定义一个limit-range, 在limit-range里面可以定义request和limit的default值

### 设置Project limitrange ###

```
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: "core-resource-limits"
spec:
  limits:
    - type: "Pod"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "200m"
        memory: "6Mi"
    - type: "Container"
      max:
        cpu: "2"
        memory: "1Gi"
      min:
        cpu: "100m"
        memory: "4Mi"
      default:
        cpu: "300m"
        memory: "200Mi"
      defaultRequest:
        cpu: "200m"
        memory: "100Mi"
      maxLimitRequestRatio:
        cpu: "10"
```
oc create -f core-resource-limit -n projectname

有了这个之后, 项目里面的pod如果没有定义memory和cpu限制, 会用limitrange的default值


## router & route ##

router支持shard, router可以定义select label, 来选择支持那些route, route需要加label, 匹配的route才能被对应的router支持

External Load Balancer Source IP Preservation Proposal #30105
https://github.com/kubernetes/kubernetes/pull/30105/files

Source IP Preservation for cloud lb services #29409
https://github.com/kubernetes/kubernetes/pull/29409


# 实践

## 生产
### 环境准备
1. 安装环境


### 权限控制
#### 管理员
*  超级管理员登入
oc login -u system:admin

*  创建管理组
oc adm group add admin

*  创建用户
htpasswd htpasswd admin  (注意, 如果ha master, 需要在每台机器上都加用户)

*  添加用户至管理组
oc adm group add-users admin admin

* 添加可创建项目role至管理组
oc adm add-cluster-role-to-group self-provisioner admin


### 创建podreader
这个用户用于备份日志文件
* 创建用户
htpasswd /etc/origin/htpasswd podreader

* 创建clusterrole
先从一个role导出一个模板:
oc export clusterrole cluster-reader > podreader.yaml

修改podreader.xml
```
apiVersion: v1
kind: ClusterRole
metadata:
  creationTimestamp: null
  name: cluster-pod-reader
rules:
- apiGroups: null
  attributeRestrictions: null
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
  ```

创建该角色:
oc create -f podreader.xml

查看该角色
oc describe clusterroles cluster-pod-reader

* 绑定用户到角色
oc adm policy add-cluster-role-to-user cluster-pod-reader podreader

* 验证
oc login podreader
oc get pods --all-namespaces=true


### 日志
#### 导出stdout/stderr到文件
run.sh | tee -a stdout.log
### emptydir



1. import image
路径规划
ose/jboss-web
航信/jboss-web

2. build s2i image
docker build

3. create


## monitor

hawkular  & cockpit


## REST API

[REST API Call](https://docs.openshift.org/latest/rest_api/index.html#rest-api-index)

两种方式, 一种是session, 一种是serviceaccount

serviceaccount方法:

首先必须注意, ServiceAccount在Openshift里面也对应到一个用户, 用户名是"system:serviceaccount:<namespace>:<sa_name>", 所有如果你需要给ServiceAccount赋权限, 需要这样写: `oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:default:test_sa`

```
$ oc create serviceaccount robot

$ oc policy add-role-to-user admin system:serviceaccounts:test:robot   #注意, 这里test是project, 另外可能需要加cluster role

$ oc describe serviceaccount robot
Name:		robot
Namespace:	test
Labels:		<none>

Image pull secrets:	robot-dockercfg-rdrpg

Mountable secrets: 	robot-token-2dsne
                   	robot-dockercfg-rdrpg

Tokens:            	robot-token-2dsne
                   	robot-token-9efwm

$  oc describe secret robot-token-2dsne
Name:		robot-token-2dsne
Namespace:	test
Labels:		<none>
Annotations:	kubernetes.io/service-account.name=robot,kubernetes.io/service-account.uid=ea70e4c7-0663-11e6-b279-fa163e610e01

Type:	kubernetes.io/service-account-token

Data
===
token:		fyJhbGciOiJSUzI1NiIyInR5cCI2IkpXVCJ9...
ca.crt:		1070 bytes
namespace:	8 bytes

$ curl -X GET -H "Authorization: Bearer <token>" https://openshift.redhat.com:8443/oapi/v1 --insecure

```

## java heap size

get_heap_size 获取cgroup配合的内存， 做一些运算


## quota

oc create quota

## openshift
oc edit 保存之后即时生效



# apache + openshift_router
问题如下: 前端还有apache, 后端是openshift router, 而router提供的应用URL都必须用DNSName, 所以这里涉及到多个router配置问题

举个例子, 应用的路由地址: mskyweb-umetrip.umetripapps.ose.travelsky.com, 后端是三个router: 10.6.103.200-202

apache端配置就必须考虑设置host header, 参考: http://serverfault.com/questions/557233/apache-proxy-http-redirect-to-ip-and-set-hostname

Possibly you could use mod_headers in conjunction with mod_proxy. I haven't tested it though.

So for your app-dev vhost you could have:

RequestHeader set Host "app.internal.domain"
and then you would add:

ProxyPreserveHost On


# nginx-kubeproxy
sudo ./openshift start
sudo ./openshift start node --kubeconfig=./openshift.local.config/node-tacyarch/node.kubeconfig --enable="proxy" --proxy-mode="travelsky" --loglevel=2
sudo ./oc --config openshift.local.config/master/openshift-master.kubeconfig create -f ~/workspace/qemu/vm/kubernetes/echo-service.yaml
sudo make clean buil

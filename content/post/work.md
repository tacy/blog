---
title: "XBMC插件之网易公开课"
date: 2013-07-23
lastmod: 2013-07-23
draft: true
tags: ["tech", "python", "xmbc", "raspberrypi"]
categories: ["tech"]
description: "自己为XBMC写的一个网易公开课插件，满足课程分类/学校分类/查找功能"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
# 电采
每个部署大概3000-4000w,

1. 业务能否跑在阿里云

2. 如何和数据中心集成

3.


# 阿里交流

## ADS
MPP产品

分钟延迟(提交的数据先buffer,然后merge)
成本, ODPS单机存储18T,ADS大概0.5-1t
查询, 大表join,尽量把大表放在一个组里面

## ODPS
hadoop类型
两种入口: stream sq / api ,api接口吞吐量150M/s, stream sql吞吐量1-2w条/秒 ##

oracle -> ODPS 走CDP同步,吞吐量在100-200m

## OSPS
storm

## kv store
hbase

## 业务场景
### 档案变更
来源与营销系统, 营销系统档案变更(用户/电表/电网), 会推送到中间库, 我们从中间库去取

相关业务需要用到档案:
* 查询:
* 电量计算: 目前计算早晚一次, 希望流式计算, 采集驱动
* 电费计算: 有了电量然后计算电费.
* 统计分析: 档案数据结合电费电量结果
* 入库: 需要和档案结合


# 英大

东单生产, 亦庄/上海, 需要做同城灾备/异地数据备份

宋体/小四 visio/ppt



# 中航信
微服务测试: 如何解决复杂依赖问题

netflix

std标准: 数据产品,类似于元数据
iam身份:
sem

flannel & coreos


1. 基础设施(iaas/物理机)
2. 传统意义上中间件(

软件资源, 偏重于:运行时能力, 运维能力


    运维:交付,

    webapp httpsvc 微服务 自定义app

    运维能力

运维平台: dependecy trace


# 航旅纵横
## env
10.6.159.37:8443   https  admin/redhat  system/admin


master: 10.6.159.37  root/123456
node1 : 10.6.159.38
node2 : 10.6.159.36
node3 : 10.6.159.188

registry: 10.6.159.239  git account: redhat/openshift


## openshift

default和openshift两个namespace是系统用的

htpasswd -b /etc/origin/passwd test test
oc get clusterrole
oc adm policy test cluster-reader

### [jsonpath](http://kubernetes.io/docs/user-guide/jsonpath/)
oc get --all-namespaces=true pods -o jsonpath='{range .items[*]}[{.metadata.name},{.metadata.uid}] {end}'
oc get --all-namespaces=true pods -o jsonpath='{range .items[?(.metadata.uid=="7166ea52-4a37-11e6-a72c-002564fbe07d")]}{.metadata.name}'

## 结论
1. 用tomcat7

2. apache + router + pod 主要讨论问题诊断问题?

3. 冻结的节点的维护需要提供对应功能

4. 容器多用户支持

5. 密码明文

6. emptydir quota

7. 日志处理, 采用emptydir, node上根据event来做日志目录quota, 日志几种服务器ssh到node, 获取日志, 同时根据pod id查询到应用信息(可以通过annotation)


key: ymyang/668158Wind





## 服务

监控中心

rpc调用协议/httpinvoker

consumer client 实现路由机制

航班动态：
通过dmz apache访问外网
航班动态暴露的服务 - 有接近10几个外部服务调用
它调用别人：通过access调用航信服务

CAT通用


资源访问：缓存memcache/redis/数据库edb/oracle




## 服务改造

monitor
1. 分组
2. openshift node 部署一套

service：
1. 修改部署描述 、 通过工具
2. 通过mark标志， 通过服务分流机制

原则是从上往下建设

### 20170802
航旅纵横工作：
1. monitor支持分组注册
2. service mark分组调用

改造要点：
front apache weight

monitor部署在node上，不采用容器方式

mid+front

front采用host网络，不走router


### configmap
更新之后, 关联的pod需要重启, 如何实现


### 关于配置
位置不确定性


## 20160817
apache切换, 用的是mod_proxy_cluster, 而openshift 出来的是haproxy 或者是 external ip,




kubectl proxy --port=8080&
open http://localhost:8080/swagger-ui
2/2


/bin/sh -c "while true; do let a=a+1; done"


库的实现:
1. 语言相关
2. 技术相关*http_rest


2016-12-7, 航信
log agent:
用途
1. pod里面运行, 扫描container文件系统
2. node运行, 扫描操作系统日志

目前状态:
目录都在统一目录下, 下层没有规范

希望:
配置文件定义:
1. 定义通配符扫描:
   (pattern, 递归, type, 文件通配符号) 两级通配符 [文件通配符用数组, type, threshold, sizeM]
   threshold超出报警

   约定: 1. 日志有且只有一个文件被打开

2. 存储容量维护

2. 文件扫描规则
   //序号滚动() 不知道在写哪个文件???

   文件扫描几个需求:
   日志文件进行分类, 确定唯一功能
   读取最新变化, 带上元数据, 然后输出

   保留一定周期日志, 如何实现?

3. 日志读取限流

3. agent自身日志处理

7. agent需要寻址配置文件, 使用环境变量寻址


4. sink(stdout/kafka), 需要配置文件

正在读取的文件换名了.

需求规格说明书

5. TBD(



问题:
1. 几种日志收集
2.


1. systemd
2. 日志文件大小, 文件限制? 一类文件限制? 超出处理策略?



2017-10-16  --()
	1. 服务目录管理  (目录管理服务器)
--发布
     --支持版本
	 --支持属性(例如隐藏)
   --接口规范

2. 开发工具整合
   -- java
   -- 别人写好的服务, 有接口契约, 访问代码生成  ------别人的已经发布的服务/生成测试桩(单元测试)
   -- 自定义微服务接口(代码) java2wsdl/wsdl2java


   微服务代码---预发布
    -- 只需要导出接口描述, 但是比较低效, 如果直接发布到服务目录管理, 需要支持更细粒度控制

3. 测试工具整合
   -- ?????

接口规范怎么定义, 代码规范怎么定义.



# 联通

## oracle
set linesize 250 pagesize 0 trims on tab off long 1000000
column report format a220

select DBMS_SQL_MONITOR.REPORT_SQL_MONITOR (sql_id=>'bcc7fda2tfuzp',report_level=>'ALL') report from dual;



java.io.InputStreamReader reader = new java.io.InputStreamReader(in);
java.io.BufferedReader iin = new java.io.BufferedReader(reader);
String readed;
while ((readed = iin.readLine()) != null) {
  System.out.println(readed);
}

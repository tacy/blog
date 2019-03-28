---
title: "Axis2 performance improve"
date: 2019-03-38
lastmod: 2019-03-28
draft: false
tags: ["tech", "tuning", "performance", "java"]
categories: ["tech"]
description: "axis2调用代码的实现导致太多的tcp连接，在高频调用的场景中，容易导致socket timeout，其实，本身axis2本身支持连接重用"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

上面一篇关于连接超时的文章，就是由于axis2客户端实现导致，下面是我用测试代码（启了4个线程，每个线程做4次调用）运行过程中的抓包：
![axis2 client](/img/axis2-1.png)
可以看到，每次调用都会新建立一个连接，无法重用

优化了一下，下面是修改过后的代码运行的抓包
![axis2 client](/img/axis2-2.png)
连接都被重用了

下面是实现代码片段：

``` java
	private static ConfigurationContext defaultConfigurationContext = createDefaultConfigurationContext();

	private static ConfigurationContext createDefaultConfigurationContext(){
		// reuse HTTP connections if possible.
		MultiThreadedHttpConnectionManager mgr = new MultiThreadedHttpConnectionManager();
		HttpConnectionManagerParams params = mgr.getParams();
		if (params == null) {
			params = new HttpConnectionManagerParams();
			mgr.setParams(params);
		}
		params.setMaxTotalConnections(40);
		params.setDefaultMaxConnectionsPerHost(20);
		IdleConnectionTimeoutThread ict = new IdleConnectionTimeoutThread();
		ict.addConnectionManager(mgr);
		ict.setConnectionTimeout(15000);
		ict.start();
		HttpClient httpClient = new HttpClient(mgr);
		ConfigurationContext defaultConfigurationContext = null;
		try {
			defaultConfigurationContext = ConfigurationContextFactory.createDefaultConfigurationContext();
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		defaultConfigurationContext.setProperty(HTTPConstants.CACHED_HTTP_CLIENT, httpClient);
		return defaultConfigurationContext;
	}

	...

	ServiceClient sender = new ServiceClient(defaultConfigurationContext,null);
	OMElement result = sender.sendReceive(method);
	sender.cleanupTransport();
```

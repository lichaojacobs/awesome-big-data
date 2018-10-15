---
title: okhttp support 100-continue for palo
date: 2018-04-24 12:40:32
tags:
	- kylin
    - Java
    - 源码
---

## 前言

虽然百度的Palo是个很强大的，基于MPP Search Engine的OLAP框架，但是由于处于开源的早期阶段，各方面都不是很完善。其中，Palo集群的稳定性对于日渐依赖Palo的核心的业务来说显得尤为重要。最近也一直在做Palo稳定性建设相关的工作。在对全链路监控这块，自然而然地想到对业务中使用频繁的http-mini-load接口进行SDK封装，以实现对请求进行失败重试以及失败率的监控报警的功能。

## 遇到的问题

#### 问题描述

- 在实际的SDK封装中，用到了流行的okhttp，发送请求如下:

	```
        PaloHttpUtil paloHttpUtil = PaloHttpUtil.builder().build();
        String bodyStr = "1 1 2018-04-12 20:13:00 4101628087476389 1\n1 1 2018-04-12 20:13:00 4205819141030267 2";
        System.out.println(paloHttpUtil
                .put(String.format("http://xxxx:8030/api/feed/comment_trend/_load?" +
                        "label=comment_trend_load_%s&columns=trend_type,user_type,timestamp,mid,count", prefix))
                .auth("feed", "feed")
                .header("Expect", "100-continue")
                .body(bodyStr, ContentType.WILDCARD)
                .asyncSend(3, 10000, TimeUnit.MILLISECONDS)
                .string()

	```

	response: code 307 Temporary {"Status":"Failed"} 也就是说发生了重定向。对于服务器为何不在一个request中直接接收PUT的数据，这块贴一下100-continue的定义。

	```
	
	100 (Continue/继续) :如果服务器收到头信息中带有100-continue的请求，这是指客户端询问是否可以在后续的请求中发送附件。
在这种情况下，服务器用100(SC_CONTINUE)允许客户端继续或用417 (Expectation Failed)告诉客户端不同意接受附件。这个状态码是 HTTP 1.1中新加入的。
	

	```
	
	至于为什么发生了重定向先不考虑，先研究一下为什么okhttp不支持307重定向。
	

#### 源码追踪

- 我们通过debug深入源码看看在哪一步处理的307重定向

![](http://ol7zjjc80.bkt.clouddn.com/realInterceptor.png)

由图，我们可以看到对于request/response的处理，okhttp采取了插件的形式，类似于Spring AOP 源码中切面invoke方法的处理方式。这种插件的方式意味着我们可以定制化请求处理逻辑。借官方原图：

![](https://raw.githubusercontent.com/wiki/square/okhttp/interceptors@2x.png)

- 由于interceptor里面是可以执行重试逻辑或直接返回response，所以，我们再深入看看在哪个Interceptor里直接返回了response。断点打到RetryAndFollowUpInterceptor里如下的代码块：

	```
 Request followUp = this.followUpRequest(response, streamAllocation.route());
 				//这里followUP返回null
            if (followUp == null) {
                if (!this.forWebSocket) {
                    streamAllocation.release();
                }

                return response;
            }

	```

	由于followUp返回了null，导致response直接返回。说明当前的redirect策略不支持307重定向，再深入具体的重定向策略followUpRequest

	![](http://ol7zjjc80.bkt.clouddn.com/redirectInterceptor.png)

	发现307，308如果request method不等于GET且不为HEAD时直接返回了null，由此对于307 的PUT重定向操作okhttp是不支持的
	
	
## 问题的解决

#### okhttp 添加自定义redirect interceptor

- 前面提到，我们可以往okhttpClient里面添加自定义的interceptor来达到对request/response灵活劫持的目的。于是考虑加一个支持palo put 307 重定向的redirect interceptor.

- 大致的策略还是跟RetryAndFollowUpInterceptor一样，对followUpRequest的方法做了修改

	```
	case 300:
   case 301:
	case 302:
	case 303:
	case 307:
	
	```
	将307放到了300-303并列的位置，进入redirect逻辑。去掉将method统一替换成GET的逻辑:
	
	```
	if (HttpMethod.redirectsToGet(method)) {
		requestBuilder.method("GET", (RequestBody)null);
	} else {
		RequestBody requestBody = maintainBody ?userResponse.request().body() : null;
		requestBuilder.method(method, requestBody);
	}
	
	```
	改为:
	
	```
	boolean maintainBody = HttpMethod.requiresRequestBody(method);
                                    RequestBody requestBody = maintainBody ? userResponse.request().body() : null;
                                    requestBuilder.method(method, requestBody);
	
	```
	为了带上用户名密码，去掉逻辑:
	
	```
	 if (!this.sameConnection(userResponse, url)){
	 	requestBuilder.removeHeader("Authorization");
	}
	
	```
	
- 至此，Palo http-mini-load put 307 Temporary 重定向问题得到了解决

#### 使用apache httpcomponents

- 在apache httpcomponents中，可以设置redirectStrategy，来达到重定向的策略，且不受http code的约束

	![](http://ol7zjjc80.bkt.clouddn.com/apacheHttpComponents.png)
	
	可以看到本身的redirect机制还是比较强大的
	
- 不过鉴于本人习惯用okhttp，且用自定义的interceptro也能解决问题，所以暂时没有采用这种方法。

## 总结

- okhttp 不支持307除get意外其他request method重定向的原因不得而知。不过对于开源的组件，也不必要满足各种各样奇怪的胃口，对于需求的定制化留好可扩展接口就行。

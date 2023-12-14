---
layout: post
tags: ["Django","csrftoken"]
title: "Csrf Token Problem"
date: 2023-12-14T17:09:45+08:00
math: false
draft: false
---

一次开发基于Django的web项目过程中，发现一个奇怪的问题：原本能正常访问的post/patch/delete等非get接口，全部都返回403了。

基本情况是这样的，`Django==4.0.10`，`djangorestframework==3.13.1`, 我的接口是基于ModelViewSet或者APIView，上午还能正常访问，后来修了几个问题（性能问题），然后就发现，所有非get请求都返回403了，检查了中间件和 settings 里 `restframework` 配置变化，没看出什么问题来。

后来通过单步Django的代码，发现在 `rest_framework.views` 模块里的 `APIView` 类里有一个方法，代码如下:
```python
def perform_authentication(self, request):
	"""
	Perform authentication on the incoming request.

	Note that if you override this and simply 'pass', then authentication
	will instead be performed lazily, the first time either
	`request.user` or `request.auth` is accessed.
	"""
	request.user
```

每次单步到这里，就会跳出程序，并raise一个403出来。
==想知道怎么实现的==

其实到这里没有太大问题，除了上午能访问，下午不能访问这件事情比较闹心之外，于是找前端同时，让传入一个头 `X-CSRFToken` 进来，就能解决问题，但是前端说之前没有传也是好的啊，😅。

后来尝试在我自己的view里重新这个方法，变成下面这个样子：
```python
def dispatch(self, request, *args, **kwargs):
	csrf_token = request.COOKIES["csrftoken"]
	request.META["HTTP_X_CSRFTOKEN"] = csrf_token
	return super().dispatch(request, *args, **kwargs)
```

也就是在进入真正的逻辑代码前，自己把这个头从cookie里拿出来，硬塞进header里，问题解决，于是重新写了个BaseAPIView类，让所有view都继承这个类。

现在的问题是，我不知道原因，这就无语了。
---
layout: post
published: true
title: Spring MVC 架构
---
# 整体结构
	servlet 
    
    Httpservlet -- java
    
    FrameworkServlet
    
    DispatcherServlet
    --初始化九大组件
    doDispatcher
    --
    mappedHandler = getHandler(processedRequest, false);
	HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
	if (!mappedHandler.applyPreHandle(processedRequest, response)) {
		return;
	}
	// Actually invoke the handler.
	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
	mappedHandler.applyPostHandle(processedRequest, response, mv);
	processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);

# 九大组件


## HandlerMapping
根据request找到Handler Interceptors

## HandlerAdapter
使用Handler处理器进行工作的人

## HandlerExceptionResolver
异常处理

## ViewResolver
视图解析

## RequestToViewNameTranslator
从requeset拿到viewName（默认视图）

## LocaleResolver
从request中解析Locale

## ThemeResolver
解析主题

## MultipartResolver
处理上传请求

## FlashMapManager
管理FlashMap。flashMap是用来在request中传递参数



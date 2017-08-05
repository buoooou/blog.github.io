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

## HandlerAdapter

## HandlerExceptionResolver

## ViewResolver

## RequestToViewNameTranslator

## LocaleResolver

## ThemeResolver

## MultipartResolver

## FlashMapManager




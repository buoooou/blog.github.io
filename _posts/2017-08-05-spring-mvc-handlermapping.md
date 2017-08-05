---
layout: post
published: true
title: Spring MVC-HandlerMapping
---
## HandlerMapping 继承关系
	HandlerMapping
    只有一个方法：通过request获取handler
    HandlerExecutionChain getHandler(HttpServletRequest request)
 	  --AbstractHandlerMapping
    	--AbstractUrlHandlerMapping
    	  --SimpleUrlHandlerMapping
          --AbstractDetectingUrlHandlerMapping
        --AbstractHanlderMethodMapping
          --
          -- 
   
### AbstractHandlerMapping----HandlerMapping的抽象实现
	 --getHandler方法实现
     通过getHandlerInternal获取handler，该方法是子类实现的
     具体不同的子类实现方法不一样。
     若没有获取到则使用默认的handler
     handler转String，来获取bean
     
#### AbstractUrlHandlerMapping----AbstractHandlerMapping的子类
	--getHandlerInternal来获取handler
    通过lookupHandler获取handler
    若没有获取到则 用rootHandler或者默认Handler
    --lookupHandler
    从map中获取handler，若没有获取到则通过Pattern匹配，获取handler
    --  注
    map获取通过registerHandler方法进行
    该方法是子类调用初始化的
    
##### SimpleUrlHandlerMapping----AbstractUrlHandlerMapping子类
	--调用registerHandler是在initAppilicationContext中调用

##### AbstractDetectingUrlHandlerMapping----AbstractUrlHandlerMapping子类
	
 
    
    
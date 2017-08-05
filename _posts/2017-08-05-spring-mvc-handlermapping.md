---
layout: post
published: true
title: Spring MVC-HandlerMapping
---
## HandlerMapping 继承关系
	**HandlerMapping**
	只有一个方法：通过request获取handler
	HandlerExecutionChain getHandler(HttpServletRequest request)
        
    AbstractHandlerMapping----HandlerMapping的抽象实现
	 --getHandler方法实现
     通过getHandlerInternal获取handler，该方法是子类实现的
     具体不同的子类实现方法不一样。
     若没有获取到则使用默认的handler
     handler转String，来获取bean
     
- AbstractHandlerMapping
- --AbstractUrlHandlerMapping
- --AbstractHanlderMethodMapping

**下面分两大类讲handlerMapping实现**

- --AbstractHandlerMapping  
- ----AbstractHanlderMethodMapping
- ------RequestMappingInfoHandlerMapping
- --------RequestMappingHandlerMapping

**总结，_常用_
层次比较清晰：**

### AbstractHanlderMethodMapping----AbstractHandlerMapping的子类
	--getHandlerInternal来获取handler



        
- --AbstractHandlerMapping
- ----AbstractUrlHandlerMapping
- ------SimpleUrlHandlerMapping
- --------AbstractDetectingUrlHandlerMapping
- ----------BeanNameUrlHandlerMapping
- ----------AbstractControllerUrlHandlerMapping

**总结，_较少用_
层次比较清晰，一层套一层：
        首先在AbstractUrlHandlerMapping中设计了整体的结构，并完成了查找Handler的具体逻辑，其中需要提供一个保存url和Handler的对应关系的Map，这个map的内容是留给子类实现的，这里提供了注册方法：registerHandler。
        初始化map有两种方法：SimpleUrlHandlerMapping通过手动在配置文件里注册
        AbstractDetectingUrlHandlerMapping在spring容器里查找。查找分两类**    
     
### AbstractUrlHandlerMapping----AbstractHandlerMapping的子类
	--getHandlerInternal来获取handler
    通过lookupHandler获取handler
    若没有获取到则 用rootHandler或者默认Handler
    --lookupHandler
    从map中获取handler，若没有获取到则通过Pattern匹配，获取handler
    --  注
    map获取通过registerHandler方法进行
    该方法是子类调用初始化的
    
### SimpleUrlHandlerMapping----AbstractUrlHandlerMapping子类
	--调用registerHandler是在initAppilicationContext中调用registerHandlers

### AbstractDetectingUrlHandlerMapping----AbstractUrlHandlerMapping子类
	--调用registerHandler是在initAppilicationContext中调用的detectHandlers
    

    
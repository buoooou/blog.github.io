---
layout: post
title: Spring MVC
image: /img/hello_world.jpeg
tags:
  - random
  - exciting-stuff
published: false
---

Spring中 抽象方法的应用

public abstract class BeanFactoryUtils {


	public static boolean isFactoryDereference(String name) {
		return (name != null && name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
	}
}
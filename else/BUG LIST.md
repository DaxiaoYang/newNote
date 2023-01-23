# BUG LIST

在listener中用@autowired注解会显示空指针异常 

因为listener的生命周期由servlet管理 servlet容器不认识autowired注解

解决方法：通过工具类获取 且在web.xml配置listener时 先配置spring的listener

```java

WebApplicationContextUtils.getWebApplicationContext(event.getServletContext()).getBean(LoadDbToRedisServiceImpl.class)

```


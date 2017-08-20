---
title: 使用Spring AOP进行token验证的统一处理
date: 2017-08-11 14:04:05
type: "categories"
categories: "Java"
---
## 了解AOP

AOP是Aspect Oriented Program的首字母缩写,也就是我们所说的面向切面编程,它能在程序运行时,动态的将代码切入到类的指定方法,指定位置上.
<!-- more -->
>面向切面编程（AOP是Aspect Oriented Program的首字母缩写） ，我们知道，面向对象的特点是继承、多态和封装。而封装就要求将功能分散到不同的对象中去，这在软件设计中往往称为职责分配。实际上也就是说，让不同的类设计不同的方法。这样代码就分散到一个个的类中去了。这样做的好处是降低了代码的复杂程度，使类可重用。      
   但是人们也发现，在分散代码的同时，也增加了代码的重复性。什么意思呢？比如说，我们在两个类中，可能都需要在每个方法中做日志。按面向对象的设计方法，我们就必须在两个类的方法中都加入日志的内容。也许他们是完全相同的，但就是因为面向对象的设计让类与类之间无法联系，而不能将这些重复的代码统一起来。    
   也许有人会说，那好办啊，我们可以将这段代码写在一个独立的类独立的方法里，然后再在这两个类中调用。但是，这样一来，这两个类跟我们上面提到的独立的类就有耦合了，它的改变会影响这两个类。那么，有没有什么办法，能让我们在需要的时候，随意地加入代码呢？这种在运行时，动态地将代码切入到类的指定方法、指定位置上的编程思想就是面向切面的编程。       
   一般而言，我们管切入到指定类指定方法的代码片段称为切面，而切入到哪些类、哪些方法则叫切入点。有了AOP，我们就可以把几个类共有的代码，抽取到一个切片中，等到需要时再切入对象中去，从而改变其原有的行为。这样看来，AOP其实只是OOP的补充而已。OOP从横向上区分出一个个的类来，而AOP则从纵向上向对象中加入特定的代码。有了AOP，OOP变得立体了。如果加上时间维度，AOP使OOP由原来的二维变为三维了，由平面变成立体了。从技术上来说，AOP基本上是通过代理机制实现的。      
   AOP在编程历史上可以说是里程碑式的，对OOP编程是一种十分有益的补充。

附:[AOP详解](http://www.cnblogs.com/hongwz/p/5764917.html)

## AOP能带来的好处

   假如没有AOP,我们想要去校验Token是否过期了,然后将数据返回给前端.我们需要在每个接口上都要写上重复的代码去校验Token,这其实是件非常烦人的事情,这对以后的维护也增加了负担.而通过AOP,我们可以将这些横向的功能抽离出来形成一个独立的小模块,然后在指定的位置上插入这些功能就可以完成校验的流程.

## 实例

这个实例主要是来实现校验用户的Token,对于使用过期的Token以及不符合的Token的请求进行过滤,也就是身份验证.这里面也涉及到Token的读取方式,因为前端调用的每个接口都是需要后端去校验Token的(除了注册登录这类接口),验证是否合法,因此如果频繁的读取数据库的话,性能不是太好,所以在这里采取本地缓存的方式.首先会在缓存中读取Token,如果不存在就从数据库中读取.
### 缓存类

首先写一个简单的缓存类.

```java
public class SimpleCache {
   private static Map<String, UserToken> cacheMap = new HashMap<String, UserToken>();
   
   private static SimpleCache simpleCache = new SimpleCache();
   
   private SimpleCache() {}
   
   public static SimpleCache sharedInstance() {
      return simpleCache;
   }
   
   public void setUserToken(String key, UserToken token) {
        cacheMap.put(key, token);
    }

    public UserToken getUserToken(String key) {
        return cacheMap.get(key);
    }
   
   //校验token是否过期
   public boolean isExpired(UserToken token) {
      System.out.println("expire:" + token.getExpire() + " expired: " +
                System.currentTimeMillis());
      return token.getExpire() < System.currentTimeMillis();
   }
   
   //定期清理过期的缓存
   pulic void clearExpiredCache() {
       //这个主要是减轻缓存压力,这个可以设计为缓存达到一定的边界就开始清除过期的token.
   }
}

```
### Token类

主要有token,expire这2个属性,token的生成我这里是根据用户名,用户id以及登陆时间的组合然后用MD5生成的一个字符串.过期时间设置的3天.

```java
pulic class UserToken {
   
   private long expire;
   
   private String token;
   
   //Getter Setter
   ...
}
```

### UserAccessValidate注解类
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface UserAccessValidate {

}
```

### UserAccessValidateHandler类
进行切面的类

```java
@Aspect
@Component
public class UserAccessValidateHandler {

    @Resource
    private UserCenterService userCenterService;

    @Around("@annotation(com.ysq.annotation.UserAccessValidate)")
    public Object userAccessValidate(ProceedingJoinPoint point) throws Throwable    {
        String token = HttpContext.getRequest().getParameter("token");
        //校验失败的话,请求就会终止.
        if (token == null || token.equals("")) {
            return new Response().failure("参数错误,请检查");
        }
        if (validateToken(token)) {
            return new Response().failure("token is invalid");
        }
        System.out.println("token: " + token);
  
        return point.proceed();
    }


    private boolean validateToken(String token) {
        SimpleCache simpleCache = SimpleCache.sharedInstance();
        UserToken userToken = simpleCache.getCacheContent(token);
        //如果缓存中不存在token,则从数据库中查询token,如果过期了则返回true,否则返回false
        if (userToken == null) {
            UserInfo info = this.userCenterService.selectByToken(token);
            if (info != null) {
                if (info.getExpire() < System.currentTimeMills()) {
                   return true;
                }
                UserToken userToken1 = new UserToken();
                userToken1.setToken(info.getToken());
                userToken1.setExpire(info.getExpire());
                simpleCache.setCacheContent(info.getToken(), userToken1);
                return false;
            }
            return true;
        } else {

            return !userToken.getToken().equals(token) || simpleCache.isExpired
                    (userToken);
        }
    }

```
在这里也说明一下上面出现的HttpContext,这个类是用来获取每次请求的HttpServletRequest,HttpServletResponse实例,具体的实现是通过ThreadLocal这个类来做的,有兴趣的可以百度,谷歌一下.我们知道在切面方法中是可以获取到接口传递过来的参数,但是会显得麻烦,因为这样的话,每个接口的要约定第几个参数是HttpServletRequest,不然的话,要一个个遍历所有的参数然后判断是否是HttpServletRequest,而且在团队开发中,是不太理想的.因此我选择了用这种方式去获取HttpServletRequest,这样的话,会显得极为方便.

### 配置自动代理

在Spring的配置文件中增加下面的配置

```xml
    <context:component-scan base-package="com.ysq.service.aspect" />
    <aop:aspectj-autoproxy proxy-target-class="true" />

```

## 运行测试

有了上面的工作之后,现在只需要在Controller类的请求方法上加上@UserAccessValidate这个注解就行了.

```java
    //查询用户信息
    @RequestMapping(value = "/query" , method = RequestMethod.POST)
    @ResponseBody
    @UserAccessValidate
    public Response queryUserInfo(String account) {
        return this.userCenterService.queryUserInfo(account);
    }
```
OK,运行程序测试一下.先登录一下分配一个Token.
![login](/images/aoplogin.png)
然后测试一下用户查询接口.
![query1](/images/aopquery1.png)
再随便输入一个token
![query2](/images/aopquery2.png)
不输入token
![query3](/images/aopquery3.png)
## 总结
通过Spring AOP,我们可以将对多个对象产生影响的公共行为,将其封装一个可重用的模块,这样就减少了系统中的重复代码,降低了模块间的耦合度,同时也提高了系统的可维护性.
<font face="黑体" color=black>下篇预告: 还是AOP,AOP收集系统的异常日志,定位线上环境存在的bug.</font>



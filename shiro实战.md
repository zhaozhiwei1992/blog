---
title: shiro实战
date: 2019-08-18
updated: 2019-08-18
tags: [springboot, shiro, swagger, 权限控制]
---

基本设置 springboot整合
=======================

1.  引入依赖包

``` {.xml}
<!-- https://mvnrepository.com/artifact/org.apache.shiro/shiro-core -->
<dependency>
  <groupId>org.apache.shiro</groupId>
  <artifactId>shiro-spring</artifactId>
  <version>1.4.0</version>
</dependency>
```

配置文件
========

1.  创建shiro.ini

``` {.ini}
#格式： 用户名=密码,角色1,角色2
[users]
ck=123456,admin

#格式：角色=权限1,权限2
[roles]
admin=add
teacher=update
```

1.  使用api测试验证

``` {.java}
public Map<String,Object> userLoginAction (String userName,String passWord){

    Map<String,Object> resultMap = new HashMap<>();
    //初始化SecurityManager对象
    Factory<SecurityManager> factory = new IniSecurityManagerFactory("classpath:shiro.ini");
    //通过SecurityManager工厂对象,获取SecurityManager实例对象.
    SecurityManager securityManager =  factory.getInstance();
    // 把 securityManager 实例 绑定到 SecurityUtils
    SecurityUtils.setSecurityManager(securityManager);
    //组建Subject主体.
    Subject subject = SecurityUtils.getSubject();
    //创建 token 令牌
    UsernamePasswordToken token = new UsernamePasswordToken(userName,passWord);
    //用户登录操作.
    try{
        subject.login(token);
        resultMap.put("code","200");
        resultMap.put("msg","用户登录成功");
    }catch (AuthenticationException e){
        //登录失败原因 1 用户不存在 2 用户密码不正确
        resultMap.put("code","-1");
        resultMap.put("msg","用户登录失败");
    }
    return resultMap;

}
```

自定义
======

1.  自定义配置类

``` {.java}
import org.apache.shiro.mgt.SecurityManager;
import lombok.extern.slf4j.Slf4j;
import org.apache.shiro.authc.*;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.realm.SimpleAccountRealm;
import org.apache.shiro.spring.LifecycleBeanPostProcessor;
import org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor;
import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
import org.apache.shiro.subject.PrincipalCollection;
import org.apache.shiro.web.mgt.DefaultWebSecurityManager;
import org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.DependsOn;
import org.springframework.web.servlet.handler.SimpleMappingExceptionResolver;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Properties;

@Slf4j
@Configuration
public class CustomShiroConfiguration {

  /**
    * Filter Name	Class
    * anon	org.apache.shiro.web.filter.authc.AnonymousFilter
    * authc	org.apache.shiro.web.filter.authc.FormAuthenticationFilter
    * authcBasic	org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter
    * perms	org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter
    * port	org.apache.shiro.web.filter.authz.PortFilter
    * rest	org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter
    * roles	org.apache.shiro.web.filter.authz.RolesAuthorizationFilter
    * ssl	org.apache.shiro.web.filter.authz.SslFilter
    * user	org.apache.shiro.web.filter.authc.UserFilter
    * anon:所有 url 都都可以匿名访问
    * authc: 需要认证才能进行访问
    * user:配置记住我或认证通过可以访问
    *
    * @param securityManager
    * @return
    */
  @Bean
  public ShiroFilterFactoryBean shiroFilter(SecurityManager securityManager) {
      log.info("注入Shiro的Web过滤器-->shiroFilter", ShiroFilterFactoryBean.class);
      ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();

      //Shiro的核心安全接口,这个属性是必须的
      shiroFilterFactoryBean.setSecurityManager(securityManager);
      //要求登录时的链接(可根据项目的URL进行替换),非必须的属性,默认会自动寻找Web工程根目录下的"/login.jsp"页面
      shiroFilterFactoryBean.setLoginUrl("/login");
      //登录成功后要跳转的连接,逻辑也可以自定义，例如返回上次请求的页面
      shiroFilterFactoryBean.setSuccessUrl("/index");
      //用户访问未对其授权的资源时,所显示的连接
      shiroFilterFactoryBean.setUnauthorizedUrl("/403");
      /*定义shiro过滤器,例如实现自定义的FormAuthenticationFilter，需要继承FormAuthenticationFilter **本例中暂不自定义实现，在下一节实现验证码的例子中体现 */

      /*定义shiro过滤链 Map结构 * Map中key(xml中是指value值)的第一个'/'代表的路径是相对于HttpServletRequest.getContextPath()的值来的 *
      anon：它对应的过滤器里面是空的,什么都没做,这里.do和.jsp后面的*表示参数,比方说login.jsp?main这种 * authc：该过滤器下的页面必须验证后才能访问,它是Shiro内置的一个拦截器org
      .apache.shiro.web.filter.authc.FormAuthenticationFilter */
      Map<String, String> filterChainDefinitionMap = new LinkedHashMap<String, String>();
      // 配置退出过滤器,其中的具体的退出代码Shiro已经替我们实现了
      filterChainDefinitionMap.put("/logout", "logout");

      // <!-- 过滤链定义，从上向下顺序执行，一般将 /**放在最为下边 -->:这是一个坑呢，一不小心代码就不好使了;
      // <!-- authc:所有url都必须认证通过才可以访问; anon:所有url都都可以匿名访问-->
      filterChainDefinitionMap.put("/login", "anon");//anon 可以理解为不拦截
      filterChainDefinitionMap.put("/reg", "anon");
      filterChainDefinitionMap.put("/plugins/**", "anon");
      filterChainDefinitionMap.put("/pages/**", "anon");
      filterChainDefinitionMap.put("/api/**", "anon");
      filterChainDefinitionMap.put("/dists/img/*", "anon");
      filterChainDefinitionMap.put("/**", "authc");

      shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);

      return shiroFilterFactoryBean;
  }

//    @Bean
//    public EhCacheManager ehCacheManager() {
//        EhCacheManager cacheManager = new EhCacheManager();
//        return cacheManager;
//    }

  @Bean
  public CustomShiroRealm customShiroRealm() {
      return new CustomShiroRealm();
  }

  @Bean
  public SecurityManager securityManager(CustomShiroRealm userRealm) {
      log.info("注入Shiro的Web过滤器-->securityManager", ShiroFilterFactoryBean.class);
      DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
      securityManager.setRealm(userRealm);
      //注入缓存管理器;
//        securityManager.setCacheManager(ehCacheManager());//这个如果执行多次，也是同样的一个对象;
      return securityManager;
  }

  /**
    * Shiro生命周期处理器 * @return
    */
  @Bean
  public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
      return new LifecycleBeanPostProcessor();
  }

  /**
    * 开启Shiro的注解
    * (如@RequiresRoles,@RequiresPermissions),需借助SpringAOP扫描使用Shiro注解的类,并在必要时进行安全逻辑验证
    * 配置以下两个bean(DefaultAdvisorAutoProxyCreator(可选)和AuthorizationAttributeSourceAdvisor)即可实现此功能
    *
    * @return
    */
  @Bean
  @DependsOn({"lifecycleBeanPostProcessor"})
  public DefaultAdvisorAutoProxyCreator advisorAutoProxyCreator() {
      DefaultAdvisorAutoProxyCreator advisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
      advisorAutoProxyCreator.setProxyTargetClass(true);
      return advisorAutoProxyCreator;
  }

  @Bean
  public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
      AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor =
              new AuthorizationAttributeSourceAdvisor();
      authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
      return authorizationAttributeSourceAdvisor;
  }

  /**
    * 异常处理
    *
    * @return
    */
  @Bean
  public SimpleMappingExceptionResolver simpleMappingExceptionResolver() {
      SimpleMappingExceptionResolver r = new SimpleMappingExceptionResolver();
      Properties mappings = new Properties();
      mappings.setProperty("DatabaseException", "databaseError");//数据库异常处理
      mappings.setProperty("UnauthorizedException", "403");
      r.setExceptionMappings(mappings);  // None by default
      r.setDefaultErrorView("error");    // No default
      r.setExceptionAttribute("ex");     // Default is "exception"
      //r.setWarnLogCategory("example.MvcLogger");     // No default
      return r;
  }

  /**
    * 自定义方式
    */
  class CustomShiroRealm extends AuthorizingRealm {

      /**
        * 授权, 通过角色授权, 这里设置角色信息即可
        * 参考: https://gitee.com/ityouknow/spring-boot-examples/blob/master/spring-boot-shiro/src/main/java/com/neo
        * /config/MyShiroRealm.java
        * https://blog.csdn.net/fu_fei_wen/article/details/77571989
        *
        * @param principals
        * @return
        */
      @Override
      protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
          //在实际开发中，开发者根据自身情况自行获取用户信息。此处简化如下：
          String role = "admin";
          String permission = "add";

          log.info("{}, role:{}, permission:{}", Thread.currentThread().getStackTrace()[1].getMethodName(), role,
                  permission);

          //设置权限信息
          SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
          simpleAuthorizationInfo.addRole(role);
          simpleAuthorizationInfo.addStringPermission(permission);

          return simpleAuthorizationInfo;
      }


      /**
        * 主要是用来进行身份认证的，也就是说验证用户输入的账号和密码是否正确。
        *
        * @param token
        * @return
        * @throws AuthenticationException
        */
      @Override
      protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token)
              throws AuthenticationException {
          //在实际开发中，开发者根据自身情况自行获取用户信息。此处简化如下：
          String name = "zhangsan";
          String pwd = "11";

          UsernamePasswordToken upt = (UsernamePasswordToken) token;
          String username = upt.getUsername();
          String password = new String(upt.getPassword());

          if (name.equals(username) && pwd.equals(password)) {
              return new SimpleAuthenticationInfo(username, password, "zhangsansan");
          }
          return null;
      }
  }
}

```

1.  登录控制器

``` {.java}
package com.lx.demo.springbootshiro.web.controller;

import com.lx.demo.springbootshiro.domain.User;
import com.lx.demo.springbootshiro.service.UserService;
import lombok.extern.slf4j.Slf4j;
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.*;
import org.apache.shiro.subject.Subject;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import javax.validation.Valid;

@Slf4j
@Controller
public class SecurityController {

  @Autowired
  private UserService userService;

  @GetMapping("/login")
  public String loginForm() {
      return "login";
  }

  @GetMapping("/index")
  public String index(){
      return "index";
  }

  @PostMapping("/login")
  public String login(@Valid User user, BindingResult bindingResult, RedirectAttributes redirectAttributes) {
      if (bindingResult.hasErrors()) {
          return "login";
      }
      String loginName = user.getLoginName();
      log.info("准备登陆用户 => {}", loginName);
      UsernamePasswordToken token = new UsernamePasswordToken(loginName,user.getPassword());
      //获取当前的Subject
      Subject currentUser = SecurityUtils.getSubject();
      try {
          //在调用了login方法后,SecurityManager会收到AuthenticationToken,并将其发送给已配置的Realm执行必须的认证检查
          //每个Realm都能在必要时对提交的AuthenticationTokens作出反应
          //所以这一步在调用login(token)方法时,它会走到MyRealm.doGetAuthenticationInfo()方法中,具体验证方式详见此方法
          log.info("对用户[" + loginName + "]进行登录验证..验证开始");
          currentUser.login(token);
          log.info("对用户[" + loginName + "]进行登录验证..验证通过");
      } catch (UnknownAccountException uae) {
          log.info("对用户[" + loginName + "]进行登录验证..验证未通过,未知账户");
          redirectAttributes.addFlashAttribute("message", "未知账户");
      } catch (IncorrectCredentialsException ice) {
          log.info("对用户[" + loginName + "]进行登录验证..验证未通过,错误的凭证");
          redirectAttributes.addFlashAttribute("message", "密码不正确");
      } catch (LockedAccountException lae) {
          log.info("对用户[" + loginName + "]进行登录验证..验证未通过,账户已锁定");
          redirectAttributes.addFlashAttribute("message", "账户已锁定");
      } catch (ExcessiveAttemptsException eae) {
          log.info("对用户[" + loginName + "]进行登录验证..验证未通过,错误次数过多");
          redirectAttributes.addFlashAttribute("message", "用户名或密码错误次数过多");
      } catch (AuthenticationException ae) {
          //通过处理Shiro的运行时AuthenticationException就可以控制用户登录失败或密码错误时的情景
          log.info("对用户[" + loginName + "]进行登录验证..验证未通过,堆栈轨迹如下");
          ae.printStackTrace();
          redirectAttributes.addFlashAttribute("message", "用户名或密码不正确");
      }
      //验证是否登录成功
      if (currentUser.isAuthenticated()) {
          log.info("用户[" + loginName + "]登录认证通过(这里可以进行一些认证通过后的一些系统参数初始化操作)");
          return "redirect:/index";
      } else {
          token.clear();
          return "redirect:/login";
      }
  }

  @GetMapping("/logout")
  public String logout(RedirectAttributes redirectAttributes) {
      //使用权限管理工具进行用户的退出，跳出登录，给出提示信息
      SecurityUtils.getSubject().logout();
      redirectAttributes.addFlashAttribute("message", "您已安全退出");
      return "redirect:/login";
  }


//    @GetMapping("/reg")
//    @ResponseBody
//    public Result<String> reg(@Valid User user, BindingResult bindingResult) {
//        if (bindingResult.hasErrors()) {
//            return Result.error("用户信息填写不完整");
//        }
//        userService.save(user);
//        return Result.ok();
//    }
}
```

1.  登录后一段时间内所有请求都可以通过

测试
----

1.  通过浏览器访问，正常登录即可
2.  如果通过curl访问需要保留session信息，下次请求带入,否则需要一直校验
    -   curl -H Content-Type:application/json;charset=UTF-8 -d
        {\"loginName\":\"zhangsan\",\"password\":\"11\"} -c mycookie -X
        POST <http://127.0.0.1:8080/login>
    -   curl -b mycookie -X GET <http://localhost:8080/users/all>

集成redis-cache
===============

集成缓存
--------

``` {.java}
  /**
* cacheManager 缓存 redis实现
* 使用的是shiro-redis开源插件
*
* @return
*/
public RedisCacheManager cacheManager() {
  RedisCacheManager redisCacheManager = new RedisCacheManager();
  redisCacheManager.setRedisManager(redisManager());
  return redisCacheManager;
}

/**
* 配置shiro redisManager
* 使用的是shiro-redis开源插件
*
* @return
*/
public RedisManager redisManager() {
  RedisManager redisManager = new RedisManager();
  redisManager.setHost("localhost");
  redisManager.setPort(6379);
  redisManager.setExpire(1800);// 配置缓存过期时间
  redisManager.setTimeout(0);
  // redisManager.setPassword(password);
  return redisManager;
}

/**
* Session Manager
* 使用的是shiro-redis开源插件
*/
@Bean
public DefaultWebSessionManager sessionManager() {
  DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();
  sessionManager.setSessionDAO(redisSessionDAO());
  return sessionManager;
}

/**
* RedisSessionDAO shiro sessionDao层的实现 通过redis
* 使用的是shiro-redis开源插件
*/
@Bean
public RedisSessionDAO redisSessionDAO() {
  RedisSessionDAO redisSessionDAO = new RedisSessionDAO();
  redisSessionDAO.setRedisManager(redisManager());
  return redisSessionDAO;
}

```

支持多客户端登录踢出
--------------------

1.  设置过滤器

``` {.java}
package com.lx.demo.springbootshiro.filter;

import com.lx.demo.springbootshiro.domain.User;
import org.apache.shiro.cache.Cache;
import org.apache.shiro.cache.CacheManager;
import org.apache.shiro.session.Session;
import org.apache.shiro.session.mgt.DefaultSessionKey;
import org.apache.shiro.session.mgt.SessionManager;
import org.apache.shiro.subject.Subject;
import org.apache.shiro.web.filter.AccessControlFilter;
import org.apache.shiro.web.util.WebUtils;

import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.io.PrintWriter;
import java.io.Serializable;
import java.util.Deque;
import java.util.HashMap;
import java.util.LinkedList;
import java.util.Map;

/**
* 踢出最好在chrome无痕模式下测试同一个用户, 这样可以保证每次都有新sessionid
* 默认只允许一个用户登录在一个地方登录，　这样第二个登录后，第一个刷新页面发现被踢出
*/
public class KickoutSessionControlFilter extends AccessControlFilter {

private String kickoutUrl; //踢出后到的地址
private boolean kickoutAfter = false; //踢出之前登录的/之后登录的用户 默认踢出之前登录的用户
private int maxSession = 1; //同一个帐号最大会话数 默认1

private SessionManager sessionManager;
private Cache<String, Deque<Serializable>> cache;

public void setKickoutUrl(String kickoutUrl) {
    this.kickoutUrl = kickoutUrl;
}

public void setKickoutAfter(boolean kickoutAfter) {
    this.kickoutAfter = kickoutAfter;
}

public void setMaxSession(int maxSession) {
    this.maxSession = maxSession;
}

public void setSessionManager(SessionManager sessionManager) {
    this.sessionManager = sessionManager;
}

//设置Cache的key的前缀
public void setCacheManager(CacheManager cacheManager) {
    this.cache = cacheManager.getCache("shiro_redis_cache");
}

@Override
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
    return false;
}

@Override
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
    Subject subject = getSubject(request, response);
    if (!subject.isAuthenticated() && !subject.isRemembered()) {
        //如果没有登录，直接进行之后的流程
        return true;
    }


    Session session = subject.getSession();
    String username = subject.getPrincipal().toString();
    Serializable sessionId = session.getId();

    //读取缓存   没有就存入
    Deque<Serializable> deque = cache.get(username);

    //如果此用户没有session队列，也就是还没有登录过，缓存中没有
    //就new一个空队列，不然deque对象为空，会报空指针
    if (deque == null) {
        deque = new LinkedList<Serializable>();
    }

    //如果队列里没有此sessionId，且用户没有被踢出；放入队列
    if (!deque.contains(sessionId) && session.getAttribute("kickout") == null) {
        //将sessionId存入队列
        deque.push(sessionId);
        //将用户的sessionId队列缓存
        cache.put(username, deque);
    }

    //如果队列里的sessionId数超出最大会话数，开始踢人
    while (deque.size() > maxSession) {
        Serializable kickoutSessionId = null;
        if (kickoutAfter) { //如果踢出后者
            kickoutSessionId = deque.removeFirst();
            //踢出后再更新下缓存队列
            cache.put(username, deque);
        } else { //否则踢出前者
            kickoutSessionId = deque.removeLast();
            //踢出后再更新下缓存队列
            cache.put(username, deque);
        }


        try {
            //获取被踢出的sessionId的session对象
            Session kickoutSession = sessionManager.getSession(new DefaultSessionKey(kickoutSessionId));
            if (kickoutSession != null) {
                //设置会话的kickout属性表示踢出了
                kickoutSession.setAttribute("kickout", true);
            }
        } catch (Exception e) {//ignore exception
        }
    }

    //如果被踢出了，直接退出，重定向到踢出后的地址
    if (session.getAttribute("kickout") != null) {
        //会话被踢出了
        try {
            //退出登录
            subject.logout();
        } catch (Exception e) { //ignore
        }
        saveRequest(request);

        Map<String, String> resultMap = new HashMap<String, String>();
        //判断是不是Ajax请求
        if ("XMLHttpRequest".equalsIgnoreCase(((HttpServletRequest) request).getHeader("X-Requested-With"))) {
            resultMap.put("user_status", "300");
            resultMap.put("message", "您已经在其他地方登录，请重新登录！");
            //输出json串
            out(response, resultMap);
        } else {
            //重定向
            WebUtils.issueRedirect(request, response, kickoutUrl);
        }
        return false;
    }
    return true;
}

private void out(ServletResponse hresponse, Map<String, String> resultMap)
        throws IOException {
    try {
        hresponse.setCharacterEncoding("UTF-8");
        PrintWriter out = hresponse.getWriter();
//            out.println(JSON.toJSONString(resultMap));
        out.flush();
        out.close();
    } catch (Exception e) {
        System.err.println("KickoutSessionFilter.class 输出JSON异常，可以忽略。");
    }
}
}
```

1.  配置过滤器

``` {.java}
  /**
* 限制同一账号登录同时登录人数控制
*
* @return
*/
@Bean
public KickoutSessionControlFilter kickoutSessionControlFilter() {
  KickoutSessionControlFilter kickoutSessionControlFilter = new KickoutSessionControlFilter();
  kickoutSessionControlFilter.setCacheManager(cacheManager());
  kickoutSessionControlFilter.setSessionManager(sessionManager());
  kickoutSessionControlFilter.setKickoutAfter(false);
  kickoutSessionControlFilter.setMaxSession(1);
  kickoutSessionControlFilter.setKickoutUrl("/logout");
  return kickoutSessionControlFilter;
}

```

1.  springboot集成问题处理及注册

``` {.java}
  /**
* org.apache.shiro.UnavailableSecurityManagerException: No SecurityManager accessible to the calling code,
* @return
*/
@Bean
public FilterRegistrationBean delegatingFilterProxy(){
  FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
  DelegatingFilterProxy proxy = new DelegatingFilterProxy();
  proxy.setTargetFilterLifecycle(true);
  proxy.setTargetBeanName("shiroFilter");
  filterRegistrationBean.setFilter(proxy);
  return filterRegistrationBean;
}

```

测试
----

通过无痕模式访问，每次变更sessionid,就被干出去了

权限控制
========

注解方式
--------

**注: 处理顺序与标题顺序一致**

### RequiresRoles:

当前Subject必须拥有所有指定的角色时，才能访问被该注解标注的方法。如果当天Subject不同时拥有所有指定角色，则方法不会执行还会抛出AuthorizationException异常。

``` {.example}
//属于user角色
@RequiresRoles("user")

//必须同时属于user和admin角色
@RequiresRoles({"user","admin"})

//属于user或者admin之一;修改logical为OR 即可
@RequiresRoles(value={"user","admin"},logical=Logical.OR)
```

### RequiresPermissions:

当前Subject需要拥有某些特定的权限时，才能执行被该注解标注的方法。如果当前Subject不具有这样的权限，则方法不会被执行。

``` {.example}
//符合index:hello权限要求
@RequiresPermissions("index:hello")

//必须同时复核index:hello和index:world权限要求
@RequiresPermissions({"index:hello","index:world"})

//符合index:hello或index:world权限要求即可
@RequiresPermissions(value={"index:hello","index:world"},logical=Logical.OR)
```

### RequiresAuthentication:

使用该注解标注的类，实例，方法在访问或调用时，当前Subject必须在当前session中已经过认证。

### RequiresUser

当前Subject必须是应用的用户，才能访问或调用被该注解标注的类，实例，方法。

### RequiresGuest:

使用该注解标注的类，实例，方法在访问或调用时，当前Subject可以是"gust"身份，不需要经过认证或者在原先的session中存在记录。

动态权限控制
------------

**这里要特别注意/\*\*的控制，必须放最后，否则后续的权限可能不起作用**

1.  增加权限控制类

``` {.java}
import lombok.extern.slf4j.Slf4j;
import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
import org.apache.shiro.web.filter.mgt.DefaultFilterChainManager;
import org.apache.shiro.web.filter.mgt.PathMatchingFilterChainResolver;
import org.apache.shiro.web.servlet.AbstractShiroFilter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping(value = "/filter")
public class FilterManageController {

//加载自定义的拦截工厂
@Autowired
private ShiroFilterFactoryBean myShiroFilterFactoryBean;

@GetMapping("/all")
public Map<String, String> findAll() {
    return myShiroFilterFactoryBean.getFilterChainDefinitionMap();
}

@GetMapping(value = "/add")
public Map<String, String> addFilter() {
    Map<String, String> filterMap = new HashMap<>();
    filterMap.put("/user/login**", "anon");
    filterMap.put("/admin/**", "anon");//把 admin 设置成不需要拦截
    filterMap.put("/super/**", "authc,roles[super],perms[super:info]");
    //管理员才可以操做
    filterMap.put("/users/**", "authc,roles[user]");
    filterMap.put("/**", "authc,kickout");
    addAndUpdatePermission(filterMap);

    return myShiroFilterFactoryBean.getFilterChainDefinitionMap();
}

/**
  * 动态更新新的权限, 更新原权限，增加新权限
  *
  * @param filterMap
  */
private synchronized void addAndUpdatePermission(Map<String, String> filterMap) {

    AbstractShiroFilter shiroFilter = null;
    try {
        shiroFilter = (AbstractShiroFilter) myShiroFilterFactoryBean.getObject();

        // 获取过滤管理器
        PathMatchingFilterChainResolver filterChainResolver = (PathMatchingFilterChainResolver) shiroFilter
                .getFilterChainResolver();
        DefaultFilterChainManager filterManager =
                (DefaultFilterChainManager) filterChainResolver.getFilterChainManager();

        //清空拦截管理器中的存储
        filterManager.getFilterChains().clear();
        /*
        清空拦截工厂中的存储,如果不清空这里,还会把之前的带进去
        ps:如果仅仅是更新的话,可以根据这里的 map 遍历数据修改,重新整理好权限再一起添加
          */
//            myShiroFilterFactoryBean.getFilterChainDefinitionMap().clear();

        Map<String, String> chains = myShiroFilterFactoryBean.getFilterChainDefinitionMap();
        chains.remove("/**");
        //把修改后的 map 放进去
        filterMap.entrySet().forEach(fm -> {
            chains.putIfAbsent(fm.getKey(), fm.getValue());
        });
//            chains.putAll(filterMap);

        //这个相当于是全量添加
        for (Map.Entry<String, String> entry : chains.entrySet()) {
            //要拦截的地址
            String url = entry.getKey().trim().replace(" ", "");
            //地址持有的权限
            String chainDefinition = entry.getValue().trim().replace(" ", "");
            //生成拦截
            filterManager.createChain(url, chainDefinition);
        }
    } catch (Exception e) {
        log.error("updatePermission error,filterMap=" + filterMap, e);
    }
}
}

```

1.  测试 修改前可直接删除用户, 修改权限后，删除用户的角色必须是user

参考
====

1.  <https://blog.csdn.net/fu_fei_wen/article/details/77571989>
2.  <https://waylau.gitbooks.io/apache-shiro-1-2-x-reference/content/>

代码路径
========

-   <https://github.com/microzhao/demo/tree/master/springboot/springboot-shiro>

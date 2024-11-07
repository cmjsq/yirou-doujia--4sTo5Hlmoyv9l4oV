[合集 \- Spring Security(7\)](https://github.com)[1\.一、Spring Boot集成Spring Security之自动装配10\-11](https://github.com/sanxiaolq/p/18458562)[2\.二、Spring Boot集成Spring Security之实现原理10\-11](https://github.com/sanxiaolq/p/18458697)[3\.三、Spring Boot集成Spring Security之过滤器链详解10\-12](https://github.com/sanxiaolq/p/18459037)[4\.四、Spring Boot集成Spring Security之认证流程10\-14](https://github.com/sanxiaolq/p/18463644):[veee加速器](https://youhaochi.com)[5\.五、Spring Boot集成Spring Security之认证流程210\-14](https://github.com/sanxiaolq/p/18464645)6\.六、Spring Boot集成Spring Security之前后分离认证流程最佳方案11\-06[7\.七、Spring Boot集成Spring Security之前后分离认证最佳实现11\-06](https://github.com/sanxiaolq/p/18531082)收起
# 二、Spring Security默认认证流程及其优缺点


## 1、Spring Security默认认证流程总结


[四、Spring Boot集成Spring Security之认证流程](https://github.com)详细介绍了认证流程，其核心流程如下


1. SecurityContextPersistenceFilter：chain.doFilter()前从安全上下文仓库中获取安全上下文，未登录状态时获取未认证的安全上下文；chain.doFilter()后从安全上下文持有者中获取安全上下文并更新到安全上下文仓库中
2. LogoutFilter：如果是登出请求，清除安全上下文认证信息并重定向到登录页面，否则不处理
3. UsernamePasswordAuthenticationFilter：如果是登录请求，校验请求参数中的用户名密码，校验成功后生成新的已认证的安全上下文并保存到安全上下文仓库中后重定向到目标URL，否则不处理
4. DefaultLoginPageGeneratingFilter：如果是登录页面请求，返回默认登录页面，否则不处理
5. DefaultLogoutPageGeneratingFilter：如果是登出页面请求，返回默认登出页面，否则不处理


## 2、优缺点


1. 提供了完整的安全的认证流程
2. 默认基于session实现非前后分离项目的认证流程，该流程已经慢慢退出历史舞台
3. 未提供前后分离认证流程


# 三、前后分离项目认证思路


## 1、前后分离项目认证流程（基于默认流程优化）


1. 前端输入用户名密码提交到后端
2. 后端获取到用户名密码并校验，校验成功后生成token（类似于sessionId）返回给前端，生成已认证的安全上下文（类似于session）存储到安全上下文仓库中
3. 前端获取到token，后续每次请求的请求头中都携带该token（类似于cookie）
4. 后端获取请求头中的token，通过token获取安全上下文，并设置到安全上下文持有者中
5. 前端提交退出请求时，后端获取请求头中的token，并通过token删除安全上下文仓库中安全上下文


## 2、前后分离项目认证流程关键组件对应的默认实现


从前后分离项目认证流程可以看出有四个关键组件


1. 每次请求时通过请求头中token从安全上下文仓库中获取安全上下文的过滤器（默认SecurityContextPersistenceFilter）
2. 登出时通过请求头中的token从安全上下文仓库中清除安全上下文的过滤器（默认LogoutFilter）
3. 登录时验证用户名密码并生成token和安全上下文，将安全上下文添加到安全上下文仓库中的过滤器（默认UsernamePasswordAuthenticationFilter）
4. 安全上下文仓库（默认HttpSessionSecurityContextRepository）


## 3、默认实现的局限性


1. UsernamePasswordAuthenticationFilter从form表单中获取请求参数，不符合RESTFUL开发规范
2. 认证的关键组件AuthenticationManager未注入到Spring容器中，导致自定义认证过滤器无法直接从Spring容器中获取
3. UsernamePasswordAuthenticationFilter只实现了认证部分，认证成功后生成的安全上下文并添加安全上下文仓库中过程无法控制，只能使用默认的HttpSession或RequestAttributes方式，无法自定义


## 4、整改思路


1. 自定义SecurityContextRepositoryImpl实现安全上下文仓库SecurityContextRepository，实现基于分布式缓存的安全上下文仓库
2. 自定义RestfulUsernamePasswordAuthenticationFilter继承AbstractAuthenticationProcessingFilter，实现符合RESTFUL开发规范的登录方式
3. 自定义UserDetailsImpl实现UserDetails接口，方便添加自定义属性
4. 自定义UserDetailsServiceImpl实现UserDetailsService接口，实现基于数据库的认证方式，并生成token设置到UserDetails中


## 5、整改后的认证流程


1. 前端输入用户名密码提交到后端
2. 后端AbstractAuthenticationProcessingFilter调用子类RestfulUsernamePasswordAuthenticationFilter的attemptAuthentication方法获取认证信息
3. RestfulUsernamePasswordAuthenticationFilter获取请求中的用户名密码，并调用UserDetailsService的loadUserByUsername获取用户信息
4. UserDetailsServiceImpl通过用户名查询用户，将用户信息设置到创建的UserDetailsImpl对象中，生成token设置到UserDetailsImpl对象中
5. AbstractAuthenticationProcessingFilter调用SecurityContextRepositoryImpl保存安全上下文
6. SecurityContextRepositoryImpl获取安全上下文及其认证信息中的token，将token和安全上下文添加到分布式缓存中
7. 将token返回到前端
8. 前端获取token，每次请求时都在请求头中携带该token
9. SecurityContextPersistenceFilter/SecurityContextHolderFilter调用SecurityContextRepositoryImpl的loadContext获取安全上下文
10. SecurityContextRepositoryImpl获取请求头中token，使用token从分布式缓存中获取安全上下文并返回
11. 前端提交登出请求
12. LogoutFilter调用SecurityContextRepositoryImpl的saveContext，其中参数安全上下文为空值安全上下文
13. SecurityContextRepositoryImpl判断出空值安全上下文，获取请求头中的token，使用token删除分布式缓存中获取安全上下文


# 四、总结


## 1、设计前后分离项目认证流程原则


1. 尽可能贴合原生Spring Security处理流程，尽量使用Spring Security提供的组件
2. 接口设计符合RESTFUL接口规范
3. 使用分布式缓存存储登录凭证，更适合分布式项目


## 2、其他说明


1. 这里说的前后分离项目认证流程最佳方案，是个人认为的最佳方案，并非行业公认的最佳方案，一千个读者就有一千个哈姆雷特，欢迎在评论区或者私信讨论你心中的最佳方案
2. 下文代码实现该方案，敬请期待



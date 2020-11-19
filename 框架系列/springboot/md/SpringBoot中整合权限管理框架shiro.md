[toc]

# SpringBoot整合shiro权限管理框架



## 本篇要点

- shiro简介及核心组件或功能介绍
- SpringBoot与shiro快速整合
- 分析身份认证和授权的流程
- 介绍shiro的拦截器机制
- 介绍shiro的权限注解

## 一、shiro是什么？用来干什么？

Apache Shiro 是 Java 的一个安全框架。Shiro 可以帮助我们完成：认证、授权、加密、会话管理、与 Web 集成、缓存等。

shiro的基本功能点有很多，详细介绍可以参照官方文档，这里分析最常用的两个：**认证Authentication与授权Authorization**，这俩功能相近，长相也差不太多，但千万注意，不要搞错。

- 何为认证？即**验证用户是否拥有响应的身份**。认证是比较好理解的，一般常见的场景就是登录场景，验证你的账号密码是否合理且合法，这就是认证的过程。
- 何为授权？即**验证某个已认证的用户是否拥有某个权限**。授权则是发生在认证之后，举个例子，超级管理拥有所有的权限便可以为所欲为，而游客可能只能拥有浏览的权限，不允许操作。

其他比较重要的功能点介绍一下：

- Subject：代表**当前用户**，准确来说是与应用代码直接交互的对象，与Subject的交互最终都会委托为SecurityManager。
- SecurityManager：安全管理器，管理所有的Subject，是shiro的核心，如果学习过SpringMVC，可以看成类似DispatcherServlet的功能。
- Realm：shiro可以从Realm中获取安全数据，如用户，角色，权限等，提供了认证与授权的主要方法接口。

## 二、SpringBoot快速整合shiro

### 导入shiro必要的依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-spring</artifactId>
            <version>1.5.3</version>
        </dependency>
```

### 创建数据源，准备加载用户数据的方法

这一步主要是为了省去和数据库交互，直接用HashMap模拟缓存用户数据。

```java
/**
 * 准备实验数据
 * @author Summerday
 */
public class DataSource {

    /**
     * k 用户名
     * v 用户信息
     */
    private static final Map<String, UserBean> data = new HashMap<>();

    static {
        data.put("hyh",new UserBean("hyh","123456","user","view"));
        data.put("sum",new UserBean("sum","123456","admin","view,edit"));
    }

    public static Map<String, UserBean> getData() {
        return data;
    }
}

@Service
public class UserService {
    
    //简单定义加载用户数据的方法，可以考虑加入缓存
    public UserBean getUser(String username) {
        return DataSource.getData().get(username);
    }

}
```

### 定义核心组件Realm

```java
/**
 * 核心组件:认证+授权
 * @author Summerday
 */

public class AuthRealm extends AuthorizingRealm {

    private static final Logger log = LoggerFactory.getLogger(AuthRealm.class);

    private static final String SESSION_KEY = "USER_SESSION";

    @Resource
    private UserService userService;

    /**
     * 只有需要验证权限时才会调用, 授权查询回调函数, 进行鉴权但缓存中无用户的授权信息时调用
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        // 获取user
        Session session = SecurityUtils.getSubject().getSession();
        UserBean user = (UserBean) session.getAttribute(SESSION_KEY);
        // 权限信息对象info,用来存放查出的用户的所有的角色（role）及权限（permission）
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        String role = user.getRole();
        String permission = user.getPermission();
        // 定义role和permission
        info.addRole(role);
        info.addStringPermissions(Arrays.asList(permission.split(",")));
        return info;
    }

    /**
     * 登录时调用该方法,返回用户名密码信息，之后将会调用CredentialsMatcher的doCredentialsMatch进	   * 行信息验证
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken auth) throws AuthenticationException {
        //获取user
        String username = (String) auth.getPrincipal();
        // 如果user为空,抛出UnknownAccountException异常
        UserBean user = Optional
                .ofNullable(userService.getUser(username))
                .orElseThrow(UnknownAccountException::new);
        SimpleAuthenticationInfo info = new SimpleAuthenticationInfo(username,user.getPassword(),getName());
        Session session = SecurityUtils.getSubject().getSession();
        session.setAttribute(SESSION_KEY,user);
        return info;
    }
}
```

用户需要提供 `principals` （身份）和 `credentials`（证明）给 shiro，从而应用能验证用户身份：

**principals**：身份，即主体的标识属性，可以是任何东西，如用户名、邮箱等，唯一即可。一个主体可以有多个 `principals`，但只有一个 `Primary principals`，一般是用户名 / 密码 / 手机号。

**credentials**：证明 / 凭证，即只有主体知道的安全值，如密码 / 数字证书等。

最常见的 `principals` 和 `credentials` 组合就是用户名 / 密码了。接下来先进行一个基本的身份认证。

### 定义shiroConfig，注入核心bean

```java
@Configuration
public class ShiroConfig {

    // shiro的生命周期处理器:用于在实现了 Initializable 接口的 Shiro bean 初始化时
    // 调用 Initializable 接口回调，在实现了 Destroyable 接口的 Shiro bean 销毁时调用 Destroyable 接口回调
    @Bean(name = "lifecycleBeanPostProcessor")
    public LifecycleBeanPostProcessor getLifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }

    //提供Realm实例
    @Bean
    public AuthRealm authRealm() {
        return new AuthRealm();
    }

    //权限管理，配置主要是Realm的管理认证
    @Bean(name = "securityManager")
    public DefaultWebSecurityManager securityManager() {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        // 设置Realm
        securityManager.setRealm(authRealm());
        return securityManager;
    }

    //Shiro 提供了相应的注解用于权限控制，配置以开启用于权限注解的解析和验证如 @RequiresRoles
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
        authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
        return authorizationAttributeSourceAdvisor;
    }
    //用于开启 Shiro Spring AOP 权限注解的支持；<aop:config proxy-target-class="true"> 表示代理类。使用cglib
    @Bean
    @ConditionalOnMissingBean
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator defaultAAP = new DefaultAdvisorAutoProxyCreator();
        defaultAAP.setProxyTargetClass(true);
        return defaultAAP;
    }

    //配置路径拦截规则
    @Bean(name = "shiroFilter")
    public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        //指定登录页面
        shiroFilterFactoryBean.setLoginUrl("/login");
        //首页：指定登录成功页面
        shiroFilterFactoryBean.setSuccessUrl("/index");
        //错误页面，认证不通过跳转
        shiroFilterFactoryBean.setUnauthorizedUrl("/denied");
        loadShiroFilterChain(shiroFilterFactoryBean);
        return shiroFilterFactoryBean;
    }

    /**
     * 定义url-filter规则，这一步可以采取从配置文件中读取，注意拦截的顺序
     */
    private void loadShiroFilterChain(ShiroFilterFactoryBean shiroFilterFactoryBean) {

        Map<String, String> map = new HashMap<>();
        //匿名拦截器，即不需要登录即可访问；一般用于静态资源过滤
        map.put("/hello","anon");
        //退出拦截器，主要属性：redirectUrl：退出成功后重定向的地址（/）
        map.put("/logout", "logout");
        //基于表单的拦截器；如 “`/**=authc`”，如果没有登录会跳到相应的登录页面登录
        map.put("/**", "authc");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(map);
    }
}

```

### 定义登录Controller，理解login的流程

```java
@Slf4j
@RestController
public class LoginController {

    @GetMapping(value = "/hello")
    public AjaxResult hello() {
        return AjaxResult.ok("不用登陆,直接访问");
    }

    @GetMapping(value = "/index")
    public AjaxResult index() {
        return AjaxResult.ok("登录成功");
    }

    @GetMapping(value = "/denied")
    public AjaxResult denied() {
        return AjaxResult.error("权限不足");
    }

    @GetMapping("/login")
    public AjaxResult login(String username, String password) {
        // 自动绑定到当前线程
        Subject subject = SecurityUtils.getSubject();
        UsernamePasswordToken token = new UsernamePasswordToken(username, password);
        try {
            //自动委托给 SecurityManager.login
            subject.login(token);
        } catch (UnknownAccountException e) {
            log.error("对用户[{}]进行登录验证,验证未通过,用户不存在", username);
            token.clear();
            return AjaxResult.error(e.getMessage());
        } catch (ExcessiveAttemptsException e) {
            log.error("对用户[{}]进行登录验证,验证未通过,错误次数过多", username);
            token.clear();
            return AjaxResult.error(e.getMessage());
        } catch (AuthenticationException e) {
            log.error("对用户[{}]进行登录验证,验证未通过,堆栈轨迹如下", username, e);
            token.clear();
            return AjaxResult.error(e.getMessage());
        }
        return AjaxResult.ok("登录成功");
    }

}
```

### 定义UserController，理解权限注解的使用

```java
@RestController
@RequestMapping("/user")
public class UserController {

    private static final Logger log = LoggerFactory.getLogger(UserController.class);

    @Resource
    private UserService userService;

    /**
     * RequiresRoles 是所需角色 包含 AND 和 OR 两种
     * RequiresPermissions 是所需权限 包含 AND 和 OR 两种
     *
     * @return msg
     */

    @RequiresRoles(value = {"admin"})
    //@RequiresPermissions(value = {"user:list", "user:query"}, logical = Logical.OR)
    @GetMapping("/rq_admin")
    public AjaxResult requireRoles() {
        return AjaxResult.ok("result : admin");
    }

    @RequiresPermissions(value = {"edit", "view"}, logical = Logical.AND)
    @GetMapping("/rq_edit_and_view")
    public AjaxResult requirePermissions() {
        return AjaxResult.ok("result : edit + view");
    }
}
```

## 三、认证流程

身份认证是项目安全中比较重要的一环，上面的登录流程中，我们看到了如下代码：

```java
        Subject subject = SecurityUtils.getSubject();
        UsernamePasswordToken token = new UsernamePasswordToken(username, password);
        try {
            //自动委托给 SecurityManager.login
            subject.login(token);
        } catch (UnknownAccountException e) {
            log.error("对用户[{}]进行登录验证,验证未通过,用户不存在", username);
            token.clear();
        }
```

我们之前提到过，SecurityManager其实是subject.login的方法的真正执行者，那真正的逻辑是怎样的呢？跟着源码一步步debug我们就能够知道具体的流程如下：

![img](img/SpringBoot%E4%B8%AD%E6%95%B4%E5%90%88%E6%9D%83%E9%99%90%E7%AE%A1%E7%90%86%E6%A1%86%E6%9E%B6shiro/4.png)

1.  `Subject.login(token)` 进行登录，其会自动委托给 `Security Manager`。
2. `Security Manager`之后委托给`Authenticator`进行身份验证：`this.authenticator.authenticate(token);`
3. 默认使用`ModularRealmAuthenticator`的`doAuthenticate`根据配置的`Realms`数量决定是否采用多`Realm`身份验证。
4. `AuthenticationInfo info = realm.getAuthenticationInfo(token);`获取info，可以实现该接口自定义realm的info获取逻辑。
5. 由于接口回调，带有用户身份信息的info在一步步往回传，如果出现`AuthenticationException`，将会认证失败，执行`onFailedLogin`逻辑。
6. 如果一路上没有错误，则创建subject，并返回。

## 四、授权流程

![img](img/SpringBoot%E4%B8%AD%E6%95%B4%E5%90%88%E6%9D%83%E9%99%90%E7%AE%A1%E7%90%86%E6%A1%86%E6%9E%B6shiro/6.png)

授权操作的本质，其实对subject主体是否具有权限和角色进行判断：

1. 调用`Subject.isPermtted/hasRole`接口，其会自动委托给 `Security Manager`。
2. `Security Manager`之后委托给`Authorizer`进行授权操作，假设我们调用`isPermitted(“user:view”)`，其首先会通过 `PermissionResolver` 把字符串转换成相应的 Permission 实例；
3. 在进行授权之前，将会调用相应的Realm为Subject授予role和permissions。
4. 默认使用`ModularRealmAuthorizer`的`doAuthenticate`根据配置的`Realms`数量决定是否采用多`Realm`身份验证。如果`isPermitted/hasRole`匹配成功，返回true。

## 五、拦截器机制

参考：[Shiro 拦截器机制](https://www.w3cschool.cn/shiro/oibf1ifh.html)

Shiro 内置了很多默认的拦截器，比如身份验证、授权等相关的。默认拦截器可以参考 `org.apache.shiro.web.filter.mgt.DefaultFilter` 中的枚举拦截器：

身份验证相关：

1. authc：FormAuthenticationFilter，基于表单的拦截器，`/**=authc`，如果没有登录会跳到相应的登录页面。
2. authBasic：BasicHttpAuthenticationFilter，BasicHTTP身份拦截器。
3. logout：LogoutFilter，退出拦截器，`/logout=logout`，退出成功后重定向。
4. user：UserFilter，用户拦截器，`/**=user`，已经身份验证或记住我登录的用户可以直接访问。
5. anon：AnonymousFilter，匿名拦截器，`/static/**=anon`，不需要登录即可访问，一般用于静态资源过滤。

授权相关：

1. roles：RolesAuthorizationFilter，角色授权拦截器，`/admin/**=roles[admin]`验证用户是否拥有所有角色。
2. perms：PermissionsAuthorizationFilter，权限授权拦截器，`/user/**=perms["user:create"]`，验证用户是否拥有所有权限。
3. port：PortFilter，端口拦截器，`/test=port[80]`，可以通过的端口为80。
4. rest：HttpMethodPermissionFilter，rest风格拦截器。
5. ssl：SslFilter，SSL拦截器，只有https协议才能通过，否则跳转会443端口。

其他：

noSessionCreation：NoSessionCreationFilter，不创建会话拦截器，调用 subject.getSession(false) 不会有什么问题，但是如果 subject.getSession(true) 将抛出 DisabledSessionException 异常。

## 六、权限注解

@RequiresAuthentication：表示当前 Subject 已经通过 login 进行了身份验证；即 `Subject.isAuthenticated()` 返回 true。

@RequiresUser：表示当前 Subject 已经身份验证或者通过记住我登录的。

@RequiresGuest：表示当前 Subject 没有身份验证或通过记住我登录过，即是游客身份。

@RequiresRoles(value={“admin”, “user”}, logical= Logical.AND)：表示当前 Subject 需要角色 admin 和 user。

@RequiresPermissions (value={“user:a”, “user:b”}, logical= Logical.OR)：表示当前 Subject 需要权限 user：a 或 user：b。

## 源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码及配置文件详细样例已经全部上传至Gitee：https://gitee.com/tqbx/springboot-samples-learn

## 参考阅读

- [一起来学SpringBoot | 第二十六篇：轻松搞定安全框架（Shiro）](http://blog.battcn.com/2018/07/03/springboot/v2-other-shiro/)

- [w3cschool：跟我学shiro](https://www.w3cschool.cn/shiro/oibf1ifh.html)
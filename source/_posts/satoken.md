---
title: satoken
date: 2025-03-10 15:50:44
tags:框架
---

# Sa-token

## 基础

### 1. 登录认证

#### 1.1**登录流程：**

1. 用户提交 `name` + `password` 参数，调用登录接口。
2. 登录成功，返回这个用户的 Token 会话凭证。
3. 用户后续的每次请求，都携带上这个 Token。
4. 服务器根据 Token 判断此会话是否登录成功。

![](F:\blog\source\_posts\satoken\Snipaste_2025-03-10_15-54-17.png)

#### **1.2 代码实现**

```java
// 会话登录接口 
@RequestMapping("doLogin")
public SaResult doLogin(String name, String pwd) {
   	QueryWrapper<User> wrapper = new QueryWrapper<>(User.class).eq("username", username);
        User user = getOne(wrapper);
        if (user != null && user.getPassword().equals(password)) {
            // 第二步：根据账号id，进行登录 
            StpUtil.login(user.getId());// 利用了 Cookie 自动注入的特性，省略了你手写返回 token 的代码
            return "登录成功";
        }
        return "登录失败";
}
```

#### 1.3 其他登录相关方法

```java
// 当前会话注销登录
StpUtil.logout();

// 获取当前会话是否已经登录，返回true=已登录，false=未登录
StpUtil.isLogin();

// 检验当前会话是否已经登录, 如果未登录，则抛出异常：`NotLoginException`
StpUtil.checkLogin();

// 获取当前会话账号id, 如果未登录，则抛出异常：`NotLoginException`
StpUtil.getLoginId();

// 类似查询API还有：
StpUtil.getLoginIdAsString();    // 获取当前会话账号id, 并转化为`String`类型
StpUtil.getLoginIdAsInt();       // 获取当前会话账号id, 并转化为`int`类型
StpUtil.getLoginIdAsLong();      // 获取当前会话账号id, 并转化为`long`类型

// ---------- 指定未登录情形下返回的默认值 ----------

// 获取当前会话账号id, 如果未登录，则返回 null 
StpUtil.getLoginIdDefaultNull();

// 获取当前会话账号id, 如果未登录，则返回默认值 （`defaultValue`可以为任意类型）
StpUtil.getLoginId(T defaultValue);

// 获取当前会话的 token 值
StpUtil.getTokenValue();

// 获取当前`StpLogic`的 token 名称
StpUtil.getTokenName();

// 获取指定 token 对应的账号id，如果未登录，则返回 null
StpUtil.getLoginIdByToken(String tokenValue);

// 获取当前会话剩余有效期（单位：s，返回-1代表永久有效）
StpUtil.getTokenTimeout();

// 获取当前会话的 token 信息参数
StpUtil.getTokenInfo();

```

### 2. 权限认证

权限认证的核心逻辑就是**判断一个账号是否拥有指定权限**，有则通过，无就禁止

#### 2.1 自定义添加权限码

因为每个项目的需求不同，其权限设计也千变万化，因此 [ **获取当前账号权限码集合** ] 这一操作不可能内置到框架中， 所以 Sa-Token 将此操作以接口的方式暴露给你，以方便你根据自己的业务逻辑进行重写。实现`StpInterface`接口

```java
/**
 * 自定义权限加载接口实现类
 */
@Component    // 保证此类被 SpringBoot 扫描，完成 Sa-Token 的自定义权限验证扩展 
public class StpInterfaceImpl implements StpInterface {

    /**
     * 返回一个账号所拥有的权限码集合 
     */
    @Override
    public List<String> getPermissionList(Object loginId, String loginType) {
        // 本 list 仅做模拟，实际项目中要根据具体业务逻辑来查询权限
        List<String> list = new ArrayList<String>();    
        list.add("101");
        list.add("user.add");
        list.add("user.update");
        list.add("user.get");
        // list.add("user.delete");
        list.add("art.*");
        return list;
    }

    /**
     * 返回一个账号所拥有的角色标识集合 (权限与角色可分开校验)
     */
    @Override
    public List<String> getRoleList(Object loginId, String loginType) {
        // 本 list 仅做模拟，实际项目中要根据具体业务逻辑来查询角色
        List<String> list = new ArrayList<String>();    
        list.add("admin");
        list.add("super-admin");
        return list;
    }

}
```

#### 2.2 其他功能实现

```java
权限校验
// 获取：当前账号所拥有的权限集合
StpUtil.getPermissionList();

// 判断：当前账号是否含有指定权限, 返回 true 或 false
StpUtil.hasPermission("user.add");        

// 校验：当前账号是否含有指定权限, 如果验证未通过，则抛出异常: NotPermissionException 
StpUtil.checkPermission("user.add");        

// 校验：当前账号是否含有指定权限 [指定多个，必须全部验证通过]
StpUtil.checkPermissionAnd("user.add", "user.delete", "user.get");        

// 校验：当前账号是否含有指定权限 [指定多个，只要其一验证通过即可]
StpUtil.checkPermissionOr("user.add", "user.delete", "user.get");   

角色校验
// 获取：当前账号所拥有的角色集合
StpUtil.getRoleList();

// 判断：当前账号是否拥有指定角色, 返回 true 或 false
StpUtil.hasRole("super-admin");        

// 校验：当前账号是否含有指定角色标识, 如果验证未通过，则抛出异常: NotRoleException
StpUtil.checkRole("super-admin");        

// 校验：当前账号是否含有指定角色标识 [指定多个，必须全部验证通过]
StpUtil.checkRoleAnd("super-admin", "shop-admin");        

// 校验：当前账号是否含有指定角色标识 [指定多个，只要其一验证通过即可] 
StpUtil.checkRoleOr("super-admin", "shop-admin");     

```

#### 2.3 拦截全局异常

防止鉴权失败抛出异常

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    // 全局异常拦截 
    @ExceptionHandler
    public SaResult handlerException(Exception e) {
        e.printStackTrace(); 
        return SaResult.error(e.getMessage());
    }
}
```

### 3. 踢人下线

#### 3.1 强制注销

```java
StpUtil.logout(10001);                    // 强制指定账号注销下线 
StpUtil.logout(10001, "PC");              // 强制指定账号指定端注销下线 
StpUtil.logoutByTokenValue("token");      // 强制指定 Token 注销下线 复制到剪贴板错误复制成功123
```

#### 3.2 踢人下线

```java
StpUtil.kickout(10001);                    // 将指定账号踢下线 
StpUtil.kickout(10001, "PC");              // 将指定账号指定端踢下线
StpUtil.kickoutByTokenValue("token");      // 将指定 Token 踢下线复制到剪贴板错误复制成功123
```

强制注销 和 踢人下线 的区别在于：

- 强制注销等价于对方主动调用了注销方法，再次访问会提示：Token无效。
- 踢人下线不会清除Token信息，而是将其打上特定标记，再次访问会提示：Token已被踢下线。

### 4.注解鉴权

#### 4.1 添加拦截器

```java
@Configuration
public class SaTokenConfigure implements WebMvcConfigurer {
    // 注册 Sa-Token 拦截器，打开注解式鉴权功能 
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 注册 Sa-Token 拦截器，打开注解式鉴权功能 
        registry.addInterceptor(new SaInterceptor()).addPathPatterns("/**");    
    }
}
```

#### 4.2 使用注解

```java
// 登录校验：只有登录之后才能进入该方法 
@SaCheckLogin                        
@RequestMapping("info")
public String info() {
    return "查询用户信息";
}

// 角色校验：必须具有指定角色才能进入该方法 
@SaCheckRole("super-admin")        
@RequestMapping("add")
public String add() {
    return "用户增加";
}

// 权限校验：必须具有指定权限才能进入该方法 
@SaCheckPermission("user-add")        
@RequestMapping("add")
public String add() {
    return "用户增加";
}

// 二级认证校验：必须二级认证之后才能进入该方法 
@SaCheckSafe()        
@RequestMapping("add")
public String add() {
    return "用户增加";
}

// Http Basic 校验：只有通过 Http Basic 认证后才能进入该方法 
@SaCheckHttpBasic(account = "sa:123456")
@RequestMapping("add")
public String add() {
    return "用户增加";
}

// Http Digest 校验：只有通过 Http Digest 认证后才能进入该方法 
@SaCheckHttpDigest(value = "sa:123456")
@RequestMapping("add")
public String add() {
    return "用户增加";
}

// 校验当前账号是否被封禁 comment 服务，如果已被封禁会抛出异常，无法进入方法 
@SaCheckDisable("comment")                
@RequestMapping("send")
public String send() {
    return "查询用户信息";
}
```

**设定校验模式**

`@SaCheckRole`与`@SaCheckPermission`注解可设置校验模式，例如：

```java
// 注解式鉴权：只要具有其中一个权限即可通过校验 
@RequestMapping("atJurOr")
@SaCheckPermission(value = {"user-add", "user-all", "user-delete"}, mode = SaMode.OR)        
public SaResult atJurOr() {
    return SaResult.data("用户信息");
}
```

mode有两种取值：

- `SaMode.AND`，标注一组权限，会话必须全部具有才可通过校验。
- `SaMode.OR`，标注一组权限，会话只要具有其一即可通过校验。

##### **联合or校验**

```java
// 角色权限双重 “or校验”：具备指定权限或者指定角色即可通过校验
@RequestMapping("userAdd")
@SaCheckPermission(value = "user.add", orRole = "admin")        
public SaResult userAdd() {
    return SaResult.data("用户信息");
}
```

orRole 字段代表权限校验未通过时的次要选择，两者只要其一校验成功即可进入请求方法，其有三种写法：

- 写法一：`orRole = "admin"`，代表需要拥有角色 admin 。
- 写法二：`orRole = {"admin", "manager", "staff"}`，代表具有三个角色其一即可。
- 写法三：`orRole = {"admin, manager, staff"}`，代表必须同时具有三个角色。

**忽略认证**

使用 `@SaIgnore` 可表示一个接口忽略认证。

- @SaIgnore 修饰方法时代表这个方法可以被游客访问，修饰类时代表这个类中的所有接口都可以游客访问。
- @SaIgnore 具有最高优先级，当 @SaIgnore 和其它鉴权注解一起出现时，其它鉴权注解都将被忽略。
- @SaIgnore 同样可以忽略掉 Sa-Token 拦截器中的路由鉴权，在下面的 [路由拦截鉴权] 章节中我们会讲到。

**批量注解鉴权**

使用 `@SaCheckOr` 表示批量注解鉴权

```java
// 在 `@SaCheckOr` 中可以指定多个注解，只要当前会话满足其中一个注解即可通过验证，进入方法。
@SaCheckOr(
        login = @SaCheckLogin,
        role = @SaCheckRole("admin"),
        permission = @SaCheckPermission("user.add"),
        safe = @SaCheckSafe("update-password"),
        httpBasic = @SaCheckHttpBasic(account = "sa:123456"),
        disable = @SaCheckDisable("submit-orders")
)
@RequestMapping("test")
public SaResult test() {
    // ... 
    return SaResult.ok(); 
}

// 当前客户端只要有 [ login 账号登录] 或者 [user 账号登录] 其一，就可以通过验证进入方法。
//         注意：`type = "login"` 和 `type = "user"` 是多账号模式章节的扩展属性，此处你可以先略过这个知识点。
@SaCheckOr(
    login = { @SaCheckLogin(type = "login"), @SaCheckLogin(type = "user") }
)
```

### 5.拦截鉴权

如果使用注解鉴权非常麻烦，sa-token提供了和security差不多的拦截方式

```java
@Configuration
public class SaTokenConfigure implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 注册 Sa-Token 拦截器，定义详细认证规则 
        registry.addInterceptor(new SaInterceptor(handler -> {
            // 指定一条 match 规则
            SaRouter
                .match("/**")    // 拦截的 path 列表，可以写多个 */
                .notMatch("/user/doLogin")        // 排除掉的 path 列表，可以写多个 
                .check(r -> StpUtil.checkLogin());        // 要执行的校验动作，可以写完整的 lambda 表达式
                
            // 根据路由划分模块，不同模块不同鉴权 
            SaRouter.match("/user/**", r -> StpUtil.checkPermission("user"));
            // 角色校验 -- 拦截以 admin 开头的路由，必须具备 admin 角色或者 super-admin 角色才可以通过认证 
            SaRouter.match("/admin/**", r -> StpUtil.checkRoleOr("admin", "super-admin"));
            / 根据 path 路由排除匹配 
            // 功能说明: 使用 .html , .css 或者 .js 结尾的任意路由都将跳过, 不会进入 check 方法
            SaRouter.match("/**").notMatch("*.html", "*.css", "*.js").check( /* 要执行的校验函数 */ );

            // 根据请求类型匹配 
            SaRouter.match(SaHttpMethod.GET).check( /* 要执行的校验函数 */ );

            // 根据一个 boolean 条件进行匹配 
            SaRouter.match( StpUtil.isLogin() ).check( /* 要执行的校验函数 */ );

            // 根据一个返回 boolean 结果的lambda表达式匹配 
            SaRouter.match( r -> StpUtil.isLogin() ).check( /* 要执行的校验函数 */ );
        })).addPathPatterns("/**");
    }
}
```

aRouter.match() 匹配函数有两个参数：

- 参数一：要匹配的path路由。
- 参数二：要执行的校验函数。

**使用 `@SaIgnore` 注解，忽略掉路由拦截认证**

### 6. Session

Session 是会话中专业的数据缓存组件，通过 Session 我们可以很方便的缓存一些高频读写数据，提高程序性能，例如：

```java
// 在登录时缓存 user 对象 
StpUtil.getSession().set("user", user);

// 然后我们就可以在任意处使用这个 user 对象
SysUser user = (SysUser) StpUtil.getSession().get("user");
```

类似ThreadLocal存储用户id，然后想用的时候获取。

在 Sa-Token 中，Session 分为三种，分别是：

- `Account-Session`: 指的是框架为每个 账号id 分配的 Session （单端）
- `Token-Session`: 指的是框架为每个 token 分配的 Session（推荐如果存在多端登录的）
- `Custom-Session`: 指的是以一个 特定的值 作为SessionId，来分配的 Session

**Account-Session**

```java
// 获取当前账号 id 的 Account-Session (必须是登录后才能调用)
StpUtil.getSession();

// 获取当前账号 id 的 Account-Session, 并决定在 Session 尚未创建时，是否新建并返回
StpUtil.getSession(true);

// 获取账号 id 为 10001 的 Account-Session
StpUtil.getSessionByLoginId(10001);

// 获取账号 id 为 10001 的 Account-Session, 并决定在 Session 尚未创建时，是否新建并返回
StpUtil.getSessionByLoginId(10001, true);

// 获取 SessionId 为 xxxx-xxxx 的 Account-Session, 在 Session 尚未创建时, 返回 null 
StpUtil.getSessionBySessionId("xxxx-xxxx");
```

**其他操作**

```java
// 返回此 Session 的id 
session.getId();                          

// 返回此 Session 的创建时间 (时间戳) 
session.getCreateTime();                  

// 返回此 Session 会话上的底层数据对象（如果更新map里的值，请调用session.update()方法避免产生脏数据）
session.getDataMap();                     

// 将这个 Session 从持久库更新一下
session.update();                         

// 注销此 Session 会话 (从持久库删除此Session)
session.logout();                         
```

### 7. yaml配置

```yaml
############## Sa-Token 配置 (文档: https://sa-token.cc) ##############
sa-token: 
    # token 名称（同时也是 cookie 名称）
    token-name: satoken
    # token 有效期（单位：秒） 默认30天，-1 代表永久有效
    timeout: 2592000
    # token 最低活跃频率（单位：秒），如果 token 超过此时间没有访问系统就会被冻结，默认-1 代表不限制，永不冻结
    active-timeout: -1
    # 是否允许同一账号多地同时登录 （为 true 时允许一起登录, 为 false 时新登录挤掉旧登录）
    is-concurrent: true
    # 在多人登录同一账号时，是否共用一个 token （为 true 时所有登录共用一个 token, 为 false 时每次登录新建一个 token）
    is-share: true
    # token 风格（默认可取值：uuid、simple-uuid、random-32、random-64、random-128、tik）
    token-style: uuid
    # 是否输出操作日志 
    is-log: true
```

## 深入

### 1. 集成redis

```xml
<!-- Sa-Token 整合 Redis （使用 jackson 序列化方式） -->
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-redis-jackson</artifactId>
    <version>1.40.0</version>
</dependency>

<!-- Sa-Token 整合 Redis （使用 jdk 默认序列化方式） -->
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-redis</artifactId>
    <version>1.40.0</version>
</dependency>
```

jackson需要构造方法添加@JsonProperty("id")

**添加redis依赖**

```xml
<!-- 提供Redis连接池 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

```yaml
spring: 
    # redis配置 
    redis:
        # Redis数据库索引（默认为0）
        database: 1
        # Redis服务器地址
        host: 127.0.0.1
        # Redis服务器连接端口
        port: 6379
        # Redis服务器连接密码（默认为空）
        # password: 
        # 连接超时时间
        timeout: 10s
        lettuce:
            pool:
                # 连接池最大连接数
                max-active: 200
                # 连接池最大阻塞等待时间（使用负值表示没有限制）
                max-wait: -1ms
                # 连接池中的最大空闲连接
                max-idle: 10
                # 连接池中的最小空闲连接
                min-idle: 0
```

**集成 Redis 后**，框架自动保存数据。集成 `Redis` 只需要引入对应的 `pom依赖` 即可，框架所有上层 API 保持不变。

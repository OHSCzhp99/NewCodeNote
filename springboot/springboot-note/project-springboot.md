## SpringBoot3
---
## 需要先知道的知识点：

### 1. SpringBoot 框架层
首先了解下 SpringBoot 框架层：  

Domain层(Pojo、Enity、Model):  
实体层，放置实体类，如Book、Person等
---
Controller层(action):  
表现层，调用service层的接口，实现接口的方法  
控制器负责接收请求并将其转发给对应的视图或服务进行处理  
它通常负责处理请求的路由和参数验证。

Service层:  
业务层，调用Dao层的接口，组织业务逻辑功能，例如数据库操作、数据转换等  
主要在这实现业务逻辑的代码开发，当然也可以在controller，但一般controller代码越少越好

Dao层(mapper):  
持久层/数据访问层，通常放置是放执行sql语句的接口类，和数据库打交道  
负责执行特定的业务逻辑，例如数据库操作、数据转换等。

### 2. 拦截器
1.实现 HandlerInterceptor 接口：   
preHandle：在请求到达处理器之前执行，可以用于权限验证、数据校验等操作。  
如果返回true，则继续执行后续操作；如果返回false，则中断请求处理。  

postHandle：在处理器处理请求之后执行，可以用于日志记录、缓存处理等操作。  

afterCompletion：在视图渲染之前执行，可以用于资源清理等操作。  

2.注册拦截器到InterceptorRegistry：   
通过实现WebMvcConfigurer接口，并重写addInterceptors方法来实现

### 1. 创建项目
使用Maven创建：   
JDK：17   
Version：1.0   
1.补充目录：在main目录创建resources文件夹，并创建application.yml   
2.引入依赖：web、mybatis、mysql驱动、lomlok依赖等...  
3.数据库连接：在application.yml页面补充   
4.完善目录：创建controller、service(里面再创建impl文件夹)、mapper、domain、utils文件夹   
5.更改启动类名字并加入@SpringBootApplication   
6.导入工具类
```xml
<!--父工程依赖-->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.1.4</version>
</parent>

//这里写笔记已经陆续导入依赖，总之后续用到的依赖都会回到这里重新更新笔记
<!--手动依赖引入-->
<dependencies>
    <!--单元测试依赖-->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.11</version>
        <scope>test</scope>
    </dependency>

    <!--web依赖-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!--mybatis依赖-->
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>3.0.2</version>
    </dependency>

    <!--mysql驱动依赖-->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
    </dependency>

    <!--lombok依赖-->
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
    </dependency>

    <!--validation依赖（用于参数校验）-->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!--jwt令牌依赖（用于用户登录验证）-->
    <dependency>
      <groupId>com.auth0</groupId>
      <artifactId>java-jwt</artifactId>
      <version>4.4.0</version>
    </dependency>

    <!--单元测试依赖-->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
    </dependency>

</dependencies>
```
application.yml：
```yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/cooking
    username: root
    password: root
```
启动类文件：
```java
@SpringBootApplication
public class SpringCookingApplication {
    public static void main( String[] args ) {
        SpringApplication.run(SpringCookingApplication.class,args);
    }
}
```

### 2.1 接口 - 用户 - 注册和登录
统一接口相应结果类 Result   
注册url：user/register   
登录url：user/login  
---
注意点：密码加密、jwt令牌、处理异常、拦截器  
1.密码加密：（需导入工具类）      
md5加密  

2.jwt令牌：（需导入工具类）  
Header(头)，记录令牌类型和签名算法  
PayLoad(荷载)，记录携带自定义的信息（不放密码）  
Signature(签名)，对头部和荷载进行加密计算得来  

3.处理异常：  
@RestControllerAdvice  
@ExceptionHandler(Exception.class) 

4.拦截器：  
创建interceptors文件夹，再创建LoginInterceptor类，用来编写 登录和注册拦截器  
创建config文件夹，再创建WebConfig类，用来注入拦截器
---
目前的文件目录：  
实体类：domain -> User、Result  
1.控制层：controller -> UserController   
2.服务层：service -> UserService，impl -> UserServiceImpl  
3.映射层：mapper -> UserMapper  
处理异常：exception -> GlobalExceptionHandle  
拦截器：interceptors -> LoginInterceptor  
拦截器注入：config -> WebConfig  

---
控制层 UserController：
```java
@RestController
@RequestMapping("/user")
@Validated //这是参数校验注解
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping("/register")
    //@Pattern() 是配合@Validated 参数校验的
    public Result register(@Pattern(regexp = "^\\S{5,16}$") String username,@Pattern(regexp = "^\\S{5,16}$") String password){
        //先查询用户再注册
        User u = userService.findByUserName(username);
        if (u==null){
            //没有被占用，可注册
            userService.register(username,password);
            return Result.success();
        }else {
            //占用
            return Result.error("用户名已被占用");
        }
    }

    @PostMapping("/login")
    public Result<String> login(@Pattern(regexp = "^\\S{5,16}$") String username,@Pattern(regexp = "^\\S{5,16}$") String password){
        //1.根据用户名查询用户
        User loginUser = userService.findByUserName(username);
        //2.判断用户是否存在
        if (loginUser==null){
            return Result.error("用户名错误");
        }
        //3.判断密码是否正确 loginUser对象中的password是密文
        //（拿输入的密码，跟数据库里的密码进行比较）
        if(Md5Util.getMD5String(password).equals(loginUser.getPassword())){
            //3.1登录成功
            //将id和username作为token中的荷载，生成token
            Map<String,Object> claims = new HashMap<>();
            claims.put("id",loginUser.getId());
            claims.put("username",loginUser.getUsername());
            String token = JwtUtil.genToken(claims);
            return Result.success(token);
        }
        return Result.error("密码错误");
    }
}
```
服务层 UserService：
```java
public interface UserService {
    //根据用户名查询用户
    User findByUserName(String username);

    //注册
    void register(String username, String password);
}
```
服务层实现类 UserServiceImpl：
```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public User findByUserName(String username) {
        User u = userMapper.findByUserName(username);
        return u;
    }

    @Override
    public void register(String username, String password) {
        //加密
        String md5String = Md5Util.getMD5String(password);
        //添加
        userMapper.add(username,md5String);
    }
}
```
映射层 UserMapper：
```java
@Mapper
public interface UserMapper {
    //根据用户名查询用户
    @Select("select * from user where username=#{username}")
    User findByUserName(String username);

    //添加
    @Insert("insert into user(username,password,create_time,update_time)" +
            " values(#{username},#{password},now(),now())")
    void add(String username, String password);
}
```
处理异常 GlobalExceptionHandle：
```java
//统一处理异常
@RestControllerAdvice
public class GlobalExceptionHandle {

    //注册参数校验失败 异常处理
    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e){
        e.printStackTrace(); //错误信息输出控制台
        //因为不是所有操作都有错误信息，所以用StringUtils.hasLength()看看是否存在错误信息
        //有则输出错误信息，没有则提示操作失败
        return Result.error(StringUtils.hasLength(e.getMessage()) ? e.getMessage() : "操作失败");
    }
}
```
拦截器 LoginInterceptor
```java
//编写一个 “访问先验证登录” 拦截器对象
@Component
public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //令牌验证
        String token = request.getHeader("Authorization");
        try {
            //验证token
            Map<String,Object> claims = JwtUtil.parseToken(token);
            //放行
            return true;
        } catch (Exception e) {
            //http响应状态码为401
            response.setStatus(401);
            //不放行
            return false;
        }
    }
}
```
拦截器注入 WebConfig
```java
//这是配置类，将拦截器对象注入到 InterceptorRegistry
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private LoginInterceptor loginInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        //注册和登录接口不拦截
        registry.addInterceptor(loginInterceptor).excludePathPatterns("/user/register","/user/login");
    }
}
```

### 2.2 接口 - 用户 - 获取用户数据
获取用户数据url：user/userinfo   

---
注意点：  
1.开启驼峰命名转换，参见application.yml   
2.给user实体类的 password属性 添加 @JsonIgnore 注解（获取用户数据时忽略密码）
3.使用 ThreadLocal 存储数据，能做到线程安全

---

拦截器 LoginInterceptor：（以下主要是添加ThreadLocal的一些代码）
```java
//编写一个 “访问先验证登录” 拦截器对象
@Component
public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //令牌验证
        String token = request.getHeader("Authorization");
        try {
            //验证token
            Map<String,Object> claims = JwtUtil.parseToken(token);
            //将token中的业务数据存储到ThreadLocal，方便各个接口用到业务数据的id，也安全
            ThreadLocalUtil.set(claims);
            //放行
            return true;
        } catch (Exception e) {
            //http响应状态码为401
            response.setStatus(401);
            //不放行
            return false;
        }
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        //清空ThreadLocal中的数据
        ThreadLocalUtil.remove();
    }
}
```

控制层 UserController：
```java
//此处忽略上面已经写好了的代码，以下代码是新添加的

@GetMapping("/userinfo")
public Result<User> userInfo(){
    //根据用户名查询用户
    //拿到ThreadLocal中的用户业务数据（在拦截器放行之前已经存入ThreadLocal了）
    Map<String,Object> map = ThreadLocalUtil.get();
    String username = (String) map.get("username");
    User user = userService.findByUserName(username);
    return Result.success(user);
}
```

### 2.3 接口 - 用户 - 更新用户数据
获取用户数据url：user/updata

---
注意点：    
1.对user实体类添加参数校验（因为是提交参数是一个对象，所以在实体类添加）   
2.接口类型为put   

实体类 User：
```java
@Data
public class User {
    //以下@NotNull、@NotEmpty、@Pattern(regexp = "")、@Email 都有由 @Validated 提供
    @NotNull //不能为空
    private Integer id;
    private String username;
    @JsonIgnore //转换json时忽略密码，就不会输出了
    private String password;

    @NotEmpty //不能为空且非空串
    @Pattern(regexp = "^\\S{1,10}$") //设置正则表达
    private String nickname;

    @Email
    private String email;
    private String userPic; //用户头像地址
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
}
```

控制层 UserController：
```java
@PutMapping("/update")
public Result<User> update(@RequestBody @Validated User user){
    //接受的参数转为实体对象接收 @RequestBody，并且校验参数 @Validated
    userService.update(user);
    return Result.success();
}
```
服务层 UserService：
```java
//更新
void update(User user);
```
服务层 UserServiceImpl：
```java
@Override
public void update(User user) {
    //更新注意要更新时间
    user.setUpdateTime(LocalDateTime.now());
    userMapper.update(user);
}
```
映射层 UserMapper：
```java
//更新
@Update("update user set nickname=#{nickname},email=#{email}," +
        "update_time=#{updateTime} where id=#{id}")
void update(User user);
```

### 2.4 接口 - 用户 - 更新用户头像
获取用户数据url：user/updataAvatar

---
注意点：    
1.对接口形参添加参数校验（因为提交参数只有部分 而且是地址）    
2.接口类型为patch  
3.不是传id的值，而是要从ThreadLocal中拿到id（须先登录） 

控制层 UserController：
```java
@PatchMapping("/updateAvatar")
public Result<User> updateAvatar(@RequestParam @URL String avatarUrl){
    //接受的参数只有部分用 @RequestParam，并且校验参数为地址 @URL
    userService.updateAvatar(avatarUrl);
    return Result.success();
}
```
服务层 UserService：
```java
//局部更新头像
void updateAvatar(String avatarUrl);
```
服务层 UserServiceImpl：
```java
@Override
public void updateAvatar(String avatarUrl) {
    //从ThreadLocal 拿到用户的id，将头像地址和id都传过去
    Map<String,Object> map = ThreadLocalUtil.get();
    Integer id = (Integer)map.get("id");
    userMapper.updateAvatar(avatarUrl,id);
}
```
映射层 UserMapper：
```java
//局部更新头像
@Update("update user set user_pic=#{avatarUrl},update_time=now() where id=#{id}")
void updateAvatar(String avatarUrl,Integer id);
```

### 2.5 接口 - 用户 - 更新用户密码
获取用户数据url：user/updataPwd

---


控制层 UserController：
```java
//更新用户密码
@PatchMapping("/updatePwd")
public Result updatePwd(@RequestBody Map<String,String> params){
    //接受的参数转为实体类 @RequestBody，但不是一一对应user实体类，所以用map接收
    //1.参数校验
    String oldPwd = params.get("old_pwd");
    String newPwd = params.get("new_pwd");
    String rePwd = params.get("re_pwd");
    if (!StringUtils.hasLength(oldPwd) || !StringUtils.hasLength(newPwd) ||!StringUtils.hasLength(rePwd)){
        return Result.error("需要填写完整的三个参数");
    }
    //2.原密码是否正确
    //先在ThreadLocal找到用户名，根据用户的原密码和新密码进行比较
    Map<String,Object> map = ThreadLocalUtil.get();
    String username = (String) map.get("username");
    User loginUser = userService.findByUserName(username);
    //比较时，原密码是加密的，新密码需要先加密
    if (!loginUser.getPassword().equals(Md5Util.getMD5String(oldPwd))){
        return Result.error("原密码填写不正确");
    }
    //3.newPwd和rePwd是否一致
    if (!rePwd.equals(newPwd)) return Result.error("两次填写的新密码不一致");
    //4.新密码和原密码相同
    if (newPwd.equals(oldPwd)) return Result.error("新密码与原密码不能相同");
    userService.updatePwd(newPwd);
    return Result.success();
}
```
服务层 UserService：
```java
//局部更新头像
void updateAvatar(String avatarUrl);
```
服务层 UserServiceImpl：
```java
//更新密码
@Override
public void updatePwd(String newPwd) {
    //将新密码加密
    String newPwdMd5 = Md5Util.getMD5String(newPwd);
    //从ThreadLocal 拿到用户的id，将新密码和id都传过去
    Map<String,Object> map = ThreadLocalUtil.get();
    Integer id = (Integer) map.get("id");
    userMapper.updatePwd(newPwdMd5,id);
}
```
映射层 UserMapper：
```java
//更新密码
@Update("update user set password=#{newPwdMd5},update_time=now() where id=#{id}")
void updatePwd(String newPwdMd5,Integer id);
```

### 3.1 接口 - 分类 - 新增美食分类
获取用户数据url：/category（post方法）


```
具体实现方法参考毕设
```

### 3.2 接口 - 分类 -  获取美食分类
获取用户数据url：/category（get方法）

---
注意点：  
1.返回给前端的时间格式，需在实体类上加上 @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")   


```
具体实现方法参考毕设
```

### 3.3 接口 - 分类 -  获取美食分类数据详情
获取用户数据url：/category/detail（get方法）


```
具体实现方法参考毕设
```  

### 3.4 接口 - 分类 -  更新美食分类
获取用户数据url：/category（put方法）

---
注意点：  
1.分组校验（在实体类看）  

```
具体实现方法参考毕设
```

### 3.5 接口 - 分类 -  删除美食分类
获取用户数据url：/category（del方法）

```
具体实现方法参考毕设
```  

### 4.1 接口 - 帖子 -  新增美食帖子
获取用户数据url：/cate（post方法）

---
注意点：  
1.需要自己写State属性的 @validation 校验注解（参考自己创建的 anno包 + validation包）  

```
具体实现方法参考毕设
```  

### 4.1 接口 - 帖子 - 获取美食帖子（分页查询）
获取用户数据url：/cate（get方法）

---
注意点：    
1.新建统一返回给前端的分页对象PageBean实体类，（该实体类属性包括total、items(List<T>类型)）  
2.借助PageHelper依赖、写动态sql的xml文件  
```
具体实现方法参考毕设
```  
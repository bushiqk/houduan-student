# 第 1 课　跑起来，并搞懂 Spring 容器

## 本课目标

学完这一课，你应该能回答：

1. 一个 Spring Boot 程序从 `main` 方法到能接收 HTTP 请求，中间发生了什么？
2. 你没有写 `new`，Controller 里的 repository 是谁塞进去的？
3. `@SpringBootApplication` 这一个注解背后到底是几个注解？
4. `application.properties` 里的每一行是什么意思？
5. 为什么自己新建的类放错包就"不生效"？

预计用时 2-3 小时（含 4 个实验）。

## 0. 先跑起来

```bash
mvn -B -DskipTests compile   # 只编译，先确认环境
mvn spring-boot:run          # 启动应用
```

启动日志里请找到三样东西，找不到就说明有问题：

- `Tomcat started on port 8080`
- `H2 console available at '/h2-console'`，以及一行 `jdbc:h2:mem:<uuid>`
- `Started PetClinicApplication in x.xx seconds`

浏览器打开 `localhost:8080` 看到欢迎页即成功。停止用 `Ctrl+C`。

## 1. 从 main 方法说起

教材文件：`src/main/java/org/springframework/samples/petclinic/PetClinicApplication.java`

```java
@SpringBootApplication
@ImportRuntimeHints(PetClinicRuntimeHints.class)
public class PetClinicApplication {

    public static void main(String[] args) {
        SpringApplication.run(PetClinicApplication.class, args);
    }

}
```

对照你熟悉的最简单 Java 程序：

```java
public class Demo {
    public static void main(String[] args) {
        System.out.println("hello");
    }
}
```

区别在于：普通程序里每个对象都是你 `new` 出来的；而 `SpringApplication.run(...)` 只交代了一件事——**"请容器把这些对象创建好、连好线、然后启动 Web 服务器"**。

注意：你没有 `new` Tomcat，没有 `new` 任何 Controller。它们都是被容器创建出来的。

`SpringApplication.run` 内部的大致顺序（知道有这么回事即可，不必背细节）：

1. 推断应用类型——这里是 SERVLET，于是准备内嵌 Tomcat
2. 创建 `ApplicationContext`（也就是 IoC 容器）
3. 组件扫描 + 装配所有 Bean（依赖注入在这一步完成）
4. 执行自动配置
5. 启动内嵌 Tomcat，开始监听 8080
6. 触发启动完成回调，打印 `Started ...`

顺带一提：`@ImportRuntimeHints(PetClinicRuntimeHints.class)` 是给 GraalVM 原生镜像用的资源提示（把模板、静态资源等告诉原生编译），先不用管，阶段 5 再回头看。

## 2. IoC 容器：本质就是一个"对象仓库"

先把术语对照表记住，后面每节课都要用：

| 术语 | 大白话 |
| --- | --- |
| Bean | 交给 Spring 管理的对象。不是所有对象都算，只有容器创建的那些 |
| IoC 容器 / ApplicationContext | 存放 Bean 的仓库，同时管理创建和生命周期 |
| 组件扫描（Component Scan） | 约定好去哪些包里找"该被管理的类" |
| 依赖注入（DI） | 容器把某个 Bean 需要的其他 Bean 塞给它，而不是它自己 new |
| 自动配置（AutoConfiguration） | Spring Boot 依据依赖，猜你需要哪些常用 Bean 并自动配好 |

为什么需要这套东西？看项目里的真实例子。

`OwnerRepository` 是一个**接口**（`src/main/java/org/springframework/samples/petclinic/owner/OwnerRepository.java`），你根本无法 `new OwnerRepository()`。但 `OwnerController` 又必须用它：

```java
class OwnerController {

    private final OwnerRepository owners;

    public OwnerController(OwnerRepository owners) {
        this.owners = owners;
    }

}
```

运行期 `this.owners` 指向的对象，是 Spring Data 在启动时生成的**动态代理**实现（你在磁盘上找不到这个 `.java` 文件，它是运行时造出来的）。手写这件事非常麻烦，交给容器就只是构造器参数的事。

**这就是 Spring 的核心价值：面向接口编程 + 由容器负责装配，调用方不需要知道实现从哪里来。**

## 3. 依赖注入的三种写法

```java
// 写法一：构造器注入（本项目全部用这种，官方推荐）
private final OwnerRepository owners;

public OwnerController(OwnerRepository owners) {
    this.owners = owners;
}

// 写法二：字段注入（不推荐，但你在老项目里会大量见到）
@Autowired
private OwnerRepository owners;

// 写法三：setter 注入（只有可选依赖才考虑）
@Autowired
public void setOwners(OwnerRepository owners) {
    this.owners = owners;
}
```

为什么推荐构造器注入：

- `final` 保证不可变：对象一旦创建，依赖就固定了
- 依赖是显式的：看构造器就知道这个类需要什么；参数太多说明职责太重，是重构信号
- 好测试：单测可以 `new OwnerController(mockRepo)`，不用启动 Spring
- 循环依赖在启动时就报错，而不是等到运行时才炸

重要细节：**如果一个类只有一个构造器，`@Autowired` 可以省略**，Spring 会自动使用它。本项目正是靠这条规则，所以构造器上都没有注解。

## 4. 把 @SpringBootApplication 拆开看

它等价于三个注解叠加：

| 组成注解 | 作用 |
| --- | --- |
| `@SpringBootConfiguration` | 本质是 `@Configuration`，声明这是配置类 |
| `@EnableAutoConfiguration` | 开启自动配置，Spring Boot"魔法"的来源 |
| `@ComponentScan` | 从当前包开始向下扫描 `@Component` 及其派生注解 |

**组件扫描的起点 = `PetClinicApplication` 所在的包**，即 `org.springframework.samples.petclinic`。所以本项目的类全部在这棵包树下：`owner`、`vet`、`system`、`model`。这就是"约定优于配置"：包放对位置，一行配置都不需要写。

自动配置怎么工作：你引入 `spring-boot-starter-webmvc`，Spring Boot 的自动配置类上写着 `@ConditionalOnClass(...)`、`@ConditionalOnMissingBean(...)` 之类的条件，条件满足就注册相应 Bean（`DispatcherServlet`、各种 `HandlerMapping`、内嵌 Tomcat 容器等）。这些自动配置类的清单位于依赖 jar 里的 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`。

一句话总结：**自动配置 = 一堆带条件的默认配置；你一旦自己定义了同类型 Bean，它就让位。**

## 5. @Component 与 @Configuration + @Bean 的区别

教材文件：`src/main/java/org/springframework/samples/petclinic/system/WebConfiguration.java`

```java
@Configuration
public class WebConfiguration implements WebMvcConfigurer {

    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        return resolver;
    }

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang");
        return interceptor;
    }

}
```

`LocaleResolver` 和 `LocaleChangeInterceptor` 都是框架提供的类，你没法在它们的源码上加 `@Component`，于是用 `@Bean` 方法自己造实例交给容器。区别总结：

- `@Component` 及其派生（`@Controller`、`@Service`、`@Repository`）：标在**自己的类**上，容器扫到就创建
- `@Bean`：标在**配置类的方法**上，方法返回值就是 Bean。适合包装第三方类，或创建前需要几行配置代码的场景

顺带看 `CacheConfiguration` 用的是 `@Configuration(proxyBeanMethods = false)`，含义是"这个配置类的方法之间不互相调用，不需要生成 CGLIB 代理"，能少一个代理类，启动更快。这是 Spring Boot 官方推荐写法，你会在几乎所有现代项目里看到。

## 6. 读懂 application.properties

教材文件：`src/main/resources/application.properties`

| 配置 | 含义 |
| --- | --- |
| `database=h2` | 自定义变量，被下面两行通过 `${database}` 引用 |
| `spring.sql.init.schema-locations=classpath*:db/${database}/schema.sql` | 启动时执行哪份建表脚本 |
| `spring.sql.init.data-locations=classpath*:db/${database}/data.sql` | 启动时执行哪份初始化数据 |
| `spring.jpa.hibernate.ddl-auto=none` | 不让 Hibernate 自动改表结构；表结构由上面的脚本负责，生产环境应当如此 |
| `spring.jpa.open-in-view=false` | 关闭 OSIV（Open Session In View）。开启它会让数据库连接一直占用到页面渲染结束，属于隐性开销 |
| `spring.jpa.properties.hibernate.default_batch_fetch_size=16` | 批量抓取，缓解 N+1 查询（第 3 课细讲） |
| `spring.messages.basename=messages/messages` | 国际化消息文件的路径前缀 |
| `management.endpoints.web.exposure.include=*` | 暴露全部 Actuator 端点。文件注释里明确写了生产环境不要这样 |
| `spring.web.resources.cache.cachecontrol.max-age=12h` | 静态资源浏览器缓存 12 小时 |

换数据库只要换 profile：`mvn spring-boot:run -Dspring-boot.run.profiles=mysql`，会额外加载 `application-mysql.properties`。

这也解释了一个设计：为什么 `db/` 下面有 `h2`、`mysql`、`postgres` 三套目录，而代码一行不用改。**表结构属于部署配置，不属于代码。**

## 7. 动手实验

### 实验 1（10 分钟）：看自动配置报告

```bash
mvn spring-boot:run -Dspring-boot.run.arguments=--debug
```

启动日志会分成 `Positive matches`（生效的自动配置）和 `Negative matches`（未生效，附带原因）。在 Positive 里找 `DataSourceAutoConfiguration`，在 Negative 里随便挑一个看它的排除原因。

观察点：体会"自动配置是条件驱动的默认值"，而不是无脑全开。

### 实验 2（10 分钟）：证明组件扫描的范围

把 `src/main/java/org/springframework/samples/petclinic/system/WelcomeController.java` 上的 `@Controller` 注释掉，重启，访问 `localhost:8080`。

预期：变成 404。

恢复注解重启，又能访问。结论：**类要被扫描到、并且带 `@Controller`，才会有路由。**

### 实验 3（15 分钟）：观察 Bean 的创建时机

在 `OwnerController` 的构造器里加一行：

```java
System.out.println("OwnerController created");
```

重启应用，观察这句在**启动阶段**只打印了一次，而不是每次访问页面都打印。

结论：Bean 默认是**单例**（singleton），一个容器里只有一个实例，所有请求共享它。因此 Controller 里绝对不能存"每个请求独有"的可变状态——这也是本项目所有依赖字段都写成 `final` 的原因。

### 实验 4（20 分钟）：亲手制造一次注入失败

1. 在 `OwnerController` 里加一个字段 `private final Foo foo;`（`Foo` 是你自己写的一个接口），并在构造器里赋值，但不给任何实现类加注解。
2. 重启 → 启动失败，报 `NoSuchBeanDefinitionException`（找不到 `Foo` 类型的 Bean）。
3. 再写一个 `@Component` 实现类，重启 → 成功。
4. 如果写了**两个**实现类，会变成另一个报错：`NoUniqueBeanDefinitionException`（找到多个，不知道用哪个）。

结论：**注入的前提是容器里恰好有一个可用的 Bean。** 这两种报错是你今后最常遇到的启动失败原因之一。

## 8. 自测题

先合上讲义作答，再对照文末答案。

1. `@SpringBootApplication` 由哪三个注解组成？各自负责什么？
2. 为什么 `OwnerController` 的构造器上没有 `@Autowired` 也能正常注入？
3. Bean 默认是单例还是每次都新建？这给写 Controller 带来什么约束？
4. 什么情况下用 `@Bean` 而不是 `@Component`？各举一个本项目中的例子。
5. `spring.jpa.hibernate.ddl-auto=none` 意味着表结构由谁负责创建？
6. 你把一个新写的 `@Service` 类放在了 `com.example.tool` 包下，应用启动后注入它时报找不到 Bean，最可能的原因是什么？

## 9. 本课要记住的五句话

1. `SpringApplication.run` 的作用是"创建容器 → 装配 Bean → 启动服务器"，不是你手动 new 一切。
2. 容器本质是一个对象仓库；依赖注入就是让容器替你连好线，这是 Spring 存在的根本理由。
3. 构造器注入 + `final` 是本项目的统一风格，也是当前的最佳实践。
4. `@SpringBootApplication` 决定了组件扫描的起点包，包放错就"不生效"。
5. 配置文件决定"用哪套数据库、表怎么建、JPA 怎么跑"，代码本身与数据库无关。

## 附：自测题答案

1. `@SpringBootConfiguration`（声明配置类）+ `@EnableAutoConfiguration`（开启自动配置）+ `@ComponentScan`（从当前包向下扫描组件）。
2. 因为该类只有一个构造器，Spring 4.3 之后会默认使用它进行注入，`@Autowired` 可以省略。
3. 默认单例，全容器共享一个实例。所以 Controller 不能保存与单个请求相关的可变状态，依赖字段应声明为 `final`。
4. 需要包装第三方类，或在创建 Bean 前要执行几行配置代码时用 `@Bean`，例如 `WebConfiguration.localeResolver()`；自己项目里的类直接用 `@Controller`、`@Service`、`@Repository` 更简洁。
5. 表结构由 `spring.sql.init.schema-locations` 指定的脚本（本项目是 `db/<database>/schema.sql`）负责，Hibernate 不参与建表。
6. 组件扫描的起点是主类所在的 `org.springframework.samples.petclinic` 包，`com.example.tool` 不在扫描范围内，所以那个 `@Service` 从来没被注册成 Bean。

# 第 5 课　工程化：测试、缓存、国际化、规范与部署

## 本课目标

1. 一个专业项目的测试为什么要分成好几层？
2. 缓存什么时候能救命，什么时候会坑你？
3. 11 个语言文件怎么协同工作？为什么有一个是空的？
4. 代码规范门禁（format / checkstyle / nohttp）在拦什么？
5. 这个项目怎么部署，怎么切换数据库？

预计用时 5-7 小时（含 6 个实验）。本课决定你和"会写代码的人"与"能交付的人"之间的差距。

## 1. 测试分层：为什么不能只有一种测试

本项目 `src/test/java/` 下的测试按"启动了多少东西"分成四层：

| 层次 | 注解 | 代表类 | 启动了什么 | 速度 |
| --- | --- | --- | --- | --- |
| 纯单元测试 | `@ExtendWith(MockitoExtension.class)` | `PetValidatorTests`、`PetTypeFormatterTests` | 什么都不启动 | 毫秒 |
| Web 切片 | `@WebMvcTest(XxxController.class)` + `@MockitoBean` | `OwnerControllerTests`、`VetControllerTests`、`VisitControllerTests`、`WelcomeControllerTests` | 只启动 Web 层，仓储被 mock 替换 | 快 |
| 数据切片 | `@DataJpaTest` + `@AutoConfigureTestDatabase(replace = Replace.NONE)` | `ClinicServiceTests` | 只启动 JPA 部分，跑真实 SQL | 中 |
| 全量集成 | `@SpringBootTest(webEnvironment = RANDOM_PORT)` | `PetClinicIntegrationTests`、`CrashControllerIntegrationTests`、`PetClinicConcurrencyTests` | 启动完整应用，真实 HTTP 请求 | 慢 |

外面还有两类"环境相关"的测试：

| 类型 | 注解 | 说明 |
| --- | --- | --- |
| Testcontainers | `@Testcontainers(disabledWithoutDocker = true)` + `@ActiveProfiles("mysql")` | `MySqlIntegrationTests` 用真实 MySQL 容器；没装 Docker 会自动跳过 |
| Docker Compose | `spring.docker.compose.start.arguments=...` + `assumeTrue(DockerClientFactory...)` | `PostgresIntegrationTests` 用 compose 起 PostgreSQL，无 Docker 时自动跳过 |

**本机未安装 Docker**，所以这两类测试会被跳过——这是预期行为，不是你的环境坏了。

分层的意义在于**反馈速度**：改一行校验逻辑，跑毫秒级的单元测试就够了；只有涉及 SQL、事务、JSON 序列化时才需要往上走一层。全都写成 `@SpringBootTest` 的项目，跑一次测试要几分钟，最终没人愿意跑。

几个细节值得注意：

- `@MockitoBean` 是新一代注解（替代旧的 `@MockBean`），用于在切片测试里替换容器中的 Bean。
- 到处出现的 `@DisabledInNativeImage`、`@DisabledInAotMode` 是为 GraalVM 原生镜像和 AOT 编译做准备——这些测试依赖运行时代理，原生镜像下不成立。
- `PetClinicIntegrationTests` 里有一个 `main()` 方法，可以**在 IDE 里当普通程序启动**，用来快速开发调试。这是官方推荐的开发方式。
- 测试里锁定 `TEST_OWNER_ID = 1` 这类固定 id，依赖 `db/h2/data.sql` 的固定初始数据。既带来了确定性，也带来了脆弱性——改 data.sql 就可能连累测试，这也是一种真实的工程权衡。

## 2. 缓存：一条注解换来的性能与风险

`src/main/java/org/springframework/samples/petclinic/system/CacheConfiguration.java`：

```java
@Configuration(proxyBeanMethods = false)
@EnableCaching
class CacheConfiguration {

    @Bean
    public JCacheManagerCustomizer petclinicCacheConfigurationCustomizer() {
        return cm -> cm.createCache("vets", cacheConfiguration());
    }

}
```

它做两件事：开启缓存能力（`@EnableCaching`），并显式创建一个名为 `vets` 的缓存（JCache / JSR-107 标准 API，底层实现是 Caffeine）。

使用侧在 `VetRepository`：

```java
@Transactional(readOnly = true)
@Cacheable("vets")
Collection<Vet> findAll() throws DataAccessException;
```

`@Cacheable("vets")` 的含义：调用前先按参数计算 key，命中就直接返回缓存值，**根本不执行方法体**（也就不会查数据库）。

为什么兽医列表适合缓存：数据几乎不变、被所有用户高频访问——这是缓存的理想场景。

但要注意三个真实的风险，这也是很多线上事故的来源：

1. **数据不一致**：如果增加了修改兽医的功能，缓存不会自动失效，页面会一直显示旧数据，必须手动 `@CacheEvict`。
2. **key 设计**：`findAll()` 和 `findAll(Pageable)` 共用一个缓存名，靠参数不同来区分 key。参数一旦变化（比如加了筛选条件），必须重新审视 key 是否还够用。
3. **内存**：缓存默认没有上限时，本质是把数据库搬进内存。

## 3. 国际化：11 个语言文件怎么协同

机制有三部分：

1. 消息文件：`src/main/resources/messages/messages*.properties`，共 12 个文件（基础 + 11 种语言）。
2. 语言选择：`WebConfiguration` 注册的 `LocaleChangeInterceptor` 监听 `?lang=` 参数，`SessionLocaleResolver` 把选择记在 Session 里。
3. 使用：模板里写 `th:text="#{owner}"`，Java 里写 `message = "{telephone.invalid}"`。

关键设计：**`messages_en.properties` 是空的**。因为 `messages.properties`（基础文件）本身就是英文，找不到对应语言时框架会回退到基础文件，所以英文不需要重复维护一份。

守护这套机制的是 `src/test/java/org/springframework/samples/petclinic/system/I18nPropertiesSyncTest.java`，它有两个测试：

- `checkNonInternationalizedStrings`：扫描 `src/main` 下的 `.java` 和 `.html`，用正则找出"没有被 `#{...}` 国际化、也没用 `th:text` 的硬编码文案"，发现就失败。
- `checkI18nPropertyFilesAreInSync`：对比基础文件与各语言文件的 key 集合，**任何一种语言少了 key 就失败**。

这解释了第 6 课改造题的连锁反应：你新增一个 `email=Email`，就必须同步补到所有语言文件，否则测试直接红。

## 4. 规范门禁：让代码风格不靠自觉

| 工具 | 作用 | 配置位置 |
| --- | --- | --- |
| spring-javaformat | 代码格式化与格式校验（缩进、import 顺序等） | `pom.xml` / `build.gradle` 插件 |
| Checkstyle | 静态代码检查 | `src/checkstyle/` |
| nohttp | 禁止出现明文链接（防止安全降级） | `src/checkstyle/nohttp-checkstyle.xml` |
| JaCoCo | 测试覆盖率统计 | `pom.xml` |

常用命令（Maven 侧）：

```bash
mvn -B spring-javaformat:apply     # 自动格式化
mvn -B spring-javaformat:validate  # 只校验不修改
mvn -B checkstyle:check            # 静态检查
```

另外项目遵循 DCO：所有提交必须带 `Signed-off-by` 尾注（见 `README.md` 的 Contributing 一节）。这类要求在大公司里同样是硬性门槛。

## 5. 可观测与部署

Actuator 已开启全部端点（`management.endpoints.web.exposure.include=*`，注释里明确警告生产环境不要这样）。启动后可以访问：

- `localhost:8080/actuator/health`：健康检查，容器编排系统的探针就用它
- `localhost:8080/actuator/metrics`：指标
- `localhost:8080/actuator/caches`：缓存状态，能直接看到 `vets` 缓存
- `localhost:8080/actuator/mappings`：所有 URL 与处理方法的映射表（排查 404 的神器）

部署方式有两种：

```bash
# 1. 打成容器镜像（需要 Docker）
mvn spring-boot:build-image

# 2. 用 compose 起数据库，再用 profile 连上去
docker compose up mysql      # 或 postgres
mvn spring-boot:run -Dspring-boot.run.profiles=mysql
```

`docker-compose.yml` 里定义了两个服务：`mysql:9.7` 和 `postgres:18.4`。对应的 profile 配置文件用环境变量做兜底：

```
spring.datasource.url=${MYSQL_URL:jdbc:mysql://localhost/petclinic}
spring.datasource.username=${MYSQL_USER:petclinic}
```

`${VAR:default}` 是标准写法：有环境变量就用环境变量，没有就用默认值。**这就是"同一份代码部署到不同环境"的实现方式**——不要把连接信息写死在代码里。

## 6. 动手实验

### 实验 1（15 分钟）：跑一个最小的测试

```bash
mvn -B test -Dtest=PetValidatorTests
```

几秒钟就结束。这就是单元测试该有的速度。

### 实验 2（10 分钟）：跑一个切片测试

```bash
mvn -B test -Dtest=OwnerControllerTests
```

比实验 1 慢，因为启动了 Web 层——但仓储是 mock 的，所以不碰数据库。

### 实验 3（20 分钟）：亲手让国际化守护测试变红

1. 打开 `src/main/resources/messages/messages_de.properties`，删掉任意一个 key（例如 `owner=...`）。
2. 运行 `mvn -B test -Dtest=I18nPropertiesSyncTest`。
3. 观察失败信息里明确指出"`messages_de.properties` 缺少哪个 key"。
4. 把 key 补回去，再跑一次，变绿。

观察点：**规范不是写在文档里的，是被测试强制执行的。** 以后你加任何界面文案，都要有心理准备同步所有语言。

### 实验 4（15 分钟）：切换语言

访问 `localhost:8080/?lang=de`，页面文案会切换成德语。再访问 `localhost:8080/?lang=zh`（不存在的中文文件），观察回退到英文。

### 实验 5（20 分钟）：看到缓存的实际效果

1. 临时打开 SQL 日志（`spring.jpa.show-sql=true`）。
2. 连续访问两次 `localhost:8080/vets`（JSON 接口）。
3. 观察控制台：第一次有 `select ... from vets`，第二次没有。
4. 再访问 `localhost:8080/actuator/caches`，能看到 `vets` 缓存。

### 实验 6（15 分钟）：用 actuator 排查问题

访问 `localhost:8080/actuator/mappings`，搜索 `owners`，对照你读过的控制器代码。

观察点：这是排查"为什么这个 URL 是 404"最直接的工具——如果映射表里找不到你要的路径，问题在注解或组件扫描；如果找得到，问题在更深的地方。

## 7. 自测题

1. `@WebMvcTest` 和 `@SpringBootTest` 的核心区别是什么？各自适合验证什么？
2. 为什么 `messages_en.properties` 是空的？
3. `I18nPropertiesSyncTest` 的两个测试分别在防什么？
4. `@Cacheable` 命中缓存时，方法体会不会执行？
5. 如果给兽医增加了"修改"功能，现有缓存会带来什么问题？怎么解决？
6. `${MYSQL_URL:jdbc:mysql://localhost/petclinic}` 这种写法解决了什么问题？
7. 排查一个 URL 返回 404 的问题，你会先看哪个 actuator 端点？

## 8. 本课要记住的五句话

1. 测试分层的唯一目的是反馈速度：能写单元测试就不要写集成测试。
2. 缓存是"用一致性换性能"的交易，加缓存前先想清楚失效策略。
3. 国际化 = 消息键 + 语言选择 + 同步校验测试，三者缺一不可。
4. 代码风格与安全约束应该由工具强制执行，而不是靠人自觉。
5. 连接信息用环境变量兜底，是同一份代码跑遍所有环境的前提。

## 附：自测题答案

1. `@WebMvcTest` 只加载 Web 层（控制器、消息转换、拦截器），依赖用 mock 替换，用于验证路由、参数绑定、校验、返回结果；`@SpringBootTest` 加载完整应用上下文，用于验证跨层协作、真实 SQL、HTTP 交互。
2. 基础文件 `messages.properties` 本身就是英文，框架在找不到对应语言的 key 时会回退到基础文件，所以英文无需重复维护。
3. 前者防"页面或代码里出现没有国际化的硬编码文案"，后者防"某种语言的翻译文件缺 key"。
4. 不会执行，直接返回缓存值——这也意味着方法里的日志、埋点、权限校验都不会运行，这是一个容易被忽略的坑。
5. 会出现数据不一致：修改后页面仍显示旧数据。解决办法是在修改方法上加 `@CacheEvict`（或改用 `@CachePut`），必要时按业务粒度清缓存。
6. 让同一份配置文件适配不同部署环境：有环境变量时用环境变量，没有时用默认值，避免把连接信息硬编码进代码或打进镜像。
7. `localhost:8080/actuator/mappings`，先确认映射是否存在，再决定往注解/扫描方向查，还是往业务代码方向查。

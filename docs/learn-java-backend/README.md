# Java 后端学习笔记（基于 Spring PetClinic）

> 学习者：你自己　开始日期：2026-10-13
> 教材：本仓库 `D:\java-spring-boot`（Spring Boot 4.1.0 + Java 17）
> 起点：会 Java 语法，没用过 Spring

## 为什么拿这个项目当教材

它是一个"麻雀虽小五脏俱全"的真实后端：Web 层、表单校验、JPA 持久化、缓存、国际化、多数据库、单元与集成测试、代码规范门禁都有，但只有 30 个源文件。你能真正读完它，而不是被几十万行代码劝退。等你看懂了它，再去看公司项目，会发现结构是同一套。

## 怎么用这套笔记

每课固定四步，缺一不可：

1. 读——看完本课讲义，不要跳行。
2. 跑——照着"动手实验"做一遍，看到预期现象。
3. 改——做课后练习，改一行代码，观察行为变化。
4. 测——回答"自测题"，答不出来就回去补。

三条学习纪律：

- 按"请求链路"读代码，不要按文件夹顺序读。目标是能说出"我点了这个按钮，服务器上依次发生了什么"。
- 每读完一段代码，回答三个问题：谁调用我？我依赖谁？这里出错会怎样？
- 改一行、跑一次、看变化。读十遍不如改一次。
- 做完实验记得还原改动，保持 `git status` 干净，避免和后面的练习混淆。

## 进度总表

| 阶段 | 主题 | 主要内容 | 预计 | 状态 |
| --- | --- | --- | --- | --- |
| 1 | 启动与容器 | Spring Boot 启动流程、IoC、依赖注入、自动配置、配置文件 | 1-2 天 | 讲义已完成 |
| 2 | Web 层 | 请求链路、注解地图、参数绑定、校验回显、重定向 | 3-5 天 | 讲义已完成 |
| 3 | 数据层 | JPA 实体映射、关联关系、派生查询、分页、事务、N+1 | 约 1 周 | 讲义已完成 |
| 4 | 校验与错误处理 | Bean Validation、自定义 Validator、错误页、异常处理 | 3-4 天 | 讲义已完成 |
| 5 | 工程化 | 测试分层、缓存、多数据库 profile、国际化、规范门禁 | 约 1 周 | 讲义已完成 |
| 6 | 综合改造 | 贯穿全栈加字段、404 改造、缓存失效、脆弱代码重构 | —— | 讲义已完成 |

课节清单：

- [ ] 第 1 课　跑起来，并搞懂 Spring 容器（`lesson-01-startup-and-container.md`）
- [ ] 第 2 课　Web 层：一次请求的完整链路（`lesson-02-web-layer.md`）
- [ ] 第 3 课　数据层：实体、关联、查询与事务（`lesson-03-data-layer.md`）
- [ ] 第 4 课　校验与错误处理（`lesson-04-validation-and-errors.md`）
- [ ] 第 5 课　工程化：测试、缓存、国际化、规范与部署（`lesson-05-engineering.md`）
- [ ] 第 6 课　结业改造：四个真实需求（`lesson-06-capstone.md`）

每完成一课，把上面那一行的 `[ ]` 改成 `[x]`。

## 环境速查（已实测）

| 项目 | 状态 |
| --- | --- |
| Java | 17.0.12（`java -version` 可验证） |
| Maven | 3.8.9，已在 PATH，可直接用 `mvn` |
| Maven Wrapper | `.mvn/wrapper` 配置为 3.9.16，可用 `.\mvnw.cmd` |
| Gradle | 未安装 CLI，但仓库自带 `gradlew.bat` |
| 依赖仓库 | 已配置国内镜像（aliyun），首次下载较慢 |
| 编译验证 | `mvn -B -DskipTests compile` 已跑通，30 个源文件，BUILD SUCCESS |
| Docker | 未安装。MySQL/PostgreSQL 的集成测试会自动跳过，属预期行为 |

常用命令：

```bash
mvn -B -DskipTests compile        # 只编译，最快确认环境
mvn spring-boot:run               # 启动应用，默认 H2 内存库
mvn -B test                       # 跑全部测试（首次较慢）
mvn spring-boot:run -Dspring-boot.run.arguments=--debug   # 打印自动配置报告
mvn spring-boot:run -Dspring-boot.run.profiles=mysql      # 切换 MySQL profile
```

应用起来后：

- 首页：`localhost:8080`
- 主人列表：`localhost:8080/owners/find`
- 兽医列表：`localhost:8080/vets.html`，同一份数据的 JSON 接口：`localhost:8080/vets`
- 数据库控制台：`localhost:8080/h2-console`，JDBC URL 用启动日志里打印的 `jdbc:h2:mem:<uuid>`
- 故意报错看错误页：`localhost:8080/oups`

## 排错备忘

- 端口被占用：启动日志报 `Port 8080 was already in use`，先关掉占用进程，或加 `--server.port=8081`。
- 启动报找不到 Bean：`NoSuchBeanDefinitionException`，说明要注入的类型在容器里不存在，检查类是否加了 `@Component` 及其派生注解，且包在扫描范围内。
- 页面 404 但代码没错：先确认类在本项目的包树下、注解没写错、URL 与方法上的路径完全一致。
- 测试很慢：第一次要下载依赖；后续增量会快很多。

## 学完这套笔记你应该能做到

- 拿到一个陌生的 Spring Boot 项目，能独立画出它的请求链路和分层结构
- 独立新增一个"实体 + 表 + 仓储 + 控制器 + 页面 + 测试"的完整小功能
- 看懂启动报错，知道去哪个配置类、哪个包、哪份脚本里找问题

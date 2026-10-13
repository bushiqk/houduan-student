# 第 6 课　结业改造：四个真实需求

## 怎么用这一课

前五课是"读懂"，这一课是"交付"。每个改造都要求你独立完成，做到验收标准才算过。**遇到不会的，回到对应课节查，不要直接抄答案。**

建议顺序：改造 1（全栈加字段）→ 改造 2（优雅 404）→ 改造 3（筛选 + 缓存）→ 改造 4（重构脆弱代码）。

## 改造 1：给主人增加邮箱字段

**需求**：主人的资料里增加邮箱，必填、格式必须合法，新建和编辑表单都要能填写，详情页要展示。三种数据库都要能跑。

这是最典型的"一个小需求牵动整个工程"，做完一遍你就理解了什么是全栈改动。

**涉及清单（自己先想，再对照）**

1. `src/main/java/org/springframework/samples/petclinic/owner/Owner.java`：加字段 `email`，用 `@NotBlank`（必填）和 `@Email`（格式）校验；补 getter/setter
2. `src/main/resources/templates/owners/createOrUpdateOwnerForm.html`：加输入框，尽量复用已有的 `fragments/inputField.html` 片段
3. `src/main/resources/templates/owners/ownerDetails.html`：展示邮箱
4. `src/main/resources/db/h2/schema.sql`、`db/mysql/schema.sql`、`db/postgres/schema.sql`：三张 `owners` 表各加一列
5. `src/main/resources/db/h2/data.sql`、`db/mysql/data.sql`、`db/postgres/data.sql`：给已有数据补上邮箱值（**漏了这步，H2 启动时的插入语句会失败**）
6. `src/main/resources/messages/messages.properties` 以及所有 `messages_*.properties`：加 `email=Email` 这类标签键（**漏了这步，`I18nPropertiesSyncTest` 会失败**）
7. 测试：`OwnerControllerTests`、`ClinicServiceTests` 里构造的 `Owner` 对象要补 `email`（**漏了这步，`@NotBlank` 会让测试失败**）

**验收标准**

- 新建主人时不填邮箱 → 表单回显"必填"错误，页面不跳走
- 填 `abc` → 提示格式错误
- 填写合法邮箱并保存 → 详情页能看到邮箱
- `mvn -B test` 全绿（包括国际化同步测试）
- 三套 `schema.sql` + `data.sql` 都能成功执行（H2 必测，MySQL/PostgreSQL 至少人工核对语法）

**提示与坑**

- 列名会自动由 `email` 映射，不需要写 `@Column(name = ...)`
- H2 脚本里的 `data.sql` 用的是 `INSERT INTO owners (...) VALUES (...)`，加列后必须同步改字段列表和值列表，两边数量要对齐
- 这个改造会同时触发第 3 课（实体映射）、第 4 课（校验 + 国际化）、第 5 课（同步测试）的知识点。**能独立做完，说明你已经具备完整的全栈改动能力。**

## 改造 2：把"找不到主人"变成优雅的 404

**现状问题**：访问 `/owners/99999` 会抛 `IllegalArgumentException`，用户看到的是通用错误页，服务端还打出一串堆栈。这既不友好，也不安全（暴露内部信息）。

**目标**：返回 HTTP 404 状态码 + 一个说明"该主人不存在"的页面，同时不再打印异常堆栈。

**两种实现思路（选一种，另一种了解即可）**

思路 A（就地处理）：在 `OwnerController.findOwner` / `showOwner` 等位置，把 `orElseThrow(() -> new IllegalArgumentException(...))` 换成 `orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Owner not found"))`。

思路 B（全局处理，更推荐）：新建一个 `@ControllerAdvice` 类，用 `@ExceptionHandler` 统一捕获"资源不存在"的异常，返回专门的视图并设置 404 状态。

**验收标准**

- 用 `curl -i localhost:8080/owners/99999`（或浏览器开发者工具的 Network 面板）确认状态码是 404，不是 500
- 页面文案友好，且走国际化（不要硬编码中文或英文）
- 后台日志里没有异常堆栈
- 正常路径（`/owners/1`）完全不受影响

**提示**：`@ControllerAdvice` 是实现全局异常处理的标准位置，做完这个改造你应该能回答"为什么企业项目里几乎都有 `@ControllerAdvice`"。

## 改造 3：兽医列表支持按专科筛选

**需求**：增加按专科（specialty）筛选兽医的能力，例如只显示做 surgery 的兽医；页面和 JSON 接口都要支持。

**步骤提示**

1. 先读 `src/main/java/org/springframework/samples/petclinic/vet/Vet.java` 和 `Specialty.java`，确认"兽医—专科"的关联映射是怎么写的（这是你第一次需要自己读实体关系）
2. 按第 3 课的派生查询规则，在 `VetRepository` 里加一个按专科名查询的方法
3. 控制器里接收筛选参数，传给仓储，放进模型
4. 模板加筛选入口

**必须处理的缓存问题**

现有 `@Cacheable("vets")` 是为"全部兽医"设计的。加了筛选后你要回答：

- 新方法的缓存 key 是什么？会不会和旧方法的 key 撞车？
- 筛选结果要不要缓存？用什么缓存名？
- 如果以后支持新增/修改兽医，该怎么失效？

**验收标准**

- `/vets?specialty=surgery` 只返回匹配的兽医（JSON 接口）
- 页面同样能筛选，不筛选时行为与改造前完全一致
- 能清楚说明你的缓存 key 设计，以及为什么不会串味

## 改造 4：重构脆弱的唯一约束判断

**现状问题**（第 3、4 课都提到）：

```java
private boolean isDuplicatePetNameViolation(DataIntegrityViolationException ex) {
    String message = ex.getMessage();
    return message != null && message.toLowerCase().contains("unique_owner_pet_name");
}
```

靠异常文本里的索引名判断错误类型。三套数据库对同一个业务约束的实现方式各不相同，改名字、换版本、换数据库都会让这段逻辑失效。

**可选方向（选一个，并说明权衡）**

- 方向 A：捕获更具体的异常并读取结构化的约束名（Hibernate 提供了约束名信息，不再依赖文本匹配）
- 方向 B：统一三套脚本的约束命名与语义，把名字抽成常量，让代码和脚本有明确的契约
- 方向 C：承认"应用层已拦截绝大多数情况"，把数据库异常统一转成友好提示——但要评估误报风险（其他完整性错误会不会被误判成重名？）

**验收标准**

- 能说清楚每种方向的利弊，而不只是"能跑"
- 重名场景在三套数据库下都得到友好提示
- 非重名的完整性错误仍然会被抛出，不被吞掉
- 改造后 `mvn -B test` 全绿

## 结业自评清单

四个改造全部完成后，逐条自问：

- [ ] 我能不看代码，说出一次页面请求的完整链路（浏览器 → DispatcherServlet → 控制器 → 仓储 → 数据库 → 模板）
- [ ] 我能解释容器、Bean、依赖注入、组件扫描、自动配置这五个词
- [ ] 我能独立新增一个"表 + 实体 + 仓储 + 控制器 + 模板 + 测试"的完整功能
- [ ] 我能看懂启动失败的原因（找不到 Bean / 多个候选 Bean / 端口占用）
- [ ] 我知道什么时候该用 `@Component`、什么时候该用 `@Bean`
- [ ] 我知道校验要分三层，防重复要"应用层 + 数据库"双保险
- [ ] 我知道 EAGER 集合会带来 N+1，并知道怎么观察和缓解
- [ ] 我知道缓存是一致性与性能的交易
- [ ] 我知道测试为什么要分层

到这里，你已经具备"在一个真实 Spring Boot 项目里独立完成需求"的能力。下一步建议：挑一个自己的小项目（比如日记本、记账工具），用这套分层结构从零写一遍。**从零搭一次，比再读十遍代码都管用。**

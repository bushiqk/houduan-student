# 第 4 课　校验与错误处理

## 本课目标

1. 用户填错数据时，错误信息是怎么被"发现"并回显到页面上的？
2. 注解式校验和手写 `Validator` 分别在什么场景用？
3. 为什么"防重复"要写两遍（Java 层一遍、数据库一遍）？
4. 错误页是怎么出来的？为什么访问不存在的 id 会看到丑页面？

预计用时 3-4 小时（含 5 个实验）。

## 1. 校验的三个层次

后端校验不是一件事，而是三件事，责任不同：

| 层次 | 谁负责 | 本项目的例子 | 失败后的表现 |
| --- | --- | --- | --- |
| 绑定/类型转换 | Spring MVC | 表单里 `birthDate` 填 `abc` | `typeMismatch.birthDate` 错误 |
| Bean Validation 注解 | Hibernate Validator | `@NotBlank`、`@Pattern` | 字段级错误，回显表单 |
| 业务规则 | 你写的代码 | 生日不能在未来、日期必须晚于今天、宠物不能重名 | `result.rejectValue(...)` 或抛异常 |

一句话：**能用注解表达的就用注解，表达不了的用代码，数据库约束作为最后一道防线。**

## 2. 注解式校验

教材文件：`src/main/java/org/springframework/samples/petclinic/owner/Owner.java`

```java
@Column
@NotBlank
private String address;

@Column
@NotBlank
@Pattern(regexp = "\\d{10}", message = "{telephone.invalid}")
private String telephone;
```

`Person` 里则是：

```java
@Column(length = 30)
@Size(max = 30)
@NotBlank
private String firstName;
```

三个要点：

- `@NotBlank` 拒绝 null、空串和纯空格；`@NotNull` 只拒绝 null；`@Size(max = 30)` 管长度。注意 `@Column(length = 30)` 是建表用的，`@Size` 才是校验用的，两者要成对维护。
- `message = "{telephone.invalid}"` 里的花括号表示"这是一个消息键"，真正的文案去 `src/main/resources/messages/messages.properties` 找：`telephone.invalid=Telephone must be a 10-digit number`。**文案与代码分离，才能做多语言**（第 5 课会看到 11 个语言文件）。
- 光写注解不生效，必须由 `@Valid` 触发（见链路 C 的 `processCreationForm(@Valid Owner owner, ...)`）。

## 3. 手写 Validator

教材文件：`src/main/java/org/springframework/samples/petclinic/owner/PetValidator.java`

```java
public class PetValidator implements Validator {

    private static final String REQUIRED = "required";

    @Override
    public void validate(Object obj, Errors errors) {
        Pet pet = (Pet) obj;
        String name = pet.getName();
        if (!StringUtils.hasText(name)) {
            errors.rejectValue("name", REQUIRED, REQUIRED);
        }
        if (pet.isNew() && pet.getType() == null) {
            errors.rejectValue("type", REQUIRED, REQUIRED);
        }
        if (pet.getBirthDate() == null) {
            errors.rejectValue("birthDate", REQUIRED, REQUIRED);
        }
    }

    @Override
    public boolean supports(Class<?> clazz) {
        return Pet.class.isAssignableFrom(clazz);
    }

}
```

它为什么不用注解？源码注释里回答得很直白：*"这里不用 Bean Validation 注解，因为在 Java 里定义这类规则更简单"*。具体来说，`pet.isNew() && pet.getType() == null` 这种**依赖对象状态的规则**，"必填"注解表达不了。

注册方式在 `PetController`：

```java
@InitBinder("pet")
public void initPetBinder(WebDataBinder dataBinder) {
    dataBinder.setValidator(new PetValidator());
    dataBinder.setDisallowedFields("id", "*.id");
}
```

`@InitBinder("pet")` 只作用于名为 `pet` 的模型属性。**注意 `setValidator` 是"替换"而不是"追加"**：一旦这样写，`Pet` 上的 Bean Validation 注解就不会再被触发。如果两者都要，得用 `addValidators(...)`。

## 4. 业务规则校验

`PetController` 里三处典型的业务规则：

```java
// 1. 同一主人下宠物不能重名（先查内存，大小写不敏感）
if (StringUtils.hasText(pet.getName()) && pet.isNew() && owner.getPet(pet.getName(), true) != null) {
    result.rejectValue("name", "duplicate", "already exists");
}

// 2. 生日不能晚于今天
LocalDate currentDate = LocalDate.now();
if (pet.getBirthDate() != null && pet.getBirthDate().isAfter(currentDate)) {
    result.rejectValue("birthDate", "typeMismatch.birthDate");
}
```

`VisitController` 里：

```java
if (visit.getDate() != null && !visit.getDate().isAfter(LocalDate.now())) {
    result.rejectValue("date", "typeMismatch.visitDate");
}
```

`result.rejectValue("字段名", "错误码", "默认消息")` 的第三个参数是**兜底文案**：国际化文件里能找到键就用翻译，找不到就用它。所以 `duplicate` 在 `messages.properties` 里是 `is already in use`，最终用户看到的是"is already in use"而不是"already exists"。

## 5. 防重复为什么要写两遍

`PetController.processCreationForm` 的最后一段：

```java
try {
    owner.addPet(pet);
    this.owners.saveAndFlush(owner);
}
catch (DataIntegrityViolationException ex) {
    if (!isDuplicatePetNameViolation(ex)) {
        throw ex;
    }
    result.rejectValue("name", "duplicate", "already exists");
    return VIEWS_PETS_CREATE_OR_UPDATE_FORM;
}
```

原因是**并发**：Java 层"先查有没有重名"和"真正写入"之间存在时间窗口。两个请求同时提交同名宠物，两边查询都说"没有重名"，然后都去写入——此时只有数据库的唯一约束能挡住。

这就是"双保险"的标准写法：

- 应用层检查 → 给用户友好的错误提示
- 数据库约束 → 保证数据绝对正确

**绝不能只依赖应用层检查。** 这是后端工程师的基本素养，也是面试高频考点。

但本项目的实现有个真实的坑（第 3 课提过）：

```java
private boolean isDuplicatePetNameViolation(DataIntegrityViolationException ex) {
    String message = ex.getMessage();
    return message != null && message.toLowerCase().contains("unique_owner_pet_name");
}
```

靠异常文本里的索引名判断，而三套数据库脚本对这个约束的写法各不相同（H2 用 `VARCHAR_IGNORECASE`、MySQL 用排序规则、PostgreSQL 用 `LOWER(name)` 函数索引）。改个名字或换个数据库版本，这段判断就会失效，导致本该提示"重名"的场景变成 500 错误。

## 6. 错误页与异常

`CrashController` 是专门用来演示异常页的：

```java
@GetMapping("/oups")
public String triggerException() {
    throw new RuntimeException("Expected: controller used to showcase what happens when an exception is thrown");
}
```

Spring Boot 会把异常交给 `templates/error.html` 渲染，页面上用到的文案来自消息文件：

```
error.404=The requested page was not found.
error.500=An internal server error occurred.
error.general=An unexpected error occurred.
```

**本项目没有全局异常处理器**（没有 `@ControllerAdvice`）。所以：

- 访问 `/oups` → 500 页面
- 访问 `/owners/99999` → `IllegalArgumentException` → 也是错误页，而不是友好的"该主人不存在"

这在生产环境是不可接受的，正是第 6 课的改造题。

## 7. 动手实验

### 实验 1（15 分钟）：看注解校验的实际效果与消息解耦

1. 访问 `/owners/new`，电话填 `123`（不是 10 位），提交。记下提示文案。
2. 打开 `src/main/resources/messages/messages.properties`，把 `telephone.invalid` 改成中文，例如 `电话必须是 10 位数字`。
3. 重启（普通方式启动即可），重新提交，观察提示变成了你写的中文。

观察点：**代码没动，文案变了。** 这就是消息键机制的价值。

### 实验 2（15 分钟）：让类型转换失败

访问 `/owners/1/edit`，把电话改成 `abcdefghij`（10 位但非数字）提交，再试 `abc`。

观察点：`@Pattern` 挡住的提示和长度不符的提示，都是回显在表单字段旁边，页面没有跳走——因为校验失败时是**转发**（第 2 课规则）。

### 实验 3（20 分钟）：验证大小写不敏感的重名

访问某个主人的详情页，添加一只宠物，名字故意用不同大小写制造冲突（例如已有 `Buddy`，就再添加 `bUdDy`）。

观察点：你会看到 `duplicate` 的错误提示。这条路径是**应用层**的 `equalsIgnoreCase` 挡下来的；想验证数据库那道防线，需要绕过应用层检查（选做：临时注释掉应用层判断，观察是否被 `DataIntegrityViolationException` 接住）。

### 实验 4（10 分钟）：对比两种错误页

分别访问 `localhost:8080/oups` 和 `localhost:8080/owners/99999`。

观察点：两者都落到通用错误页，对用户毫无指导意义。记住这个体验，第 6 课我们来改。

### 实验 5（15 分钟）：拆掉校验看后果

把 `PetValidator.validate` 里生日那段注释掉，重启，提交一个**不填生日**的宠物。

观察点：校验没了，坏数据就可能进库（或抛出更难懂的异常）。**校验代码是数据质量的守门人，删它一时爽。**

## 8. 自测题

1. `@NotNull`、`@NotBlank`、`@Size` 各自管什么？
2. `message = "{telephone.invalid}"` 里的花括号是什么意思？文案最终从哪里来？
3. 什么时候该用自定义 `Validator` 而不是注解？
4. `dataBinder.setValidator(...)` 和 `addValidators(...)` 有什么区别？
5. `result.rejectValue` 的第三个参数是什么作用？
6. 应用层已经检查过重名了，为什么还要数据库唯一约束？
7. 本项目为什么访问 `/owners/99999` 会看到通用错误页？正确做法应该是什么？

## 9. 本课要记住的五句话

1. 校验分三层：类型绑定、注解校验、业务规则，各司其职。
2. 错误文案用消息键，代码与文案分离，才可能做多语言。
3. 依赖对象状态的规则用自定义 `Validator`，简单规则用注解。
4. 防重复必须双保险：应用层给体验，数据库保正确。
5. 没有全局异常处理的项目，用户只能看到通用错误页，体验不合格。

## 附：自测题答案

1. `@NotNull` 只拒绝 null；`@NotBlank` 拒绝 null、空串和纯空白；`@Size` 管长度（可同时约束最小和最大）。
2. 花括号表示这是消息键，最终文案从 `messages/messages*.properties`（按语言选择）里取，取不到才用代码里的默认文案。
3. 规则依赖对象状态（如"新建时才要求类型必填"）、或逻辑复杂用注解表达不清时。
4. `setValidator` 是替换默认校验器，会丢弃 Bean Validation 注解校验；`addValidators` 是追加，两者都会生效。
5. 兜底提示文案，在消息文件里找不到对应键时使用。
6. 因为并发：两个请求可能都通过了应用层检查后同时写入，只有数据库唯一约束能保证最终数据正确。
7. 因为项目没有全局异常处理，`IllegalArgumentException` 被统一交给 `error.html` 渲染。正确做法是用 `@ControllerAdvice` + `@ExceptionHandler`（或返回 404 状态）把它转换成一个有意义的页面。
